# Setup

This is instructions for setting up gentoo on a system

* intel i7 processor
* wifi is primary network
* UEFI

# Setup network

Find your wireless card id by doing `ifconfig`

The fastest way to get wifi going in a basic way is `net-setup <wireless id>`

* set the ESSID to your wifi you want to connect to
* set password to WPA/WPA2 and type in your wifi password

# Setup disk

We're going to use a tool `parted` to setup your drives. For UEFI to work, your boot drive is going to need to be FAT32.

First, find your hard drive ( it might be `nvme0N0` if laptop hard drive or `sda` if big hard drive).

```
parted -a optimal /dev/nvme0N0
```

Once in the UI you can see what partitions already exist with `print`

Delete all current partitions with `rm <partition number>`

Once everything is cleared

```
mklabel gpt
unit mib
mkpart primary 1 130
name boot
set 1 boot on
mkpart primary 130 -1
name rootfs
print
quit
```

This creates a partition for your boot and the rest for your files.  In order for UEFI to read your boot partition it needs to be FAT32.

```
mkfs.fat -F 32 /dev/nvme0N0P1
mkfs.ext4 -T small /dev/nvme0N0P2
```

# Preparing the system for setup

We're going to setup our gentoo system by using `chroot` to change the root of the local file system to our new drive mounted at a subdirectory `/mnt/gentoo`

```
mount /dev/nvme0N0P2 /mnt/gentoo`
```

We also need to map some of the Linux special directories to this subdirectory

```
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount /dev/nvme0N0P1 /mnt/gentoo/boot
```

# Get basic Gentoo tools

We need to get the `stage3` bundle of tools

```
cd /mnt/gentoo
links http://gentoo.cs.utah.edu/releases/amd64/autobuilds/current-stage3-amd64/
```

Find the latest `stage3-amd64-<date>.tar.xz` and download.

Unzip stage3 using this command to preserve file settings:

```
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```

# Environment settings

In case your network has special DNS resolution configs

```
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
```

Setup the mirrors you want to use

```
mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf
```
Change the make config to use modern core i7 with a larger number of threads used in `make`

```
nano -w /mnt/gentoo/etc/portage/make.conf
```

```Makefile
# Compiler flags to set for all languages
COMMON_FLAGS="-march=corei7-avx -O2 -pipe"
# Use the same settings for both variables
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"

MAKEOPTS="-j8"
```

# Chroot into our environment

```
chroot /mnt/gentoo /bin/bash
```

We are now in our environment of our hard drive, everything from here on out will affect your final system!

# Update stage3 tools

Before we get going, let's update our existing tools

```
source /etc/profile
emerge-webrsync
```

# Setup wireless

```
# extra steps
modules="wpa_supplicant" 
wpa_supplicant_wlan0="-Dnl80211"

# static wifi
config_wlan0="192.168.0.20/24 brd 192.168.0.255"
routes_wlan0="default via 192.168.0.1."

# for dhcp, but dhcp is the default  
config_wlan0="dhcpcd"
```