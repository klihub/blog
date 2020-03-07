---
layout: post
title:  "All your disk are belong to NAS..."
date:   2020-02-29 14:45:31 +0200
categories: linux network nas home-server
---

# Table of Contents

1. [Introduction](#introduction)
2. [Original Exercise](#original-exercise)
3. [Argh... Cut the Crap and Get to the Point Already](#going-almost-diskless-with-aoe-dhcp-tftp-and-ipxe)

# Introduction

After a [recent chain of events][sidegrading-fedora] which eded up
with me eventually updating my home server/NAS box, I noticed that
my living room media box started crashing. It booted up fine, all the
way through runlevel 3 (a.k.a multi-user.target), but then it crashed
and stopped responding to [pings][ping] when switching to runlevel 5
(a.k.a graphical.target) trying to start up Xorg and eventually Kodi.

For a short while I did a half-arsed attempt and pretended trying to
figure out what was wrong with the box. It was running an ancient
Fedora 16, hacked to boot from my recently updated NAS box using
[ATA-over-Ethernet][AoE]. After the NAS update it took me quite a
while to notice that the custom AoE systemd service was not running.
All that while, for a week or so, the media box itself was up and
running. I know this for a fact because I have a bit strange setup
of other gadgets with audio being routed from my PS4 to the speakers
shared by all living room devices via the media box. If the media box
is down I simply don't get any audio from the PS4. Yet I was watching
Netflix every evening without any problems the whole week.

Since the box was set up with two LVM volumes served over AoE, a system
one for the distro itself and a storage one for any other data, I was
pretty sure there is no danger of losing data by reinstalling the box
from scratch, so I quickly decided to do just that. Especially, since
this would also give me the opportunity to check how it compares to the
original exercise and if I could fix some of the annoyances still
present in the current setup.


# Original Exercise

I set up the media box originally in 2011 or 2012. The hardware itself
is one of the early NUCs with a single Gigabit Ethernet port, two HDMI
outputs, two USB 2.1 ports and nothing else on the outside except for
the power supply input.

#### Box Setup Procedure

The procedure to set up the box with AoE went something like this:

 1. put in a real hard disk, install everything on that
 2. create a dedicated LVM volume for the distro on the server
 3. copy the disk content to the new volume
 4. set up everything for serving AoE in the server
 5. hack an AoE dracut module until it finally boots

Step 5 was a little bit more involved because I couldn't figure out
at that time how to get the bootloader bootstrapped over AoE. I was
using a custom-configured gPXE image for the AoE boot with only a
single hardware driver enabled, the right one for the NUC. Eventually
I ended up enabling HTTP support in gPXE and booting the media box
by fetching the kernel and initramfs images with HTTP from the NAS
server.

#### Bootup Sequence

I ended up with the following multi-stage bootup sequence, everything
hosted from the NAS server, with PXE enabled in the BIOS in the media
box for network boot:

  1. use DHCP to tell the box to come back and TFTP-fetch a boot image
  2. TFTP-serve an (AoE-) and HTTP-enabled gPXE image
  3. use a baked in gPXE script to fetch kernel and initramfs with HTTP
  4. boot the kernel, with root set to root=etherd/...
  6. use a custom dracut module to modprobe kernel aoe module and do
    device discovery
  7. normal boot eventually pivot_rooting to etherd/...

#### Remaining Annoyances

While this setup was working fine, it had some undesirable properties.
The biggest one was that automatic kernel updates required extra glue,
either to copy the kernel to the fixed location the gPXE scripts were
looking for, or to set up symlinks for the HTTP server from the right
kernel and initramfs images to the fixed gPXE locations after every
update.

This turned out to not be a problem for me at that time for a very
simple reason: every single stock Fedora kernel in 16, and every single
release for years to come, had a large RedHat-/Fedora-specific patch set
which broke and rendered AoE support completely useless. Whenever doing
even moderately heavy block I/O over AoE the kernel OOPS'ed almost
immediately.

My go to test for this became simply trying to build the kernel with
`make -j4 bzImage` or something more parallel. For a while, I tried
the latest fedora kernels on the box to see if the problem is gone,
but I gave up once the exercise became very boring with a 100% predictable
outcome.

Since my media box was an appliance-like fixed-function device with the
single purpose of rendering media and it was running inside my home
network behind a firewall (actually a chain of two firewalls) never really
having to access the internet to perform its core duties, I ended up
disabling all automatic updates and locking the box down to the package
versions it had been running with.

Of course what I should have done was to report the bug to RedHat and
see if they care enough to try to bisect it and narrow down to the
probelmatic set of patches... or try doing that myself. However, I
strongly suspected at the time that my personal fixation of 'Ethernet
should be a natural extension of the PCI bus' made me neglibly marginal
user segment of 1. I should have tried nevertheless...


# Going (Almost) Diskless with AoE, DHCP, TFTP, and iPXE

Here is a brief description of how I recreated my diskless NUC media
box, directly installing Fedora 31 to a remote disk image using
[ATA Over Ethernet][aoe], and patched it up to be able to boot
from it.

### Collateral You'll Need

There are a number of thinsgs you'll need to get started.

#### Hardware

1. Gigabit Ethernet network, at least
  - WiFi: forget about it kiddo, this is real networking for grown up men...
2. a server with enough physical disk space
3. a media/HTPC box with network boot (PXE) enbled in the BIOS

#### Software

1. server: [ISC DHCP server][isc-dhcpd], I use the stock Fedora one
2. server: any TFTP server, I use the stock on Fedora
3. server: [userspace AoE server][vblade]
4. client: [iPXE boot firmware][ipxe]
5. client: [Fedora 31 installation media][fedora-media], I used the
           server netboot image

You might want to try [a kernel-space AoE server][kvblade] instead,
especially if your server is already heavily loaded. I know will take
a closer look at it later and if it proves to be stable and actively
maintained, I'll switch to it.

#### Configuration and Extra Software Bits

1. server: DHCP server: [configuration block](#dhcpd-config-block) for your
                media box
2. server: TFTP server set up to serve iPXE image for your media box
3. server: [AoE server][vblade] [custom aoe systemd service][aoe-systemd-service]
4. client: [dracut][dracut] [custom AoE module][aoe-dracut-module]
5. client: boot loader [entry][bootloaderspec] for testing AoE boot

### Configuring The Server

#### DHCP

#### TFTP

#### AoE

#### Final Checks

Finally, make sure that access to your TFTP server is not blocked.
If your server is in a safe network beind firewalls, the easiest
way to do this is to simply make sure firewalld is not running:

```
systemctl stop firewalld
systemctl disable firewalld
# or the big axe: rpm -e firewalld
```

### Installing Fedora On the Media Box


### Fixing Up the Media Box to Boot


[sidegrading-fedora]: 2020-02-29-siderading-fedora.html
[ping]: https://en.wikipedia.org/wiki/Ping_(networking_utility)
[AoE]: https://en.wikipedia.org/wiki/ATA_over_Ethernet
[isc-dhcpd]: https://www.isc.org/dhcp
[vblade]: https://github.com/OpenAoE/vblade
[kvblade]: https://github.com/john-sharratt/kvblade
[ipxe]: https://ipxe.org/
[fedora-media]: http://www.nic.funet.fi/pub/mirrors/fedora.redhat.com/pub/fedora/linux/releases/31/Server/x86_64/iso/
[aoe-systemd-service]: https://github.com/klihub/aoe-systemd-service
[dracut]: https://dracut.wiki.kernel.org/index.php/Main_Page
[aoe-dracut-module]: https://github.com/klihub/aoe-dracut-module
[bootloaderspec]: https://www.freedesktop.org/wiki/Specifications/BootLoaderSpec

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
