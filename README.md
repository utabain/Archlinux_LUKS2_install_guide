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
Zap the disk,
(**WARNING** This will irreversibly delete all data on your disk, please backup any important data):
```
sgdisk --zap-all /dev/nvme0n1
```
Create the partitions using the `sgdisk` command:
Number |           Size          | Code |    Type    |
-------|-------------------------|------|------------|
   1   | 555.0 MiB               | EF00 | EFI System |
   2   | Remainder of the device | 8309 | Linux LUKS |
```
sgdisk --clear \
       --new=1:0:+550MiB --typecode=1:ef00 \
       --new=3:0:0       --typecode=2:8309 /dev/nvme0n1
```
### Yubikey 2fa setup (Pre-chroot)
Update the package database:
```
# pacman -Sy
```
Install required packages:
```
# pacman -S make git yubikey-personalization cryptsetup udisks2 expect
```
Clone the `yubikey-full-disk-encryption repository`:
```
# git clone https://github.com/agherzan/yubikey-full-disk-encryption.git
```
`cd` into the directory created by `git`:
```
# cd yubikey-full-disk-encryption
```
Install using the `make` command:
```
# make install
```
Prepare the second slot of your YubiKey for the challenge response authentication:
```
ykpersonalize -v -2 -ochal-resp -ochal-hmac -ohmac-lt64 -ochal-btn-trig -oserial-api-visible
```
Get the UUID of `/dev/nvme0n1p2`:
```
# partprobe
# ls -l /dev/disk/by-uuid/ | grep nvme0n1p2
```
Edit `/etc/ykfde.conf`:
```
### Configuration for 'yubikey-full-disk-encryption'.
### Remove hash (#) symbol and set non-empty ("") value for chosen options to
### enable them.

### *REQUIRED* ###

# Set to non-empty value to use 'Automatic mode with stored challenge (1FA)'.
#YKFDE_CHALLENGE="insert-password-here"

# Use 'Manual mode with secret challenge (2FA)'.
YKFDE_CHALLENGE_PASSWORD_NEEDED="1"

# YubiKey slot configured for 'HMAC-SHA1 Challenge-Response' mode.
# Possible values are "1" or "2". Defaults to "2".
YKFDE_CHALLENGE_SLOT="2"

### OPTIONAL ###

# UUID of device to unlock with 'cryptsetup'.
# Leave empty to use 'cryptdevice' boot parameter.
#YKFDE_DISK_UUID="UUID of /dev/nvme0n1p2"

# LUKS encrypted volume name after unlocking.
# Leave empty to use 'cryptdevice' boot parameter.
#YKFDE_LUKS_NAME="cryptlvm"
```
Create the LUKS2 encrypted container on the Linux LUKS partition with `ykfde-format`:
```
ykfde-format --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 5000 --type luks2 /dev/nvme0n1p2
```
Unlock the encrypted partition with `ykfde-open`:
```
# ykfde-open -d /dev/nvme0n1p2 -n cryptlvm
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
Install necessary packages:
```
# pacstrap /mnt base linux linux-firmware mkinitcpio lvm2 vim dhcpcd wpa_supplicant network_manager sbsigntools dracut efibootmgr git
```
### Configure the system (Pre-chroot)

Generate an fstab file:
```
genfstab -U /mnt >> /mnt/etc/fstab
```
(optional) Change relatime option to noatime
| /mnt/etc/fstab                                                  |
| --------------------------------------------------------------- |
| # /dev/mapper/vg-root                                           |
| UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx / ext4 rw,noatime 0 1 |

Copy the `ykfde.conf` file for later use:
```
# cp /etc/ykfde.conf /mnt/
```
### Change root into the new system
Use the `chroot` command to acomplish the above:
```
# arch-chroot /mnt
```
At this point you should have the following partitions and logical volumes:

('100%FREE' is just a place holder for the remainder of your drive aswell as 'chosen ammount' being a place holder for the ammount of swap space you dedicated to the swap partition)

`lsblk`
NAME           | MAJ:MIN | RM  |  SIZE          | RO  | TYPE  | MOUNTPOINT |
---------------|---------|-----|----------------|-----|-------|------------|
nvme0n1        |  259:0  |  0  | 465.8G         |  0  | disk  |            |
├─nvme0n1p1    |  259:3  |  0  | 512M           |  0  | part  | /efi       |
├─nvme0n1p2    |  259:4  |  0  | 100%FREE       |  0  | part  |            |
..└─cryptlvm   |  254:0  |  0  | 100%FREE       |  0  | crypt |            |
....├─vg-swap  |  254:1  |  0  | chosen ammount |  0  | lvm   | [SWAP]     |
....├─vg-root  |  254:2  |  0  | 100%FREE       |  0  | lvm   | /          |

### Configuring the system (Post-chroot)
#### Timezone
To configure the timezone correctly replace `Europe/London` with your respective timezone found in `/usr/share/zoneinfo`
```
ln -sf /usr/share/zoneinfo/Europe/:ondon /etc/localtime
```
Run `hwclock` to generate `/etc/adjtime`:

(Assumes hardware clock is set to UTC)
```
# hwclock --systohc
```
#### Localization
Uncomment `en_GB.UTF-8 UTF-8` in `/etc/locale.gen` and generate the locale,

('en_GB.UTF-8 UTF-8' is just a placeholder, replace with your respective locale):
```
# locale-gen
```
The default console keymap is US. Available layouts can be listed with:
```
# ls /usr/share/kbd/keymaps/**/*.map.gz
```
If you set the console keyboard layout, make the changes persistent in `vconsole.conf`,

('KEYMAP=us' is just a placeholder see command above to view all avalible layouts):
```
echo KEYMAP=us > /etc/vconsole.conf
```
### Network configuration
Create the hostname file,

('myhostname' is just a placeholder, rename it to anything that you'd like):
```
echo myhostname > /etc/hostname
```
Create the hostname file:
`/etc/hosts`
```
127.0.0.1 localhost
::1 localhost
127.0.1.1 myhostname.localdomain myhostname
```
This is a unique name for identifying your machine on a network.
### Initramfs
Add the `keyboard`, `ykfde`, `consolefont`, `keymap` and `lvm2` hooks to `/etc/mkinitcpio.conf`,

(ordering matters):
```
HOOKS=(base udev autodetect consolefont modconf block keymap lvm2 filesystems fsck keyboard ykfde)
```
Additionally, the ext4 module is needed. Add ext4 to the MODULES:
```
MODULES=(ext4)
```
Recreate the initramfs image:
```
mkinitcpio -p linux
```
### Install microcode
> Processor manufacturers release stability and security updates to the processor microcode. These updates provide bug fixes that can be critical to the stability of your system. Without them, you may experience spurious crashes or unexpected system halts that can be difficult to track down.
> 
> All users with an AMD or Intel CPU should install the microcode updates to ensure system stability.
```
pacman -S intel-ucode
```
### Install and configure systemd-boot
Use `bootctl` to install systemd-boot into the EFI system partition:
```
# bootctl --path=/boot install
```
Create a loader entry `/boot/loader/entries/arch.conf` of your Arch Linux installation:
```
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img # only if you are using an Intel CPU
initrd /initramfs-linux.img
options root=/dev/mapper/root
```
Edit `/boot/loader/loader.conf` to look like this:
```
default arch
timeout 4
console-mode max
editor no
```
### Install secure boot
Install `sbctl`:
```
pacman -S sbctl
```
Check the status of `sbctl`:

`sbctl status`
```
Installed:    Sbctl is not installed
Setup Mode:   Enabled
Secure Boot:  Disabled
```
Verify file database and EFI images in `/efi`:

`# sbctl verify`
```
Verifying file database and EFI images in /efi...
✘ /boot/vmlinuz-linux is not signed
✘ /efi/EFI/BOOT/BOOTX64.EFI is not signed
✘ /efi/EFI/BOOT/KeyTool-signed.efi is not signed
✘ /efi/EFI/Linux/linux-linux.efi is not signed
✘ /efi/EFI/arch/fwupdx64.efi is not signed
✘ /efi/EFI/systemd/systemd-bootx64.efi is not signed
```
Sign the required files:

