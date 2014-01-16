---
layout: post
tags: [OSX]
title: OSX as a PXE-Boot Server
---
I actually wrote this up a few months ago as a
[reply](http://techdebug.com/blog/2008/12/08/using-your-macbookpro-to-pxeboot-openbsd/comment-page-1/#comment-39340)
to a
[blog entry](http://techdebug.com/blog/2008/12/08/using-your-macbookpro-to-pxeboot-openbsd/),
detailing my own personal experience and variations on this process, but I'm
reposting it here on my blog now for my own reference and for anyone else who
is interested.  This process can, of course, also be adapted to PXE-boot other
things such as a CentOS [Kickstart](http://www.smtps.net/pxe-kickstart.html)
install.

Here are the steps I used to successfully
[PXE](http://en.wikipedia.org/wiki/Preboot_Execution_Environment)-boot
[OpenBSD](http://www.openbsd.org/) from OSX.  My MacBook Pro is connected to
the Internet via the AirPort, and my soon-to-be OpenBSD box is connected to
my Mac via the Ethernet port.  As a slight added complication, my WLAN uses
the 192.168.2.x subnet (which conflicts with the address range generally used
by OSX's Internet Sharing), so Internet Sharing needed to be adjusted to use
a non-default address range.

So first, I fixed my Internet Sharing address conflict:

*   Disable Internet Sharing
*   Close any System Preferences windows
*   Edit `/Library/Preferences/SystemConfiguration/com.apple.nat.plist` to add:  
    `SharingNetworkNumberStart 192.168.3.0`

Then I configured and launched tftpd:
*   Edit `/System/Library/LaunchDaemons/tftp.plist`
*   Remove the `Disabled` key+value
*   Add `-i` to `ProgramArguments`
*   Invoke launchd:  
    `launchctl load -w /System/Library/LaunchDaemons/tftp.plist`

I downloaded the appropriate OpenBSD `pxeboot` files to `/private/tftpboot`:

{% highlight sh %}
lwp-download http://ftp5.usa.openbsd.org/pub/OpenBSD/5.1/amd64/pxeboot
lwp-download http://ftp5.usa.openbsd.org/pub/OpenBSD/5.1/amd64/bsd.rd
mv bsd.rd bsd
{% endhighlight %}

Edited the `bootpd.plist` file:

*   Launch Internet Sharing and make a copy of `/etc/bootpd.conf`.
*   Stop Internet Sharing and copy your bootpd.conf back to `/etc`
*   Edit `/etc/bootpd.conf`:
*   Add `dhcp_option_66`
    *   Do an ifconfig to find the ip address of the interface that the
        OpenBSD box will be connecting to. In my case because of the
        modification I made to the InternetSharing settings, this is
        `192.168.3.1`, but for a default OSX install it would be
        `192.168.2.1`.
    *   Compute the Base64 string to use with this command:  
        `perl -MMIME::Base64 -e'print encode_base64(pack("C*",192,168,3,1))."\n"'`
*   Add `dbcp_option_67`
    *   The proper string to use is `cHhlYm9vdAA=` for `pxeboot`.
    *   If you want to use a different file, you can compute the string like
        this:  
        `perl -MMIME::Base64 -e'print encode_base64("pxeboot")."\n"'`

Start up Internet Sharing.

Make sure PXE-boot is enabled for the NIC on the OpenBSD box.

Boot it up!

NB: If your OpenBSD box is on the same LAN as your OSX box rather than behind
Internet Sharing as in my setup, you will need to adjust the IPs
appropriately, and manually configure and launch the bootpd service. (and be
careful to avoid DHCP server conflicts!)