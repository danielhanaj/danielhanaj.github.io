+++
author = "Danielhanaj"
title = "Create vCenter Role for Packer using Terraform"
date = "2021-06-25"
description = "Create vCenter Role for Packer using Terraform"
featured = true
categories = ["Packer"]
tags = [
    "Packer",
    "vCenter",
    "Role",
    "Terraform",
    "vSphere"
]
thumbnail = "images/2021/06/25/terraform.png"
featureImage = "/images/2021/06/25/terraform.png"


+++
In this blog post I would like to demonstrate how to create dedicated role for HashiCorp Packer and assign permissions on vCenter objects using Terraform.
<!--more-->
## **Custom Packer Role**
Most posts that demonstrate Packer use Administrator vCenter role to provision your Packer Images. This is good for demo or home lab usage, but not necessarily for production. In production you want to have just limited permission for account that provision your Packer Templates.
In old times where Packer vSphere-iso provisioner was delivered and maintained by JetBrains group, a few people asked that question. What is minimum access rights that needs to be assigned to make Packer to provision VM images? The answer was create role named Packer and assign fallowing permissions to that role:

* Datastore
  * Allocate space
  * Browse datastore
  * Low level file operations
* Host
  * Configuration -> System Management
* Network
  * Assign Network
* Resource
  * Assign virtual machine to resource pool
* Virtual Machine
  * All

Next assign vSphere account to that role as fallowing:

Object     | Role      | Propagation
-----------|-----------|------------
Datacenter | Read-Only | No
Host       | packer    | No
Network    | packer    | Yes
Datastore  | packer    | Yes
Folder     | packer    | Yes

I've decided to automate that process as much as possible using Terraform. Probably it would be easier to do it with PowerCLI but my goal is to learn Terraform as much as possible.

For this reason I've used Terrafrom vSphere Provider version 1.26

Unfortunately at time of writing this article Terraform vSphere Provider does not allowed me to create local vSphere accounts so you have to do it manually in vCenter Inventory. You can also use PowerCLI for that. In my example I've created local account named Packer@vsphere.local. Hopefully that's the only manual step you need to do, rest will be done by Terraform.

