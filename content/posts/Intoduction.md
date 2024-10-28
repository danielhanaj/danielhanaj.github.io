+++
author = "Daniel Hanaj"
title = "Fully automated Windows Templates with Packer, Terraform and Azure DevOps"
date = "2021-06-21"
description = "Fully automated Windows Templates with Packer, Terraform and Azure DevOps"
featured = true
categories = ["Packer"]
tags = [
    "Packer",
    "Azure DevOps"
]
thumbnail = "images/2021/06/21/devops1.png"
featureImage = "/images/2021/06/21/devops1.png"
+++
This is my first blog post and in next few blog entries I will demonstrate entire process how you can fully automatically build and customize vSphere Windows templates using Terraform, Packer and Azure DevOps. My goal is to deliver Windows templates that will be production ready - meaning, you will be able use those to provision production systems. Windows templates will be always up to date since we will rebuild those every month once Microsoft will release their patches. Let's get started then!
 <!--more-->

## **But Why?**
You may ask that question - why do I even need to bother? I already have my templates created and there is no need to touch them. My templates have all required pieces of software in place, there is no point to spend my time on developing new template process. If there is a need to transfer those I can export my templates to ova/ovf format and copy/replicate it to remote locations. I can also use vSphere Content Library feature to replicate my templates across the site.

There is few important reasons why you want to automate template creation process:

1. **Template corruption** - this can happen, you may have some luck and get an template copy from remote location so you can bring it back, but it's time consuming process and require a lot of manual steps
   
2. **Accidental template deletion** - your colleague has accidentally deleted your production template. Same you can bring template back from remote location, but it's time consuming. You can also call to your backup admin asking to restore your template. He will probably tell you he is too busy or that your template was not backed up as it was not treated as critical.
3. **Updates** - you have to manually update your templates every single month on every location to make those up to date. You don't need to do it, but once you deploy VM from outdated template your SCCM client will need ages to bring the system up to date and apply all missing security patches. Hope you have SCCM or any other endpoint management tool, right?
> I remember few years back when I've worked for a company which had multiple sites across UK-EMEA region. After each VM creation we had to wait few hours until newly deployed machine was fully patched. When we asked our SCCM team why this process take so long, they said that if we are not happy we should update our templates with recent patches. Guess what - no one even touched those templates. No one wanted to spend few hours each month to update all our templates on every location. We accepted that VM patching will take few hours and business will have to wait a little bit longer to get their systems up and running.
> 
4. **Your templates become larger** - each Windows update process keeps some additional files on your system. With each update your template become larger and that's something you want to avoid. You want to make those templates light. Rebuilding your templates every single month will make them light without keeping unnecessary files. 
   
5. **Avoid human mistakes** - how many times you had to make some changes on your templates? If you had multiple tasks to do on every template on every location there is a chance you forget to do something on one of them. Automated template creation remove human mistake risk.
6. **French office requires French Windows** - you may need to have localized version of your production template, so you need create it and maintain it. Do you speak French? No - then you have a problem as you will have to spend twice as much time to create it. Real challenge? Try to build Chinese version of your template. Packer will solve that as building process is language agnostic.
7. **You can learn something new** - Packer is a universal DevOps tool that is used across different public clouds and on-premise environments. Have you noticed I used DevOps term? That's right learning Packer you learn DevOps! Everyone loves DevOps, right?
8. **You can automate** - automation is a must have in every organization. Imagine you tell your manager: "Hey - from now on our template creation process will be fully automated. You will always have your templates up to date without spending any additional time on this. I may leave the company but each month you will get your fresh, updated Windows template". I guarantee that he will love the idea and will give you a green light to continue on this project(at least I would do that if I would be your manager).
9. **It's Free** - Packer is an Open Source product, mean you can use it for free. Everyone loves that word - right? As for Azure DevOps it's also free up to 5 users with one self-hosted agent and one parallel job. And that's exactly what we need, so you can build your template creation solution at no costs.
## First things first
But what the Hashicorp Packer really is? I will try to answer that question in my next blog post.

<!-- Thanks to Hashicorp Packer and Azure DevOps entire process can be fully automated. Your templates will be always the same and you will be able to provision those on multiple sites and vCenter instances. You will be able to configure different schedule for building process depending on time zone so you will make sure that building process won't affect production.
## What problems Packer solves
VMware uses Content Libaries concept to replicate resources across vCenter instances. It use the replication between sites, but it's not very convenient in my opinion. You also needs to have multiple vCenter instance to perform replication. What if you have single vCenter server that manages multiple datacenters?

 -->
