+++
author = "Daniel Hanaj"
title = "Create Docker image with Packer, PowerShell, PowerCLI and Azure DevOps self-hosted agent"
date = "2021-12-28"
description = "Create Docker image with Packer, PowerShell, PowerCLI and Azure DevOps self-hosted agent"
featured = true
categories = ["Packer"]
tags = [
    "Packer",
    "DevOps",
    "Azure",
    "Docker",
    "PowerCLI",
    "Automation"
]
thumbnail = "images/2021/12/27/docker.png"
featureImage = "/images/2021/12/27/docker.png"
+++
Today we will create a Docker image that will contain Packer, Windows Update Provisioner for Packer, PowerShell, PowerCLI and Azure DevOps self-hosted agent.  
 <!--more-->
To automate Packer image creation process we will utilize Azure DevOps, which is a service hosted in public cloud. What we want to do is to create Packer image on-premise in our datacenter, so there is a problem with communication between vCenter server and Azure DevOps. 

# Azure DevOps agent

## Why Azure DevOps agent is needed?
Normally Azure DevOps does not have any connection with our datacenter. 
To establish connection between on-premise vCenter server and Azure DevOps, a dedicated agent placed inside our datacenter is needed. That agent needs to have network connection with our vCenter server and ESXi hosts. It can be placed in different VLAN, but that VLAN needs to be routable with vCenter and ESXi VLAN. 

## What Azure Devops agent does? 
It actually triggers a tasks initiated from Azure DevOps portal locally in our infrastructure. All files reuired to create our builds needs to be first copied on Azure Devops agent and agent triggers the build job. The files copied on agent from Azure DevOps server are called artifacts.
Because agent triggers a build jobs, it needs to be equipped with all necessary tools that are required to create Packer template.
Those are:
* Packer
* Packer Windows Update provisioner
* PowerShell
* PowerCLI

## Self-hosted Azure DevOps agent
Azure DevOps agent can be hosted in Azure or it can be hosted locally in our DC. Obviously we will need self-hosted agent.  
First of all it allows direct communication with our datacenter. Second thing - it's free. For Azure DevOps agent hosted in Azure you normally needs to pay some money.  
Of course everything depends on subscription level you own, but my assumption is that you don't have any available subscription and want to minimize operational costs.  
`Remember, or goal is to build secure templates and minimize operational costs!`  
To establish communication between DevOps server and on-premise vCenter server a dedicated token will be used. This token will have to be generated on Azure Devops portal.

## Support platforms for DevOps agents
Azure DevOps agent can be installed on Windows or on Linux. We will choose Linux as our destination platform and there is few reasons for that:
* When utilizing Linux you don't need to pay for Windows license. Since Azure DevOps agent will just trigger our build tasks, there is no need to pay Windows license for such service.
* When using Linux as agent platform we can utlize Docker and automate process of creating a docker image will all necessary tools. 
>You could probably use some bash script to automate process of installing and configuring all necessary tools on Linux, but using Docker will be much faster and efficient way to achieve expected result.
* Since DevOps agent keeps all artifacts locally in clear text (including passwords), security is a major concern here. It's very easy to erase Windows password using some small 3rd party tool and get access to all artifacts and all passwords on that agent. Using Linux and Docker container make a things much harder for potential attacker. We will use other methods to secure data on agent machine but Linux is much better option in terms of security.

