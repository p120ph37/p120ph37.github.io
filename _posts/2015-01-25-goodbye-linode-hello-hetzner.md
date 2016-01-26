---
layout: post
tags: [Security,Meta]
title: Goodbye Linode, Hello Hetzner
---
> There's an old saying in Tennessee — I know it's in Texas, probably in Tennessee —
> that says, fool me once, shame on — shame on you. Fool me — you can't get fooled again.  
> -_[George W. Bush](https://en.wikipedia.org/wiki/George_W._Bush)_

A long time ago I used to run Linux on a [beige box](https://en.wikipedia.org/wiki/Beige_box)
at home as a webserver and router.  Over the years, the box evolved, eventually being
migrated to a [VPS](https://en.wikipedia.org/wiki/Virtual_private_server).

At first, I used [Slicehost](http://37signals.com/founderstories/slicehost), and was very
happy with them for a while, but eventually they were [acquired by RackSpace and shut down](http://readwrite.com/2011/05/03/rackspace-shutting-down-sliceh).

At that point, I migrated to [Linode](https://en.wikipedia.org/wiki/Linode), but they never
quite felt as dedicated to doing things "right" as Slicehost had been.  Over the next few
years, they experienced several
[major](https://threatpost.com/linode-resets-customer-passwords-after-breach-ddos-attack/115790/)
[security](https://threatpost.com/linode-hacked-through-coldfusion-zero-day-041613/77733/)
[breaches](https://threatpost.com/linode-resets-customer-passwords-after-breach-ddos-attack/115790/),
and still have not been able to explain to my satisfaction how these came about or what
steps are being taken to avoid them in the future.

Additionally, the recent [Snowden](https://en.wikipedia.org/wiki/Edward_Snowden) leaks
call into question the [integrity of all US-based providers](https://en.wikipedia.org/wiki/PRISM_(surveillance_program))
with regard to engineered and incidental back doors for NSA access - if a back door is a
known part of the system design, [how secure can you really expect the system to be?](http://dspace.mit.edu/bitstream/handle/1721.1/97690/MIT-CSAIL-TR-2015-026.pdf?sequence=8)
Germany may not be as [anti-NSA as Bolivia](http://www.cbsnews.com/news/edward-snowden-offered-asylum-in-bolivia-by-president-evo-morales/),
but they are certainly [not as buddy-buddy](http://www.nytimes.com/2014/07/11/world/europe/germany-expels-top-us-intelligence-officer.html?_r=0)
as [the UK](http://www.theguardian.com/uk-news/2015/feb/06/gchq-mass-internet-surveillance-unlawful-court-nsa)...

So after [the most recent Linode incident](https://blog.linode.com/2016/01/05/security-notification-and-linode-manager-password-reset/),
I decided enough was enough and that I should find a new VPS provider.
One possibility was to use the French company [Gandi](https://www.gandi.net/hosting/iaas),
through whom I already manage my DNS, but their VPS pricing is not terribly competitive,
and they began operating a [US-based datacenter](http://www.gandibar.net/post/2010/12/20/US-Data-Center-Open-for-Business),
which implicitly subjects them to NSA purview.

Then I cam across a [discussion on Hacker News about European VPS options](https://news.ycombinator.com/item?id=5993947&utm_source=dlvr.it&utm_medium=tumblr),
and the [offering from Hetzner](https://www.hetzner.de/us/hosting/produktmatrix_vserver/vserver-produktmatrix)
looked promising: for the same price as I had been spending on Linode, I could get a much
larger VPS.  Account creation process went smoothly - they did request a photo of my ID
before approving the account creation, but that was quick and painless via email.
Server provisioning was not quite as fast as Linode, but not bad.  Their management
console is all custom built in-house, and though not quite as polished as Linode's yet, is
entirely functional.  What it lacks in polish it makes up for in utility - for example,
provisioning SSH keys from the web interface, including support for
[Ed25519](https://en.wikipedia.org/wiki/Curve25519) keys, which I am now using.

I now have both a Linode and a Hetzner instance up and running, and have cut over DNS
already.  If all goes well until the end of the month (end of the Linode billing cycle),
I will cancel my Linode service entirely.

So, goodbye Linode - nice try, but you'll have to do better to keep business these days.
