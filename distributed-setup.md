# Overview
This document provides a quick walkthrough of using the DC/OS community edition advanced installer to stand up a basic DC/OS cluster on a set of CentOS 7.3 boxes (built using the 1611 minimal ISO).

---

*Hint: I prefer the advanced installation method.  In my opinion, it's much easier to set up, use, and troubleshoot.*

---

Requirements: 6 systems, all configured as follows:
CentOS 7.3 (CentOS-7-x86_64-Minimal-1611.iso)
Static IP addresses
SSH access, with sudo permissions (using linux user 'admin' with sudo access)

### Architecture:
This walkthrough details the deployment of a 3-master cluster, with one public agent and one private agent.  For the purposes of this walkthrough, these are the IP addresses used:

Bootstrap node: 
172.16.125.20

Master nodes:
172.16.125.21
172.16.125.22
172.16.125.23

Public Agent node:
172.16.125.25

Private Agent node:
172.16.125.26

---

*Hint: DC/OS requires an odd number of masters.*

---

## Setup of prerequisites.

* Time synchronization
All DC/OS nodes must be synchronized via a standard time synchronization mechanism.  centos7.3 comes with `chrony` configured out of the box.  `ntpd` will also work.  You can verify chrony sync state with this:

```bash
$ chronyc tracking
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
