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
Now it's time to prepare the second slot of your YubiKey for the challenge response authentication:
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

# Device to unlock with 'cryptsetup'. If left empty and 'YKFDE_DISK_UUID'
# is enabled this will be set as "/dev/disk/by-uuid/$YKFDE_DISK_UUID".
# Leave empty to use 'cryptdevice' boot parameter.
#YKFDE_LUKS_DEV=""

# Optional flags passed to 'cryptsetup'. Example: "--allow-discards" for TRIM
# support. Leave empty to use 'cryptdevice' boot parameter.
#YKFDE_LUKS_OPTIONS=""

# Number of times to try assemble 'ykfde passphrase' and run 'cryptsetup'.
# Defaults to "5".
#YKFDE_CRYPTSETUP_TRIALS="5"

# Number of seconds to wait for inserting YubiKey, "-1" means 'unlimited'.
# Defaults to "30".
#YKFDE_CHALLENGE_YUBIKEY_INSERT_TIMEOUT="30"

# Number of seconds to wait after successful decryption.
# Defaults to empty, meaning NO wait.
#YKFDE_SLEEP_AFTER_SUCCESSFUL_CRYPTSETUP=""

# Verbose output. It will print all secrets to terminal.
# Use only for debugging.
#DBG="1"
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
Install necessary packages
```
# pacstrap /mnt base linux linux-firmware mkinitcpio lvm2 vim dhcpcd wpa_supplicant network_manager sbsigntools dracut efibootmgr git [Your processor manufacturer (x86 based cpu's only)]-ucode
```
### Configure the system (Pre-chroot)

Generate an fstab file:
```
genfstab -U /mnt >> /mnt/etc/fstab
```
(optional) Change relatime option to noatime
| /mnt/etc/fstab |
| -------------  |
| WIP            |
