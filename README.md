MontaVista Installer Program
============================

This program is designed to install a yocto build onto a disk or set
of disks.  It is the main part of a three part section, the other
parts being uipartition, which is a partitioning program, and
mvmkramdisk, which is a ramdisk creation program.

You will need to create an image with this installer on it and start
the installer somehow.  It will need to have a file named
`/to_install.tar.bz2` that gets installed as the root filesystem on
the target.  You would generally create a .wic filesystem from yocto,
or an ISO filesystem, but the details for that are not included here.

Depending on what you use from the installer, installed image will
need to have grub, grub-efi, efibootmgr, mvmkramdisk, mdadm, lvm2 and
perhaps other things.

mvmkramdisk is optional.  If it is on the installed system, the
installer will use it to create a ramdisk that the boot system will
use.  This is required to use RAID, LVM, or if the disk drivers are
modules.  If you don't have mvmkramdisk, you can only have normal disk
devices (no RAID or LVM) and you must compile support for all the disk
drivers into the kernel.

The kernel for the installer will probably need to have the disk
driver support built in to the kernel, but it can be modules in the
installed system ad mvmkramdisk

Using the Installer
-------------------

When the installer comes up, you will get:

    Welcome to the MontaVista Installer!  This installer will allow
	you to partition your disk and install a generated MontaVista Linux
	image onto your machine.
	What would you like to do? (EFI system)
	 1 - Partition your hard drive(s)
	 2 - Install image and GRUB on the system 
	 3 - Recovery sub-menu
	 4 - Toggle swap on autoinstall (False)
	 5 - Auto-install
	 9 - Restart the installer
	 0 - Reboot
	Press the key then enter: 

It knows if your system is EFI or not and will adjust automatically.
You can partition your drive with the partitioner and tell where you
want the various partitions mounted.  Once you have a partitioned
system, you can install the image an grub on it.

You can also auto-install the system.  This will scan the system for
boot devices.  If it finds one, it partitions it as a `/boot` device
and an LVM device, then creates a root logical volume on LVM for the
`/` device.  If it finds two, it creates two partitions on each, and
RAIDs each partition together.  One is `/boot`, the other is an LVM
again used to create a device for `/`.

For EFI systems and RAID, /boot is not RAID-ed.  There will be two
individual /boot partitions, mounted on `/boot/efi` and `/boot/efi2`,
but only `/boot/efi` is set up.  This is because EFI requires a vfat
filesystem and it can't be on RAID.

If you enable swap on autoinstall, the autoinstall process will also
create a swap filesystem twice the size of main memory.

The recovery menu is:

     1 - Set up from an existing root partition
	 2 - Re-install grub
	 3 - Run a shell in the installed system
	 4 - Run a shell in the install host's root directory
	 5 - Set the installed system's hostname
	 0 - Return to main menu
	Press the key then enter:

You can load the partition table and setup from an existing partition
created with this installer, then modify the partition table,
re-install, redo grub on it, etc.

Once you have set up from an existing partition, you can also run a
shell in a chroot on the target, useful for some recovery.  You can
also run a shell in the installer's filesystem.  The installed system
will be mounted on `/t` if you have set it up.

Testing the Installer With QEMU
-------------------------------

qemu is extremely handy for testing, it can test almost any
configuration of hardware for installation.  For these examples, we
will have the installer image
`installer-image-x86-generic-64.rootfs.wic` along with two qemu qcow2
images created with:

    qemu-img create -f qcow2 tmp1.qcow2 20G
	qemu-img create -f qcow2 tmp2.qcow2 20G

This installer file is a wic file, meaning it is a raw disk image with
partitions and everything.  The image specification for that is:

	installer-image-x86-generic-64.rootfs.wic,format=raw
	
The qcow images image specification is:

	tmp1.qcow2,format=qcow2

To mount a normal SCSI/SATA drive, you would do

	-drive file=<image>
	
There are lots of ways to add drives in various configuration, the
above is the simplest.  If you have multiple drives, they will
generally boot in the order you put them in unless you override with
`bootindex`, as discussed later.

The basic system for testing this just sets up a machine and a network
connections and a serial port.  You can remove the `-nographic` if you
want to use a regular console that will come up as a new X window:

    qemu-system-x86_64 --machine q35 -m 1G -enable-kvm -nographic \
       -net nic,model=virtio,macaddr=52:54:00:12:34:61 \
	   -net user,hostfwd=tcp::5561-10.0.2.15:22 \
	   
To enable EFI, on Ubuntu you would install the ovmf package and add:

    -drive if=pflash,format=raw,readonly=on,file=/usr/share/qemu/OVMF.fd

though this will vary depending on the system.  For USB drives, you
can attach them with:

	-drive if=none,file=<image>,id=usbstick \
	-device usb-ehci,id=ehci \
	-device usb-storage,bus=ehci.0,drive=usbstick,bootindex=1 \
	   
The `bootindex` controls the boot order and can be added to any device
except the ide one.  To add an NVME drive, do:

	-drive if=none,file=<image>,id=nvm1 \
	-device nvme,serial=deafbeef,drive=nvm1,bootindex=2 \

As long as you create separate image files for them, you can create as
many drives as you like and attach them.
