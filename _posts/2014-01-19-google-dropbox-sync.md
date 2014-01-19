---
layout: post
tags: [OSX]
title: Google Drive and Dropbox Sync
---
> I also noticed that my harddisk ... seems to be going, so keep your
> fingers crossed.  I thought I'd better upload what I have now, rather than
> notice that I lost everything when I get back to work on Monday..
> (Only wimps use tape backup: _real_ men just upload their important stuff
> on ftp, and let the rest of the world mirror it ;)  
> - _[Linus Torvalds](https://groups.google.com/forum/#!msg/linux.dev.kernel/2OEgUvDbNbo/bTk-VE1zrnYJ)_

Having lost many a hard drive myself, most recently just last month, I've spent
a lot of time looking for a good, modern Internet-based backup solution.  Years
ago, [Mozy][] began offering such a service, and I tried it out at the time,
but for whatever reason, I didn't keep using it.  Other products have since
appeared, such as [Dropbox][] and [Google Drive][], and with the popularization
of the concept of "The [Cloud][]," focus has shifted from a simple replacement
for tape-backups, to a full document store.  Features such as online
[document](http://www.google.com/drive/apps.html)
[editing](https://write-box.appspot.com)
have also become common.  Synchronization to multiple machines is also a
becoming common use case.

Many of these products offer a free service tier that is sufficient for storing
a reasonable amount of data - perhaps not a full disk image with applications,
but at least my entire documents folder will fit.  So the other day I decided
to try out Dropbox and Google Drive.  I signed up for a free account on each,
and installed the free software on my Mac.

Everything went well, but I was a bit frustrated that I couldn't set them to
use the same folder on my Mac.  I ended up with two separate folders: one named
`Google Drive`, and one named `Dropbox`.  My first attempt to circumvent this
limitation involved one a [symlink][] to the other, but the apps weren't to be
fooled so easily, and simply refused to acknowledge the symlink as a valid
folder.

Next, I tried making a [hardlink][], but discovered that the [ln][] command on
OSX refuses to create directory hardlinks.  Not to be deterred, I dug up a
[utility](https://github.com/selkhateeb/hardlink) that bypassed this limitation,
compiled it, and tried it.  At first, it seemed to be a success - files that I
saved in one local folder appeared instantly in the other.  But after further
testing, I realized that depending on which folder name I used, only one of the
synchronization apps would trigger and upload the file.  My best guess is that
the [filesystem-monitoring facilities][FSEvents] in OSX which the apps use to
receive notification of file changes do not understand hardlinks, so even
though the filesystem itself may be storing only a single folder, a change
event is only triggered for the particular path name which was used in the
syscall that produced the change.

I began to consider other alternatives - writing my own script that monitored
file change events and propagated changed between the two folders seemed simple
at first, until I realized I would have to keep some additional state
information to properly handle propagation of file deletes.  [rsync][] also has
the same limitation, so I looked into [unison][].  Unison seemed to have all
the correct features for synchronizing the two folders, but no facility for
continuously monitoring for changes.

Coming from a Linux background, running unison repeatedly from [cron][]
immediately came to mind as one possible way to do it.  But I seemed to
remember that on OSX, things are done a little differently.  Sure enough, it
turned out that [launchd][] is the preferred tool for scheduled tasks.

As I was reading up on how to create a [launchd.plist][] file, I came across
the `WatchPaths` key.  Bingo!  Using this, I could use `launchd` to schedule
a `unison` launch not based on a timed schedule, but rather triggered by any
change made within a set of directories.  One final more I had was
getting `~` to expand to my user directory when passed to `unison` from the
`launchd.plist` file (I didn't want to hard-code my username into that
file so that it would be portable to other user accounts.)  The solution
was to enable the `EnableGlobbing` option.

Finally, there were a few files in those folders that weren't part of the
data I wanted synchronized: `.dropbox.cache`, `.DS_Store`, and `Icon?`

So here is how to set it up on your Mac:

1. Make sure you have [Homebrew](http://brew.sh) installed.
2. `brew install unison`
3. Make sure you have the [Dropbox app for Mac](https://www.dropbox.com/downloading?os=mac) installed.
4. Make sure you have the [Google Drive app for Mac](https://tools.google.com/dlpage/drive) installed.
5. Save [this gist](https://gist.github.com/8470353) to
   `~/Library/LaunchAgents/gdrive-dropbox-sync.plist`
6. Run this command to load the config:  
   `launchctl load -w ~/Library/LaunchAgents/gdrive-dropbox-sync.plist`

A file in your home directory named `unison.log` should now start recording
any synchronization events that occur.  You can save files to either your
Dropbox folder or your Google Drive folder, and these changes will instantly
propagate to the other.  This also applies to changes you make to these files
from another computer or your mobile device - as long as your Mac is online
(or as soon as it reconnects if it was offline), the changes will carry over.

This means that you can edit a document on your Google Drive via Google
Documents, and as soon as you save it, the change will appear on your Mac and
then be copied over to your Dropbox automatically.

[Mozy]: http://mozy.com/free
[Dropbox]: https://www.dropbox.com
[Google Drive]: https://drive.google.com
[Cloud]: http://en.wikipedia.org/wiki/Cloud_computing#Origin_of_the_term
[symlink]: http://en.wikipedia.org/wiki/Symbolic_link
[hardlink]: http://en.wikipedia.org/wiki/Hard_link
[ln]: https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man1/ln.1.html
[FSEvents]: https://developer.apple.com/library/Mac/documentation/Darwin/Reference/FSEvents_Ref/Reference/reference.html
[rsync]: http://rsync.samba.org
[unison]: http://www.cis.upenn.edu/~bcpierce/unison/
[cron]: http://en.wikipedia.org/wiki/Cron
[launchd]: https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man8/launchd.8.html
[launchd.plist]: https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man5/launchd.plist.5.html