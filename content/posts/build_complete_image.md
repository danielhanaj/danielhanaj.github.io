+++
author = "Daniel Hanaj"
title = "Build complete Windows image"
date = "2021-08-24"
description = "Build complete Windows image"
featured = true
categories = ["Packer"]
tags = [
    "Packer",
    "DevOps",
    "Automation",
    "Windows",
    "Windows updates"
]
thumbnail = "images/2021/11/01/updates.png"
featureImage = "/images/2021/11/01/updates.png"
+++
In this blog post we will build complete production ready Windows image. Let's crack it out!  
 <!--more-->
If you had a chance to read my previous posts you should be already aware how to build basic Packer Windows image using variables. This is a good start but we are just at the beginning. Currently our image does not contain any Windows updates installed and still has WinRM enabled which is serious security bridge in term of using it in production. Beside that image that has been created contains two virtual CD-ROM devices which is ok, but the goal is to leave only one virtual CD-ROM. Let's start fixing those gaps.


# Production ready image
Main question is what ideal Packer image should contain? The answer is - it depends on your needs. In my opinion production ready Windows image should not have any third party software installed. Instead security should be our priority. `Remember that image you are creating can be used to deploy hundreds production systems!`
The image which will be created will have fallowing features:
* All current Windows updates installed,
* Enable RDP port to allow Remote Desktop connectivity with image that is being created,
* Disable WinRM once all configuration work will be done,
* Install .NET 4.6 on the image,
* Cleanup the image removing windows updates cache,
* Disable WinRM once entire customization will be completed,
* Remove virtual CD-ROM leaving only one drive available

Of course you can customize your image as you want, it's completely up to you and you have unlimited possibilities with Packer, but try to make it simple as possible.

## Apply Windows updates
I will use Windows Update provisioner created by @rgl.
Simple download 



