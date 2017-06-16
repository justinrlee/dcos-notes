# Overview
This document provides a quick walkthrough of using the DC/OS community edition advanced installer to stand up a basic DC/OS cluster on a set of CentOS 7.3 boxes (built using the 1611 minimal ISO).

This document isn't meant to be used to build a production-ready stack (it's missing a lot of the security configurations, etc.).  Rather, this is a quick-start guide to stand up a basic DC/OS cluster with distributed masters; it's meant to familiarize new users with the installation method in general.

---

*I prefer the advanced installation method.  In my opinion, it's much easier to set up, use, and troubleshoot.*

---

Requirements: 6 systems, all configured as follows:
CentOS 7.3 (CentOS-7-x86_64-Minimal-1611.iso)
Static IP addresses
SSH access, with sudo permissions (using linux user 'admin' with sudo access)

## Architecture:
This walkthrough details the deployment of a 3-master cluster, with one public agent and one private agent.  For the purposes of this walkthrough, these are the IP addresses used:

Bootstrap node:
* 172.16.125.20

Master nodes:
* 172.16.125.21
* 172.16.125.22
* 172.16.125.23

Public Agent node:
* 172.16.125.25

Private Agent node:
* 172.16.125.26

---

*Hint: DC/OS requires an odd number of masters.*

---

# Prerequisites Installation / Configuration

This section details the system configurations that should be made and/or verified before installing DC/OS.

## Time synchronization (all nodes)

All DC/OS nodes must be synchronized via a standard time synchronization mechanism.  centos7.3 comes with `chrony` configured out of the box.  `ntpd` will also work.  You can verify chrony sync state with the `chronyc tracking` command:

```bash
chronyc tracking
Reference ID    : 45.33.84.208 (christensenplace.us)
Stratum         : 3
Ref time (UTC)  : Fri Jun 16 17:14:31 2017
System time     : 0.000268338 seconds slow of NTP time
Last offset     : -0.000494987 seconds
RMS offset      : 0.125952706 seconds
Frequency       : 8.250 ppm slow
Residual freq   : -0.038 ppm
Skew            : 1.565 ppm
Root delay      : 0.024103 seconds
Root dispersion : 0.002404 seconds
Update interval : 516.8 seconds
Leap status     : Normal
```

## Disable the firewall (all nodes)

Disable the firewall on all nodes:

```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```


## Enable the overlay kernel module (all nodes)

DC/OS requires the use of the overlay linux kernel module.  It can be enabled by running this command, which will create the overlay configuration file at /etc/modules-load.d/overlay.conf:

```
sudo tee /etc/modules-load.d/overlay.conf <<-'EOF'
overlay
EOF
```

This will not take effect until the system is rebooted.  The next step (disable SELinux) includes a reboot.

## Set SELinux to 'permissive' mode (all nodes)

As of version 1.9.0, DC/OS currently does not support SELinux.  SELinux must be set to permissive mode (or disabled) in order to install and run DC/OS.

This is a two step process:
- Change the /etc/selinux/config file SELINUX mode to 'permissive'
- Reboot the system

You can change the config file with this sed command:

```bash
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
```

Then, reboot the system:
```
sudo init 6
```

---

*Once the systems have finished rebooting, you can verify that the above two steps were successful by running `getenforce` and `lsmod | grep overlay`*

---


## Install Docker (all nodes)

While Docker images can be run on DC/OS using the Universal Container Runtime (UCR), as of 1.9.0 the Docker engine is still a prerequisite for the installation process to complete.  This is because we actually use Docker containers to deploy some of the packages.

