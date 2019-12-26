Title: Gentoo Encrypted Root File System
Date: 2016-07-27T22:41-07:00
Modified: 2016-11-19T15:56-08:00
Category: blog
Tags: linux,gentoo
Authors: Robert Joslyn
Summary: Installing Gentoo with an Encrypted Root File System

This post describes how to setup a Gentoo GNU/Linux system with an encrypted
file system. The root file system, along with the kernel, are encrypted. This
leaves the bootloader as the only unencrypted component. The resulting system
has the following features:

    - x86-64 architecture with multilib support
    - Use EFI firmware to boot
    - Use a GPT partition table
    - No swap partition. For my purposes, I do not need a separate swap
      partition.
    - dm-crypt/LUKS encrypted root partition, including the kernel. Every part
      of the system is encrypted except for the bootloader.

# Boot a Live CD

To start the process, the system must be booted using a Live CD or similar. The
present Gentoo minimal install disc does not support EFI booting, but two
alternatives are:

    - [SystemRescueCd](http://system-rescue-cd.org), a Gentoo-based system
      frequently used for system recoery. This boots to a CLI.
    - The normal [Ubuntu](https://ubuntu.com) installer, which boots to a
      graphical environment. This is useful for easy access to a web browser.

Using the Live CD of your choice, boot the system and open a terminal.

# Disk Setup

Once booted into a Linux environment, the disk can be partitioned. For this
setup, three partitions are used:

    - GRUB Partition
    - EFI System Partition
    - Root Partition

For the rest of this document, the instalation drive is assumed to be
`/dev/sda`. This may be different depending on the system, especially if there
are multiple drives. The `lsblk` command can help determine the correct device
to use.

## Partitioning

Start `parted` and direct it to the desired disk.

    parted -a optimal /dev/sda

To see any existing partitions, use the `print` command. Create a new GPT
partition label. In this case, a GPT label is used.


    mklabel gpt

Set the units displayed to Mibibytes rather than sectors. This makes partition
sizes easier to understand.

    unit mib

Create the GRUB partition with a 2 MiB size.

    mkpart primary 1 3

Set a name for the GRUB partition.

    name 1 grub

Turn on the `bios_grub` flag for the GRUB partition.

    set 1 bios_grub on

Next, create the EFI system partition with a size of 128 MiB.

    mkpart primary 3 131

Set a name for the EFI system partition.

    name 2 efi

Finally, create the root partition. This partition uses the rest of the disk.

    mkpart primary 131 -1

Set a name for the root partition.

    name 3 rootfs

## Create Encrypted Device

With the partitions created, a LUKS encrypted container can be created inside
the root partition created above.

    $ cryptsetup --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 5000 --use-random --verify-passphrase --verbose luksFormat /dev/sda3

The following options are used:

    - `--cipher aes-xts-plain64` selects AES encryption in XTS mode using the
      plain64 initialization vector.
    - `--key-size 512` uses a 512 bit encryption key. Because XTS mode
      effectivly halves the key size, this results in AES-256 encryption. To
      use faster AES-128 encryption, set the key size to 256.
    - `--hash sha512` uses the SHA-512 hash algorithm for the encryption keys.
    - `--iter-time 5000` sets the number of milliseconds to spend on PBKDF2
      password processing. Increasing this value makes it more difficult to
      brute-force the password, but adds a time delay between password
      attempts. A setting of 5,000 milliseconds means it takes 5 seconds for
      the device to unlock after entering the password. This should be set to
      be as long as is tolerable.
    - `--use-random` forces the use of `/dev/random` as the entropy source.
      This is generally considered more secure than using `/dev/urandom`, as
      it ensures high quality random data. The downside is that using
      `/dev/random` can take significantally longer, as once the entropy pool
      is depleated, the system will wait until there is more entropy. When
      using `/dev/urandom`, the system will generate pseudo-random data once
      the entropy pool is depleated.
    - `--verify-passphrase` forces the user to enter the new passphrase twice
      to confirm that it is correct.
    - `--verbose` produces additional output.
    - `luksFormat` specifies to use a LUKS formatted container.
    - `/dev/sda3` is the partition to use.

This may take a long time to run. Once it completes, the new encrypted
container can be opened. This command opens the container and names it "root".
It will prompt for the password used when creating the container.

    $ cryptsetup open /dev/sda3 root

When this is complete, a new block device will appear at `/dev/mapper/root`.
This device can now be used to install Gentoo.

## Create File Systems

With the new block device available at `/dev/mapper/root`, it can be formatted
with the file system of choice. In this example, the btrfs file sytem is used
and gives the partition a label of "rootfs". The label is optional and not
required.

    $ mkfs.btrfs -L rootfs /dev/mapper/root

Mount the root file system. If `/mnt/gentoo` does not exist, create it first.

    $ mkdir /mnt/gentoo
    $ mount -t btrfs -o compress /dev/mapper/root /mnt/gentoo

Create a FAT32 file system on the EFI system partition. This is where the
bootloader is stored and is not encrypted. This example gives the partition
a name of "ESP" (short for EFI system partition).

    $ mkfs.vfat -F 32 -n ESP /dev/sda2

Create the EFI directory and mount the EFI system partition.

    $ mkdir -p /mnt/gentoo/boot/efi
    $ mount /dev/sda2 /mnt/gentoo/boot/efi

# Gentoo Setup

With the disks configured, Gentoo can mostly be installed as usual. These
instructions mirror those found in the
[Gentoo Handbook](https://wiki.gentoo.org/wiki/Handbook:Main_Page).

## Prepare Directories

Change to the `/mnt/gentoo` directory.

    $ cd /mnt/gentoo

Download the latest stage 3 tarball to the `/mnt/gentoo` directory. For a list
of mirrors, see the [Gentoo Mirror List](https://gentoo.org/downloads/mirrors).
Select a mirror and download the latest stage 3 tarball located in the
`releases/amd64/autobuilds` directory on the mirror.

    $ wget ............................

Extract the stage 3 tarball.

    $ tar xvjpf stage3-*.tar.bz2 --xattrs

Modify the `/mnt/gentoo/etc/portage/make.conf` file as desired. This is a good
time to set variables such as CFLAGS, INPUT\_DEVICES, L10N, LINGUAS, MAKEOPTS,
or VIDEO\_CARDS. Since this is an EFI system, set `GRUB_PLATFORMS="efi-64"`
within `make.conf` as well.

    $ nano -w /mnt/gentoo/etc/portage/make.conf

Create the `repos.conf` file and modify as desired.

    $ mkdir /mnt/gentoo/etc/portage/repos.conf
    $ cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
    $ nano -w /mnt/gentoo/etc/portage/repos.conf/gentoo.conf

Copy the `resolv.conf` file into the new system to ensure the network is
accessible after running `chroot`.

    cp -L /etc/resolv.conf /mnt/gentoo/etc/

Mount necessary file systems and enter the new environment.

    $ mount -t proc proc /mnt/gentoo/proc
    $ mount --rbind /sys /mnt/gentoo/sys
    $ mount --rbind /dev /mnt/gentoo/dev
    $ chroot /mnt/gentoo /bin/bash
    $ source /etc/profile
    $ export PS1="(chroot) $PS1"

## Configure the New System

Within the new environment, the rest of the system can be configured. First,
update the portage tree.

    $ emerge-webrsync

Set the timezone. Change "America/Los\_Angeles" as desired. Possible values can
be found in the "TZ" column in this list on
[Wikipedia](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).

    $ echo "America/Los_Angeles" > /etc/timezone
    $ emerge --config sys-libs/timezone-data

Adjust the locale settings as desired and regenerate the locales.

    $ nano -w /etc/locale

Select the desied locale with eselect.

    $ eselect locale list
      [1]   C
      [2]   en_US
      [3]   en_US.iso88591
      [4]   en_US.utf8
      [5]   POSIX
      [ ]   (free form)
    $ select locale set 4
    $ env-update

### Configure The initramfs

The kernel alone is not able to decrypt the root file system, so an initramfs
is required. The initramfs is a minimal file system that contains the tools
required in order to unlock the full root file system and boot the system.
There are many ways to create an initramfs, but the method shown here is
called Early Userspace. Technically, the modern Linux kernel always uses an
initramfs, but it defaults to being empty. If the initramfs is empty, the
kernel reverts back to older methods of finding the root file sysetm and
executing `/sbin/init`. It is also possible (and perhaps more common) to create
an external initramfs, but that will not be covered here. In this section, the
built-in initramfs is populated with the tools required to unlock the encrypted
root file system.

First, install `cryptsetup`, which provides the tools needed to decrypt the
root file system. The `cryptsetup` binary needs to be included in the
initramfs to decrypt the disk.

    $ emerge -av cryptsetup

Create a directory to contain the initramfs configuration files. The exact
location does not matter, but this document uses `/usr/src/initramfs`.

    $ mkdir /usr/src/initramfs

If you use a keyboard layout other than the en-US QWERTY layout, generate a
keymap binary to load in the initramfs. Here, the Dvorak layout is used.

    $ loadkeys -b dvorak > /usr/src/initramfs/dvorak.keymap

Create an init script that initializes the system inside the initramfs.

    $ nano -w /usr/src/initramfs/init

The primary purpose of the init script is to mount necessary file systems
(such as `/dev`, `/proc`, and `/sys`), unlock the root file system, and then
continue the boot process by switching to the "real" root file system. The
following example does this, along with a few convience features such as
setting the keyboard layout.

    #!/bin/busybox sh

    rescue_shell() {
    	printf "%s\n" "$@"
	printf "%s\n" "Something went wrong, dropping to a shell."
	exec /bin/sh
    }

    # Mount dev, proc, and sys
    mount -t devtmpfs devtmpfs /dev
    mount -t proc proc /proc
    mount -t sysfs sysfs /sys
    # Mount devpts
    mkdir /dev/pts
    mount -t devpts devpts /dev/pts

    # Install BusyBox symlinks. This is required in order to create the
    # symlinks for the various tools provided by BusyBox.
    busybox --install -s

    # Load keymap.
    loadkmap < /root/dvorak.keymap || rescue_shell "Loading keymap failed"

    # Disable kernel messages from writing to the screen
    printf "0\n" > /proc/sys/kernel/printk

    # Unlock root file system.
    cryptsetup open /dev/sda3 root || rescue_shell "cryptsetup failed"

    # Mount unlocked device as read-only.
    mount -t btrfs -o compress,ro /dev/mapper/root /mnt/root || rescue_shell "Mounting rootfs failed"

    # Done with these, so unmount them.
    umount /dev
    umount /proc
    umount /sys

    # Switch to the newly mounted root file system.
    exec switch_root /mnt/root /sbin/init

Create a configuration file describing what needs to be added to the initramfs.
The following example shows the bare essentials, along with providing a custom
keyboard layout. For additional information on the format of this file, see the
kernel documentation for
[Early Userspace](https://kernel.org/doc/Documentation/early-userspace/README)
and [initramfs](https://kernel.org/doc/Documentation/filesystems/ramfs-rootfs-initramfs.txt).

    $ nano -w /usr/src/initramfs/initramfs-list

Example configuration:

    # Main directory structure
    dir /bin	755 0 0
    dir /dev	755 0 0
    dir /etc	755 0 0
    dir /lib	755 0 0
    dir /lib32	755 0 0
    dir /lib64	755 0 0
    dir /mnt	755 0 0
    dir /mnt/root	755 0 0
    dir /proc	755 0 0
    dir /root	700 0 0
    dir /sbin	755 0 0
    dir /sys	755 0 0
    dir /usr	755 0 0

    # Main init script
    file /init	/usr/src/initramfs/init	755 0 0
    # Node required for the console. This is required to get a shell in the initramfs.
    nod /dev/console	644 0 0 c 5 1
    # Keyboard layout (optional)
    file /root/dvorak.keymap	/usr/src/initramfs/dvorak.keymap	755 0 0

    # BusyBox
    slink /bin/sh	/bin/busybox	777 0 0
    file /bin/busybox	/bin/busybox	755 0 0

    # Libraries roquired by cryptsetup (use ldd to discover)
    dir /usr/lib64	755 0 0
    file /sbin/cryptsetup           /sbin/cryptsetup        755 0 0
    dir	/lib64/ld-linux-x86-64.so.2    /lib64/ld-linux-x86-64.so.2 755 0 0
    file /lib64/libc.so.6           /lib64/libc.so.6        755 0 0
    file /lib64/libdevmapper.so.1.02    /lib64/libdevmapper.so.1.02 755 0 0
    file /lib64/libpthread.so.0     /lib64/libpthread.so.0      755 0 0
    file /lib64/libudev.so.1        /lib64/libudev.so.1     755 0 0
    file /lib64/libuuid.so.1        /lib64/libuuid.so.1     755 0 0
    file /usr/lib64/libcryptsetup.so.4  /usr/lib64/libcryptsetup.so.4   755 0 0
    file /usr/lib64/libgcrypt.so.20     /usr/lib64/libgcrypt.so.20  755 0 0
    file /usr/lib64/libgpg-error.so.0   /usr/lib64/libgcrypt.so.20  755 0 0
    file /usr/lib64/libpopt.so.0        /usr/lib64/libpopt.so.0     755 0 0

Note that in addition to the `cryptsetup` binary, all libraries required by
the binary are also included in the initramfs. These libraries can be found
using `ldd`. If additional utilities are required in the initramfs, they must
be added to this configuration file.

With the initramfs configuration defined, the kernel can be configured.

### Configure the Kernel

Install and configure the Linux kernel.

    $ emerge -av gentoo-sources
    $ cd /usr/src/linux
    $ make menuconfig

Be sure to enable the following options as built-in, not as modules:

    CONFIG_BLK_DEV_INITRD
    CONFIG_BLK_DEV_DM
    CONFIG_DM_CRYPT
    CONFIG_CRYPTO_AES_X86_64
    CONFIG_CRYPTO_SHA512
    CONFIG_CRYPTO_XTS

In order to build the initramfs into the kernel binary, the 
`CONFIG_INITRAMFS_SOURCE` value must be set in the kernel configuration to the
file created previously.

    (/usr/src/initramfs/initramfs-list) Initramfs source file(s)

After configuring the kernel, build and install it.

    $ make
    $ make modules_install
    $ make install

### Configure System Settings

If your system requires firmware blobs, such as those required by many
wireless network adapters, install the `linux-firmware` package.

    $ emerge -av linux-firmware

Configure the `/etc/fstab` file to mount the file system as read-write. Add
any additional file systems that should be mounted automatically once
booted into the full system.

    /dev/mapper/root	/	btrfs	noatime,compress	0 1

Set a hostname in `/etc/conf.d/hostname`.
