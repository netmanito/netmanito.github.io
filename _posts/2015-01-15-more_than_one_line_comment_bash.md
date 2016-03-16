---
layout: post
title: More Linux Tips
date: 2015-01-14 17:03
categories: jekyll update
---

####More than one line comment bash


	#!/bin/bash
	echo "Say Something"
	<<COMMENT1
    your comment 1
    comment 2
    blah
	COMMENT1
	echo "Do something else"



####Check tcp open ports 
While doing this, you're checking if the port 1521 is open at the ip address, very useful when you don't have `nc` netcat, mmap or other stuff.

	[root@host]# echo "" > /dev/tcp/172.10.0.15/1521


####PFX Certificates

PFX files are PKCS#12 Personal Information Exchange Syntax Standard files. They can include arbitrary number of private keys with accompanying X.509 certificates (public keys) and a Certificate Authority Chain.

If you want to extract client certificates (not the CA certificates), you can use OpenSSL's PKCS12 tool.

	openssl pkcs12 -in xxxx.pfx -out mycertificates.crt -nokeys -clcerts

The command above will output the certificate(s) in PEM format. The ".crt" extension known to both Mac OS X and Windows operating systems and will be usable. You mention ".cer" extension your question which is the DER format equivalent. Same certificate but different encoding. Try the ".crt" file first and if it doesn't help, it's easy to convert from PEM to DER format.

	openssl x509 -inform pem -in mycertificates.crt -outform der -out mycertificates.cer


####Network routing


You can add static route using following command:
ip route add {NETWORK} via {IP} dev {DEVICE}
For example network 192.168.100.0/24 available via 192.168.1.254:
     
     # ip route add 192.168.100.0/24 via 192.168.1.254 dev eth1

Alternatively, you can use old good route command:

     # route add -net 192.168.100.0 netmask 255.255.255.0 gw 192.168.1.254 dev eth1



####Grep the nth line of a file?

Good old-fashioned awk is incredibly useful for these small jobs:

	awk '{ if (NR==45) print $0 }' file

Or, you can pass in n as a parameter if you want to make a script:

	#!/bin/bash
	# Usage getn N filename

	awk '{ if(NR==n) print $0 }' n=$1 $2

**$0 is the whole line. $1 would be the first column, $2 the 2nd etc.**
