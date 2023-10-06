---
layout: post
title: ""
aliases: 
date: 2023-10-06
tags: 
draft: false
---
I'm using Ubuntu for my everyday work. Recently, I have some tasks relating SELinux. Unfortunately, SELinux is by default not enabled on Ubuntu, and the ecosystem is not as full as it in Fedora.

In this article, I wrote down the steps to setup an Fedora VM via [Virtual Maching Manager](https://virt-manager.org/), and using `ssh` to login Fedora.

<!--more-->

## Install Virtual Machine Manager

```shell
sudo apt-get install virt-manager
```

## Download the Fedora Workstation ISO

I'm using Fedora 38. Download from the [official site](https://fedoraproject.org/workstation/download/) or simple using the command below.

```shell
wget https://download.fedoraproject.org/pub/fedora/linux/releases/38/Workstation/x86_64/iso/Fedora-Workstation-Live-x86_64-38-1.6.iso
```

## Create Fedora VM and Install

Click "Create a new virtual machin".

![](https://i.imgur.com/cMUgqOr.png)

Click "Forward".

![](https://i.imgur.com/hXZssOT.png)

Follow the steps below to select the Fedora ISO we just downloaded.

![](https://i.imgur.com/n91oauG.png)

Set proper memory and CPU and disk size.

![](https://i.imgur.com/AmmHwdF.png)


![](https://i.imgur.com/EtfWp6A.png)

Set the name of the VM, click "Finish".

![](https://imgur.com/CK39ncF.png)

Now the boot process starts. We firstly boot from the ISO and then install to the disk.

![](https://i.imgur.com/nnbYVDp.png)

Install Fedora.
![](https://i.imgur.com/JHsPzgD.png)
The installation destination is not set. Click "Installation Destination" to set.

![](https://i.imgur.com/PdKYWhF.png)

Double click the only disk in "Local Standard Disks", which is created for us in the VM creation steps, then click "Done". Then click "Begin Installation".

After installation is done and reboot, we can boot from the disk.

![](https://i.imgur.com/ccdttga.png)

After reboot and login, open "Terminal" app. Then go to the next section.

## Setup ssh login

Since `openssh-server` is already installed by default, we could simply enable it.

```shell
# Guest terminal
systemctl enable sshd
systemctl start sshd
```

Copy public key from host.

```shell
# Host terminal
cat ~/.ssh/id_rsa.pub
```

Paste the output to the guest machine.

```shell
# Guest terminal
mkdir -p ~/.ssh

# Paste the output to ~/.ssh/authorized_keys
```

Obtain guest's IP address

```shell
# Guest terminal
ifconfig | grep 192
> 192.168.123.25
```

Login guest from host.

```shell
ssh <username>@192.168.123.25
```
