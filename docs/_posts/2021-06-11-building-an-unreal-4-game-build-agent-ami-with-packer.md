---
layout: post
title:  "Building an Unreal 4 Game Build Agent AMI with Packer"
date:   2021-06-11 17:34:12 -0800
categories: aws packer windows unreal
---

You've been building your game in UE4, and you're finally ready to stop building locally. Or you have a Jenkins/Azure DevOps/Gitlab/Bamboo CI/CD platform and you want to scale your builds into AWS so you don't have to buy more metal in your office. Maybe your "build machine" is just another development workstation that could absolutely be more useful with your new Engine Developer who starts tomorrow. AWS has a lot of compute at their disposal in the EC2 service, and you can offload your builds there. This is a guide on how to get the bare minimum going so you have a working build machine image with your specific UE4 version.

If you're using a Jenkins/Bamboo/CircleCI tool, this can be used for your elastic build agents. If you use Azure DevOps, then you can use [Terraform](https://terraform.io) to spin up a cluster of agents that self register and are ready to take builds. This guide won't walk through setting up First-Party SDKs from Microsoft or Sony, or the Third Party tools like Havoc or Wwise. It's a starting point, everyone's game is a little different.

# Repo
<https://github.com/alexholliz/w2019-ue-agent>

# Prereqs
1. Have an AWS Account and VPC to spin stuff up in, note your region and VPC ID
2. A *private* S3 Bucket for your Unreal Engine versions
2. Have Packer installed


# AWS

## Packer IAM Role

