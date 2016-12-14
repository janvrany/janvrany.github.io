---
title: "Fun with ZFS, part 1: Installing Debian Jessie on ZFS Root"
created_at: 2016-10-06 21:29:27 +0100
kind: article
published: true
---

## Preface 

Couple weeks ago I decided to make myself a small home server. Home sever, 
sounds weird, doesn't it? Home's not work place, no? 

Well, it is, in my case. I have no need for media center, downloading and 
streaming movies or even playing on TV, nothing like this. But I need a  storage
 for backups and host to run virtual machines on. 

<!-- more -->

Initially I thought about buying some NAS but then a friend pointed out, 
correctly, that I need none of the features of small NAS solutions for home
sector so why paying for that. He suggested to go for a 
[HP ProLiant MicroServer Gen8][1]. I could get an entry-level model for a reasonable price so I got
myself one. 

This means, however, that I had to install and setup OS and everything myself.
Since some people expressed interest in my experience, I'll try to write down
some interesting bits. 

## Why ZFS? 

Why not? Since I want to run virtual machines on the server, I need something
that allows me to allocate storage for them, extend it, take snapshots and so
on. I used in the past Linux LVM, LUKS and ext4 / xfs on top of it but overall
experience was not-so-good. A lot of hassle to resize volumes, many commands,
super-easy to mess it up (I managed to mess it up couple times in the past). 
ZFS looked interesting, all-in-one solution<sup id="a1">[*](#f1)</sup>, people 
reported it is stable, battle-hardened and CPU and memory hungry. So I decided 
to give it a go. 

## Installing Debian Jessie.

People behind [ZFS On Linux][2] did an amazing job and wrote down an 
idiot-proof step-by-step guide 
[HOWTO install Debian GNU Linux to a Native ZFS Root Filesystem][3]. 
The only little issue is that it uses ZFS from ZOL project repository. 

Since ZFS found its way to Debian Project [4], I decided use Debian packages. 
Everything was pretty much straightforward but there were some gotchas. 

This is how I did it: 

 1. Boot from [Debian Live 8.6 Live CD][6]
 2. Log in to live session (username is "user", password "live") and
    install package [zfs-dkms][4] from [jessie-backports][7]:

    * Edit `/etc/apt/sources.list` and add following:
      
      <pre><code>deb http://ftp.debian.org/debian jessie-backports main
      deb-src http://ftp.debian.org/debian jessie-backports main
      deb http://ftp.debian.org/debian jessie-backports  contrib
      deb-src http://ftp.debian.org/debian jessie-backports  contrib</code></pre>

    * Update package database:

      <pre><code>sudo apt-get update</code></pre>

    * Install `zfs` module (userland tools will be pulled as dependencies):

      <pre><code>sudo apt-get install zfs-dkms</code></pre>   

 1. Now follow [ZOL guide][3] up to [Step 3: Disk Formatting][8]. My server
    has an on-board MicroSD card slot and I had old, unused 2GB MicroSD card
    so I decided to use it for while `/boot`, not only for `/boot/grub` as the guide suggest. This looked to me as safer option since this way kernels and
    initrd images are stored a good old ext3 so GRUB does not need to 
    understand ZFS.                     
 1. Then continue with ZOL guide up to [Step 6: Cleanup and First Reboot][5]. 
    Just before leaving chroot do following:

    * Edit `/etc/default/grub` and add `boot=zfs` to 
      `GRUB_CMDLINE_LINUX_DEFAULT` variable. This is required
      for code in initrd to setup ZFS before mounting root
      filesystem. Actually, this took me some time for figure this
      out and I only find out after reading the scripts in generated 
      initrd. 

    * In `/etc/grub.d/10_linux` fix the code that determines current 
      filesystem. Comment the original code and add new one that does it by
      simply parsing output of `mount` command. 

      <pre><code>  xzfs)
          # Original code (does not work)
          #rpool=`${grub_probe} --device ${GRUB_DEVICE} --target=fs_label 2&gt;/dev/null || true`
          #bootfs="`make_system_path_relative_to_its_root / | sed -e "s,@$,,"`"
          #LINUX_ROOT_DEVICE="ZFS=${rpool}${bootfs}"

          # Workaround          
          zfsroot=`mount | grep '/ type zfs' | cut -d ' ' -f 1`
          LINUX_ROOT_DEVICE="ZFS=${zfsroot}"
          ;;</code></pre>

      Indeed this is an ugly hack and fragile, but works for me for now. ZOL
      team seems to be aware of this and is working on a fix, AFAIK. 
    
    * Create new file `70-zfs-grub-fix.rules` with following contents: 

      <pre><code>KERNEL=="sd*", ENV{ID_SERIAL}=="?*", SYMLINK+="$env{ID_BUS}-$env{ID_SERIAL}"
