# arch linux installation
## Environment
**vm** : vmware workstation 12 pro
**ISO** : arch linux 2017.03.01-dual
**Guest Operation System (vm)** : Other Linux 3.x kernel 64-bit
**Name** : Arch
**Processors** : 1
**cores per processor** : 1
**Memory** : 2048MB
**Network connection** : NAT
**SCSI Controller** : LSI Logic
**Virtual disk type** : SCSI
**Storage Size** : 20GB
**check** : boot with efi without bios

## Partition
### Layout
|partition|size|note|
|:------:|:------:|:------:|
|boot|512M||
|swap|50M||
|LVM|5G||
|/|14.5G||


### MBR
利用fdisk製作出mbr分割表 
```shell
fdisk /dev/sda
```
切出給boot分區    #Mission1
```shell
# create /boot
n 
p
1
\n
+1G # set size
a # enable boot
```
將boot分區的hex標為efi啟動(<mark>重要</mark>)
```shell
t
ef 
```
分割swap分區
```
n
p
2
\n
+50M
```
分割lvm分區 #Mission2
```
n
p
3
\n
+10G
```
分割root分區
```
n
p
\n
\n
w
```

### LVM
將sda3製作成lvm，並且分割出home與var。 #Mission 2
```shell
# Create Physical Volume
pvcreate /dev/sda3

# Create Volume group
vgcreate vg1 /dev/sda3

# Create Logical Volumes
lvcreate -L 4G vg1 -n lvolhome
lvcreate -L 2G vg1 -n lvolvar
```

```
4g
2g
```

### Mount volumes
將硬碟掛載，並將var的10%設保護。 #Misson1, 2, 3
```shell
# mount root
mkfs.ext4 /dev/sda4
mount /dev/sda4 /mnt

# mount boot
cd /mnt
mkdir boot
mkfs.vfat /dev/sda1
mount /dev/sda1 /mnt/boot

# mount home
mkdir home
mkfs.ext4 /dev/mapper/vg1-lvolhome
mount /dev/mapper/vg1-lvolhome /mnt/home

#mount var
mkdir var
mkfs.ext4 /dev/mapper/vg1-lvolvar
mount /dev/mapper/vg1-lvolvar /mnt/var

# var protect
tune2fs -m 10 /dev/mapper/vg1-lvolvar
```
### Create swap
建立swap，不過最後並沒有enable，因為在虛擬機內，成效不大。#Mission 4
```shell
mkswap /dev/sda2
```

## Install
安裝真正的kernel，並且因為是使用vmware，所以並不用特別設定DHCP，其中會先刷新key，避免為認證之情況。 #Mission 5, 7
```shell
# Refresh Public key list
pacman-key --refresh-keys
# Install base package
pacstrap /mnt base
# Generate the fstab file
# fstab defines how disk partitions, 
# various other block devices, or remote 
# filesystems should be mounted into the filesystem.
genfstab -U /mnt >> /mnt/etc/fstab
# switch in the mounted system
arch-chroot /mnt
# generate locale stuff
locale-gen
# install essential tools
pacman -S vim grub efibootmgr grub-bios

```


## boot setting
我先製作bios booting，主機板先設定成bios開機，做完以下指令後，切換成efi模式，然後再用**live CD**開機，重新掛載，然後arch-root，再使用安裝efi。 #Mission 7
### bios booting setting
```shell
# Install the bios boot loader at the first IDE disk 
grub-install --target=i386-pc --recheck --debug /dev/sda
mkdir -p /boot/grub/locale
cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo \
/boot/grub/locale/en.mo
# Generate grub.cfg
# Which is a setting file for grub
grub-mkconfig -o /boot/grub/grub.cfg
``` 
### Efi booting
\# Mission 8
```shell
# Install GRUB UEFI application to /boot/EFI/arch_grub and its modules 
# to /boot/grub/x86_64-efi:
grub-install --target=x86_64-efi \
--efi-directory=/boot --bootloader-id=grub
# Generate grub.cfg
grub-mkconfig -o /boot/grub/grub.cfg
```
## Resize for home 
\# Mission 9
```shell
# add size
lvresize -L +496M /dev/mapper/vg1-lvolhome
```


