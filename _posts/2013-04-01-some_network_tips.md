---
layout: post
title: Some Network Tips
date: 2013-04-01 12:30
categories: update
---

#### Iptables
Do you have a firewall or some computer with any kind of exposure to the Internet? Then you might want to use iptables for **filtering/blocking/forwarding** your connections. 
Lets see some examples.

#### Block an IP address

	iptables -I INPUT -i eth0 -s IP_address -j REJECT


#### Basic Init.d script for firewalling your network.

	#!/bin/sh
	case $1 in
	start)
	echo "miscelanea"
	echo "Configuring Router"
	echo 1 > /proc/sys/net/ipv4/ip_forward
	modprobe ip_conntrack_ftp
	modprobe ip_nat_ftp
	modprobe ip_conntrack_irc
	
	## put here your iptables rules
	iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
	
	## end rules
	 ;;
	 stop)
	 echo "Removing iptables rules"
	 iptables -t nat -F
	 ;;
	 esac	 

#### Port redirection

	 iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8082 -j DNAT --to 12.0.0.1:8080

#### Firehol, the definitive firewall 

Creating firewall rules can get you MAD, one of the easier way to create rules for your FW based on iptables is using [firehol](http://firehol.sourceforge.net/ "firehol homepage"). It has a very easy syntax to create rules as shown below.

root@raspi ~ # cat /etc/firehol/firehol.conf
	
	version 5
	
	# my own rule named clan forwarding port 5223
	server_clan_ports="tcp/5223"
	client_clan_ports="default"

	#dnat 10.0.0.2 proto tcp dst 192.168.56.2 dport 80

	interface eth0 internet
    	client "dns http https ssh clan" accept
		server "ssh clan" accept

	interface wlan0 wifi
        server "dhcp ssh dns" accept
        client "dns http https ssh clan" accept
	
	#interface eth2 wifi
	#       server "ssh" accept
	#       client "dns http https ssh" accept

	router wifi2internet inface wlan0 outface eth0		masquerade
    	route "dns http https clan" accept
	
	#router lan2internet inface eth2 outface eth0
	#       masquerade
	#       route "dns http https" accept

	#router internet2dmz inface eth0 outface wlan0
	#       route "http" accept

