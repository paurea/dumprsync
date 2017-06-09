#simple dump made with rsync

The backup system consists of three scripts (under ./scripts) and some configuration files
(under ./systemd).
The main script is scripts/dumpme. Dumpme takes a snapshot of the
root of the origin system (MAIN) and creates a directory under the dump filesystem
(DUMP) in a directory (and all the tree if needed) of the form

	/dump/yyyy/dd/mm/hhmm/rootname

for example:

	/dump/2017/0523/1031/NEWAGE

This directory is synchronized using rsync from MAIN but using the current dump to hard link
directories without changes so unchanged directories occupy very little space.

The script plays well with systemd logging the start and the end of the dump.
In the root of the dump filesystem a soft link called current will be kept pointing to the current dump
without the root name.

This script is run periodically (normally an hour, but it can be less often, and at most once per minute)
from systemd by the user (not root). The files dump.service and dump.timer
To do this, two files are provided, ./systemd/dump.service and ./systemd/dump.timer
which configure the unit and the timer for systemd respectively.

There is very little redundancy in the dump, so if you are a little bit paranoid (as you should be
with backups) you would like that less often (for example, once a month) you make a completely
clean copy of the directory instead of using hard links. This effectively duplicates the space of the last
dump copy. We call this a check point. and there is also a soft link in the dump pointing to it without
the root name. The script which creates the checkpoint is scripts/chkptme and the systemd configuration
units and timer are ./systemd/chkpt.service and ./systemd/chkpt.timer.

To configurate a dump, we need a root for main and a root for the dump. I normally use /newage and
/dump. In the fileserver I bind my directories to that places by using the fstab entries (paurea is my user name):

	/media/paurea/my_main_disk /newage none uid=paurea,gid=paurea,rw,nosuid,nodev,relatime,user,allow_other,bind 0 0
	/media/paurea/dump	/dump	none	uid=paurea,gid=paurea,rw,nosuid,nodev,relatime,user,allow_other,bind 0 0

Then I make manually the first dump

	mkdir /dump/first
	cp -a /newage/NEWAGE /dump/first
	
Create the soft links for the current dump and the checkpoints
	cd /dump
	ln -s first/NEWAGE current
	ln -s first/NEWAGE current_chk

Create the configuration files for rsync
	mkdir /dump/.rsync/
	touch /dump/.rsync/exclude

Create the configuration files for systemd

	mkdir -p $HOME/bin
	cp scripts/dumpme scripts/chkptme $HOME/bin
	cp systemd/*.service systemd/*.timer $HOME/.config/systemd/user
	chmod -x $HOME/.config/systemd/user/*.service $HOME/.config/systemd/user/*.timer

Test the units

	cd $HOME/.config/systemd/user

	systemctl --user start dump.service

Debug it
	du -a /dump
	systemctl --user status dump.service	#see what happens
	systemctl --user daemon-reload		#reload
	systemctl --user start dump.service		#try again

Once fixed and running

	systemctl --user enable dump.service
	systemctl --user enable dump.timer
	systemctl --user start dump.timer

To stop it 

	systemctl --user disable dump.service
	systemctl --user stop dump.timer
	systemctl --user disable dump.timer

The same can be done for chkpt.timer and chkpt.service.

The logs can be monitored continuously with
	
	journalctl -f
	

The commands to navigate the dump can be found at:

	[commands to navigate the dump] (https://github.com/paurea/dump)

I got many of the ideas from:

	[time machine with rsync](https://blog.interlinked.org/tutorials/rsync_time_machine.html)
	[plan 9 history](http://man.cat-v.org/plan_9/1/history)