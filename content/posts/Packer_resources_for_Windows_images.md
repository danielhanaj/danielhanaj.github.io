+++
author = "Danielhanaj"
title = "Packer resources for Windows images"
date = "2021-08-03"
description = "Packer resources for Windows images"
featured = true
categories = ["Packer"]
tags = [
    "Packer",
    "DevOps",
    "Automation"
]
thumbnail = "images/2021/08/03/github.png"
featureImage = "/images/2021/08/03/github.png"
+++
In this blog post I will try to provide you some interesting links related to Packer and Windows server builds. Using those and modifying according your needs will make your life much simpler when creating your first Packer image.
 <!--more-->

   
# Packer Resources
You can find plenty Packer resources on Internet. Some of them are better than others. In this section I will point you to the one which I find very useful.

## Matt Wrock HashiConf session on YouTube
The must watch at first place is a Matt Wrock session named "Lightweight Windows Vagrant Boxes with Packer". We do not build Vagrant Boxes here but Matt provides plenty of useful information how to build Windows templates using Packer. This session is quite old, but all information pointed there are accurate.
{{< youtube tKj5plMcfTY >}}

The main thing I have remembered from Matt's session is "Bring back WinRM to pristine state once you end up configuring your template". Placing WinRM in insecure mode (which we need to do during template creation as Packer needs to talk with Windows) brings huge risk to you and your company. Templates you create can be used to deploy hundreds of VMs across your entire organization, that is why it's very important to properly secure it. Our goal is to build secured, up do date, customized templates in fully automated way. So remember to **disable WinRM at the end of building process!!!**

## Rui Lopes Github repo
The second resource which I found very useful is a [Rui Lopes](https://twitter.com/ruiglopes) [github](https://github.com/rgl/windows-vagrant) repository. Actually this is my favorite one as Rui's repo contains multiple packer template files, Windows answer files and multiple useful configuration scripts that you can use on Windows. Having a scripts (mostly PowerShell) that you can rely on is a must have when you build your production templates.

>For example for some time I have used a simple cmd script that silently installed VMtools on VM. The script worked well on all Windows versions, except Windows Server 2012 where it didn't worked at all. As I didn't wanted to use separate script for Windows Server 2012 and other Windows versions I have used a PowerShell script from Rui's repository which worked well on all Windows versions.
>
Additionally Rui has developed a extremely useful Packer provisioner called [packer-plugin-windows-update](https://github.com/rgl/packer-plugin-windows-update). As you can imagine this provisioner allows you to install most recent MS patches and apply those on the image you create on. Frankly speaking this provisioner is a must have for every Sysadmin that creates Windows Packer images. I use it all the time and it works really well.

## Stefan Scherer Github repo
Third repo I find very useful is [Stefan Scherer](https://github.com/StefanScherer/packer-windows). Stefan works for Docker and updates his Packer template files quite recently.This is my second choice repo as Stefan has toked different approach and a lot of configuration scripts are installed through answer file (Autounattend.xml). In my opinion answer file should be clean and simple as much as possible, only critical scripts should be installed through answer file.

## Boxcutter Github repo
[This one](https://github.com/boxcutter/windows) has been mentioned during Matt Wrock HashiConf session. A lot of scripts for various operating systems, not only Windows one

Those are the ones you should have in mind when looking for examples and configuration scripts for your Packer images.

Thanks for reading.



