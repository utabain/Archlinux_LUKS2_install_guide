# Arch Linux Full-Disk Encryption Installation Guide
### Set the console keyboard layout
The default console keymap is US. Available layouts can be listed with:
```
# ls /usr/share/kbd/keymaps/**/*.map.gz
```
To set the keyboard layout, pass a corresponding file name to loadkeys(1), omitting path and file extension. For example, to set a German keyboard layout:
```
# loadkeys de-latin1
```
### Connect to the internet
Ethernet should be automatic in the arch iso, if any errors occour be sure to consult the archwiki or community fourm.
#### Connect to wifi (optional):
```
# iwctl
```
The interactive prompt is displayed with the prefix of `[iwd]#`.

Firstly, if you do not know your wireless device name, list all Wi-Fi devices:
```
[iwd]# device list
```
Then, to initiate a scan for networks (note that this command will not output anything):
```
[iwd]# station device scan
```
You can then list all available networks:
```
[iwd]# station device get-networks
```
Finally, to connect to a network:
```
[iwd]# station device connect SSID
```
If a passphrase is required, you will be prompted to enter it. Alternatively, you can supply it as a command line argument:
```
# iwctl --passphrase passphrase station device connect SSID
```
The connection may now be verified with ping,

(Use the `Ctrl Z` shortcut to kill the command once the connection has been confirmed):
```
# ping archlinux.org
```
### Update the system clock
To ensure that the system clock is accurate, please run this command:
```
# timedatectl set-ntp true
```
### Partition the disk
When recognized by the live system, disks are assigned to a block device such as `/dev/sda`, `/dev/nvme0n1` or `/dev/mmcblk0`. To identify these devices, use the
command bellow:
```
# fdisk -l
```
Create EFI System and Linux LUKS partitions:
Number |           Size          | Code |    Type    |
-------|-------------------------|------|------------|
   0   | 512.0 MiB               | EF00 | EFI System |
   1   | Remainder of the device | 8309 | Linux LUKS |

Run the gdisk with the name of the block device you want to use. This example uses /dev/nvme0n1:
```
# gdisk /dev/nvme0n1
```
When you get into your desired block device type in the following commands,

(**THIS WILL IRREVEESIBLY ERASE YOUR DATA, MAKE SURE THAT YOU HAVE ALL IMPORTANT FILES BACKED UP**!):
```
o
n
[Enter]
[Enter]
+512M
ef00
n
[Enter]
[Enter]
[Enter]
8309
w
```
Create the encrypted container on the Linux LUKS partition and set the passphrase:
```
# cryptsetup --hash sha512 --iter-time 5000 --key-size 512 luksFormat /dev/[Your Device]p2
```
Open the container (decrypt it and make available at /dev/mapper/cryptlvm)
```
# cryptsetup luksOpen /dev/[Your device]p3 cryptlvm
```
### Preparing the logical volumes
Create a physical volume on top of the opened LUKS container:
```
# pvcreate /dev/mapper/cryptlvm
```
Create the volume group and add physical volume to it:
```
# vgcreate vg /dev/mapper/cryptlvm
```
Create logical volumes on the volume group for swap and root,
(Swap size is a matter of personal preference, 8GB is just a placeholder):
```
# lvcreate -L 8G vg -n swap
# lvcreate -l 100%FREE vg -n root
```
Create a fileystem for ext4 to `/dev/vg/root` using the mkfs command:
```
# mkfs.ext4 /dev/vg/root
```
Create a filesystem for `/dev/vg/swap` using the mkswap command:
```
# mkswap /dev/vg/swap
```
Create a filesystem for the EFI System partition:
```
# mkfs.fat -F32 /dev/nvme0n1p1
```
Disabling journaling:
```
# tune2fs -O "^has_journal" /dev/vg/root
```
Mount `/dev/[Your Device]p2` to `/mnt/efi`
```
# mount /dev/[Your device]p2 /mnt/efi
```
Mount `/dev/vg/root` to `/mnt`:
```
# mount /dev/vg/root /mnt
```
Activate the swap file:
```
# swapon /dev/vg/swap
```
### Installation
Install necessary packages
```
# pacstrap /mnt base linux linux-firmware mkinitcpio lvm2 vim dhcpcd wpa_supplicant network_manager sbsigntools dracut efibootmgr

git [Your processor manufacturer (x86 based cpu's only)]-ucode

```
### Configure the system (Pre-chroot)

Generate an fstab file:
```
genfstab -U /mnt >> /mnt/etc/fstab
```
(optional) Change relatime option to noatime
| /mnt/etc/fstab |
| -------------  |
| col 3 is       |