This is a three step process:
- Configure CentOS with the Docker yum repo
- Configure Docker to use OverlayFS (basically, we're configuring a systemd override file before Docker is first installed and started
- Install and the Docker engine

1. Set up the Docker yum repo:

```bash
sudo tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
```

2. Configure Docker to use OverlayFS (we're creating a systemd override configuration file in a docker.service.d directory)

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d

sudo tee /etc/systemd/system/docker.service.d/override.conf <<- 'EOF'
[Service]
Restart=always
StartLimitInterval=0
RestartSec=15
ExecStartPre=-/sbin/ip link del docker0
ExecStart=
ExecStart=/usr/bin/dockerd --graph=/var/lib/docker --storage-driver=overlay
EOF
```

3. Install Docker engine (1.13.1), using yum, then enable the systemd unit and start it.

```bash
sudo yum install -y docker-engine-1.13.1

sudo systemctl enable docker
sudo systemctl start docker
```


---

*Once the systems have finished rebooting, you can verify docker is running with overlay by running `sudo docker info | grep Storage`*

---

## Other requirements (all nodes)

DC/OS also has a couple other small requirements: you must install `ipset` and `unzip` and you must add the linux group `nogroup`.

Install `unzip` and `ipset`:

```bash
sudo yum install -y unzip ipset
```

Create the group `nogroup`:

```bash
sudo groupadd nogroup
```

# Install DC/OS

Now that all of the requirements are set up, the basic installation process for DC/OS is as follows:

Set up the bootstrap node
- Create a workspace directory on your bootstrap node
- Download the installer to your bootstrap node
- Create the `genconf` directory in your workspace
- Populate your `genconf/ip-detect` file
- Populate your `genconf/config.yaml` file
- Generate the configuration generation script
- Host the `genconf/serve` directory via nginx

On each master:
- Create workspace directory
- Download the `dcos_install.sh` script from the bootstrap node
- Run the `dcos_install.sh` script with the `master` option


On each Agent:
- Create workspace directory
- Download the `dcos_install.sh` script from the bootstrap node
- Run the `dcos_install.sh` script with the `slave` option

On each Public Agent:
- Create workspace directory
- Download the `dcos_install.sh` script from the bootstrap node
- Run the `dcos_install.sh` script with the `slave_public` option

---

*DC/OS uses the `pkgpanda` package manager, instead of yum or apt or some other package management tool, in order to be fully cross-platform compliant.  The above process basically turns your bootstrap node into a pkgpanda repository (hosted over http on nginx)*

*The installation script is then downloaded to each node, and run from each node.  The installation script essentially runs a bunch of pkgpanda download and installs to install of the DC/OS components*

---

## Set up the bootstrap node:

Create a workspace directory on your bootstrap node, and cd to it (I'm using 190 to refer to DC/OS version 1.9.0; you can use whatever directory you want):

```bash
mkdir 190
cd 190
```

In the dcos_190 directory, download the installer (or use some mechanism such as scp to get it over to the bootstrap node and put it in dcos_190):

```bash
curl -LO https://downloads.dcos.io/dcos/stable/commit/0ce03387884523f02624d3fb56c7fbe2e06e181b/dcos_generate_config.sh
```

Create the genconf directory within the 190 directory

```bash
mkdir genconf
```

Use vi to create a genconf/ip-detect file.  This is used to self-identify the IP address that will be used for internal communication between nodes.  When run, it should output an IP address that is reachable from all nodes within the cluster (this is also used as the 'hostname' for the DC/OS UI.  Examples of ip-detect files for different environments are available here: https://dcos.io/docs/1.8/administration/installing/custom/advanced/

For the purposes of this document, I'm using a very simple ip-detect file that, when run, outputs the first ip address.  This may or may not be suitable for your environment (for example, in some environments, depending on the shell, this will return a different IP address, which will cause issues).

This script is copied to each node in your cluster, so it should work on all cluster nodes.

> vi genconf/ip-detect

```
#!/bin/sh
hostname -I | awk '{print $1}'
```

Use vi to create a genconf/config.yaml.  Make sure that the bootstrap URL matches the output of ip-detect when run from the bootstrap node, and make sure the master IPs match the outputs of ip-detect run on your mater nodes.

> vi genconf/config.yaml

```yml
---
bootstrap_url: http://10.10.0.69
cluster_name: 'dcos-oss'
exhibitor_storage_backend: static
master_discovery: static
master_list:
 - 10.10.0.201
resolvers:
- 8.8.4.4
```



<!--- # Todo: Add '>' style commands for better markdown output. --->