# Build Ubuntu VM
## Create Ubuntu 21.04 Packer image
We could use Windows Subsystem for Linux (WSL) to speedup Docker image creation process but we want to build our final solution which will utilize Ubuntu VM. For this reason we will use Ubuntu 21.04. To create Ubuntu VM Template using Packer use this [link](https://www.virtualizationhowto.com/2021/05/packer-build-ubuntu-21-04-for-vmware-vsphere/).

## Customize Ubuntu VM
To make life easier first enable root account and setup new root account password.

```console
sudo passwd root
```
Once done switch to root account typing command:
```console
su root
```
Username will change from your existing user to root. This means we have elevated privileges on the system.
### Set static IP
```console
cd /etc/netplan/
ls
```
Once done a configuration file will be displayed

Edit the file using nano editor:
```console
nano 00-installer-config.yaml
```
Edit IP, gateway and DNS settings to match your network configuration:
```yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens192:
      addresses:
      - 192.168.1.222/24
      gateway4: 192.168.1.1
      nameservers:
        addresses: [192.168.1.1, 8.8.8.8]
        search: [lab.local]
  version: 2
```
Type Ctrl+x and confirm saving changes typing y.
```console
netplan apply
```
This will apply network settings that has been modified.
From now putty can be used to interact with VM through SSH console

### Change hostname
Perform fallowing command to change hostname to ld9-vcsabackups01
```console
sudo hostnamectl set-hostname devopsagent01
```
Provide server description using fallowing command:
```console
sudo hostnamectl set-hostname "Azure DevOps Self-hosted Agent" --pretty
```
Once applied verify that changes have been applied
```console
hostnamectl
```
### Upgrade system
Perform server upgrade
```console
sudo apt-get update
sudo apt-get-upgrade
```
## Install Docker Engine and Docker Compose
### Install Docker Engine
Docker Engine allow us to run standalone containers.

First remove existing Docker Engine if exists:
```console
sudo apt-get remove docker docker-engine docker.io containerd runc
```
Update the apt package index and install packages to allow apt to use a repository over HTTPS:
```console
 sudo apt-get update
 sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
Add Docker’s official GPG key:
```console
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
Use the following command to set up the stable repository.
```console
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

```
Install Docker Engine
```console
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```
Check Docker version:
```console
docker --version

Docker version 20.10.10, build b485636
```
### Install Docker Compose
Compose is a file format for describing distributed Docker apps, and it’s a tool for managing them.
Docker Compose relies on Docker Engine for any meaningful work, so make sure you have Docker Engine installed either locally or remote.

On Linux, you can download the Docker Compose binary from the [Compose repository release page on GitHub](https://github.com/docker/compose/releases).

Run this command to download the current stable release of Docker Compose:
```console
sudo curl -L "https://github.com/docker/compose/releases/download/v2.2.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
Apply executable permissions to the binary:
```console
sudo chmod +x /usr/local/bin/docker-compose
```
Note: If the command docker-compose fails after installation, check your path. You can also create a symbolic link to /usr/bin or any other directory in your path.

```console
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```
Test the installation:
```console
docker-compose --version

Docker Compose version v2.2.2
```
Once Docker Engine and Docker Compose will be installed we need to add existing, non-root user to docker group to perform docker tasks without privileged mode:

```console
sudo usermod -aG docker your-user-name
```
Once added, restart your SSH session to apply changes.

## Preparing for Docker container image creation
Inside your home directory create new folder named "Azure-DevOps-Agent":
```console
mkdir Azure-DevOps-Agent
cd Azure-DevOps-Agent/
```
Once we are inside devopsagent folder create new file named `Dockerfile`:
```console
nano Dockefile
```
To build container image we need a file named `Dockerfile` which will include fallowing content:

```docker
# Indicates the base image.
FROM ubuntu:18.04

# Define Args for the needed to add the package
ARG PACKER_VERSION="1.7.8"
ARG PACKER_WINDOWSUPDATE_VERSION="0.14.0"
ARG PS_VERSION=7.1.3
ARG PS_PACKAGE=powershell_${PS_VERSION}-1.ubuntu.18.04_amd64.deb
ARG PS_PACKAGE_URL=https://github.com/PowerShell/PowerShell/releases/download/v${PS_VERSION}/${PS_PACKAGE}
ARG TARGETARCH=amd64
ARG AGENT_VERSION="2.195.2"

# Define ENVs for Localization/Globalization
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false \
    LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    # set a fixed location for the Module analysis cache
    PSModuleAnalysisCachePath=/var/cache/microsoft/powershell/PSModuleAnalysisCache/ModuleAnalysisCache \
    POWERSHELL_DISTRIBUTION_CHANNEL=PSDocker-Ubuntu-18.04
ENV PACKER_VERSION=${PACKER_VERSION}
ENV PACKER_WINDOWSUPDATE_VERSION=${PACKER_WINDOWSUPDATE_VERSION}
ENV DEBIAN_FRONTEND=noninteractive
RUN echo "APT::Get::Assume-Yes \"true\";" > /etc/apt/apt.conf.d/90assumeyes

# Install dependencies and clean up
RUN apt-get update \
    && apt-get install --no-install-recommends -y \
    # curl is required to grab the Linux package
        curl \
    # less is required for help in powershell
        less \
    # requied to setup the locale
        locales \
    # required for SSL
        ca-certificates \
        gss-ntlmssp \
    # PowerShell remoting over SSH dependencies
        openssh-client \
    # Install python
    python3 \
    python3-pip\
    python3-boto\
    # Install unzip
    unzip \
    # Install DevOps Agent packages
    jq \
    git \
    iputils-ping \
    libcurl4 \
    libicu60 \
    libunwind8 \
    netcat \
    libssl1.0 \
    # Download the Linux package and save it
    && echo ${PS_PACKAGE_URL} \
    && curl -sSL ${PS_PACKAGE_URL} -o /tmp/powershell.deb \
    && curl -LO https://releases.hashicorp.com/packer/${PACKER_VERSION}/packer_${PACKER_VERSION}_linux_amd64.zip \
    && curl -LO https://github.com/rgl/packer-plugin-windows-update/releases/download/v${PACKER_WINDOWSUPDATE_VERSION}/packer-plugin-windows-update_v${PACKER_WINDOWSUPDATE_VERSION}_x5.0_linux_amd64.zip \
    && unzip '*.zip' -d /usr/bin \
    && chmod +x /usr/bin/packer-plugin-windows-update_v${PACKER_WINDOWSUPDATE_VERSION}_x5.0_linux_amd64 \
    && rm *.zip \
    && mv /usr/bin/packer-plugin-windows-update_v${PACKER_WINDOWSUPDATE_VERSION}_x5.0_linux_amd64 /usr/bin/packer-plugin-windows-update \
    && curl -LsS https://aka.ms/InstallAzureCLIDeb | bash \
    && apt-get install --no-install-recommends -y /tmp/powershell.deb \
    && apt-get dist-upgrade -y \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && locale-gen $LANG && update-locale \
    # remove powershell package
    && rm /tmp/powershell.deb \
    # intialize powershell module cache
    # and disable telemetry
    && export POWERSHELL_TELEMETRY_OPTOUT=1 \
    && pwsh \
        -NoLogo \
        -NoProfile \
        -Command " \
          \$ErrorActionPreference = 'Stop' ; \
          \$ProgressPreference = 'SilentlyContinue' ; \
          while(!(Test-Path -Path \$env:PSModuleAnalysisCachePath)) {  \
            Write-Host "'Waiting for $env:PSModuleAnalysisCachePath'" ; \
            Start-Sleep -Seconds 6 ; \
          }"

RUN apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

WORKDIR /azp
RUN if [ "$TARGETARCH" = "amd64" ]; then \
      AZP_AGENTPACKAGE_URL=https://vstsagentpackage.azureedge.net/agent/${AGENT_VERSION}/vsts-agent-linux-x64-${AGENT_VERSION}.tar.gz; \
    else \
      AZP_AGENTPACKAGE_URL=https://vstsagentpackage.azureedge.net/agent/${AGENT_VERSION}/vsts-agent-linux-${TARGETARCH}-${AGENT_VERSION}.tar.gz; \
    fi; \
    curl -LsS "$AZP_AGENTPACKAGE_URL" | tar -xz

COPY ./start.sh .
RUN chmod +x start.sh


# Install the VMware.PowerCLI Module
SHELL [ "pwsh", "-command" ]
RUN Install-Module VMware.PowerCLI, PowerNSX -Force -Confirm:0;
RUN Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -confirm:$false

## Install DevOps Agnet
#CMD [ "pwsh" ]
#CMD    ["/bin/bash"]
ENTRYPOINT [ "./start.sh" ]
```
In same directory as `Dockerfile` create new file named `start.sh`
```console
$ nano start.sh
```
and paste there fallowing content:
```sh
#!/bin/bash
set -e
if [ -z "$AZP_URL" ]; then
	  echo 1>&2 "error: missing AZP_URL environment variable"
	    exit 1
fi
if [ -z "$AZP_TOKEN_FILE" ]; then
	  if [ -z "$AZP_TOKEN" ]; then
		      echo 1>&2 "error: missing AZP_TOKEN environment variable"
		          exit 1
			    fi
			      AZP_TOKEN_FILE=/azp/.token
			        echo -n $AZP_TOKEN > "$AZP_TOKEN_FILE"
fi
unset AZP_TOKEN
if [ -n "$AZP_WORK" ]; then
	  mkdir -p "$AZP_WORK"
fi
export AGENT_ALLOW_RUNASROOT="1"
cleanup() {
	  if [ -e config.sh ]; then
		      print_header "Cleanup. Removing Azure Pipelines agent..."
		          # If the agent has some running jobs, the configuration removal process will fail.
			      # So, give it some time to finish the job.
			          while true; do
					        ./config.sh remove --unattended --auth PAT --token $(cat "$AZP_TOKEN_FILE") && break
						      echo "Retrying in 30 seconds..."
						            sleep 30
							        done
								  fi
							  }
						  print_header() {
							    lightcyan='\033[1;36m'
							      nocolor='\033[0m'
							        echo -e "${lightcyan}$1${nocolor}"
							}
						# Let the agent ignore the token env variables
						export VSO_AGENT_IGNORE=AZP_TOKEN,AZP_TOKEN_FILE
						source ./env.sh
						print_header "1. Configuring Azure Pipelines agent..."
						./config.sh --unattended \
							  --agent "${AZP_AGENT_NAME:-$(hostname)}" \
							    --url "$AZP_URL" \
							      --auth PAT \
							        --token $(cat "$AZP_TOKEN_FILE") \
								  --pool "${AZP_POOL:-Default}" \
								    --work "${AZP_WORK:-_work}" \
								      --replace \
								        --acceptTeeEula & wait $!
						print_header "2. Running Azure Pipelines agent..."
						trap 'cleanup; exit 0' EXIT
						trap 'cleanup; exit 130' INT
						trap 'cleanup; exit 143' TERM
						# To be aware of TERM and INT signals call run.sh
						# Running it with the --once flag at the end will shut down the agent after the build is executed
						./run.sh "$@" --once &
						wait $!
```
Alternatively once you are under your home directory run fallowing command:
```console
git clone https://github.com/danonh/Azure-DevOps-Agent
```
Once done you should see Azure-DevOps-Agent folder which will contain three files:
```
├── Dockerfile
├── Readme.md
└── start.sh
```

You can find both files on my [GitHub account](https://github.com/danonh/Azure-DevOps-Agent)

### Changing product versions
Within Dockerfile I've hardcoded particular product versions:
* Packer version 1.7.8
* Windows Update Provisioner version 0.14.0
* PowerShell version 7.1.3
* Azure DevOps agent version 2.195.2

Obviously you may change product versions according your needs. This can be done modifying ARG values directly in the Dockerfile or provide alternative values during building process which will be passed to Dockerfile.

## Build Docker image
Finally we are at stage where building process can be initiated. 

If product versions has been manually changed directly inside Dockerfile then you can simply run fallowing command:
```console
cd Azure-DevOps-Agent/
docker build -t packercontainer:local .
```
This will initiate building process that may take up to 15 minutes.

Alternatively you can pass particular product versions into docker build command:
```console
docker build -t packercontainer:local --build-arg PS_VERSION="7.1.4" --build-arg PACKER_VERSION="1.7.7" .
```
With those arguments we are telling Docker to include PowerShell version 7.1.4 instead 7.1.3 and Packer version 1.7.7 instead 1.7.8.

Don't forget to include `.` at the end of build command. This tells where Docker should look for Dockerfile.

Now sit down and relax.

If everything will go well you should see fallowing console output:
```console
Removing intermediate container 281635ad7c01
 ---> fc1495d19d30
Step 24/24 : ENTRYPOINT [ "./start.sh" ]
 ---> Running in f2d498a10b01
Removing intermediate container f2d498a10b01
 ---> ae65ba27dd06
Successfully built ae65ba27dd06
Successfully tagged packercontainer:local
```
This would mean that our docker image has been created successfully. You can check if container image is visible running fallowing command:
```console
docker images
```
The output should look like this:
```console
docker images
REPOSITORY        TAG       IMAGE ID       CREATED          SIZE
packercontainer   local     ae65ba27dd06   15 minutes ago   2.25GB
```
To run the container we would have to pass few specific values from our Azure DevOps account, but that will be topic for some other blog entry.

Thanks for reading.


