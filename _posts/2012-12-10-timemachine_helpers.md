---
layout: post
title: timemachine_helpers
date: 2012-12-10 11:35
categories: default update
---

![TimeMachine icon](https://dl.dropbox.com/u/4473739/seriousman.org/time-machine.png)

 Imagine You use Apple OSX, imagine you like to play in ***Terminal***, imagine you're trying to fix some issue with a certain **app** in your **~/Library** folder and you make a mistake, doing what's not supposed to do:

	rm -rf ~/Library/   
	
This can cause big damage in your user account, even if it's only for a few seconds, but it will probably be to late.[^1]

[^1]: Reminder: don't touch things you shouldn't touch.

This is just to introduce some tips and helps I've collected regarding Apple's ***TimeMachine*** backup software.

Ok, lets start, first of all give thanks and link to all articles I've been looking at and saved in my del.icio.us are [here](http://delicious.com/netman2/timemachine "Time Machine links"). 

### Linux AFP server

If you have your own Linux serving AFP, edit your **/etc/netatalk/AppleVolumes.default** and add this options to your TimeMachine directory:

	~/.TimeMachine "$u Backup" allow:YOUR_USERNAME cnidscheme:dbd options:usedots,upriv,tm 

### Enable Time Machine Network Volumes

If your network storage is not detected as a TimeMachine destination, type this in your terminal.

	defaults write com.apple.systempreferences MShowUnsupportedNetworkVolumes 1

### Starting new TimeMachine backup?

Mount your Network Shared Folder where you want time machine to be stored.
In your localhost, open a terminal and create an image disk as follows:

	hdiutil create -size 120g -fs HFS+J -volname "Backup of sheldon" sheldon_00111233a2da.sparsebundle

The image must be in format ***hostname_macaddress.sparsebundle***; where ***00111233a2da*** is the mac address from you computer default network adapter, used by TimeMachine as and *ID*.
Find it through your terminal by typing 

	ifconfig en0 |grep ether | sed 's/\://g'

Copy the image in the shared folder as this way

	rsync -avE sheldon_00111233a2da.sparsebundle /Volumes/TimeMachineNetworkBackupDestination/.

Disconnect from network volume and connect again, then you can go back to TimeMachine and configure your backup destination as your network volume should appear. If not, and you're in Lion or MLion, you can use ***tmutil*** to configure from terminal.

	sudo tmutil setdestination -p /Volumes/TimeMachineNetworkBackupDestination
or
	
	sudo tmutil setdestination afp://user:pass@host/share

### Change TimeMachine backup time Intervals

Last number is the seconds you want between each backup *(14400 = 4h)*.

	sudo defaults write /System/Library/LaunchDaemons/com.apple.backupd-auto StartInterval -int 14400
	

### Problems removing TimeMachine Backups

Trying to remove *Backups.backupdb* and you receive  "operation not permitted" errors?

Use the following through command line for a quick and clean remove in Lion.

	sudo /System/Library/Extensions/TMSafetyNet.kext/Contents/MacOS/bypass rm -rfv /Volumes/[disk]/Backups.backupdb/[path]

In Mountain Lion **bypass** has changed in place so you must type:

	sudo /System/Library/Extensions/TMSafetyNet.kext/Helpers/bypass rm -rfv /Volumes/[disk]/Backups.backupdb/[path]
	
… And last one is 

### Fix Time Machine Sparsebundle NAS Based Backup Errors

Please follow this [link](http://www.garth.org/archives/2011,08,27,169,fix-time-machine-sparsebundle-nas-based-backup-errors.html "Fix Time Machine Sparsebundle NAS Based Backup Errors") for the whole process as I'm only describing the quicks.

 Open terminal and become **root**
	
		sudo su -
 Dive to directory where sparsebundle is stored and type this and wait:

		chflags -R nouchg sheldon_00111233a2da.sparsebundle
 Once is done, type the next:
		
		hdiutil attach -nomount -noverify -noautofsck sheldon_00111233a2da.sparsebundle
	
You will then see something like:

		/dev/diskx Apple_partition_scheme
		/dev/diskxs1 Apple_partition_map
		/dev/diskxs2 Apple_HFSX

Check if fsck is working with:
	
		tail -f /var/log/fsck_hfs.log
Then, run ***fsck*** manually in the partition holding the data which should be *Apple_HFSX* or *Apple_HFS*.
		
		fsck_hfs -drfy /dev/diskxs2
	
This will take a while and at the end, you'd either see:
		
		`The Volume was repaired successfully´
or

		`The Volume could not be repaired´

At the end, if all went well, just detach de sparsebundle 

		hdiutil detach /dev/diskxs2
… and give TimeMachine a new chance.
		
		tmutil startbackup
		
