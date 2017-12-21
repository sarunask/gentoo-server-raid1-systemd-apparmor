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
