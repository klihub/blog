---
layout: post
title:  "Sidegrading 32-bit Fedora to 64-bit"
date:   2020-02-29 14:45:31 +0200
categories: linux distro fedora
---

# Upgrading 32-bit Fedora 16 to 64-bit Fedora 31

A while ago one of the disks in the RAID array of my home file server was
taken automatically offline after it has encountered a number of read and
write errors. I bought a new disk, swapped out the broken one and brought
the array back up to fully operational mirrored state. With money and time
well spent and a job well done, I thought I'm through with the ordeal...

However, after aimlessly looking around in the box for a while I decided
to turn my attention to some of its other vital signs. It turned out that
it had 4-5 years of continuous uptime under its belt. More importantly and
somewhat more shockingly it was still running Fedora 16 and a 3.7.x series
kernel. Suddenly I found myself in a situation where the long-neglected box
was in desperate need of some gentle love and serious care.

Beside being my file server hosting among other things our full music and
family photo collection, the box was doubling as the DHCP- and DNS-servers
in the network and was also serving disks to my diskless living room media
box using [ATA over Ethernet (AoE)][AoE]. I was not set up to recreate the
box from scratch with a fresh installation of the latest available distro
version. I did not have RPM packages for my custom tweaks for serving AoE,
let alone Ansible playbooks to set up the rest of the box with the right
configuration.

I quickly decided that I will do what I almost always end up doing: upgrade
instead of wiping out and reinstalling. I've done this a countless times on
a number of Fedora boxes both at work and at home, development workstations
and test servers alike. Because of its duties and the fact that the box was
tucked away, under and behind a big pile of junk, in our storage room with-
out easy physical access, with no keyboard or monitor attached I knew that
this is more risky than usual.

I also realised that because of the amount of ground I needed to cover this
probably will be more laborous than usual. I had to go from Fedora 16 to 31
switching in between from YUM to DNF. Even if I stick to my habit of hopping
over several releases at a time, contrary to what the official documentation
recommends the distance is significant.

As icing on this cake of misery, i686 was downgraded already in Fedora 27 to
the status of 'community supported' and the latest Fedora 31 has given the
boot for 32-bit x86 support for good. Luckily, my box was built around the
Intel D510MO, a passively cooled mini-ITX board with a 64-bit processor. So
I decided that while I upgrade to the latest distro version, I will also
sidegrade from a 32- to a 64-bit version of the distro.

To ease my mind and cover some ground I decide that I will first upgrade to
a farily recent version before I attempt the sidegrade. Having done it a
number of times, the upgrade went smoothly. I triple-jumped forward, hopping
over several versions at a time, slowing down only around the critical spot
where the distro switched from YUM to DNF.

By the time I reached Fedora 25 I got both comfortable and confident with
the progress, but a bit bored by the repetitive DNF command sequence, and
also a bit anxious as I was quickly running out of 32-bit versions to go
to. I decided it was high time to find out what exactly it takes to switch
from 32-bit to a 64-bit version. Sidegrading a distro is something I had
never attempted before.

It only took a single google search to find out that officially it is not
supported at all. A bit more googling turned up a lot of people asking how
to do this in practice. Even more googling turned up very limited anecdotal
evidence, actually more closer to folklore, by a few folks claiming they
have done it successfully. What did not turn up, no matter how much I
searched, was any detailed description about what the problems in the
process are and how to overcome them.

Well, no worries, at least not too much... It was well documented, that the
necessary first step was to switch to a hybrid setup running a 64-bit kernel
and a 32-bit userspace. I pulled in a 64-bit kernel, a bunch of other stuff
it needed, rebooted and was up and running with such a setup in no time.

Cool, great progress so far. Well, how hard can the rest be... apparently
very. Here is a rough outline of what I remember I had to do to make it all
eventually work. It is all from my memory as unfortunately I did not take
notes of the process or my progress. In the end you'll find a link to the
.bash_history file I later saved trying to recall some of the details.

TODO: add description and .bash_history...

[AoE]: https://en.wikipedia.org/wiki/ATA_over_Ethernet