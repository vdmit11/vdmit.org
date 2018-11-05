---
title: Debian Live USB using debootstrap
description: How to install a Debian Linux to USB stick from another (already running) Debian system.
category: linux
tags: linux debian sysadmin debootstrap
---

In this text, I'm going to describe how to create a kind of Live USB Debian system.

Traditionally, "Live USB" is made using a "Hybrid" approach, where a
Live CD image (an `.iso` file) is written to a USB drive. 
And when such Live USB system boots, you get a simulated CD drive.

This is [the way used by Debian Installer](https://www.debian.org/releases/stable/amd64/ch04s03.html.en#usb-copy-isohybrid),
and this is what you get if you use [UNetbootin](https://unetbootin.github.io/) to create a Live USB
stick, and this approach is described in existing guides I've seen on the Internet.

And I don't quite like this "hybrid" technique, because it involves CD
emulation, which seems a bit necro to me. Not to say that the CD image
is hard to modify (especially from inside the running Live CD system).

What I like to do instead, is to install a first-class citizen Debian
system directly onto USB drive (no CD emulation) as if it was an
ordinary HDD/SSD installation. Like what is done by Debian Installer,
but with no need for any installer CD.


[`debootstrap` is the tool](https://linux.die.net/man/8/debootstrap) designed for that purpose.
It allows to install a Debian sytem from already running Debian system.

And actually, [this is how Debian Installer CD works](https://salsa.debian.org/installer-team/base-installer):
it boots a Debian Linux system from CD, and runs `debootstrap` to install another
system on HDD. And if you have an already running Debian system,
then you don't need a special installer. You can just call `debootstrap` directly.

So all you need to do that is a USB stick plugged in, and a running
Debian system (Ubuntu, Mint, and other Debian-based distors are
suitable).

And yes, before you start, you need to install the `debootstrap` package on your host system:

```sh
~$ sudo apt-get install debootstrap
```


## 1. Prepare the device


#### Check the device and erase existing data

[`badblocks` is a tool](https://linux.die.net/man/8/badblocks)
that searches devices for bad blocks.
When I'm getting a new device into use, I usually run such check, because
otherwise, it may be an unpleasant surprise to discover a defect later,
after you put some files in there.

With `-w` flag, it will perform a read-write test that erases all
existing data. So make sure you don't have any important files on the
device.

```sh
~$ sudo badblocks -s -w -t 0 /dev/sdX
```

 - `-s` shows the progress percentage in the console
 - `-w` means to perform a read-write test (default is a read-only test)
 - `-t 0` means to make the single write pass using zeros
   (default is to do 4 different passes)
 
`/dev/sdX` is a path to the USB stick device. In my case it is `/dev/sdc`, 
and on your machine it may be different of course. Use `dmesg` to figure
it out. And be careful, as you may erase a wrong device.


#### Create new partition table

Erasing the whole device means that there is no partition table
anymore. That is, if you had `/dev/sdc1` - it is gone, and now there is
just `/dev/sdc` without any partitions. And you need to create one.

The one-liner spell below creates the single partition
that occupies the whole device:


```sh
~$ echo ";" | sudo sfdisk /dev/sdX
```

If you don't know how to use [`sfdisk`](https://linux.die.net/man/8/sfdisk),
but want a different partitioning scheme, then try [`cfdisk`](https://en.wikipedia.org/wiki/Cfdisk)
as a more user-friendly partition manager


#### Format the new partition

Now you need to format the new partition to create a file system in there:

```sh
~$ sudo mkfs.ext2 -L liveusb_root /dev/sdX1
```

 - `-L liveusb_root` is the label (will be needed later for configuration)
 
I have chosen `ext2` file system because it is a native system for
Linux, and it has no journaling, so it performs less writes to my slow
USB Flash drive. If you're installing to HDD or SSD, you probably want
to use `ext4`, or any other FS you believe is good.
 

#### Mount partition as a directory

OK, now you have an empty ready-to-use partition. You need to mount it
as a directory, because later you will put in some files.

```sh
~$ sudo mkdir /mnt/liveusb
~$ sudo mount /dev/sdX1 /mnt/liveusb
```


## 2. Debootstrap

The `debootstrap` is the main tool here: it installs a Debian system from
already running Debian system.

This is how you run it:

```sh
~$ sudo debootstrap --arch=amd64 stable /mnt/liveusb
```

- `stretch` is the release name.
   You can see the list of available release scripts in `/usr/share/debootstrap/scripts/`

That will create a root file system inside `/mnt/liveusb`
(with all those `/usr` and `/bin` and `/lib` directories),
with a bunch of core `.deb` packages installed.

Essentially, this is a small Debian system installed to a sub-directory.
It is not yet bootable: the bootloader and the Linux kernel itself is missing,
but it contains all necessary executable files to run a `bash` shell.

And you soon will utilize this shell to finish the installation.


#### Basic configuration

But before you can switch to the new shell, you need to copy some basic
configuration from your host system.

```sh
~$ sudo cp /etc/resolv.conf /mnt/liveusb/etc/resolv.conf
~$ sudo cp /etc/locale.gen /mnt/liveusb/etc/locale.gen
~$ sudo cp /etc/default/locale /mnt/liveusb/etc/default/locale
```


#### Mount special file systems

And one more thing that will be needed later is the special virtual
file systems: `/dev` and `/proc` and `/sys`. They are not required to
run a shell, but some `debconf` scripts (triggered when you do `apt
install`) will fail if they are missing.

So here how do you mount them:

```sh
~$ sudo mount --bind --make-rslave /dev /mnt/liveusb/dev
~$ sudo mount --bind --make-rslave /proc /mnt/liveusb/proc
~$ sudo mount --bind --make-rslave /sys /mnt/liveusb/sys
```


#### Chroot

Finally, you're ready to switch to the installed system:

```sh
~$ sudo chroot /mnt/liveusb /bin/bash
```

you may notice your shell prompt changed:

```
root@localhost:/# 
```

Now you're insdie the target system.

It may seem strange if you didn't work with `chroot` before, but that
is right: you don't need to reboot. If you have another Linux system
installed into some sub-directory, you can easily run a shell in there.


Now `/mnt/liveusb` is your root directory `/`, and all subsequent
commands will be executed inside this isolated root environment. 


## 3. Install packages

Now, when you're inside your newly installed system, you can execute
any commands, including `apt install`. And this is how you're going to
install packages.

But before you can install packages, you have to tune APT repository list:

```sh
cat <<EOF > /etc/apt/sources.list
deb http://deb.debian.org/debian/ stable main contrib non-free
deb http://deb.debian.org/debian/ stable-updates main contrib non-free
deb http://deb.debian.org/debian-security stable/updates main
EOF
```

If you want a different release, then you may want to use:

 - [Debian Sources List Generator](https://debgen.simplylinux.ch/)
 - [Ubuntu Sources List Generator](https://repogen.simplylinux.ch/)

#### Locales

And another thing is to install localization data.
This is not strictly required to operate, but it helps to avoid a bunch of noisy
warning messages that printed when you install packages with missing locale data.

```sh
apt update
apt upgrade
apt install tzdata locales
apt install localepurge
```

And `localepurge` is a little trick that helps to save disk space
(useful for USB sticks). 

#### Install packages

Finally, you can use `apt` to install some packages.

Here is a list of packages that I include to the base system.
Mostly these are administration tools, device drivers, networking packages,
and other "basic" packages useful for any installation:


```sh
apt install \
  aide anacron bzip2 ca-certificates cifs-utils cryptsetup curl debootstrap \
  dnsutils dosfstools e2fsprogs e2fsck-static elinks file firmware-atheros \
  firmware-iwlwifi firmware-linux firmware-linux-nonfree firmware-realtek \
  hdparm mcelog procinfo sysstat chkrootkit cmospwd foremost screen tmux \
  secure-delete bash ecryptfs-utils ifupdown iproute2 iptables iputils-ping \
  isc-dhcp-client iw kbd less linux-image-amd64 lm-sensors lshw lynx man mdadm \
  lvm2 lsof htop iotop mc modemmanager nano ncurses-term net-tools netcat nmap \
  ntfs-3g openssh-client openssl p7zip-full p7zip-rar pciutils pm-utils ppp \
  pppconfig pppoeconf psmisc resolvconf rng-tools rsync sdparm smbclient socat \
  sshfs sudo tcpdump telnet testdisk traceroute unzip usb-modeswitch usbutils \
  vim wicd-curses wireless-regdb wireless-tools wpasupplicant wvdial
```

## Configure the Bootloader

Now you have a nice Debian system with a bunch of base packages installed.
But... it is not bootable yet.

In order to make the device bootable, you have to install a
bootloader, Grub in my case.

But, before you install, you need to write the `/etc/fstab` file
(needed for configuration script executed automatically right after
installation):

```sh
cat <<EOF > /etc/fstab
LABEL=liveusb_root    /        ext2    noatime  0 1
EOF
```

And now you can install the bootloader itself:

```sh
apt-get install grub2
```

After the installation, the configuration script will offer you a
menu, where you have to choose the device to install the bootloader.

You should select your USB Stick device there (sic: the whole device,
not a partition).

This is needed to actually write the bootloader code to special area
on the device, which is triggered when computer boots.

Be careful to choose the right device. Otherwise, you will get a
surprise after you reboot your host system.


#### Install desktop environment

Now, if you're not comfortable with just bootable shell,
you can also install a Desktop Environment, maybe some web browser, 
and any other extra packages you need:

```sh
apt-get install lxde
```

In my case, I don't have enough free space on my USB Stick (it is an
old device), so I skipped this step, and left my installation with
only base set of CLI tools.  That should be good enough for a rescue
system.


## Final configuration

You're almost done.
But before you exit, you need to leave some configuration to make the system usable after reboot.

First, some very basic networking configuration:

```
cat <<EOF > /etc/network/interfaces.d/lo
auto lo
iface lo inet loopback
EOF

echo liveusb > /etc/hostname
sed -i 's/localhost/localhost liveusb/' /etc/hosts
```

And second, you need to create a user account.
Otherwise, you will not be able to log in.

```sh
adduser username
usermod -G sudo,netdev username
```


#### Exit from chroot

Before exit, you can clean up `.deb` files download from the internet.
They are not needed after installation, so deleting them allows to win some free space on USB stick:

```
rm -rvf /var/cache/apt/archives/*.deb
```

Finally, you can exit from your `chroot` environment to your host system:

```sh
exit
```

After this point, you're on your host system again.

You're almost done. Before ejecting the USB device, you need to
un-mount all the file systems:

```sh
~$ fuser  --verbose --kill /mnt/liveusb/
~$ sudo umount --recursive /mnt/liveusb/
```

- `fuser --kill` command is needed to stop background processes
   that are sometimes spawned during package installation.


## Done

Once you un-mounted the device, you can re-boot and test it. You
should have a full-weighted first-class-citizen Debian Linux system
that boots and works directly from the USB Flash device (as if it were
an HDD).

The nice thing about it is that the system is writable (in contrast to hybrid Live CD/USB systems).
You can install new packages and update the configuration on the fly,
straight from the running system (as you normally do on a desktop system).

And the bonus nice thing is that now you leraned this `debootstrap`
and `chroot` techniques, and now you can make a lot:
 
  - infest your friend's USB sticks using `debootstrap`
  - use such USB sticks to infest more machiens with Debian
  - in case you break something, you can boot from such USB stick and then use `chroot` 
    to switch to the broken system, and maybe even fix it
  
There is a lot of useful uses. 

Have fun.


### Known Issues


#### Architecture

This whole technique can only work if your host architecture matches the target architecture.
If you have an `amd64` host, and for example you want to bootstrap an `armel` system, then you're in trouble.

This is possible ([using multistrap](https://wiki.debian.org/EmDebian/CrossDebootstrap)),
but the process is commplicated, and it is really easier to download
a prepared virtual machine image (ir an `.iso` file) in this case.


#### BIOS vs UEFI

This guide assumes you have a motherboard that supports BIOS bootloader mechanism.
Yes, it is a bit old, but it is simple, and it is still the lowest common denominator.
Majority of motherboards (I've seen to this day) do support BIOS mode booting, at least as a compatibility option.

If you want to use [UEFI](https://wiki.debian.org/UEFI) (which I do recommend honestly),
then the process will be slightly more complicated: you will need to allocate a special system partition,
and use `grub-efi`, and configure it differently, and so on.

I've omitted this option for simplicitly, because the text is already long.


#### Performance

The described technique involves running `apt install` on a USB Flash drive.
If the device is slow, the instllation process may take hours.

To speed it up, I prefer to do this whole process in a temporary
directory on my main SSD, and then copy it to USB stick at the very
last moment.

But this scenario is more complicated: you have to enter/exit `chroot`
multiple times, and mount-unmount file systems, and maybe some extra
Grub bootloader manipulations, and so on. So I omitted it for
simplicity, but you can try it.