ENV{DEVTYPE}=="partition", IMPORT{parent}="ID_*", ENV{ID_FS_TYPE}=="zfs_member", SYMLINK+="$env{ID_BUS}-$env{ID_SERIAL} $env{ID_BUS}-$env{ID_SERIAL}-part%n"</code></pre>

      and refresh `dev` by issuing:

      <pre><code>udevadm trigger</code></pre>

      For whatever reason (perhaps a bug) GRUB scripts does no look for devices
      in `/dev/disk/by-id` but in `/dev` and fails when it cannot find them. This ugly indeed but again, works for me for now. Again, ZIL team works
      on this too. 

    * Finally, update GRUB: 

      <pre><code>update-grub</code></pre>

  1. And continue with ZOL guide. 

  1. Have fun with ZFS! 

Hope it helps.

## A follow up...

Hajo Noerenberg wrote a [script][9] that automates the above process and was so nice
to share it: 

> Hi, 
>
> Debian's grub has problems with detecting ZFS pools with
> feature@hole_birth=enabled and feature@embedded_data=enabled. If you
> disable those features (only possible for new pools), grub installs
> correctly without errors (and /etc/grub.d/10_linux correctly detects
> ZFS
> as well).
>
> I've put together a fully automatic script for native ZFS
> installation: https://github.com/hn/debian-jessie-zfs-root 

I haven't tried myself, but clearly worth having a look. I'll certainly give
it a go as soon as [native ZFS encryption][10] finds it's way to ZFS on Linux.

---

<b id="f1">*)</b>
I can hear so-called UNIXers screaming this is bad, against UNIX 
philosophy, blah blah blah. You know, each tool should do exactly one 
thing then you can combine them the way you like, not the way developer 
like. This is the one and true "UNIX way". It might well be against 
UNIX philosophy, but man: I can't care less :-) As long as it works well
and is easy to use, I'm fine with that. 
[↩](#a1)



[1]: https://www.hpe.com/us/en/product-catalog/servers/proliant-servers/pip.hpe-proliant-microserver-gen8-server.5379860.html#
[2]: http://zfsonlinux.org/
[3]: https://github.com/zfsonlinux/zfs/wiki/HOWTO-install-Debian-GNU-Linux-to-a-Native-ZFS-Root-Filesystem
[4]: https://packages.debian.org/jessie-backports/zfs-dkms
[5]: https://github.com/zfsonlinux/zfs/wiki/HOWTO-install-Debian-GNU-Linux-to-a-Native-ZFS-Root-Filesystem#step-6--cleanup-and-first-reboot
[6]: http://cdimage.debian.org/debian-cd/current-live/amd64/bt-hybrid/debian-live-8.6.0-amd64-standard.iso.torrent
[7]: https://backports.debian.org/Instructions/
[8]: https://github.com/zfsonlinux/zfs/wiki/HOWTO-install-Debian-GNU-Linux-to-a-Native-ZFS-Root-Filesystem#step-3-disk-formatting
[9]: https://github.com/hn/debian-jessie-zfs-root
[10]: https://github.com/zfsonlinux/zfs/pull/4329