`# sbctl sign -s /efi/EFI/BOOT/BOOTX64.EFI`
```
✔ Signed /efi/EFI/BOOT/BOOTX64.EFI...
```
`# sbctl sign -s /efi/EFI/arch/fwupdx64.efi`
```
✔ Signed /efi/EFI/arch/fwupdx64.efi...
```
`# sbctl sign -s /efi/EFI/systemd/systemd-bootx64.efi`
```
✔ Signed /efi/EFI/systemd/systemd-bootx64.efi...
```
`# sbctl sign -s /usr/lib/fwupd/efi/fwupdx64.efi -o /usr/lib/fwupd/efi/fwupdx64.efi.signed`
```
✔ Signed /usr/lib/fwupd/efi/fwupdx64.efi...
```

Verify that you have signed the required packages:

`# sbctl verify`
```
Verifying file database and EFI images in /efi...
✔ /usr/lib/fwupd/efi/fwupdx64.efi.signed is signed
✔ /efi/EFI/BOOT/BOOTX64.EFI is signed
✔ /efi/EFI/arch/fwupdx64.efi is signed
✔ /efi/EFI/systemd/systemd-bootx64.efi is signed
✘ /boot/vmlinuz-linux is not signed
✘ /efi/EFI/BOOT/KeyTool-signed.efi is not signed
✘ /efi/EFI/Linux/linux-linux.efi is not signed
```
`# sbctl list-files`
```
/boot/vmlinuz-linux
Signed:		✘ Not Signed

/efi/EFI/BOOT/KeyTool-signed.efi
Signed:		✘ Not Signed

/efi/EFI/Linux/linux-linux.efi
Signed:		✘ Not Signed

/efi/EFI/arch/fwupdx64.efi
Signed:		✔ Signed

/efi/EFI/BOOT/BOOTX64.EFI
Signed:		✔ Signed

/usr/lib/fwupd/efi/fwupdx64.efi
Signed:		✔ Signed
Output File:	/usr/lib/fwupd/efi/fwupdx64.efi.signed

/efi/EFI/systemd/systemd-bootx64.efi
Signed:		✔ Signed
```
