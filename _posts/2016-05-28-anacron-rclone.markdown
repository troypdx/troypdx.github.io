---
layout: post
title:  "File Synchronization with Linux Anacron and Google Drive"
date:   2016-05-28 22:45:36 -0700
categories: linux cron anacron laptop google drive
---

Here's an approach to file synchronization between Linux on one or more laptops and Google Drive. This serves me well to keep my [Keepass][keepass-url] password file synchronized across the various laptops I use. The samples below show my setup with Debian-based Linux Mint.

<h3> Step 1: Install rclone - rsync for cloud storage </h3>
Download and install rclone and then follow the instructions [for Google Drive][rclone-drive].  

<h3> Step 2: Create bash script to synchronize the local and Google Drive files </h3>
This script determines which version of the Keepass database file is most recent between the local copy and what exists at Google Drive. The script logic copies the password file from Google Drive to a local temporary directory. If the temporary file timestamp is more recent than the local version at the home directory then the local version is replaced. Otherwise the local version replaces the Google Drive's version at remote:Documents.        

{% highlight bash %}
#!/bin/bash

echo "synclocal: Get copy of .kdbx from Google Drive"
rclone --config=/home/snoopy/.rclone.conf --log-file=/home/snoopy/rclone-copy.log --include /accounts_snoopy.kdbx copy remote:Documents ~/Documents/rclonetemp

echo "synclocal: Compare file time stamps then rclone if necessary"
if [ ~/Documents/rclonetemp/accounts_snoopy.kdbx -nt ~/Documents/accounts_snoopy.kdbx ];
then
  echo "synclocal: Replacing local drive's .kdbx with Google Drive's version"
  mv ~/Documents/rclonetemp/accounts_snoopy.kdbx ~/Documents/accounts_snoopy.kdbx 
else
  echo "synclocal: Sync Google Drive's .kdbx with local version"
  rclone --config=/home/snoopy/.rclone.conf --log-file=/home/snoopy/rclone-sync.log --include /accounts_snoopy.kdbx sync ~/Documents remote:Documents   
fi

{% endhighlight %}

<h3> Step 3: Update /etc/anacrontab </h3>
The sample anacrontab below runs the synclocal bash script. Field 1 is recurrence period where 1 indicates daily and Field 2 is the number of minutes delay before the script is executed. This works well for me but if I update the Keepass file on multiple laptops during the same day there's a risk of losing an update because the synchronization is happening less often than the file update. 

{% highlight bash %}
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
HOME=/home/snoopy
LOGNAME=snoopy

# These replace cron's entries
1	5	cron.daily	run-parts --report /etc/cron.daily
7	10	cron.weekly	run-parts --report /etc/cron.weekly
@monthly	15	cron.monthly	run-parts --report /etc/cron.monthly

1	5	cron.daily	/bin/sh ~/Scripts/synclocal

{% endhighlight %}
 
Note how the anacrontab file declares environment variables for HOME which allowed the synclocal bash script to use ~/ notation for home which cut down on how verbose the command parameters are. This worked well except for location of the .rclone.conf file. I found that with rclone v1.29 I needed to explicitly provide the --config parameter with an explicit path in order for rclone to locate the .rclone.conf file.

To troubleshoot the output of anacron see /var/log/syslog. 

[rclone-drive]: http://rclone.org/drive/
[keepass-url]: http://keepass.info/
