---
layout: post
title: Lots of Linux Tips
date: 2016-10-28
categories: default linux
---

Let's store more tips I've been collecting during last times.

### Export http proxy linux shell

	export http_proxy=http://10.0.0.1:61010 
	export https_proxy=http://10.0.0.1:61010 

### Apt-Cache madison

It displays available versions of a package in a tabular format.

Try with

    apt-cache madison myPackage
 
### Use of Grep

Try the following:

	grep -v -e '^$' foo.txt

The -e option allows regex patterns for matching.

The single quotes around ^$ makes it work for Cshell. Other shells will be happy with either single or double quotes.

UPDATE: This works for me for a file with blank lines or "all white space" (such as windows lines with "\r\n" style line endings), whereas the above only removes files with blank lines and unix style line endings:

	grep -e -v '^[[:space:]]*$' foo.txt 
	
#### Exlcude # and empty lines

	egrep -v "^#|^$" foo.txt

## Git tips 

#### bash completion

There are other ways to enable git-bash-completion depending on your OS.

For linux you can just do the following:

First get the git-completion.bash script (view it here) and put it in your home directory:

	curl https://raw.githubusercontent.com/git/git/master/contrib/completion/git-completion.bash -o ~/.git-completion.bash

Next, add the following lines to your .bash_profile. This tells bash to execute the git autocomplete script if it exists.

	if [ -f ~/.git-completion.bash ]; then
	. ~/.git-completion.bash
	fi

End with

	source ~/.bash_profile

#### add file from one branch to other

	git checkout master               # first get back to master
	git checkout experiment -- app.js # then copy the version of app.js 
                                      # from branch "experiment"
---

### Use of SED

If you want to delete lines 5 through 10 and 12:

	sed -e '5,10d;12d' file
	
This will print the results to the screen. If you want to save the results to the same file:

	sed -i.bak -e '5,10d;12d' file
	
This will back the file up to file.bak, and delete the given lines.

To delete line 5, do:

	sed -i -e '5d' file.txt

For a variable line number:

	sed -i -e "${line}d" file.txt

If the -i option isn't available in your flavor of sed, you can emulate it with a temp file:

	sed "${line}d" file.txt > file.tmp && mv file.tmp file.txt

### PV shows progress on dd, tar and others

Decompressing with TAR

	tar -czf - ./Downloads/ | (pv -p --timer --rate --bytes > backup.tgz)

Using DD to write image on disk

	pv -tpreb debian-live-8.5.0-amd64-standard.iso | sudo dd of=/dev/disk1 bs=1m
	
	dd if=/dev/urandom | pv | dd of=/dev/null

### mdadm raid

First examine your drives to see their *uuids* with

	mdadm --examine --scan

Enable or assemble the raid volume

	mdadm --assemble --uuid <uuid> /dev/md0

Mount your drive

	mount /dev/md0 /my/destination/folder

### Iptables

Blocking access to a certaing port

	iptables -A OUTPUT -p tcp -d 192.168.1.2 --dport 5050 -j DROP

How to forward correctly a port, I finally found how-to. First, I had to add -i eth1 to my "outside" rule (eth1 is my WAN connection). I also needed to add two others rules. Here in the end what I came with :

	iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 8080 -j DNAT --to 10.32.25.2:80
	iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to 10.32.25.2:80
	iptables -t nat -A POSTROUTING -p tcp -d 10.32.25.2 --dport 80 -j MASQUERADE
	

### Echo just for fun

	echo "I'm a Unix Creationist, the world was created on $(TZ=GMT date -d @0) and it will end on $(TZ=GMT date -d @$((2**31-1)))"
	
## CENTOS tips

#### change timezone centos

	# cp /usr/share/zoneinfo/GB /etc/localtime

#### ifconfig

	yum install net-tools

#### static network address

	vi /etc/sysconfig/network-scripts/ifcfg-eth0

	I will change  the file like this:

	#My IP description
	# IPv-4

	DEVICE="eth0"
	NM_CONTROLLED="yes"
	ONBOOT=yes
	HWADDR=20:89:84:c8:12:8a
	TYPE=Ethernet
	BOOTPROTO=static
	NAME="System eth0"
	UUID=5fb06bd0-0bb0-7ffb-45f1-d6edd65f3e03


#### find files from packages

	yum whatprovides ifconfig
	
---
	
### Send data through netcat

