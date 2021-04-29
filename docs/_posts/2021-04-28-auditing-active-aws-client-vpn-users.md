---
layout: post
title:  "Auditing Active AWS Client VPN Users"
date:   2021-04-28 13:34:12 -0800
categories: aws snippet
---

Get a valid token for your target aws account. For those of us using SAML + MFA for our connections: (https://github.com/jmhale/okta-awscli)

Get a list of all AWS Client Vpn Endpoints in your region

{% highlight bash %}
aws ec2 --region us-west-1 describe-client-vpn-endpoints --query 'ClientVpnEndpoints[*].ClientVpnEndpointId' --output text --no-cli-pager
{% endhighlight %}

List all the users connected to each endpoint

{% highlight bash %}
aws ec2 --region us-west-1 describe-client-vpn-connections --client-vpn-endpoint-id cvpn-endpoint-yourverycoolendpointid --query 'Connections[?Status.Code==`active`].Username' --output text --no-cli-pager
{% endhighlight %}