---
layout: post
title: Some Linux Tips
date: 2012-10-09 13:23
categories: jekyll update
---

Update: 2014-02-06 17:46

#### Find hard link files

	$ find /usr/local/ -type f -a \! -links 1

	$ find <SOURCEDIR> -type f -links +1 -printf "\n\n %n HardLinks of file : %H/%f \n" -exec find<TARGETDIR> -type f -samefile {} \;

#### SED adding ' ' at beginning and end

	$ cat example.txt
	
	2348572389752389
	
	8234759823745899
	
	2389458923458923

Let's modify

	$ cat example.txt |sed "s/^/'/g" |sed "s/$/',/g"
	
	'2348572389752389',
	
	'8234759823745899',
	
	'2389458923458923',


#### Network checks

	$ netstat -lntp | grep 8080

	$ iftop

Check port connectivity

	$ echo "" > /dev/tcp/10.0.2.135/1521

#### Quick RESIZE bulk images and rename.

	$ for i in *; do convert $i -resize 60% $i.out;done
	$ for i in *; do k=`echo $i |cut -b 1-12`; r=$k; mv $i $r;done

#### Debian Locale unset

	perl: warning: Setting locale failed.
	perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	
	$ dpkg-reconfigure locales -p critical
	$ export LC_ALL="C"
	$ dpkg-reconfigure locales
	$ locale-gen

#### How-To flush the DHCP server lease cache

Delete the temporary file dhcpd.leases~:

    $ sudo rm dhcpd.leases~

Flush the lease cache dhcpd.leases:

    $ sudo echo "" > dhcpd.leases

#### Convert All Text In a File From UPPER to lowercase:
Type the following command at shell prompt:

	$ tr '[:upper:]' '[:lower:]' < input.txt > output.txt
	$ cat output.txt

also you can use see

	$ sed -i -e 's/\(.*\)/\L\1/' input.txt > output.txt 

#### Batch rename

	$ for i in *; do k=`echo $i|cut -b 19-`; r=$k; mv $i $r;done

#### Export Xserver 

On server side:
X11 installed and ssh X11 forwarding enabled

On client side:

	ii xserver-xephyr 2:1.7.7-14 nested X server

	$ Xephyr -ac -screen 1024x768 -br -reset -terminate 2> /dev/null :1 & 
	$ ssh -X mooo blackbox

#### Find options and remove

	$ find -depth -type d -empty -exec rmdir {} \;
##### Secure but may be slow due to -exec
	$ find /path/to/dir -type d -empty -print0 -exec rmdir -v "{}" \;

The syntax is as follows to delete all empty files:
secure and fast version

	$ find /path/to/dir/ -type f -empty -print0 | xargs -0 -I {} /bin/rm "{}"

or secure but may be slow due to -exec

	$ find . -type f -empty -print0 -exec rm -v "{}" \;

###### Find multiple files at once

	$find ./ -name "*.jpg" -o -name "*.css" -ls  

#### SSH ninja

###### SCP Tunnel with one hop

First, open the tunnel

	$ ssh -L 1234:remote2:22 -p 45678 user1@remote1

Then, use the tunnel to copy the file directly from remote2

	$ scp -P 1234 user_remote2@localhost:file .


Lets connect to our remote host through a middle host which can get out to the internet.

	$ ssh -L -N 127.0.0.1:7777:remote_host:22 middle_host
Now we've a ssh connection on localhost:7777 to our remote host in his port 22. 	
We can now ssh to the remote host by:

	$ ssh -P 7777 remoteuser@localhost
	
If we want to navigate from remote hosts network with our web browser, we can use the ssh to create the tunnel:

		$ ssh -D 8081 -fN -p 7777 localhost

Then, on your Web Browser, set network settings to point 

	proxy_socks forward localhost:8081

#### Batch remote processing

	for machine in `cat liveboxes.txt`; do ssh $machine "uname -n;rpm -qa|grep python";done | tee output_pythonLive.txt

	while read machine; do ssh $machine "uname -n;rpm -qa|grep python"; done < test | tee output2.txt

	root@host:~# for machine in stage{portal,batch}{1,2}; do ssh ur@$machine curl ifconfig.me; done

