# Overview
This document provides a quick walkthrough of using the DC/OS community edition advanced installer to stand up a basic DC/OS cluster on a set of CentOS 7.3 boxes (built using the 1611 minimal ISO).

This document isn't meant to be used to build a production-ready stack (it's missing a lot of the security configurations, etc.).  Rather, this is a quick-start guide to stand up a basic DC/OS cluster with distributed masters.

---

I prefer the advanced installation method.  In my opinion, it's much easier to set up, use, and troubleshoot.*

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

Once the systems have finished rebooting, you can verify that the above two steps were successful by running `getenforce` and `lsmod | grep overlay`*

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
