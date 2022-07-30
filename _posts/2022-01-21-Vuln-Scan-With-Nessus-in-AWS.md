---
title: Vulnerability Management with Nessus in AWS
date: 2022-01-20 12:00:00 -0400
image: 
  path: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/Vulnerability%20Management%20with%20Nessus%20in%20AWS%2090286706ffaf46128c3727fb6f6c7e58/banner.jpg
  height: 1100
  width: 500
categories: [AWS, Blogging]
tags: [security, lab, cloud, aws]
---


## Introduction

In this tutorial we will cover vulnerability scanning and vulnerability remediation. These are two of the main steps in the Vulnerability Management Lifecycle. We will be using Nessus Essentials to scan local VMs hosted on VMWare Workstation in order run credentialed scans to discover vulnerabilities, remediate some of the vulnerabilities.

## EC2 Instance Setup

first step is launch an EC2 instance, the recommended requirements are:

- windows OS

![Untitled](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/Vulnerability%20Management%20with%20Nessus%20in%20AWS%2090286706ffaf46128c3727fb6f6c7e58/Untitled.jpg)

- basic: t3 medium
- recommended: t3 xlarge

Decrypt your password to login in a RDP session and use this to access your EC2 instance

![Untitled](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/Vulnerability%20Management%20with%20Nessus%20in%20AWS%2090286706ffaf46128c3727fb6f6c7e58/Untitled%201.jpg)

## Installing Nessus Essentials

Then we install nessus in the windows EC2 instance, we will select 10.0.2_x64 version, use your code activation that they sent you in your account registration.

![Untitled](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/Vulnerability%20Management%20with%20Nessus%20in%20AWS%2090286706ffaf46128c3727fb6f6c7e58/Untitled%202.jpg)

When we finished installing Nessus this image will apper

![Untitled](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/Vulnerability%20Management%20with%20Nessus%20in%20AWS%2090286706ffaf46128c3727fb6f6c7e58/Untitled%203.jpg)

## Setup Inbound Rules to our EC2 instance

After launch the ec2 and download Nessus we have to add inbound rules to our machine in order to perform a credential scanning

We are going to use this rules:

- https 443       ec2 ip/32
- dns (tcp 53)   ec2 ip/32
- custom TCP 8834 ec2 ip/32
- ssh(22)  e2 ip/32
- custom TCP 139 ec2ip/32
- SMB (445) ec2ip/32
- custom TCP 8835 ec2ip/32
- all ICMP -IPv4 ec2ip/32
- custom TCP 49152-65535 0.0.0.0/0
- https(443) 0.0.0.0/0
- rdp (3389) 0.0.0.0/0

Then we have to give a name to our credential scan an the ip of the EC2 instance

## Credential Vulnerability Scan

![Untitled](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/Vulnerability%20Management%20with%20Nessus%20in%20AWS%2090286706ffaf46128c3727fb6f6c7e58/Untitled%204.jpg)

Then select run in the dashboard of scans and wait to complete the scan

![Untitled](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/Vulnerability%20Management%20with%20Nessus%20in%20AWS%2090286706ffaf46128c3727fb6f6c7e58/Untitled%205.jpg)

## Inspecting First Scan

At the end we will be able to see the vulnerabilities that the host windows has, in this case it has the samba port open without authentication by password, when we click on the vulnerability it shows us more details of this one.

![Untitled](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/Vulnerability%20Management%20with%20Nessus%20in%20AWS%2090286706ffaf46128c3727fb6f6c7e58/Untitled%206.jpg)

## Remediation

if we visit the remediation tab it will show us the tasks we need to do to fix this vulnerability, in our case we will see that we must update Firefox, because we have a version that is very old and it contains vulnerabilities, we will also see about protecting the samba service with a password or close the port when you are not using this service that helps us to share files on the local network.

After updating Firefox or uninstalling this vulnerability won´t appear in next scans

## Note

If you don´t want to spend money on this lab, you also can install vmware and create a windows virtual machine, you will only need the ip of this machine to perform the scan in Nessus, is more faster and cheap, but you have to provide your own hardware.

Thanks for reading, happy learning!