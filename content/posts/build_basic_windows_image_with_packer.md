+++
author = "Danielhanaj"
title = "Build basic Windows image with Packer on vSphere"
date = "2021-08-16"
description = "Build basic Windows image with Packer on vSphere"
featured = true
categories = ["Packer"]
tags = [
    "Packer",
    "DevOps",
    "Automation",
    "VMware",
    "vSphere",
    "Windows"
]
thumbnail = "images/2021/08/07/windows.png"
featureImage = "/images/2021/08/07/windows.png"
+++
In this blog post I will demonstrate how to build basic Windows 2016 image on vSphere using Packer.
 <!--more-->
This will be very simple Windows installation to demonstrate how Packer works. In this blog post we will use only Packer vSphere builder, without any provisioner, post-processor or variables. This mean our image won't be customized at all. We will look how to use provisioners, post-processors and variables in later posts. If you don't know what Packer builders, provisioners or post-processors are, check out my previous [blog](https://blog.danielhanaj.com/post/2021/08/02/packer_explained/) entry.
# Installing HashiCorp Packer
Packer is delivered as single binary that is available for Windows, Linux or Mac.  
## Installing Packer on Windows
Before we dive in into building our first image we need to install Packer on our machine.  
If you use Windows as I am, simplest way to install Packer is to use [Chocolatey](https://chocolatey.org/install).
If you are not familiar with Chocolatey, it's a package manager for Windows. On Linux you use yum or apt to obtain packages on your machine. Chocolatey use same concept, but on Windows. I guarantee that once you will start using Chocolatey you will never again go back and install applications through Windows installer.
After you install Chocolatey, run fallowing command from PowerShell:
``` ps
choco install -y packer
```
It's a good practice to install Visual Studio Code and Git to edit Packer files and obtain my github repositories. Using Chocolatey you can install multiple packages with single command:
``` ps
choco install -y packer, visualstudiocode, git
```
With this command Chocolatey will install Packer, Visual Studio Code and Git packages on your Windows machine.  
You can verify that packer has been installed running fallowing command:
```txt
>packer version
Packer v1.7.4
```
If you install Packer through Chocolatey, packer will be automatically added to environment variables on your local PC, means you can use packer command from any path on your Windows.

## Installing Packer on Linux
Run fallowing commands from your Linux session:
``` bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install packer
```

# Build basic Windows 2016 image on vSphere
I assume you have already created dedicated account with proper role and permissions on your vSphere environment. If you didn't check [this](https://blog.danielhanaj.com/post/2021/06/25/vcenterrole/) blog post how to create it using Terraform.  
To build very basic Windows image using Packer we will need just few files. Those are:
  - [Packer Template File](/post/2021/08/16/build_basic_windows_image_with_packer/#packer-template-file)
  - [Windows answer file for unattended installation](/post/2021/08/16/build_basic_windows_image_with_packer/#windows-answer-file-for-unattended-installation)
  - [Windows Server 2016 installation iso file](/post/2021/08/16/build_basic_windows_image_with_packer/#windows-server-2016-installation-iso-file)
  - [VMware Tools iso file](/post/2021/08/16/build_basic_windows_image_with_packer/#vmware-tools-iso-file)
  - [WinRM enable script](/post/2021/08/16/build_basic_windows_image_with_packer/#winrm-enable-script)
  - [VMware Tools install script](/post/2021/08/16/build_basic_windows_image_with_packer/#vmware-tools-install-script)

## Packer Template File
Main JSON template file where we specify your image configuration and parameters. This file is quite simple to read. First we define [builder](https://blog.danielhanaj.com/post/020821_packer_explained/#packer-builder) that we want to use. 
>HashiCorp delivers vmware, vsphere-iso and vsphere-clone builders. VMware builder is used to build images on VMware Workstation/Fusion platforms. A vsphere-iso and vsphere-clone builders are designed for vSphere environment although those are used for different purposes. vSphere-iso is used to build the image from iso file, and we will use this in our case. The vsphere-clone builder is used to clone existing image and perform necessary changes on existing template. 
>

``` yaml
"builders": [
    {
      "type": "vsphere-iso",
```
We use proper configuration options to establish communication with our vSphere environment using packer@vsphere.local account, then we define parameters for our image:
```yaml
"vcenter_server": "vcsa01.lab.local",
      "username": "packer@vsphere.local",
      "password": "VMware1!",
      "insecure_connection": "true",

      "vm_name": "Windows2016-Template",
      "folder": "Packer",
      "datacenter": "DC01",

      "cluster": "CL01",
      "datastore": "DS-Intel-SSD",
      "network_adapters": [
        {
          "network": "VMNetwork",
          "network_card": "vmxnet3"
        }
      ],
      "convert_to_template": "true",

      "vm_version": "15",
      "guest_os_type": "windows9Server64Guest",

      "communicator": "winrm",
      "winrm_username": "Admin",
      "winrm_password": "VMware1!",

      "CPUs": "2",
      "CPU_hot_plug": "true",
      "RAM": "4096",
      "RAM_hot_plug": "true",
      "RAM_reserve_all": false,
      "firmware": "bios",

      "disk_controller_type": "lsilogic-sas",
      "storage": [
        {
          "disk_size": 40960,
          "disk_thin_provisioned": true
        }
      ],
      "ip_settle_timeout": "120s",

```
Most of options are self-explanatory. The `ip_settle_timeout` defines amount of time to wait for VM's IP to settle down, sometimes VM may report incorrect IP initially, then its recommended to set that parameter to apx. 2 minutes.

Next we define iso paths for Windows and VMware Tools:
```yaml
"iso_paths": [
        "[DS-Intel-SSD] iso/en_windows_server_2016_vl_x64_dvd_11636701.iso",
        "[DS-Intel-SSD] iso/VMwareTools_11_0_1.iso"
      ],
```
Last part is to provide path to scripts that will be included on virtual floppy disk:
```yaml
 "floppy_files": [
        "Autounattend_Windows2016/autounattend.xml",
        "scripts/winrm.ps1",
        "scripts/vmtools.ps1"
      ]
```
To configure our basic image we need answer file with two scripts to configure WinRM and VMware Tools installation.  
>It's a good practice to keep configuration files in separate folders. This will make more sense when you start using variables. Using variables you will be able to point Packer to different folder to obtain different answer file.
>


## Windows answer file for unattended installation
Autounattend.xml file which is answer file used by windows for unattended installation. Answer file is very sensitive and can be very complex. It may contain a lot of settings that can be configured during image creation. Answer file can also contain a list of scripts that can be triggered automatically during image creation. Those scripts needs to exist on virtual floppy disk that is attached to your image. A good thing is that Packer creates such virtual floppy disk for you once specifying file paths and attach it automatically during image creation. Another good thing about scripts specified in answer file is that those do not need to use any Packer provisioners to make initial image configuration. In our example we will enable WinRM communicator on VM through script that will be triggered by autounattend.xml. Same thing will do for VMware Tools installation.  
You can build your own autounattend.xml file using dedicated tool called [Windows System Image Manager](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install), but this task is quite complex and requires a lot of time. I suggest to use my answer file or use one which you will find here [Windows Packer github repos](https://github.com/danonh/Packer-vSphere-Windows.git) under section "02 - Basic Windows Template with Packer/Autounattend_Windows2016/". If you want to check other answer files created, check my previous [blog entry](https://blog.danielhanaj.com/post/2021/08/03/packer_resources_for_windows_images/), where I lisk a useful Packer repositories.
There is no point to invent the wheel if others did great work creating those for us and sharing with community.
> Windows 2016 and Windows 2019 answer files are almost identical so you can use it to build both types of images. The only difference are Windows versions that needs to be specified in two places under answer file and license key that is OS specific. Windows Server 2012, Windows10 answer files are completely different, so if you decide to build those, use dedicated answer files.
>

## Windows Server 2016 installation iso file
You can specify iso path directly in Packer Template File using evaluation media provided by Microsoft. Packer will then download file locally on your machine and attach it to VM during image creation. It's quite convenient functionality but I prefer to download iso first, and then upload it on vSphere datastore. I know, that's one additional manual step you need to do, but this iso file will be used multiple times by Packer so it's worth spend some additional effort and do it prior image creation.

## VMware Tools iso file
VMware Tools are released separately from ESXi and you can grab proper version [here](https://packages.vmware.com/tools/esx/index.html). Same as Windows iso file, download version that is compatible with your current vSphere version and place it on datastore where you store your iso files.

## WinRM enable script
A script that will enable WinRM on our image. This script will be placed on virtual floppy disk that will be attached to our image during OS installation and will be triggered by answer file. Once triggered Packer will be able to establish communication with image to perform some additional customization
## VMware Tools install script
A script that will install VMware Tools fully automatically. VMware VMTools are released separately from ESXi and you can grab proper version [here](https://packages.vmware.com/tools/esx/index.html)
> Both WinRM and VMware Tools install scripts are crucial for building process. Without properly installed VMware Tools, VMXNET network adapter won't be detected by Windows. Without networking, Packer cannot obtain image IP and perform custom on image itself.

Great WinRM and VMware Tools installation scripts are provided by Rui Lopes github repo. I use those all the time and those works great.
To obtain all files needed to build your first Windows 2016 image use fallowing (github) repo.


## Packer and IP Address
If network you use to provision Packer image already have DHCP server configured, it's fine and everything will work as expected.  
If your network does not have DHCP server configured things gets more complicated. To connect with image we are creating, Packer use WinRM communicator. Anyhow to establish WinRM communication, our Windows image needs to have valid IP assigned. That IP needs to be accessible from machine where Packer build process is triggered.  
Packer template file does not allow us to specify static IP for our image. To workaround this issue you can do some trick which will assign static IP to our image through autounattend.xml file.  
Our template will use VMXNET3 network adapter which is recommended by VMware. Windows iso file does not contain VMXNET3 driver build-in, so to assign static IP for our system we need to install VMware Tools first. Once network adapter will be detected we can then assign IP to our VM. Both operations can be specified in autounattend.xml file under FirstLogonCommands section. This section defines what operation will be triggered once system will be installed.  
```xml
<FirstLogonCommands>
            <SynchronousCommand wcm:action="add">
                    <Order>1</Order>
                    <CommandLine>PowerShell "Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Force"</CommandLine>
                    <Description>Change the default PowerShell Execution Policy from Restricted to Unrestricted</Description>
                </SynchronousCommand>
                 <SynchronousCommand wcm:action="add">
                    <Order>2</Order>
                    <CommandLine>PowerShell -File a:\vmtools.ps1</CommandLine>
                </SynchronousCommand>
                <SynchronousCommand wcm:action="add">
                    <CommandLine>cmd.exe /c netsh interface ip set address "Ethernet0" static 192.168.1.88 255.255.255.0 192.168.1.1</CommandLine>
                    <Description>Assign static IP</Description>
                    <Order>3</Order>
                    <RequiresUserInput>true</RequiresUserInput>
                </SynchronousCommand>
                <SynchronousCommand wcm:action="add">
                    <CommandLine>cmd.exe /c netsh interface ip add dns "Ethernet0" 192.168.1.5</CommandLine>
                    <Description>Assign static DNS</Description>
                    <Order>4</Order>
                    <RequiresUserInput>true</RequiresUserInput>
                </SynchronousCommand>
                <SynchronousCommand wcm:action="add">
                    <!-- Enable WinRM service -->
                    <CommandLine>powershell -ExecutionPolicy Bypass -File a:\winrm.ps1</CommandLine>
                    <Order>5</Order>
                    <RequiresUserInput>true</RequiresUserInput>
                </SynchronousCommand>
                <SynchronousCommand wcm:action="add">
                    <CommandLine>cmd.exe /c "net user administrator /active:no"</CommandLine>
                    <Description>Disable local Administrator Account</Description>
                    <Order>6</Order>
                    <RequiresUserInput>true</RequiresUserInput>
                </SynchronousCommand>
            </FirstLogonCommands>
```
As you suspect Order defines what order listed commands will be triggered. In our example this will be fallowing:
1. Set PowerShell execution policy to "Unrestricted"
2. Install VMware Tools from virtual floppy disk using vmtools.ps1 script
3. Assign static IP, network mask, default gateway on Ethernet0 (change that according your needs)
4. Assign DNS server on Ethernet0 (change that according your needs)
5. Enable WinRM from virtual floppy disk using winrm.ps1 script
6. Disable local Administrator account
   > A good security practice is to disable default Administrator account. We will fallow that rule as our goal is to build secured image that will be used in production environment. Answer file which you used to build your first image creates additional account named Admin and adds it to local Administrators group. Admin account is the one that will be used to establish communication with our image and perform customization.  
   >
You can name you account as you want. To change it you need to modify answer file:
```xml
 <LocalAccounts>
                    <LocalAccount wcm:action="add">
                        <Description>Admin User</Description>
                        <DisplayName>Admin</DisplayName>
                        <Group>Administrators</Group>
                        <Name>Admin</Name>
                        <Password>
                            <Value>VMware1!</Value>
                            <PlainText>true</PlainText>
                        </Password>
                    </LocalAccount>
                </LocalAccounts>
```
and Packer template file:
```yaml
"winrm_username": "Admin",
      "winrm_password": "VMware1!",
```

   
## Perform build process
Once all files are properly modified, go to file directory with Packer template file and trigger build process running `packer build .\Windows-Server-2016.json` command:
```txt
PS C:\packer\Blog> packer build .\Windows-Server-2016.json

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
```
Build process will start and under defined directory in your vSphere environment you should see new VM that has been powered on. Once installation will complete you will get an image created and converted to template.

Like mentioned at the beginning this is just a basic configuration, definitely **not ready for production**, but it gives you overview how Packer works.  

In next blog I will demonstrate how to use variables inside Packer template file and why it is so helpful and useful to use those. 

Thanks for reading!