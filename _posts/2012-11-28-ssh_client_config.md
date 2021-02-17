---
layout: post
title: ssh_client_config
date: 2012-11-28 11:14
categories: linux updated tips
---

Managing to get into several different severs, has provoked my **.ssh/config** file to be modified.

First of all, we want to keep alive our ssh connections with the following option:
	
	ServerAliveInterval	60
	ForwardAgent yes

This are global settings. Now we're going to configure some specific hosts with it's own settings.

	Host home
		Hostname server.home.net
		Username user
		IdentityFIle ~/.ssh/id_rsa
		Port 2200
 
**Host** is the short name you'll use to connect.
	
	ssh home

**Hostname** is the real name for the host, can be also an IP address.
**Username** is the one used to connect to server, it can be different from your local user.
**IdentityFile** is the private_key used in trust with server if configured.