On your server (A):

	nc -l -p 1234 -q 1 > something.zip < /dev/null

On your "sender client" (B):

	cat something.zip | netcat server.ip.here 1234

### Smbcredentials

Using a text editor, create a file for your remote servers logon credential:

	gedit ~/.smbcredentials

Enter your Windows username and password in the file:

	username=msusername
	password=mspassword

Save the file, exit the editor.

Change the permissions of the file to prevent unwanted access to your credentials:

	chmod 600 ~/.smbcredentials

Mount with the following options in /etc/fstab

	//your_server/Your_Share /mnt/samba/Your_Mount cifs credentials=/home/your_home_directory/.smbcredentials,uid=yourlinuxaccount,gid=users 0 0

Or through command line with

	mount -t cifs //your_server/Your_Share -o credentials=/home/your_home_directory/.smbcredentials


### disable ipv6

To disable ipv6, you have to open /etc/sysctl.conf using any text editor and insert the following lines at the end:

	net.ipv6.conf.all.disable_ipv6 = 1
	net.ipv6.conf.default.disable_ipv6 = 1
	net.ipv6.conf.lo.disable_ipv6 = 1

If ipv6 is still not disabled, then the problem is that sysctl.conf is still not activated.

To solve this, open a terminal(Ctrl+Alt+T) and type the command,

	sudo sysctl -p

You will see this in the terminal:

	net.ipv6.conf.all.disable_ipv6 = 1
	net.ipv6.conf.default.disable_ipv6 = 1
	net.ipv6.conf.lo.disable_ipv6 = 1

After that, if you run:

	$ cat /proc/sys/net/ipv6/conf/all/disable_ipv6

It will report:

	1

If you see 1, ipv6 has been successfully disabled.

### Vlan debian

	$ sudo apt-get install vlan 
	$ sudo modprobe 8021q

##### vconfig manages vlan

	# vconfig rem eth1.10
	Removed VLAN -:eth1.10:-
	# ip a l eth1.10
	Device "eth1.10" does not exist.
	
### Forward to a Gmail mail server

To configure SSMTP, you will have to edit its configuration file (/etc/ssmtp/ssmtp.conf) and enter your account settings:

	/etc/ssmtp/ssmtp.conf


	# The user that gets all the mails (UID < 1000, usually the admin)
	root=username@gmail.com

	# The mail server (where the mail is sent to), both port 465 or 587 should be acceptable
	# See also https://support.google.com/mail/answer/78799
	mailhub=smtp.gmail.com:587

	# The address where the mail appears to come from for user authentication.
	rewriteDomain=gmail.com
	
	# The full hostname
	hostname=localhost
	
	# Use SSL/TLS before starting negotiation
	UseTLS=Yes
	UseSTARTTLS=Yes
	
	# Username/Password
	AuthUser=username
	AuthPass=password
	
	# Email 'From header's can override the default domain?
	FromLineOverride=yes
	
Note: Take note, that the shown configuration is an example for Gmail, You may have to use other settings. If it is not working as expected read the man page man 8 ssmtp, please.

Create aliases for local usernames (optional)

	/etc/ssmtp/revaliases

	root:username@gmail.com:smtp.gmail.com:587
	mainuser:username@gmail.com:smtp.gmail.com:587

To test whether the Gmail server will properly forward your email:

	$ echo test | mail -v -s "testing ssmtp setup" tousername@somedomain.com

### nmap snmp

You can use Nmap's snmp-brute something like

	nmap -sU -p161 --script snmp-brute --script-args snmplist=community.lst 192.168.1.0/24

### check_mk

	root@linux# cmk -II df xyzsrv01

This first removes all checks of type df on host xyzsrv01 and then does inventory.

Example 2:

	root@linux# cmk -II xyzsrv01

This removes all agent based of host xyzsrv01 before doing inventory.

## ras-pi best practices

#### raspi-config

* expand filesystem
* change timezone, keyboard, hostname

First make sure (if you haven’t already) that your raspian is up to date.
type:
	
	sudo apt-get update
	sudo apt-get upgrade

It is also a good idea to make sure your firmware is up to date

	sudo apt-get install rpi-update

and run it 
	
	sudo rpi-update

a reboot is required when this is finished

	sudo shutdown -r now

(you will have to reconnect via ssh when the Rpi has restarted if you are using a remote terminal)

#### rpi-cam

To enable video0 on raspi

	sudo modprobe bcm2835-v4l2 