First, install Terraform. I use Windows so used Chocolatey to install Terraform binary. Once you have it, clone [my](https://github.com/danonh/Packer-vSphere-Windows.git) GitHub repo.  Under "01 - Terraform vCenter Role" you will find three files: main.tf file and two variable files. The one you are interested with is terrafrom.tfvars where you need to specify your vSphere objects values. In my case file looks like this:

```hcl
username       = "administrator@vsphere.local"
password       = "VMware1!"
vcenter        = "vcsa01.lab.local"
datacenter     = "DC01"
cluster        = ["CL01"]
esxihost       = ["esx01.lab.local"]
datastore      = ["datastore1"]
network        = ["VM Network"]
folder         = ["/DC01/vm/Packer"]
vsphereaccount = "vsphere.local\\Packer"
```

## **Square Brackets Values**
You may wonder what the square brackets are and what they are for? Square brackets in variable file means those are treated as lists of values. Your Datacenter object may contain multiple clusters and those contain multiple ESXi hosts. Additionally different clusters usually contain different datastores, networks and folder structures.
Normally you want to assign extended rights to be assigned on single ESXi object. This script supports it so means you can put multiple clusters, ESXihosts, datastores, networks and folders. The condition is that those must belong to same Datacenter object. So considering single DC that contain multiple clusters you put fallowing values:

```hcl
username       = "administrator@vsphere.local"
password       = "VMware1!"
vcenter        = "vcsa01.lab.local"
datacenter     = "DC01"
cluster        = ["CL01", "CL02"]
esxihost       = ["esx01.lab.local", "esx02.lab.local"]
datastore      = ["datastore1", "datrastore2"]
network        = ["VM Network", "DPortGroup"]
folder         = ["/DC01/vm/Packer", "/DC01/vm/Packer1"]
vsphereaccount = "vsphere.local\\Packer"
```
## **Apply configuration**
Open folder where your Terraform configuration is stored and run `terraform init` command:

```text
PS C:\Terraform\Role1\Packer vSphere Role> terraform init

Initializing the backend...

Initializing provider plugins...
- Checking for available provider plugins...
- Downloading plugin for provider "vsphere" (hashicorp/vsphere) 1.26.0...

The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, it is recommended to add version = "..." constraints to the
corresponding provider blocks in configuration, with the constraint strings
suggested below.

* provider.vsphere: version = "~> 1.26"


Warning: registry.terraform.io: This version of Terraform has an outdated GPG key and is unable to verify new provider releases. Please upgrade Terraform to at least 0.12.31 to receive new provider updates. For details see: https://discuss.hashicorp.com/t/hcsec-2021-12-codecov-security-event-and-hashicorp-gpg-key-exposure/23512



Warning: Interpolation-only expressions are deprecated

  on main.tf line 3, in provider "vsphere":
   3:   password       = "${var.password}"

Terraform 0.11 and earlier required all non-constant expressions to be
provided via interpolation syntax, but this pattern is now deprecated. To
silence this warning, remove the "${ sequence from the start and the }"
sequence from the end of this expression, leaving just the inner expression.

Template interpolation syntax is still used to construct strings from
expressions when the template includes multiple interpolation sequences or a
mixture of literal strings and interpolations. This deprecation applies only
to templates that consist entirely of a single interpolation sequence.

(and 7 more similar warnings elsewhere)

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

Once done, Terraform vSphere provider should be downloaded and placed in same directory as your terraform configuration files.

Next run `terraform plan` command. The output will tell you what and how it will be generated.

```text
PS C:\Terraform\Role1\Packer vSphere Role> terraform plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.vsphere_folder.folder["/DC01/vm/Packer"]: Refreshing state...
data.vsphere_role.readonly: Refreshing state...
data.vsphere_datacenter.dc: Refreshing state...
data.vsphere_datastore.datastore["datastore1"]: Refreshing state...
data.vsphere_host.host["esx01.lab.local"]: Refreshing state...
data.vsphere_network.net["VM Network"]: Refreshing state...
data.vsphere_compute_cluster.cluster["CL01"]: Refreshing state...

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # vsphere_entity_permissions.p1 will be created
  + resource "vsphere_entity_permissions" "p1" {
      + entity_id   = "datacenter-2"
      + entity_type = "Datacenter"
      + id          = (known after apply)

      + permissions {
          + is_group      = false
          + propagate     = false
```

Finally run `terraform apply --auto-approve` command and this will apply configuration on your vCenter server:

```text
PS C:\Terraform\Role1\Packer vSphere Role> terraform apply --auto-approve
data.vsphere_datacenter.dc: Refreshing state...
data.vsphere_folder.folder["/DC01/vm/Packer"]: Refreshing state...
data.vsphere_role.readonly: Refreshing state...
data.vsphere_host.host["esx01.lab.local"]: Refreshing state...
data.vsphere_network.net["VM Network"]: Refreshing state...
data.vsphere_datastore.datastore["datastore1"]: Refreshing state...
data.vsphere_compute_cluster.cluster["CL01"]: Refreshing state...
vsphere_entity_permissions.p1: Creating...
vsphere_role.role2: Creating...
vsphere_entity_permissions.p1: Creation complete after 1s [id=datacenter-2]
vsphere_role.role2: Creation complete after 1s [id=-1879812451]
vsphere_entity_permissions.p4["datastore1"]: Creating...
vsphere_entity_permissions.p6["/DC01/vm/Packer"]: Creating...
vsphere_entity_permissions.p3["esx01.lab.local"]: Creating...
vsphere_entity_permissions.p2["CL01"]: Creating...
vsphere_entity_permissions.p5["VM Network"]: Creating...
vsphere_entity_permissions.p6["/DC01/vm/Packer"]: Creation complete after 0s [id=group-v82]
vsphere_entity_permissions.p4["datastore1"]: Creation complete after 0s [id=datastore-54]
vsphere_entity_permissions.p2["CL01"]: Creation complete after 0s [id=domain-c7]
vsphere_entity_permissions.p3["esx01.lab.local"]: Creation complete after 0s [id=host-12]
vsphere_entity_permissions.p5["VM Network"]: Creation complete after 0s [id=network-17]

Warning: Interpolation-only expressions are deprecated

  on main.tf line 3, in provider "vsphere":
   3:   password       = "${var.password}"

Terraform 0.11 and earlier required all non-constant expressions to be
provided via interpolation syntax, but this pattern is now deprecated. To
silence this warning, remove the "${ sequence from the start and the }"
sequence from the end of this expression, leaving just the inner expression.

Template interpolation syntax is still used to construct strings from
expressions when the template includes multiple interpolation sequences or a
mixture of literal strings and interpolations. This deprecation applies only
to templates that consist entirely of a single interpolation sequence.

(and 7 more similar warnings elsewhere)


Apply complete! Resources: 7 added, 0 changed, 0 destroyed.
```

## **Results**
As you can see Terraform has successfully created 7 resources and you can verify those under vCenter that Packer role has been created and Packer@vsphere.local account has that role assigned on lister vSphere objects.

## **Multiple datacenters**
You may manage multiple datacenter objects under single vCenter server. My Terraform configuration allow you to specify only single datacenter objects and does not allow you to specify multiple datacenter objects.. So what to when you want to assign Packer role on multiple datacenter objects? Simply delete .tfstate file and re-run configuration specify new datacenter values in terrafrom.tfvars. 

Thanks,

