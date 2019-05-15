---
layout: post
title: After a fresh minimal Debian install
date: 2015-02-05 17:32
categories: default update
---

I do this many times when I play with VM's (virtual machines); so I've to leave written all I normally do after a **Fresh minimal Debian Install** 

First of all, I've not set the *apt* repositories yet for a quicker install. 

	$ echo "deb http://ftp.de.debian.org/debian stable main contrib non-free" >> /etc/apt/sources.list
	$ echo "deb-src http://ftp.de.debian.org/debian stable main contrib non-free" >> /etc/apt/sources.list

Once you've added the url for repositories you can start to install all you need.

```
$ apt-get update && apt-get -y upgrade
$ dpkg-reconfigure tzdata
$ export LC_ALL="C"
$ dpkg-reconfigure locales -p critical
$ locale-gen
```

For basics:

```	
$ apt-get update
$ apt-get install -y vim vim-scripts mc screen less links curl sudo rsync openssh-server
$ update-alternatives --all (this changes your default editor and more)
```

Check whether there's an enabled group in sudoers

	$ visudo 
	
Check for the following line, normally enable in Debian.

	## Allows people in group wheel to run all commands
	%sudo  ALL=(ALL) NOPASSWD:ALL
	
Add your user to this group

	$ adduser 'myuser' sudo

Check login with your new user through **ssh** and *sudo* something.

If so, you're done!! 

Once here, you've to decide if it's a server or a desktop machine, so you'll have to choice what you want to install.

Remember to install virtual-machine guest tools!
