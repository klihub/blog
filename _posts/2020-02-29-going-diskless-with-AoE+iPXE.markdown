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

I have a [forked version][klihub-vblade] of [AoE server][vblade] with
rudimentary fedora packaging, if someone needs/fancies that.

### Configuring The Server

The main idea of our setup is that we enable our box, which has no native
capability to boot using AoE, do exactly that by first chain-loading an
AoE-enabled iPXE image using the native PXE support present in the
firmware of our box.

With this setup there is a sequence of several DHCP requests hitting our
server for any single boot from firmware to Linux:

1. DHCP request by firmware: response should direct to iPXE TFTP image
2. DHCP request by iPXE image: response should direct to AoE-boot
3. DHCP request by linux client: response with normal configuration

#### DHCP

You need to set up your DHCP server according to this idea: first let the
native PXE implementation chainload iPXE using TFTP, then let iPXE AoE-boot,
and finally let the linux DHCP client configure the IP stack, DNS servers,
etc.

I do this with the following fragment in `/etc/dhcpd/dhcpd.conf`:

```
  ...
  # Kodi/XBMC living room media box
  host kodi.dexlab.net {
      hardware ethernet ec:a8:6b:f1:ff:ad;
      fixed-address 192.168.16.8;

      # for iPXE clients don't pass an image, otherwise point to iPXE image
      if exists user-class and option user-class = "iPXE" {
          filename "";
          option root-path "aoe:e11.0";
      } else {
          filename "nuc/nucipxe";
      }
  }
  ...
```

We use here the DHCP `user class` option, which the iPXE implementation
sets by default to "iPXE", to differentiate between the native PXE and
iPXE phases. We don't treat the linux client in any special way, so it'll
get served by the else branch getting the iPXE filename. This is not a
problem since the client simply ignores this option altogether. The most
importat point in all this is to *not direct* the iPXE image to itself.
Failing to do so ends up in an infinite loop with iPXE chainloading
itself over TFTP forever.

Since I serve TFTP from the same server I serve DHCP requests, I don't need
to tell clients where the TFTP server is. If this was not the case I would
have included a next-server option there pointing to the TFTP server, like
this
```
...
      } else {
          filename "nuc/nucipxe";
          next-server server-address;
      }
...
```

Finally, in the above setup, we also pass to iPXE the AoE device is should
try to boot from. Although this could be done using an iPXE script embedded
in the iPXE image itself, this gives us more flexibility as we can freely
change where the box is booting from without having to rebuild the iPXE
image itself.

#### TFTP

For tftp I use the one provided by the sotck Fedora tftp-server package,
which looks to be the [one hosted on kernel.org][kernel-org-tftpd]. To
enable it, just install `tftp-server` and `xinetd`.

```
dnf install xinetd tftp-server
systemctl enable xinetd
systemctl start xinetd
```

Make sure you have tftp enabled by taking a look at `/etc/xinet.d/tftp`.
It should look something like this:

```
service tftp
{
        disable = no
        socket_type             = dgram
        protocol                = udp
        wait                    = yes
        user                    = root
        server                  = /usr/sbin/in.tftpd
        server_args             = -s /var/lib/tftpboot
        disable                 = yes
        per_source              = 11
        cps                     = 100 2
        flags                   = IPv4
}
```

The TFTP server will look for files relative to the directory
given by the `-s` options. In this case it is the default
`/var/lib/tftpboot`. Therefore, the full absolute path of the iPXE
binary given as `nuc/nucipxe` will be `/var/lib/tftpboot/nuc/nucipxe`
in the servers filesystem. If you prefer to keep it somewhere else,
change the xinetd TFTP configuration accordingly.


#### iPXE

You can compile and install a stock iPXE binary with the following
set of commands:

```
git clone git://git.ipxe.org/ipxe.git
cd ipxe/src
make bin/ipxe.pxe
mkdir -p /var/lib/tftpboot/nuc
cp bin/ipxe.pxe /var/lib/tftpboot/nuc/nucipxe
```

