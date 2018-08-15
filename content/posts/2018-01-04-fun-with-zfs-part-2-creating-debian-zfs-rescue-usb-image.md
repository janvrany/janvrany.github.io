---
title: "Fun with ZFS, part 2: Creating Debian ZFS Rescue USB Image"
created_at: 2018-01-04 22:36:26 +0000
kind: article
published: true
---

While ZFS is supposedly very stable and reliable it's not an integral part
of Linux and likely never will be (due to licensing issues). Because of that,
upgrading kernel or the  operating system may go wrong. Well, so far in my
case of Debian Jessie it was very smooth, but you never know. I have heard of
people whose experience was not that great.

Since I plan to do an OS upgrade, I want to be ready for the case the upgrade
go wrong and I won't be able to boot the OS stored on ZFS filesystem. In that
case, one needs a a bootable live system with ZFS ready-to-use. Ideally the
same version you're already running (or planning to run) on the system one wants
to upgrade.

In case of Debian, creating such an image turned out to be very very simple...

<!-- more -->

## Creating Debian Live Rescue Image with ZFS

Debian have tool named *live-build* [1]. The easies way to get is just

    sudo apt-get install live-build

Once installed:

  1. Create a (temporary) working directory where it all happens:

         mkdir /tmp/debian-streatch-live+zfs
         cd /tmp/debian-streatch-live+zfs

  2. Generate initial configuration:

         lb config \
           --distribution stretch \
           --architectures=amd64 \
           --binary-images iso-hybrid \
           --iso-volume "Debian Stretch ZFS Rescue Live" \
      	   --archive-areas "main contrib" \
      	   --linux-packages "linux-image linux-headers"


     We should include *contrib* in archives since that's where ZFS packages are.
     You may want to pass `--backports` if you want to include backports
     repositories. The `--linux-packages` is to include kernel headers necessary to
     compile ZFS kernel modules.

  3. Configure additional packages to be installed, most importantly the ZFS packages:

         echo "
         zfs-dkms
         less
         curl
         wget
         openssh-client
         openssh-server
         openssh-sftp-server
         " >> config/package-lists/my.list.chroot

   4. Build the live root filesystem:

          sudo lb bootstrap
          sudo lb chroot

   5. Build the hybrid ISO binary:

          sudo lb binary

      After this step, you should find the resulting ISO image in `live-image-amd64.hybrid.iso`.

   6. *Profit!*

## Using the Rescue Image


By default, ZFS kernel module is not loaded, so:

    sudo modprobe zfs

Then you need to import pools:

    sudo zpool import

It may happen that pools were not cleanly exported. Then you'd have to import them with
`-f` flag (a.k.a. force):

    sudo zpoool import -f tank

You may check status of pools by:

    sudo zpool status

Only do so if you're absolutely sure no other system is accessing the pool at the
time. But if you run ZFS normal PC with couple SATA disks attached like me and
the PC is currently running the live rescue image than you should be fine.

Mount host's root filesystem to `/tmp/host`:

    mkdir /tmp/host
    mount -t zfs tank/host/root /dev/host

Now you can `chroot /dev/host /bin/bash` and do what you want to create ZFS
snapshot, whatever you wish. Just make sure that once you're done you export
pools so they can be cleanly imported by the host.

    sudo zpool export tank

## Some Comments

  1. The configuration for `lb` command is stored in `config` directory. The
     `lb config` with various parameters just modifies (or creates if it does
      not exit) contents of the `config`. It may be a good idea to put the
      `config` into under version control such as Mercurial or git.

  2. The live image produced as above is really minimal, there's not even a
     `man`. You may want to add more packages. To do so, simply add them
     to the `config/package-lists/my.list.chroot`. You may pull in a whole
     GNOME by adding `gnome-desktop` package (though I have not tried).

To wrap it up, the process of building live Debian image is surprisingly easy.
Kudos to Debian people!


[1]: https://debian-live.alioth.debian.org/live-manual/stable/manual/html/live-manual.en.html




