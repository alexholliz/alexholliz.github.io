---
layout: post
title:  "Building A Windows Server AMI with Packer"
date:   2021-05-17 17:34:12 -0800
categories: aws packer windows
---

# Prereqs
1. Have an AWS Account and VPC to spin stuff up in
2. Have an idea of what windows AMI you want to start with
3. Have Packer installed

# AWS
In your AWS Account, start by making a policy for Packer to use. Using Packer's documentation here: <https://www.packer.io/docs/builders/amazon#iam-task-or-instance-role> we can arrive at something like the following:

{% highlight json %}
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PackerSecurityGroupAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:CopyImage",
        "ec2:CreateImage",
        "ec2:CreateKeypair",
        "ec2:CreateSecurityGroup",
        "ec2:CreateSnapshot",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:DeleteKeyPair",
        "ec2:DeleteSecurityGroup",
        "ec2:DeleteSnapshot",
        "ec2:DeleteVolume",
        "ec2:DeregisterImage",
        "ec2:DescribeImageAttribute",
        "ec2:DescribeImages",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus",
        "ec2:DescribeRegions",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSnapshots",
        "ec2:DescribeSubnets",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DetachVolume",
        "ec2:GetPasswordData",
        "ec2:ModifyImageAttribute",
        "ec2:ModifyInstanceAttribute",
        "ec2:ModifySnapshotAttribute",
        "ec2:RegisterImage",
        "ec2:RunInstances",
        "ec2:StopInstances",
        "ec2:TerminateInstances",
        "ec2:CreateLaunchTemplate",
        "ec2:DeleteLaunchTemplate",
        "ec2:CreateFleet",
        "ec2:DescribeSpotPriceHistory",
        "ec2:DescribeVpcs"
      ],
      "Resource": "*"
    }
  ]
}
{% endhighlight %}

Once you've got your policy created (name it something you can track down), make a new IAM User for running automated tasks. This will give you an Access Key ID, and a Secret Key. Store those somewhere secure, since you can only see them once, and you'll need them later.

Also, if you want to start with an AMI that _isn't_ the latest windows server 2019 with container support, now's the time to hop into the "AMIs" function of EC2 and play with the filters until you find something you like.

# Packer

Start by preparing your environment. You can set your aws access key ID, and secret key as environment variables. It's inherently a little insecure, but the more limited permissions make it a little more safe than having admin rights or something. Packer uses specific environment variable notation: <https://www.packer.io/docs/templates/legacy_json_templates/user-variables#environment-variables> and <https://www.packer.io/guides/hcl/variables#from-environment-variables>

In my example, that would look something like:

```
export PKR_VAR_AKID=AMADEUPACCCESSKEY
export PKR_VAR_SKEY=0101MADEUPSECRETKEY42069
```

With those in place, you're ready to get cracking. Packer provides a pretty solid starting point: <https://www.packer.io/docs/builders/amazon/ebs#connecting-to-windows-instances-using-winrm> and for our purposes we are going to use WinRM to communicate. It's pretty slick, only requiring the powershell to enable winrm to be present when you're doing the packer build.

What follows is _my_ packer script for setting up a Windows 2019 Server with the Desktop Experience and Container support. Here's the important bits:

* Variables: Set your subnet ID, VPC ID, region, etc... in the file
* force_deregister: set this to true to overwrite your existing AMI when you rebuild (great for images you plan to keep up to date automatically)
* Source AMI Filter: This is how you set the different filter options to autoselect your AMI based on that query. In our case I want a Windows 2019 image, Full experience, with Container Support. The options in the filter match what you'd select in the AMIs sub-selection of the EC2 area of AWS.
* most_recent: This is just below the filter for your AMI, but it says if the filter returns multiple results, use the latest one.
* Owners: That string means Amazon. In the old JSON format for Packer, you could just write amazon. I am sure I could figure that out eventually, but the string works, and I am lazy.
* user_data_file: the LOCAL path to your WinRM enabling script.
* WinRM options: you'll see those are required in the WinRM enabling script, ez.
* Provisioner powershell: This runs after the instance is up, but before it is turned into an AMI. Specifically I run the recommended tasks to set a random password on next boot and sysprep the image so it looks like a new machine everytime something is spun up from this AMI.

```
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "subnet_id" {
  type    = string
  default = "subnet-coolstring"
}

variable "vpc_id" {
  type    = string
  default = "vpc-alsocoolstring"
}

variable "AKID" {
  type    = string
}

variable "SKEY" {
  type    = string
}

# source blocks are generated from your builders; a source can be referenced in
# build blocks. A build block runs provisioner and post-processors on a
# source. Read the documentation for source blocks here:
# https://www.packer.io/docs/from-1.5/blocks/source
source "amazon-ebs" "w2019-packer-fullcontainers-clean" {
  access_key = "${var.AKID}"
  secret_key = "${var.SKEY}"
  ami_name         = "w2019-packer-fullcontainers-clean"
  communicator     = "winrm"
  force_deregister = true
  instance_type    = "t2.micro"
  region           = "${var.aws_region}"
  source_ami_filter {
    filters = {
      architecture        = "x86_64"
      name                = "Windows_Server-2019-English-Full-ContainersLatest-*"
      root-device-type    = "ebs"
      virtualization-type = "hvm"
    }
    most_recent = true
    # the "Amazon" ami owner
    owners      = ["801119661308"]
  }
  subnet_id      = "${var.subnet_id}"
  user_data_file = "./scripts/autogenerated_password_https_bootstrap.txt"
  vpc_id         = "${var.vpc_id}"
  winrm_insecure = true
  winrm_use_ssl  = true
  winrm_username = "Administrator"
}

# a build block invokes sources and runs provisioning steps on them. The
# documentation for build blocks can be found here:
# https://www.packer.io/docs/from-1.5/blocks/build
build {
  sources = ["source.amazon-ebs.w2019-packer-fullcontainers-clean"]

  provisioner "powershell" {
    inline = ["C:/ProgramData/Amazon/EC2-Windows/Launch/Scripts/SendWindowsIsReady.ps1 -Schedule", "C:/ProgramData/Amazon/EC2-Windows/Launch/Scripts/InitializeInstance.ps1 -Schedule", "C:/ProgramData/Amazon/EC2-Windows/Launch/Scripts/SysprepInstance.ps1 -NoShutdown"]
  }
}

```

# Building the AMI

You're kinda... done here. It seems like a lot, but it's really not too bad. In your directory with your packer-name.pkr.hcl file, run some setup and validation commands:

{% highlight bash %}
packer init packer-name.pkr.hcl
packer validate packer-name.pkr.hcl
{% endhighlight %}

Then just go ahead and build it

{% highlight bash %}
packer build packer-name.pkr.hcl
{% endhighlight %}

Mine take about 10 minutes before they are ready, but yeah, you did it!