There are a number of things you can tweak with iPXE. The default
image should work just fine for us, as it includes everything we
need, most importantly support for AoE.

If you want to have an embedded iPXE script baked into your image
you can pass the the `EMBED=path-to-script` option to make. See
the documentation for more information about what you can do with
iPXE scripts.

#### AoE

You can install the user-space AoE server implementation from a
number of sources. I chose to clone and compile it from OpenAoE
on github, `https://github.com/OpenAoE/vblade.git`:

```
git clone https://github.com/OpenAoE/vblade.git
make
make install
```

#### Exporting AoE Devices

To install a simple script and systemd services to help you export
AoE devices you can:

```
git clone git@github.com/klihub/aoe-storage-export-service
cd aoe-storage-export-service
make rpm
rpm -ivh ~/rpmbuild/RPMS/noarch/aoe-storage-export*.rpm
```

Alternatively, you might want to take at the contributed presistence
add-on to [AoE server][vblade-persistence].

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

There is surprisingly little extra fiddling is necessary to
get a stock Fedora 31 image installed to an AoE devices. The
procedure is roughly this:

1. start installation as normally
2. set up keyboard, timezone, etc.
2. before chosing any disks, drop into a root shell by cycling through
Ctrl-Alt-F1...Fn until you find a shell.
3. pull in the aoe module by `modprobe aoe`
4. trigger AoE device discovery by `echo "" > /dev/etherd/discover`
5. Go to chosing disks. Here I had to go back and forth a few times
to the pre-disk selection before the AoE devices showed up. However,
I'm not sure if this is really necessary if you trigger AoE device
discovery early enough.
6. Set up your disks. It's the safest to go with manual partitioning.
Anaconda crashes with a Python exception if you let it go by its
default which is LVM. Since you'll be serving your disks over ethernet,
it is anyway a much better idea to have/use LVM on that side of the network.
7. Let the installation run to the finish.

At this point you have succesfully installed fedora on an AoE device.
However, at this point it cannot boot because

1. Your machine cannot natively boot from AoE devices.
2. Fedora did not realize that the AoE device is not an
   ordinary local block devices so it needs extra preparation
   for booting.

We'll be fixing these problems next.

### Fixing Up the Media Box to Boot


### Adding AoE to Your Initramfs(-generation)

For installing a simple dracut helper module you can:

```
git clone git@github.com/klihub/dracut-aoe
cd dracut-aoe
make rpm
rpm -ivh ~/rpmbuild/RPMS/noarch/dracut-aoe*.rpm
```



[sidegrading-fedora]: 2020-02-29-siderading-fedora.html
[ping]: https://en.wikipedia.org/wiki/Ping_(networking_utility)
[AoE]: https://en.wikipedia.org/wiki/ATA_over_Ethernet
[isc-dhcpd]: https://www.isc.org/dhcp
[vblade]: https://github.com/OpenAoE/vblade
[vblade-persistence]: https://github.com/OpenAoE/vblade/tree/master/contrib/persistence
[klihub-vblade]: https://github.com/klihub/aoe-vblade
[kvblade]: https://github.com/john-sharratt/kvblade
[ipxe]: https://ipxe.org/
[fedora-media]: http://www.nic.funet.fi/pub/mirrors/fedora.redhat.com/pub/fedora/linux/releases/31/Server/x86_64/iso/
[aoe-systemd-service]: https://github.com/klihub/aoe-systemd-service
[dracut]: https://dracut.wiki.kernel.org/index.php/Main_Page
[aoe-dracut-module]: https://github.com/klihub/dracut-aoe
[bootloaderspec]: https://www.freedesktop.org/wiki/Specifications/BootLoaderSpec
[kernel-org-tftpd]: http://www.kernel.org/pub/software/network/tftp

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
