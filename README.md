# Description how to install Gentoo Linux with RAID1 root partition and UEFI and Systemd and AppArmor in VirtualBox

## Partitioning Scheme

|Partition|Filesystem|Size|Description    |
|---------|----------|----|---------------|
|sd*1     | (bootloader)| 2M|BIOS boot partition |
|sd*2|fat32 as UEFI is being used|128M|Boot/EFI system partition|
|sd*3|(swap)|512M|Swap partition|
|sd*4|JFS|Rest of the disk|Root FS|

## Partitioning

Would be using Gentoo LiveCD. Would start with first disk partitioning.
```sh
parted -a optimal /dev/sda
mklabel gpt
unit mib
mkpart primary 1 3
name 1 grub
set 1 bios_grub on
mkpart primary 3 131
name 2 boot
set 2 boot on
mkpart primary 131 643
name 3 swap
mkpart primary 643 -1
name 4 rootfs
print
```
You should see something like this:
```
Model: ATA VBOX HARDDISK (scsi)
Disk /dev/sda: 8192MiB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start    End      Size     File system  Name    Flags
 1      1.00MiB  3.00MiB  2.00MiB               grub    bios_grub
 2      3.00MiB  131MiB   128MiB                boot    boot, esp
 3      131MiB   643MiB   512MiB                swap
 4      643MiB   8191MiB  7548MiB               rootfs
```
Now copy GPT partition to /dev/sdb
```
sgdisk /dev/sda -R /dev/sdb
partprobe /dev/sdb
sgdisk -G /dev/sdb
```

## Create RAID1
```
mdadm --create --verbose /dev/md0 --level=mirror --raid-devices=2 --metadata=1.0 --name="boot_raid1" /dev/sda2 /dev/sdb2
mdadm --create --verbose /dev/md1 --level=mirror --raid-devices=2 --metadata=1.2 --name="swap_raid1" /dev/sda3 /dev/sdb3
mdadm --create --verbose /dev/md2 --level=mirror --raid-devices=2 --metadata=1.2 --name="rootfs_raid1" /dev/sda4 /dev/sdb4
```
Now you could check that all your RAID devices are done correctly
```
watch cat /proc/mdstat
```
If you would need to remove array for some reason:
```
mdadm --stop /dev/md0
mdadm --remove /dev/md0
```
To get more details:
```
mdadm --detail /dev/md0
```
If you need to replace failed disk, example
```
mdadm --add /dev/md0 /dev/sdb2
```
Generate /etc/mdadm.conf from current config
```
mdadm --detail --scan >> /etc/mdadm.conf
```


## Create file systems and install Gentoo

```bash
mkfs.vfat -n boot /dev/md0
mkswap -L swap /dev/md1
mkfs.jfs -L rootfs /dev/md2
swapon /dev/md1
mount /dev/md2 /mnt/gentoo
mkdir /mnt/gentoo/boot
mount /dev/md0 /mnt/gentoo/boot/
cp /etc/mdadm.conf /mnt/gentoo/etc/
```

**NOTE: Please edit /etc/mdadm.conf and remove 'name=xxxxxx:xxxxx' parameter in array config, otherwise you would not be able to boot** 

Follow [Gentoo Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Stage) and use stage3-amd64-hardened+nomultilib till you install stage3 and see Systemd mentioned.
After that point start following [Systemd Handbook](https://wiki.gentoo.org/wiki/Systemd).

**NOTE: Configuring Gentoo-sources you should use recommended settings from [here](http://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project/Recommended_Settings#x86_64_2) to enhance security.** 

## Some static binaries
You would need some static utils, so before emerge make use of those flags:
```bash
echo "sys-fs/jfsutils static" >> /etc/portage/package.use/system
echo "sys-apps/busybox static" >> /etc/portage/package.use/system
echo "sys-fs/mdadm static" >> /etc/portage/package.use/system
echo "sys-apps/util-linux static-libs" >> /etc/portage/package.use/system
```

## Enable SystemD and AppArmor

```bash
emerge -1 gentoolkit
euse -E systemd
euse -E -ipv6
euse -E apparmor
mkdir /etc/portage/profile
echo "-apparmor" >> /etc/portage/profile/use.mask
emerge -DuavN @world
emerge apparmor apparmor-utils
```

## Generate own initfs image

```bash
mkdir /usr/src/initramfs
```

You would need to emerge utils in such order as used below.
Note: jfsutils would fail build statically without static-libs USE flag on util-linux
```bash
emerge -1 sys-apps/util-linux
emerge -1 jfsutils mdadm busybox
```
Copy initramfs files to /usr/src/initramfs
Generate initramfs:
```
cd /usr/src/linux
make -C /usr/src/linux/usr/ gen_init_cpio
./scripts/gen_initramfs_list.sh -o /boot/initrd.cpio.gz /usr/src/initramfs/initramfs_list
```

## Install Grub:2
```
echo 'GRUB_PLATFORMS="efi-64"' >> /etc/portage/make.conf
emerge -av grub:2
grub-install --target=x86_64-efi --efi-directory=/boot --removable
grub-mkconfig -o /boot/grub/grub.cfg
```

## Final touches before reboot
```
passwd
useradd -m -g users,wheel some_user
passwd some_user
```

# Useful links
Mdadm:
1. http://www.ducea.com/2009/03/08/mdadm-cheat-sheet/
1. https://ubuntuforums.org/showthread.php?t=1947275
1. https://ubuntuforums.org/showthread.php?t=1950154
1. https://www.howtoforge.com/replacing_hard_disks_in_a_raid1_array

SystemD:
1. https://wiki.gentoo.org/wiki/Systemd

Kernel Protection (as GRSecurity is not OS now):
1. http://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project/Recommended_Settings
