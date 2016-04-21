---
layout: post
title: VirtualBox Manage
date: 2014-01-24 14:07
categories: jekyll update
---

**VBoxManage** is the command line way to manage your VirtualBox hosts.

	VBoxManage showvminfo
	VBoxManage list vms
	VBoxManage list runningvms
	
	VBoxManage startvm $vm --type=headless
	VBoxManage startvm $vm --type=gui
		
	VBoxManage controlvm $vm savestate
	VBoxManage controlvm $vm poweroff

	VBoxManage modifyvm $vm --accelerate3d=off
	VBoxManage modifyvm backend --memory 256

#### Mount Shared folder

	mount -t vboxsf -o uid=1000,gid=1000 Shared Shared/

#### Shrink Virtualbox VM

If your virtual disk looks full but not really as much, the you can shrink Virtualbox VM disk with the following:

	sudo apt-get zerofree

reboot and start in **Init 1** mode

	mount -n -o remount,ro -t ext3 /dev/sda1 /
	zerofree /dev/sda1
	shutdown -h now
	
Once done, go back to the master host and

	VboxManage modifyvdi /path/to/your/VM.vdi compact

#### Clone VirtualMachine

	VBoxManage clonevm --name newvm --register --basefolder VirtualBox\ VMs/oldvm
