# Overview
This document provides a quick walkthrough of using the DC/OS community edition advanced installer to stand up a basic DC/OS cluster on a set of CentOS 7.3 boxes (built using the 1611 minimal ISO).

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
*Hint* DC/OS requires an odd number of masters.
---

## Setup of prerequisites.