[Take a look here for further detail](https://alexanderhollis.com/aws/packer/windows/2021/05/18/building-a-windows-server-ami-with-packer.html)

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

## S3 IAM User

Create another IAM user, with a memorable s3 related name, so you can find them, and generate a second Access Key ID and Secret Access Key; this is what you'll use to push content into S3 to use during your AMI build. Assign a policy that allows R/W to S3 to that IAM User. Out of the box, that's an Amazon Provided Policy called `AmazonS3FullAccess`.

## S3 Bucket

Create a new *PRIVATE* bucket. S3 Buckets are globally addressable, so if you make it public, people can find and pull from your S3 Bucket. This will cost you a significant amount of money. Don't do that. S3 Buckets must have a globally unique name; go ahead and give it a name that's relevant to you, and note that down, you'll need it later. Also note the region you're using, as you'll reference your S3 Bucket and EC2 instances by region.

## Local AWS Config

Install the [AWS CLI](https://aws.amazon.com/cli/)

Then hop over to your home directory (where that is depends on your OS) and make the [config and credentials files](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)

Those will look something like the following, with your S3 User's access key ID, secret access key, and region in place.


`~/.aws/credentials`
```
[default]
aws_access_key_id=AKIAIOSFODNN7EXAMPLE
aws_secret_access_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

`~/.aws/config`
```
[default]
region=us-west-2
output=json
```

Do a quick test of your AWS CLI config once you've got that in place by running an AWS CLI command to list all the buckets in your account.
```
aws s3 ls
```

# Unreal Engine

## Getting the Unreal Engine

1. Install the [Epic Games Launcher](https://www.epicgames.com/store/en-US/download)
2. Log in and hop over to the "Unreal Engine" header, and accept the license agreements, etc...
3. Under the Library tab, click the '+' sign next to Engine Versions and add the target engine version for your project
4. Click Install on the chiclet for your version, and pick a spot with 40G+ of space. Note the form for the install dir, you should have a UE_4.XX directory being added.
5. Wait for the install to finish, it takes a while

## Pushing Unreal Engine to S3

With your AWS Config and Credentials in place, this is pretty easy!

1. Open up your Powershell/Cmder/Terminal etc... and change directories to the install dir for the engine you set up earlier. `X:\installdir\UE_4.XX`
2. Do an S3 sync from inside the install directory, to your bucket with a subdir of the same name
```
aws s3 sync . s3://bucketname/UE_4.XX/
```
3. Wait a long time again, this is ~35GB so it can take a while

# Packer

## Packer Environment Variables

In my previous walkthrough of packer, we just needed 2 environment variables, the Access Key ID and Secret Access Key for your IAM User with EC2 access. We need more this go-round. Packer has a syntax you can use to set environment variables that will be analyzed at runtime. In a CI/CD platform, these would be secrets. For local development on your machine, set these locally like this:

```
export PKR_VAR_AKID=AMADEUPACCCESSKEY
export PKR_VAR_SKEY=0101MADEUPSECRETKEY42069
```

Or on Windows:

```
setx PKR_VAR_AKID AMADEUPACCCESSKEY
setx PKR_VAR_SKEY 0101MADEUPSECRETKEY42069
```

However you set them, your list of variables needs to include:
* PKR_VAR_AWS_REGION
* PKR_VAR_PACKER_IAM_AKID
* PKR_VAR_PACKER_IAM_SKEY
* PKR_VAR_S3_BUCKET
* PKR_VAR_S3_IAM_AKID
* PKR_VAR_S3_IAM_SKEY
* PKR_VAR_UNREAL_ENGINE_VERSION
* PKR_VAR_VPC_ID

## Packer File

Packer provides a pretty solid starting point: <https://www.packer.io/docs/builders/amazon/ebs#connecting-to-windows-instances-using-winrm> and for our purposes we are going to use WinRM to communicate. It's pretty slick, only requiring the powershell to enable winrm to be present when you're doing the packer build.

What follows is _my_ packer script for setting up a Windows 2019 Server with the Desktop Experience and Container support. Here's the important bits:

* Variables: Set your subnet ID, VPC ID, region, etc... in the file
* force_deregister: set this to true to overwrite your existing AMI when you rebuild (great for images you plan to keep up to date automatically)
* Source AMI Filter: This is how you set the different filter options to autoselect your AMI based on that query. In our case I want a Windows 2019 image, Full experience, with Container Support. The options in the filter match what you'd select in the AMIs sub-selection of the EC2 area of AWS.
* most_recent: This is just below the filter for your AMI, but it says if the filter returns multiple results, use the latest one.
* Owners: That string means Amazon. In the old JSON format for Packer, you could just write amazon. I am sure I could figure that out eventually, but the string works, and I am lazy.
* user_data_file: the LOCAL path to your WinRM enabling script.
* WinRM options: you'll see those are required in the WinRM enabling script, ez.
* Provisioner powershell: This runs after the instance is up, but before it is turned into an AMI. We're going to use this to install all of our build dependencies. Then we'll run the recommended tasks to set a random password on next boot and sysprep the image so it looks like a new machine everytime something is spun up from this AMI.

```
variable "aws_region" {
  type = string
}

variable "subnet_id" {
  type = string
}

variable "vpc_id" {
  type = string
}

variable "AKID" {
  type = string
}

variable "SKEY" {
  type = string
}

variable "S3_BUCKET" {
  type = string
}

variable "UNREAL_ENGINE_VERSION" {
  type = string
}

variable "S3_AKID" {
  type = string
}

variable "S3_SKEY" {
  type = string
}

# source blocks are generated from your builders; a source can be referenced in
# build blocks. A build block runs provisioner and post-processors on a
# source. Read the documentation for source blocks here:
# https://www.packer.io/docs/from-1.5/blocks/source
source "amazon-ebs" "packer-w2019-ue-agent" {
  access_key       = "${var.AKID}"
  secret_key       = "${var.SKEY}"
  ami_name         = "packer-w2019-ue-agent"
  communicator     = "winrm"
  force_deregister = true
  instance_type    = "c5.large"
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
    owners = ["801119661308"]
  }
  launch_block_device_mappings {
    device_name           = "/dev/sda1"
    volume_size           = 100
    volume_type           = "gp3"
    delete_on_termination = true
  }
  subnet_id      = "${var.subnet_id}"
  user_data_file = "./packer/scripts/autogenerated_password_https_bootstrap.txt"
  vpc_id         = "${var.vpc_id}"
  winrm_insecure = true
  winrm_use_ssl  = true
  winrm_username = "Administrator"
  shutdown_behavior = "terminate"
}

# a build block invokes sources and runs provisioning steps on them. The
# documentation for build blocks can be found here:
# https://www.packer.io/docs/from-1.5/blocks/build
build {
  sources = ["source.amazon-ebs.packer-w2019-ue-agent"]

  provisioner "powershell" {
    script = "./packer/scripts/disable_uac.ps1"
  }

  provisioner "powershell" {
    script = "./packer/scripts/packages.ps1"
  }

  provisioner "powershell" {
    script = "./packer/scripts/visual_studio.ps1"
  }

  provisioner "powershell" {
    script = "./packer/scripts/awscli.ps1"
  }

  provisioner "powershell" {
    environment_vars = ["BUCKET=${var.S3_BUCKET}", "UNREAL_ENGINE_VERSION=${var.UNREAL_ENGINE_VERSION}", "AWS_ACCESS_KEY_ID=${var.S3_AKID}", "AWS_SECRET_ACCESS_KEY=${var.S3_SKEY}", "AWS_DEFAULT_REGION=${var.aws_region}"]
    script           = "./packer/scripts/unreal_engine.ps1"
  }

  provisioner "powershell" {
    inline = ["C:/ProgramData/Amazon/EC2-Windows/Launch/Scripts/SendWindowsIsReady.ps1 -Schedule", "C:/ProgramData/Amazon/EC2-Windows/Launch/Scripts/InitializeInstance.ps1 -Schedule", "C:/ProgramData/Amazon/EC2-Windows/Launch/Scripts/SysprepInstance.ps1 -NoShutdown"]
  }
}
```

## Provisioner Scripts

These series of Powershell Scripts will install Chocolatey, a package manager that will then let us install Visual Studio and the AWS CLI. Then later provisioners will pull down and set up the Unreal Engine so we can build games with this AMI.

### Disabling UAC

UAC prompts popping up would prevent automatic installers from completing. So we disable that in `scripts/disable_uac.ps`

{% highlight posh %}
$ErrorActionPreference = 'Stop'

try {
  Write-Host "Disabling UAC…"
  New-ItemProperty -Path HKLM:Software\Microsoft\Windows\CurrentVersion\Policies\System -Name EnableLUA -PropertyType DWord -Value 0 -Force
  New-ItemProperty -Path HKLM:Software\Microsoft\Windows\CurrentVersion\Policies\System -Name ConsentPromptBehaviorAdmin -PropertyType DWord -Value 0 -Force
}

catch {
  Write-Error "Failed to disable UAC"
  exit 1
}
{% endhighlight %}

### Chocolatey

[Chocolatey](https://chocolatey.org/install) Is a package manager for Windows. Like homebrew on MacOS or Apt/Yum for flavors of Linux. Our first powershell provisioner under `scripts/packages.ps` installs Chocolatey so we can use it later to install more packages.

{% highlight posh %}
$ErrorActionPreference = 'Stop'

try {
  #PowerShellGet requirest NuGet installation.
  Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force

  # Install AWS CLI
  Import-Module AWSPowerShell

  # Install Chocolatey
  Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

  # Globally Auto confirm every action
  choco feature enable -n allowGlobalConfirmation
}

catch {
  Write-Error "Failed to disable UAC"
  exit 1
}
{% endhighlight %}

### Visual Studio

In `scripts/visual_studio.ps` we use Chocolatey to install Visual Studio 2019 Community edition, as well as the C++ Game Development Workload, as those are requirements for building games with the Unreal Engine. It should be noted that the Community Edition of Visual Studio is not licensed for commercial projects. If you're leaning that way, Spin up your own internal chocolatey for packages to be installed from. Then you can securely install the licensed version of Visual Studio automatically.

{% highlight posh %}
$ErrorActionPreference = 'Stop'

try {
  # Install Visual Studio 2019 Community
  choco install visualstudio2019community

  #Install Game Development for C++ Workload for Visual Studio 2019
  choco install visualstudio2019-workload-nativegame
}

catch {
  Write-Error "Failed to install packages"
  exit 1
}
{% endhighlight %}

### AWS CLI and how it's different from AWS Powershell

Officially provided Windows AMI include AWS Powershell, which didn't work for me. so in `scripts/awscli.sh` I opted to use the previously installed Chocolatey package manager to install the AWS CLI we already used to push the Unreal Engine into S3 to also be responsible for pulling it back out.

{% highlight posh %}
$ErrorActionPreference = 'Stop'

try {
  choco install awscli
}

catch {
  Write-Error "Failed to install awscli"
  exit 1
}
{% endhighlight %}

### Pulling from S3

Finally, in `scripts/unreal_engine.ps` we pull down the Unreal Engine from S3 and drop it in `C:\Unreal\UE_4.XX`.

{% highlight posh %}
$ErrorActionPreference = 'Stop'

try {
  Set-Location -Path "C:\"
  mkdir Unreal
  mkdir Unreal\$env:UNREAL_ENGINE_VERSION

  aws --version
  aws s3 ls

  Write-Host "Pulling down Unreal Engine version $env:UNREAL_ENGINE_VERSION from S3"
  Write-Host "aws s3 sync s3://$env:BUCKET/$env:UNREAL_ENGINE_VERSION C:\Unreal\$env:UNREAL_ENGINE_VERSION --only-show-errors"

  aws s3 sync s3://$env:BUCKET/$env:UNREAL_ENGINE_VERSION C:\Unreal\$env:UNREAL_ENGINE_VERSION --only-show-errors

  Get-ChildItem -Path C:\Unreal\ -Name
}

catch {
  Write-Error "Failed to pull Unreal Engine from S3"
  exit 1
}
{% endhighlight %}

## What's next?

Once your build is done, you'll have your own Amazon AMI to make build machine instances out of. I think [Terraform](https://www.terraform.io/) is a great tool for that. Additionally, with Terraform, if you were to have a CI/CD platform you wanted these instances to register with so they could take jobs automatically, you could have the agent provisioning as a component of that setup!

If manual is more your speed, you just start EC2 Instances using that AMI as a base, and then set up Perforce or Git or whatever to pull in your source code, add your first party SDKs, or third party middleware installers, and start building.