Packer variables allow you re-use same Packer template file multiple times within different vSphere environments. If we would hardcode all settings inside template file, like we did in my previous [post](https://blog.danielhanaj.com/post/2021/08/16/build_basic_windows_image_with_packer/), we would then have to create Packer template file for each vSphere instance. You will probably agree that this is not very efficient as we need change only few settings inside existing template file , mostly for vCenter server objects, to re-use it within different environment.. 
>My assumption is that you want to build your images across different vCenter instances. Even with single vCenter instance variables can be quite helpful as you could use same template file to build Windows Server 2016 and Windows Server 2019. 
>
Since we want to fully automate image build process using Azure DevOps, it's advised to use variables for security and flexibility reasons.

## Define variable inside template file
To define variables we need to first define variables section inside Packer template file. Variable naming convention is up to you, but I suggestto use names that are self explanatory. Also do not use spaces naming your variables.  

Below is an example how to define variable section:
```yaml
"variables": {
    "vsphere_server": "",
    "vsphere_user": "",
    "vsphere_password": ""
  },
```
In this example we want to set three variables for vCenter server name, vCenter username and vCenter password. Next we need to define those variables inside Packer template file:
```yaml
 "builders": [
    {
      "type": "vsphere-iso",

      "vcenter_server":      "{{user `vsphere_server`}}",
      "username":            "{{user `vsphere_user`}}",
      "password":            "{{user `vsphere_password`}}",
      "insecure_connection": "true"
    }
 ]
 ```
## Define variable values
 To define values for our variables we need to create dedicated variable file. Same as Packer template file, variable file is stored in JSON format. You can use random naming convention, but for simplicity, let's name our file `vars.json` and store it in same directory as Packer template file. Variable file should look like this: 
```yaml
{
    "vsphere_server": "vcsa01.lab.local",
    "vsphere_user": "packer@vsphere.local",
    "vsphere_password": "VMware1!"
    }
```
## Which values should you store as variable?
Packer is very flexible and allow to store each option in template file as variable. This is very handy. For example, you can store patch to you answer file as variable, place answer files in dedicated directories and point Packer to proper answer file depending which OS you want to create. From the other hand I don't see the point of storing options like CPU number or amount of memory as variable. Basically those values are not important as, at the end, we change those values during provision process based on system/user requirements.
So what values should be stored as variable? From my experience those should be:  
Variable     | Description
-----------|-----------
vsphere_server         | Stores vCenter server name.
vsphere_user           | Stores vCenter username.
vsphere_password       | vSphere password for account that has required permissions to create images.
vsphere_folder         | Stores vCenter Folder path where our image will be placed.
vsphere_dc_name        | Stores vCenter datacenter name.
vsphere_cluster        | Stores vCenter cluster name where our image will be placed.
vsphere_host           | Stores ESXi host name where our image will be placed
vsphere_portgroup_name | Stores vCenter portgroup name where our image will be connected to. 
vsphere_datastore      | Store vCenter datastore name where our image will be deployed.
winadmin_password      | Windows password used by WinRM to connect with image to perform customization.
os_iso_path            | Path to datastore where Windows installation media is placed.
vmtools_iso_path       | Path to datastore where VMware Tools installation media is placed.
autounattend_path      | Path to autounattend.xml answer file.
template_name          | Image name that we want to apply on the image.
version                | VM hardware version that we want to apply on our image.

You may wonder why we need to put so many variables into single template file? Well - because each vSphere environment can be different. You may have vSphere 6 and 6.7 that you manage so for each environment image hardware version can be different etc. It will make more sense later on once we start build our images fully automatically, trust me.  
>Ok, but why I have specified "vsphere_host" since "vSphere _cluster" has been set as variable? The thing is that account that we specified and assigned through [terraform](https://blog.danielhanaj.com/post/2021/06/25/vcenterrole/), provides permissions on cluster level without propagation. We assign sufficient permissions on host level to better secure our environment. 
>
At the end you should get one additional file inside your folder and all listed variables should be offloaded from your template file.  
Under [03 - Packer variables](https://github.com/danonh/Packer-vSphere-Windows.git) folder you will find working set of files with offloaded variable files. You can simply place your sensitive values inside `vars.json` file and run: `packer build -var-file=".\vars.json" windows-server-2016.json` command once you are in directory with your template file.  
This command will tell Packer to use *vars.json* as variable file and *windows-server-2016.json* as main template file.

```txt
PS C:\Packer\Blog_variables> packer build -var-file=".\vars.json" windows-server-2016.json

vsphere-iso: output will be in this color.

==> vsphere-iso: Creating VM...
==> vsphere-iso: Customizing hardware...
==> vsphere-iso: Mounting ISO images...
==> vsphere-iso: Adding configuration parameters...
==> vsphere-iso: Creating floppy disk...
    vsphere-iso: Copying files flatly from floppy_files
    vsphere-iso: Copying file: Autounattend_Windows2016/autounattend.xml
    vsphere-iso: Copying file: scripts/winrm.ps1
    vsphere-iso: Copying file: scripts/vmtools.ps1
    vsphere-iso: Done copying files from floppy_files
    vsphere-iso: Collecting paths from floppy_dirs
    vsphere-iso: Resulting paths from floppy_dirs : []
    vsphere-iso: Done copying paths from floppy_dirs
==> vsphere-iso: Uploading created floppy image
==> vsphere-iso: Adding generated Floppy...
==> vsphere-iso: Set boot order temporary...
==> vsphere-iso: Power on VM...
==> vsphere-iso: Waiting for IP...
==> vsphere-iso: IP address: 192.168.1.88
==> vsphere-iso: Using winrm communicator to connect: 192.168.1.88
==> vsphere-iso: Waiting for WinRM to become available...
    vsphere-iso: WinRM connected.
==> vsphere-iso: Connected to WinRM!
==> vsphere-iso: Shutting down VM...
==> vsphere-iso: Deleting Floppy drives...
==> vsphere-iso: Deleting Floppy image...
==> vsphere-iso: Eject CD-ROM drives...
==> vsphere-iso: Convert VM into template...
==> vsphere-iso: Clear boot order...
Build 'vsphere-iso' finished after 18 minutes 28 seconds.

==> Wait completed after 18 minutes 28 seconds

==> Builds finished. The artifacts of successful builds are:
--> vsphere-iso: Windows2016-Template
```

## Variables inside Windows answer file
You are probably wonder if variables can be used inside answer file to re-use autounattend.xml file with different Windows versions? Autounattend.xml does not support variables natively, but with help of Azure DevOps third party tools we can bypass variables into autounattend.xml as well. This will be demonstrated in later posts once we will start to create our images using Azure DevOps.

In next blog we will tweak our image using provisioners and post-processors.

Thanks for reading!