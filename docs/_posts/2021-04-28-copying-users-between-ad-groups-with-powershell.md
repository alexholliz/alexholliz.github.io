---
layout: post
title:  "Copying Users between AD Groups with PowerShell"
date:   2021-04-28 10:58:12 -0800
categories: powershell snippet
---

List the members of your source group:

{% highlight posh %}
Get-ADGroupMember -Identity "source_ad_group" | Select-Object Name | Sort-Object Name
{% endhighlight %}

Do a foreach-object to write every member of your source group to your target group

{% highlight posh %}
Get-ADGroupMember -Identity "source_ad_group" | Foreach-object {Add-ADGroupMember -Identity "target_ad_group" -Members $_.distinguishedName}
{% endhighlight %}

