---
layout: post
title: MacPorts Maintenance
date: 2013-10-28 10:36
categories: macos osx updated
---

To maintain you macports repository clean and without unused stuff, proceed with:

	port clean --all all
	sudo port -f uninstall inactive	
	
If you can't wait for the clean one to run in the background, there are a few commands you can run manually and faster.

Remove leftover build files (this is done automatically by default):

	sudo rm -rf /opt/local/var/macports/build/*

Remove download files:

	sudo rm -rf /opt/local/var/macports/distfiles/*
	
Remove archives (these aren't created by default):

	sudo rm -rf /opt/local/var/macports/packages/*
