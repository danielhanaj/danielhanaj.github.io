+++
author = "Daniel Hanaj"
title = "Packer Explained"
date = "2021-08-02"
description = "Packer Explained"
featured = true
categories = ["Packer"]
tags = [
    "Packer",
    "DevOps",
    "Automation"
]
thumbnail = "images/2021/08/02/packer.png"
featureImage = "/images/2021/08/02/packer.png"
+++
In this blog post I will explain what Packer is and basic terminology that you need to get fimiliar with before we will start using it.
 <!--more-->
 ## **What Hashicorp Packer is?**
 Packer is an open source template creation tool. Packer is delivered as single binary and It's been designed to automate process of creating and customizing various types of templates in fully automated way.

## **What Hashicorp Packer is not?**
 * Packer is not a provisioning/deployment tool. A good example of provisioning tool is Terraform, CloudFormation, Pulumi.  
  * Packer is not a configuration management tool like Ansible, Puppet or DSC, although it can utilize those technologies to customize your templates.
 
 Packer has been designed to build and configure your templates. That's it!
 
 Once you start working with it you will realize how powerful tool it is. 

> I remember few years back when I worked in Globalization group we had a project where we had to create a massive number of localized VM templates for our vSphere environment. Multiple Windows OS versions with multiple language versions. Our localized products had to be tested on localized operating systems. Imagine the challenge - build all those images in a same way so they would behave exactly the same. Chinese, Japanese, Korean, French, Hungarian, Polish, Russian, name it. 
>
> First I have created a document that described what each template should contain, what options should be enabled, and which one should be disabled, what patches should be applied etc. That document was two A4 pages long.
>
>Second, someone had to create those templates for us. At that time we had a vendor which performed a product testing for us. I have provided that vendor my document and asked to create those templates for us. I did get a massive number of questions from vendor side, creating templates was very slow. Even completed our templates were not consistent as those have been created manually by different people. Manually means you can make a multiple mistakes within the process. 
>
>At the end we dropped off manual process as it was inefficient, expensive and time consuming. Hopefully we had a very talented scripting engineer within the team who spend around two months to create our own dedicated set of Powershell/PowerCLI scripts which did exactly what we wanted to do. The concept was similar to one that packer currently uses.
>
>Today you don't have to be a scripting master or spend significant amount of time to develop your own template creation tool, because you have Hashicorp Packer.

## **Infrastricture as a code**
Packer is an infrastructure as a code tool where your entire template configuration is defined within a code. Basically you need only few files and scripts which should be enough to build a template.

## **Declarative approach**
 To build your images Packer uses a template files that defined declaratively. Declarative, means you tell Packer what you want to get, not how to get it. 
 That's the difference between imperative and declarative.
 In imperative programing languages, like PowerShell, you need to tell your script what it needs to do step by step. All commands are performed in order defined within the script itself. If you miss one step your entire script will fail.
 In declarative programming languages you only needs to specify what you want to get, so basically you don't need to keep specify any order.

 ## **Cloud agnostic**
 Because Packer has been designed as a "swiss army knife" it allows you to provision your templates for various environments. You can create your templates on-premise or in public cloud. VMware, Hyper-V, Nutanix, Azure, GCP, AWS, Alibaba, name it. Because of that it's very flexible.
 > You may ask why I need to build my templates within public cloud? Cloud provider already provides me set of pre-defined images that I can use it straight away. The problem is that images delivered by cloud vendors are very often outdated, they do not include recent patches, or do not have necessary tools you need At the end very often after deployment you need to customize your OS, which is not very efficient.
 >
 ## **Multiplatform**
 Packer runs on multiple platforms – Windows, Linux, MacOS and can create templates for Windows, Linux, MacOS, Docker

 ## **Packer template format**
 To build your template you need to define it within a code. You can do it using JSON or HCL format. HCL is fairly new and it's been implemented quite recently.Frankly speaking it's good that HashiCorp standardize their tools using one common language, but at the moment most of template files you will find still use JSON format. I will focus on explaining how to define your templates in JSON which then can be converted to HCL natively by Packer.

 ## **Basic Terminology**
 Let's start from terminology which is very important as it will be used later one. 
 ### Packer Template 
 **Packer Template** is a file in JSON format where you define your image.
 ### Image
**Image** is the final output like vSphere template file you may see under "VM and Templates" section.
### Packer Builder 
**Packer Builder** is required parameter you need to define inside your **template**. You may think of builder as an engine that talks to your underlying infrastructure. Some builders are build in into packer binary and some you need to download separately and place under same location as Packer binary file. You may find a list of supported builders [here](https://www.packer.io/docs/builders). There is around 30 builders available on Hashicorp website, but that's not all. Thanks to community there is a massive number of third party builders created by specific vendors and published on github. So if you won't find specific builder on Hashicorp website, just search for it on Internet, there is a good chance you will find it.
### Packer Provisioner
**Packer Provisioner** is an optional parameter you may define in your **template**. Provisioners are used to install and configure software for machines created by builders. Same as builders some provisioners are build in into Packer binary, but most of them you will have to download separately and place in same location as Packer binary file. You can use multiple provisioners within same template file which makes them very flexible. There is a number of third party provisioners created by community, like Ansible, Chef, Puppet, DSC.
### Post-processor
**Post-processor** is an optional parameter you may define in your template file. Post-processors is an array of objects defining various steps to take once builder and provisioners parts are completed. As an example of post-processor operation is exporting template to ova/ovf format or perform operation directly on VM object, like removing additional CD-ROM from VM with using PowerCLI.
### Communicators
**Communicators** are protocols used for communication between Packer and machine that is being created (ssh for Linux and winrm/ssh for Windows) When you think of it, after machine has been created, Packer needs to connect with it to customize our OS, apply some scripts etc. That's what communicators are used for.
### User variables
**User variables** are key/value strings that defines user variables included in packer **template** file.
User variables allow to pass different parameters to Packer template file. Variables helps to apply different parameters to same JSON file. This is extremely helpful if we want to use same JSON template to build images on different vCenter servers.
Variables needs to be defined in packer JSON file within variables section and then defined in builders section 
>It is a good practice to place your sensitive information, like passwords, usernames as a variable and not placing those directly inside **template** file. Basically each value inside template file can be stored as variable. For example paths to your scripts, path to iso files, vSphere datastore names etc. If you do it wisely your template file can be universal and you will be able to use it in multiple environments, just changing specific variables that are stored separately.
>
Variables file has fallowing structure and it needs to be saved as separate JSON file:
```json
{
	    "vsphere_server": "vcenter01.lab.local",
	    "vsphere_user": "packer@vsphere.local"
	}
```
Now you know basic Packer terminology. In next blog post we will build our first Windows template on vSphere using Packer.




 

