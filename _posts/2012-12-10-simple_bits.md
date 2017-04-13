---
layout: post
title: simple_bits
date: 2012-12-10 11:42
categories: default update
---

This are little **bits** I frequently use and sometimes forgive.

### Enable Spotlight on encfs volumes

If you're using encfs or fuse volumes on OSX, depending on how you mount the volume, Spotlight might have problems to index on it.

Use the options **-oallow_other** and **-olocal** so that spotlight correctly indexes the mounts in Max OSX Lion and Mountain Lion.

	encfs ~/encrypted_dir ~/mount_point_dir -oallow_other -olocal

### Search in gmail:
This is how I sometimes search at gmail with multiple variables, assuming you use ***labels/folders*** at gmail.

This will search in current folder with before and after specific date, concrete email sender and subject containing the word *Alert*. 

	before:2012/10/22 after:2012/09/30 from:user@example.com subject:ALERT
	
Ok, lets do it a bit more powerful, include search at anywhere in your gmail account, this means any folder. **anywhere** can be substituted by any of your mail folders.

	in:anywhere before:2012/10/01 after:2012/11/01 from:user@example.com subject:ALERT

And now we want the ones that are unread.

	in:anywhere before:2012/10/01 after:2012/11/01 from:user@example.com subject:ALERT label:unread
	
Find all unread mails that are not ***starred***.

	label:unread NOT label:starred
	
### Textedit

Change the default iCloud save preference from Textedit in OSX

	defaults write NSGlobalDomain NSDocumentSaveNewDocumentsToCloud -bool false

### Locate database in OSX

Been a *NIX user you might sometimes use ***locate*** when you want to find files in a system.
OSX has also this option but has to be enabled.


	locate Curls

	WARNING: The locate database (/var/db/locate.database) does not exist.

To create the database used by *locate*, run the following command:

	sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.locate.plist

This will take a while until database is created.

### Adding network routes in OSX

**route** version in OSX is a bit old, and does not work same as LINUX.
Adding a new route in your Mac is as follows:

	route -n add -net 12.0.0.0 192.168.1.200 255.0.0.0 
This adds network access to 12.0.0.0 network through gateway 192.168.1.200 with 255.0.0.0 netmask.

Now we just want to route to a certain IP address.

	route add -host 12.11.10.9 192.168.1.200 255.255.240.0

This IP is not accessible through the default network gateway, so we add this ***route*** over another gateway to access it.

### Show Hiden Files in Finder

In the Terminal type:

	defaults write com.apple.finder AppleShowAllFiles TRUE/FALSE 

depending if you want to enable or disable.
After that, remember to reload Finder

	killall Finder



From the Terminal, type 

	ps aux | grep username 

In the results, look for the process called **loginwindow** associated with the affected username.

Note the process ID (the first column of the output) of *loginwindow* -- for purposes of this example, assume it was 1234. To kill the logged-in user, just type:

	 % sudo kill -9 1234
The user will immediately disappear from the list of logged-in users. It's not elegant, but it works.
