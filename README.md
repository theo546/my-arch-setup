<h1 align="center">
	My Arch Linux setup
</h1>

This git depot will teach you how to make my Arch Linux UEFI setup that features encryption, Secure Boot, btrfs and AppArmor.

![The installed KDE Plasma setup](plasma.webp)

## Synopsis

Here is a step by step guide to replicate my Arch Linux setup that includes the instructions to setup:
- A LUKS partition on top of LVM on top of a btrfs partition with two subvolumes.
- An encrypted SWAP partition.
- Secure Boot using sbupdate to prevent the evil maid attack.
- Flatpak for most applications.
- AppArmor on top of Firejail.
- USBGuard to prevent unauthorized USB devices to be plugged on the system.
- Optionally a way to automate decryption using a TPM.

*Note:* most of the informations in this guide were pulled from [this gist](https://gist.github.com/huntrar/e42aee630bee3295b2c671d098c81268), [this Medium post](https://medium.com/@pawitp/full-disk-encryption-on-arch-linux-backed-by-tpm-2-0-c0892cab9704), [this reddit post](https://www.reddit.com/r/archlinux/comments/7np36m/detached_luks_header_full_disk_encryption_with/) and from the [Arch Linux Wiki](https://wiki.archlinux.org).

*Note2:* all the works made on this repo are licensed under the GPLv3, however, the wallpaper shown in the `plasma.webp` image, the `angeldust.webp` image and the three wallpapers in the `wallpapers` folder belong to their respective owners, I do not own or claim ownership of those said images.

*Note3*: I tried to gather as much useful informations as possible to insert them all in this guide, I verified and tested all these steps myself on a VM and my own PC.  
If you spotted an error or if you have any recommendation to make, please open an issue, I'll be glad to take a look.

*Note4:* unless specified with a `$`, run all the commands as root.

# Table of contents

0. [Verifying that everything is OK](#0-verifying-that-everything-is-ok)

1. [Download and verify the Arch ISO](#1-download-and-verify-the-arch-iso)

2. [Boot the system and connect to the internet](#2-boot-the-system-and-connect-to-the-internet)

	a. [BIOS settings](#a-bios-settings)

	b. [Connect to the internet](#b-connect-to-the-internet)

	c. [Synchronize the system date](#c-synchronize-the-system-date)

	d. [(optional) Configure the keyboard layout](#d-optional-configure-the-keyboard-layout)

3. [Make the partitions and the file system](#3-make-the-partitions-and-the-file-system)

	a. [(optional) Erase the disk with random data before doing anything](#a-optional-erase-the-disk-with-random-data-before-doing-anything)

	b. [Create the partitions using gdisk](#b-create-the-partitions-using-gdisk)

	c. [Create the LUKS2 partition](#c-create-the-luks2-partition)

	d. [Create the LVM partition and volume group on top of the LUKS partition](#d-create-the-lvm-partition-and-volume-group-on-top-of-the-luks-partition)

	e. [Create the partitions on the LVM volume group](#e-create-the-partitions-on-the-lvm-volume-group)

	f. [Create the SWAP, btrfs and EFI (FAT32) partitions](#f-create-the-swap-btrfs-and-efi-fat32-partitions)

	g. [And mount them](#g-and-mount-them)

4. [Time to install the system](#4-time-to-install-the-system)

	a. [Pre-Chroot](#a-pre-chroot)

	b. [In the Chroot](#b-in-the-chroot)

5. [On the now working setup](#5-on-the-now-working-setup)

	a. [Flatpak](#a-flatpak)

	b. [USBGuard](#b-usbguard)

	c. [Firejail and AppArmor](#c-firejail-and-apparmor)

	d. [(optional) Set the keyboard layout for x11](#d-optional-set-the-keyboard-layout-for-x11)

	e. [(optional) Enable the numlock at boot for SDDM](#e-optional-enable-the-numlock-at-boot-for-sddm)

	f. [(optional) KDE Plasma customization](#f-optional-kde-plasma-customization)

	g. [(optional) Fix KDE Plasma refresh rate with a 144 Hz screen](#g-optional-fix-kde-plasma-refresh-rate-with-a-144-hz-screen)

	h. [Make sure that the system is syncing to NTP servers](#h-make-sure-that-the-system-is-syncing-to-ntp-servers)

	i. [(optional) Install libvirt](#i-optional-install-libvirt)

6. [Enroll the keys in the BIOS](#6-enroll-the-keys-in-the-bios)

	a. [(optional) Sign the Microsoft keys](#a-optional-sign-the-microsoft-keys)

	b. [Copy the certificates and the KeyTool utility](#b-copy-the-certificates-and-the-keytool-utility)

	c. [Enroll the keys](#c-enroll-the-keys)

	d. [Restart and set a BIOS password](#d-restart-and-set-a-bios-password)

7. [Post-install](#7-post-install)

	a. [Backup your LUKS partition header](#a-backup-your-luks-partition-header)

	b. [Automatically unlock your encrypted LUKS partition using your motherboard TPM](#b-automatically-unlock-your-encrypted-luks-partition-using-your-motherboard-tpm)

- [Self-notes](#self-notes)

- [Final words](#final-words)

# And so it begins...

## 0. Verifying that everything is OK

Verify that your system support UEFI as this setup is only for those systems.

Benchmark your system encryption capabilities using `cryptsetup benchmark`, you should prioritize aes-xts, if the results are satisfying, you may carry on with the guide.  
For this setup, I'll pick aes-xts using a 512 bits key.

## 1. Download and verify the Arch ISO

Find and download the latest Arch ISO on https://archlinux.org/download/  
Verify the ISO checksum using sha1sum:
```
curl -s https://archlinux.org/iso/latest/sha1sums.txt | sha1sum -c --ignore-missing
```

## 2. Boot the system and connect to the internet

### a. BIOS settings
Disable Secure Boot and allow only UEFI systems to boot in your BIOS settings.  
You may also add a BIOS password to prevent modifications of the Secure Boot settings.

### b. Connect to the internet
You can connect your computer to the internet by plugging an ethernet cable or by using the internet sharing feature on your phone.  
You can get more ways to connect to the internet using the [Arch Linux Wiki](https://wiki.archlinux.org/index.php/installation_guide#Connect_to_the_internet).

### c. Synchronize the system date
Allow the system clock to synchronize with some NTP server on the internet using
`timedatectl set-ntp true`.

### d. (optional) Configure the keyboard layout
If you are using an AZERTY keyboard just like me, you will need to run ````loadkeys fr```` to be able to use the AZERTY layout.

## 3. Make the partitions and the file system

For this section, I will stick to this scheme.

Number | Start (sector) | End (sector) |    Size    | Code |        Name         |
-------|----------------|--------------|------------|------|---------------------|
   1   |   2048         |   514047    | 250M   | EF00 | EFI System          |
   2   |   514048       |   41943006  | 19.8G  | 8309 | Linux LUKS          |

### a. (optional) Erase the disk with random data before doing anything
You can use dd:
```
dd if=/dev/urandom of=/dev/sdX bs=4096 status=progress
```

You can also use blkdiscard:
```
blkdiscard /dev/sdX
```

But also shred:
```
shred -n1 -v /dev/sdX
```

*Note:* replace X with your disk letter.

Then sync the now erased partitions (just in case):
```
sync
```

### b. Create the partitions using gdisk
Type `gdisk /dev/sdX` (where X is your disk letter).

Type all these commands in this very specific order to create a 250M EFI partition while using all the remaining disk space for the LUKS partition:

```
o (create a new GUID partition table)
y (confirm the disk erasing)
n (create a new partition)
[Enter] (pick the default option)
0 (try to use the very first available sector)
+250M (make the partition size 250M more than the first selected sector)
ef00 (the code that correspond to the EFI System Partition partition type)
n (create a new partition)
[Enter] (pick the default option)
[Enter] (pick the default option which is the first available sector after the EFI partition)
[Enter] (pick the default option which is the last available sector on the disk)
8309 (the code that correspond to the Linux LUKS partition type)
w (write the changes to the disk)
y (confirm the changes)
```
*Note:* if gdisk ask you about an "invalid MBR" or "corrupt GPT", press `2`.  
*Note2:* from this point, we'll assume that `/dev/sda` is our disk, make the appropriate changes from the commands below if needed.

### c. Create the LUKS2 partition
In this setup, I will use `aes-xts-plain64` (AES-512) as the cipher, sha512 as the hash function, 5000 as the chosen iteration time (higher than the cryptsetup default) and argon2id as the key derivation function.  
Please make your own opinion about those settings, I am not here to talk about cryptography here.

The final command should look like this:
```
cryptsetup luksFormat -c aes-xts-plain64 -h sha512 -S 1 -s 512 -i 5000 --use-random --type luks2 --pbkdf argon2id /dev/sda2
```
*Note:* if you wanna go overkill, change the iteration time from 5000 to 30000.

We are now able to open the LUKS partition using:
```
cryptsetup open --allow-discards /dev/sda2 cryptlvm
```
*Note:* in this guide, the LUKS partition will be referenced as `cryptlvm`.  
*Note2:* remove the `--allow-discards` option if you are not using an SSD or if you do not wish to use TRIM for "security reasons".

### d. Create the LVM partition and volume group on top of the LUKS partition
You can do so by typing:
```
pvcreate /dev/mapper/cryptlvm
```

You can now create a volume group named `vg` by typing:
```
vgcreate vg /dev/mapper/cryptlvm
```

*Note:* In this guide, the LVM volume group will be referenced as `vg`.

### e. Create the partitions on the LVM volume group
In this setup, I will make a 8G SWAP partition named `swap`:
```
lvcreate -L 8G vg -n swap
```
*Note:* if you wish to use hibernation, make your SWAP partition size even or bigger than your system RAM size.

Now a partition that will use all the remaining space named `root`:
```
lvcreate -l 100%FREE vg -n root
```

### f. Create the SWAP, btrfs and EFI (FAT32) partitions
This one doesn't even need any explanation:
```
mkfs.btrfs /dev/vg/root
mkswap /dev/vg/swap
mkfs.fat -F32 /dev/sda1
```

### g. And mount them
But before that, let's save the mount options in a variable so we don't have to type them a thousand time:
```
btrfs_options=rw,noatime,ssd,compress-force=zstd:2,space_cache=v2,discard=async
```
*Note:* if you are not using an SSD, remove the `ssd` and `discard=async` option for this and every commands that are about mounting your partitions, however, you may add the `autodefrag` option.  
*Note2:* please take into consideration that using compression may decrease your system performance, sometimes by *a lot*.  
*Note3:* remove the `discard=async` option if you do not wish to use TRIM for "security reasons".  
*Note4:* you can use [this spreadsheet](https://docs.google.com/spreadsheets/d/1x9-3OQF4ev1fOCrYuYWt1QmxYRmPilw_nLik5H_2_qA/edit#gid=0) to get the best compression ration that you seem good.

Then mount the btrfs pool partition:
```
mount -o $btrfs_options,subvolid=5 /dev/vg/root /mnt
```
Let's create the `@` and `@home` subvolumes:
```
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
```
Umount the btrfs pool:
```
umount /mnt
```

Then mount the subvolumes:
```
mount -o $btrfs_options,subvol=@ /dev/vg/root /mnt
```

Create the `/home`, `/btrfs_pool` and `/efi` directories before mounting the subvolumes:
```
mkdir /mnt/{home,btrfs_pool,efi}
```
Now we can mount them:
```
mount -o $btrfs_options,subvol=@home /dev/vg/root /mnt/home
mount -o $btrfs_options,subvolid=5 /dev/vg/root /mnt/btrfs_pool
```

Set the permissions of the btrfs pool mount directory to 700 for security reasons:
```
chmod 700 /mnt/btrfs_pool
```

Mount the ESP:
```
mount /dev/sda1 /mnt/efi
```
And also activate the SWAP:
```
swapon /dev/vg/swap
```

## 4. Time to install the system

### a. Pre-Chroot
You know the drill:
```
pacstrap /mnt base base-devel linux linux-headers linux-firmware mkinitcpio lvm2 nano dhcpcd wpa_supplicant btrfs-progs sudo efibootmgr efitools git wget
```
*Note:* you can also install a compatible microcode for your system:
`intel-ucode` or `amd-ucode`.

Generate the fstab:
```
genfstab -U /mnt >> /mnt/etc/fstab
```

Time to chroot:
```
arch-chroot /mnt
```

### b. In the Chroot
Change the timezone of your setup:
```
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
```

Synchronize the system time:
```
hwclock --systohc
```

Uncomment `en_US.UTF-8 UTF-8` from `/etc/locale.gen`:
```
sed -i 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/g' /etc/locale.gen
```

You can also uncomment for a language of your choice, ex: `fr_FR.UTF-8 UTF-8`:
```
sed -i 's/# fr_FR.UTF-8 UTF-8/fr_FR.UTF-8 UTF-8/g' /etc/locale.gen
```

Then run:
```
locale-gen
```

Add the wished system language into `/etc/locale.conf`:
```
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```
or:
```
echo "LANG=fr_FR.UTF-8" > /etc/locale.conf
```

Set a variable containing your system hostname:
```
hostname=myhostname
```

Add your system hostname to the hostname file:
```
echo $hostname > /etc/hostname
```
*Note:* In this guide, the system hostname will be referenced as `myhostname`.

Then in the host file:
```
echo -e "127.0.0.1 localhost\n::1 localhost\n127.0.1.1 $hostname.localdomain $hostname" > /etc/hosts
```

Edit the `/etc/mkinitcpio.conf` to add the `crc32c` (or `crc32c-intel`) module and the `keyboard`, `keymap`, `encrypt`, `lvm2` and `resume` hooks.  
Let's first add the `crc32c` or `crc32c-intel` module:
```
MODULES=(crc32c / crc32c-intel)
```
*Note:* obviously, use the `crc32c-intel` module if you are using an Intel CPU that support Intel SSE4.2, otherwise, use the `crc32c` module.

For the hooks, **the order matter**, it should be as follow:
```
HOOKS=(base udev autodetect keyboard keymap modconf block encrypt lvm2 filesystems resume fsck)
```
*Note:* if you do not wish to use the hibernation feature, remove the `resume` hook.  
*Note2:* if you are only using the QWERTY keyboard layout, you can not include the `keymap` hook.

You can now save and exit the file.

(optional) enable the zstd compression for the initramfs:
```
sed -i 's/#COMPRESSION="zstd"/COMPRESSION="zstd"/g' /etc/mkinitcpio.conf
```

Add your keyboard layout in `/etc/vconsole.conf`:
```
echo "KEYMAP=en" > /etc/vconsole.conf
```
Or for an AZERTY keyboard:
```
echo "KEYMAP=fr" > /etc/vconsole.conf
```

Change your system root password:
```
passwd
```

Allow the wheel group to execute any sudo command using:
```
nano /etc/sudoers
```

(optional) install the Intel GPU drivers on the system:
```
pacman -S mesa vulkan-intel
```

(optional) install the NVIDIA drivers on the system:
```
pacman -S nvidia
```

(optional) install the VirtualBox guest utilities on the system:
```
pacman -S virtualbox-guest-utils xf86-video-vmware
```

(optional) install the ATI drivers on the system:
```
pacman -S xf86-video-ati
```

(optional) let the system only SWAP when at least 90% of the RAM is used:
```
echo "vm.swappiness=10" >> /etc/sysctl.d/99-swappiness.conf
```

(optional) prevent coredumps from generating:
```
echo "Storage=none" >> /etc/systemd/coredump.conf
```

(optional) prevent journald from taking more than 250M of storage:
```
sed -i 's/#SystemMaxUse=/SystemMaxUse=250M/g' /etc/systemd/journald.conf
```

(optional) allow the user to force a system reboot after smashing the CtrlAltDel keys 7 times:
```
sed -i 's/#CtrlAltDelBurstAction=reboot-force/CtrlAltDelBurstAction=reboot-force/g' /etc/systemd/system.conf
```

(optional) reduce the timeout of systemd for a service shutdown to 10 secondes:
```
sed -i 's/#DefaultTimeoutStopSec=90s/DefaultTimeoutStopSec=10s/g' /etc/systemd/system.conf
```

(optional if you are doing a dual boot with Windows) set the RTC date to save the time with your timezone:
```
timedatectl set-local-rtc 1 --adjust-system-clock
```

Time to create your user:
```
useradd -m -G wheel -s /bin/bash myuser
passwd myuser
```
*Note:* In this guide, the main user username will be referenced as `myuser`.

Get into your newly created user and cd into your home directory:
```
su myuser
$ cd
```

In this step, we'll install the `yay` AUR helper:
```
$ git clone https://aur.archlinux.org/yay.git
$ cd yay
$ makepkg -si
```

Once yay is installed, remove its install folder:
```
$ cd ..
$ rm -rf yay
```

Ask yay to always loop in case the sudo command would time-out:
```
$ yay --sudoloop --save
```

Now install the `sbupdate-git` package:
```
$ yay -S sbupdate-git
```

Then exit the su mode:
```
$ exit
```

Add the boot options in `sbupdate.conf`, this command will automatically fetch the UUID of the LVM partition, add the boot flags for AppArmor and set the kernel loglevel to 3.
```
sed -i 's/#CMDLINE_DEFAULT=""/CMDLINE_DEFAULT="cryptdevice=UUID='"$(blkid -s UUID -o value /dev/sda2)"':cryptlvm:allow-discards root=\/dev\/vg\/root resume=\/dev\/vg\/swap rootflags=subvol=@ rw loglevel=3 rd.udev.log_priority=3 rd.systemd.show_status=false systemd.show_status=false quiet splash apparmor=1 lsm=lockdown,yama,apparmor"/g' /etc/sbupdate.conf
```
*Note:* if you are using an HDD or if you do not wish to use TRIM for "security reasons", remove the `allow-discards` option.  
*Note2:* if you do not wish to use the hibernation feature, remove the `resume` option.  
*Note3:* add the `i915.fastboot=1` option if you are using the integrated GPU of an Intel CPU.  
*Note4:* add the `nvidia-drm.modeset=1` option if you wish to use Wayland with an NVIDIA graphics card.

Set the ESP dir in the `sbupdate.conf` as `/efi`:
```
sed -i 's/#ESP_DIR="\/boot"/ESP_DIR="\/efi"/g' /etc/sbupdate.conf
```

Set the keys directory for Secure Boot to 
```
sed -i 's/#KEY_DIR="\/etc\/efi-keys"/KEY_DIR="\/root\/secureboot_keys"/g' /etc/sbupdate.conf
```

*Note:* you can also set the `SPLASH` value to match a BMP file you wanna get to show up at the system boot-up.

Create and cd into the key folder for Secure Boot:
```
mkdir /root/secureboot_keys && cd /root/secureboot_keys
```

Generate a random UUID for the certificates and keys:
```
uuidgen --random > GUID.txt
```

Generate the platform key:
```
openssl req -newkey rsa:4096 -nodes -keyout PK.key -new -x509 -sha256 -days 3650 -subj "/CN=my Platform Key/" -out PK.crt
openssl x509 -outform DER -in PK.crt -out PK.cer
cert-to-efi-sig-list -g "$(< GUID.txt)" PK.crt PK.esl
sign-efi-sig-list -g "$(< GUID.txt)" -k PK.key -c PK.crt PK PK.esl PK.auth
```

Generate the key exchange key:
```
openssl req -newkey rsa:4096 -nodes -keyout KEK.key -new -x509 -sha256 -days 3650 -subj "/CN=my Key Exchange Key/" -out KEK.crt
openssl x509 -outform DER -in KEK.crt -out KEK.cer
cert-to-efi-sig-list -g "$(< GUID.txt)" KEK.crt KEK.esl
sign-efi-sig-list -g "$(< GUID.txt)" -k PK.key -c PK.crt KEK KEK.esl KEK.auth
```

Generate the signature database key:
```
openssl req -newkey rsa:4096 -nodes -keyout db.key -new -x509 -sha256 -days 3650 -subj "/CN=my Signature Database key/" -out db.crt
openssl x509 -outform DER -in db.crt -out db.cer
cert-to-efi-sig-list -g "$(< GUID.txt)" db.crt db.esl
sign-efi-sig-list -g "$(< GUID.txt)" -k KEK.key -c KEK.crt db db.esl db.auth
```
*Note:* you may change the common name to whatever you want.

Get the `96-sbupdate-move.hook` file and move it into the `/usr/share/libalpm/hooks` directory:
```
wget -O /usr/share/libalpm/hooks/96-sbupdate-move.hook https://raw.githubusercontent.com/theo546/my-arch-setup/master/96-sbupdate-move.hook
```

Create the `/efi/EFI/BOOT` directory to store the future EFI image and set the `/boot` directory permissions to 700:
```
mkdir -p /efi/EFI/BOOT
chmod 700 /boot
```

Install all the packages I want for my setup:
```
pacman -S plasma plasma-wayland-session kwalletmanager spectacle flatpak nautilus xdg-user-dirs xsettingsd firefox firefox-i18n-fr virtualbox packagekit-qt5 konsole gnome-disk-utility gnome-keyring pavucontrol adapta-gtk-theme materia-gtk-theme papirus-icon-theme xcursor-vanilla-dmz noto-fonts-emoji ttf-dejavu ttf-liberation ttf-droid ttf-ubuntu-font-family noto-fonts networkmanager usbguard firejail apparmor htop bpytop dnscrypt-proxy syncthing jre8-openjdk jre-openjdk ldns gvfs-mtp gocryptfs compsize whois openbsd-netcat net-tools usbutils dnsmasq libcups cups ghostscript avahi xsane earlyoom iw ncdu partitionmanager wireguard-tools bluez
```
It include:
- KDE Plasma
- The Plasma Wayland session package so a Wayland session under Plasma may be started
- Firefox and the French language package
- Firejail, AppArmor and USBGuard
- Flatpak
- VirtualBox
- The Adapta theme, the Materia theme, the Papirus icon pack and the dmz cursor
- A bunch of fonts to avoid the missing font squares
- Nautilus, Spectacle, the amazing GNOME Disks utility and GNOME Keyring
- packagekit-qt5 to be able to update Arch directly from KDE Discover
- htop and bpytop
- dnscrypt-proxy to prevent your ISP from knowing where you're going
- Syncthing to sync files between some devices
- Java 8 and the latest Java release because of **Minecraft**
- The ldns package for the `drill` command
- The gvfs-mtp package so I can access the files of my Android phone
- gocryptfs so the Vault feature of KDE Plasma is available
- compsize for btrfs
- A whois client
- openbsd-netcat to be able to connect to SSH server using a SOCKS proxy
- net-tools for the netstat tool
- usbutils to get informations about connected USB devices
- dnsmasq so the internet connection can be shared through ethernet
- libcups, cups, ghostscript and avahi so you can print with a printer
- xsane so you can scan using a printer
- earlyoom to prevent system freeze when running out of RAM
- The iw CLI tool to manage wireless devices
- The ncdu tool to know where the storage is used
- The KDE Partition Manager tool
- The wireguard-tools package to be able to configurate Wireguard
- bluez for Bluetooth support

*Note:* installing EasyEffects will cause PulseAudio to get replaced by pipewire-pulse which is a working drop-in replacement of PulseAudio.

Enable the `NetworkManager`, `SDDM`, `AppArmor`, `cups`, `avahi-daemon`, `earlyoom` and the `bluetooth` services:
```
systemctl enable NetworkManager sddm apparmor cups avahi-daemon earlyoom bluetooth
```

**It is now time to restart your PC, your setup is able to boot!**

## 5. On the now working setup

### a. Flatpak
Open a terminal, then install some Flatpak applications:
```
flatpak install -y com.bitwarden.desktop com.discordapp.Discord org.signal.Signal com.github.Eloston.UngoogledChromium com.github.micahflee.torbrowser-launcher com.github.tchx84.Flatseal com.spotify.Client org.audacityteam.Audacity org.filezillaproject.Filezilla org.gnome.baobab org.gimp.GIMP org.kde.krita org.libreoffice.LibreOffice org.gnome.Geary org.telegram.desktop org.videolan.VLC com.valvesoftware.Steam com.valvesoftware.Steam.CompatibilityTool.Proton com.valvesoftware.Steam.CompatibilityTool.Proton-GE com.obsproject.Studio org.remmina.Remmina org.mozilla.firefox org.qbittorrent.qBittorrent org.kde.kdenlive com.visualstudio.code-oss org.gtk.Gtk3theme.Adapta-Nokto-Eta org.gtk.Gtk3theme.Materia-dark-compact org.gnome.eog org.gnome.FileRoller io.github.peazip.PeaZip com.github.wwmm.easyeffects org.gnome.seahorse.Application
```

Here are the installed applications:
- Bitwarden
- Discord
- Signal
- ungoogled-chromium
- Tor Browser launcher
- Flatseal
- Spotify
- Audacity
- FileZilla
- Disk Usage Analyzer
- GIMP
- Krita
- LibreOffice
- Geary
- Telegram
- VLC
- Steam
- Steam Proton
- Steam Proton-GE
- OBS Studio
- Remmina
- Firefox
- qBittorrent
- Kdenlive
- Code-OSS
- The Adapta Nokto Eta theme
- The Materia Dark Compact theme
- Eyes of GNOME
- File Roller
- The PeaZip archive manager
- EasyEffects
- Seahorse

You can find more instructions to properly setup some of these applications in their respective folders.

(optional and only if Flatpak applications doesn't respect the applied GTK theme) let's make sure that the Flatpak applications will use the Materia-dark-compact theme:
```
$ echo "xsettingsd &" >> ~/.xinitrc
```

### b. USBGuard
Generate the policy for the currently connected devices:
```
usbguard generate-policy > /etc/usbguard/rules.conf
```

Enable the USBGuard service:
```
systemctl enable usbguard
```

**Important:** you can consult the [Arch Linux Wiki](https://wiki.archlinux.org/index.php/USBGuard) for more instructions.

### c. Firejail and AppArmor
Activate the Firejail profile for AppArmor:
```
apparmor_parser -r /etc/apparmor.d/firejail-default
```

Get the `firejail.hook` file and move it into the `/usr/share/libalpm/hooks` directory:
```
mkdir /etc/pacman.d/hooks
wget -O /etc/pacman.d/hooks/firejail.hook https://raw.githubusercontent.com/theo546/my-arch-setup/master/firejail.hook
```

### d. (optional) Set the keyboard layout for x11
For an AZERTY keyboard:
```
localectl set-x11-keymap fr
```
*Note:* you don't need to do this step if you're using a QWERTY keyboard.

### e. (optional) Enable the numlock at boot for SDDM
```
echo -e "[General]\nNumlock=on" >> /etc/sddm.conf
```

### f. (optional) KDE Plasma customization
There is two dark themes that I enjoy, [Breeze Transparent Dark](https://store.kde.org/p/1170816/) and [Breeze Darker Transparent Plasma Theme](https://store.kde.org/p/1303414/).  
You can also use a customized version of the Breeze Darker Transparent theme that you can find inside the themes folder, simply drag 'n drop it in Appearance then Plasma Style.

*Appearance*  
Global Theme -> Breeze Dark  
Plasma Style -> Breeze-Darker-Transparent  
Application Style -> Application Style -> Configure GNOME/GTK Application Style -> GTK theme: Materia-dark-compact  
Icons -> Papirus-Dark  
Cursors -> DMZ (White)

*Workspace*  
Workspace Behavior -> General Behavior -> Click behavior: Single-click to open files and folders  
Workspace Behavior -> Dekstop Effects -> Window Open/Close Animation: Glide  
Workspace Behavior -> Dekstop Effects -> Appearance: Untick Maximize  
Workspace Behavior -> Dekstop Effects -> Window Management: Untick Present Windows  
Workspace Behavior -> Screen Locking -> Appearance: Configure... -> Positioning: Scaled, Keep Proportions  
Workspace Behavior -> Screen Locking -> Appearance: Configure... -> Solid colour: #282828  
(My wallpaper is the one in the wallpapers folder, it's the green Angel Dust.)  
Startup and Shutdown -> Login Screen (SDDM) -> Breeze  
Startup and Shutdown -> Desktop Session -> On Login: Start with an empty session  
Startup and Shutdown -> Splash Screen -> None

*Hardware* 
Display and Monitor -> Night Color -> Tick Activate Night Color  
Audio -> Applications -> Mute Notification Sounds  
**(optional, to prevent games from blocking the compositor)**
Display and Monitor -> Compositor -> Untick Allow applications to block compositing

*Right click on desktop*  
Configure Desktop and Wallpaper -> Background -> Positioning: Scaled, Keep Proportions  
Configure Desktop and Wallpaper -> Background -> Solid colour: #282828  
My wallpaper is the one in the wallpapers folder, it's the green Angel Dust.

*Right click on the task bar*  
Configure Icons-only Task Manager... -> Behavior -> Middle-clicking any task: Minimizes/Restores window or group

*Click on the volume icon in the task bar*
Tick Raise maximum volume

*Right-click on the "Start" menu*  
Show Alternatives... -> Select Application Menu  
Configure Application Menu... -> Untick Recent Applications and Recent files, Expand search to bookmarks, files and emails

(optional if the system is already in french) *Right-click on the date in the task bar*  
Configure Digital Clock... -> Time display: 24-Hour
Configure Digital Clock... -> Date format: Custom -> dd/MM/yyyy

### g. (optional) Fix KDE Plasma refresh rate with a 144 Hz screen
Open a terminal then type:
```
$ mkdir -p ~/.config/plasma-workspace/env && cd ~/.config/plasma-workspace/env
$ echo -e '#!/bin/sh\nexport KWIN_X11_REFRESH_RATE=144000\nexport KWIN_X11_NO_SYNC_TO_VBLANK=1\nexport KWIN_X11_FORCE_SOFTWARE_VSYNC=1' > kwin_env.sh
$ chmod +x kwin_env.sh
$ kwriteconfig5 --file kwinrc --group Compositing --key MaxFPS 144
$ kwriteconfig5 --file kwinrc --group Compositing --key RefreshRate 144
```
*Note:* you can set a custom refresh rate in `KWIN_X11_REFRESH_RATE` by multiplying your monitor refresh rate by a thousand.

Then restart your Plasma session.

### h. Make sure that the system is syncing to NTP servers
```
timedatectl set-ntp true
```

### i. (optional) Install libvirt
Open a terminal then type:
```
pacman -S libvirt virt-manager ebtables ovmf swtpm
```
This will provide libvirt itself, virt-manager to manage the VM more easily, ebtables as the firewall back-end, ovmf so your VM will be able to boot in EFI mode and a software TPM emulator.

Now enable and start the libvirtd service:
```
systemctl enable --now libvirtd
```

Now ask virsh to automatically start the default network at system boot:
```
virsh net-autostart default
```

Then start said network:
```
virsh net-start default
```

You're now good to go with libvirt!

## 6. Enroll the keys in the BIOS

### a. (optional) Sign the Microsoft keys
In case of a dual boot with a Windows system, you'll need to enroll the `Microsoft Windows Production PCA 2011` and `Microsoft Corporation UEFI CA 2011` certificates too.  
Get in your `/root/secureboot_keys` folder then sign them using your KEK:
```
cd /root/secureboot_keys
wget -U="" https://www.microsoft.com/pkiops/certs/MicWinProPCA2011_2011-10-19.crt
wget -U="" https://www.microsoft.com/pkiops/certs/MicCorUEFCA2011_2011-06-27.crt
sbsiglist --owner 77fa9abd-0359-4d32-bd60-28f4e78f784b --type x509 --output MS_Win_db.esl MicWinProPCA2011_2011-10-19.crt
sbsiglist --owner 77fa9abd-0359-4d32-bd60-28f4e78f784b --type x509 --output MS_UEFI_db.esl MicCorUEFCA2011_2011-06-27.crt
cat MS_Win_db.esl MS_UEFI_db.esl > MS_db.esl
sign-efi-sig-list -a -g 77fa9abd-0359-4d32-bd60-28f4e78f784b -k KEK.key -c KEK.crt db MS_db.esl add_MS_db.auth
```

### **Important note**
On some motherboard, you will need to add the `Microsoft Corporation UEFI CA 2011` certificate as without it, your motherboard will probably not boot.  
For exemple, not adding this certificate on a MSI H110M PRO-VD motherboard with an NVIDIA graphics card will prevent the motherboard to post as the BIOS will attempt to validate the signature of the graphics card BIOS before booting, in vain.  
If this situation occur on your setup, you will need to remove the CMOS battery to reset the BIOS to its default state.

If you want to sign only this certificate, you will need to sign the said certificate and enroll it in the BIOS:
```
cd /root/secureboot_keys
wget -U="" https://www.microsoft.com/pkiops/certs/MicCorUEFCA2011_2011-06-27.crt
sbsiglist --owner 77fa9abd-0359-4d32-bd60-28f4e78f784b --type x509 --output MS_UEFI_db.esl MicCorUEFCA2011_2011-06-27.crt
sign-efi-sig-list -a -g 77fa9abd-0359-4d32-bd60-28f4e78f784b -k KEK.key -c KEK.crt db MS_UEFI_db.esl add_MS_db.auth
```

### **Very important note**
If you are using a Lenovo ThinkPad, **do not under any circumstance** remove the default Secure Boot keys or [you'll brick your motherboard](https://wiki.archlinux.org/title/Lenovo_ThinkPad_T14s_(AMD)_Gen_1#Secure_boot).  
This does apply on every modern ThinkPad that were / are currently sold by Lenovo)  
Please insert the keys alongside the already present ones.

### b. Copy the certificates and the KeyTool utility
Make sure that you have Secure Boot in setup mode.  
Copy the certificates to a FAT formatted USB drive:
```
cd /root/secureboot_keys; cp *.cer *.esl *.auth /path/to/your/usb/drive
```
*Note:* obviously, rename `/path/to/your/usb/drive` to the actual path of your USB drive.

Now copy the `KeyTool.efi` utility in a folder to let your motherboard know that there is a bootable utility:
```
mkdir -p /path/to/your/usb/drive/EFI/BOOT
cp /usr/share/efitools/efi/KeyTool.efi /path/to/your/usb/drive/EFI/BOOT/BOOTX64.EFI
```
*Note:* if your motherboard is good enough, you may not need the KeyTool utility to enroll the keys.

### c. Enroll the keys
Now, restart your PC, open the boot options from your motherboard then boot off the USB drive to start the KeyTool utility.

Enroll the keys in this order:
1. db
2. KEK
3. PK

*Note:* make sure to last enroll your platform key.  
*Note2:* always prefer the `.auth` certificates over the others certificate type.  
*Note3:* do not forget to include the `add_MS_db.auth` certificate as you may not be able to boot afterward.

___

Once on the main menu, select `Edit Keys` then `The Allowed Signatures Database  (db)`.  
Now select `Add New Key` then select the drive and the db certificate on the file list.

*Note:* if you are doing a dual boot with Windows or if you need the `Microsoft Corporation UEFI CA 2011` certificate enrolled, add `add_MS_db.auth` as a db certificate.

Repeat these steps for KEK.

For the platform key, select `Replace Key(s)` then select your PK certificate just like earlier.

Once done, press ESC to get back on the main menu, if you have done those steps properly, at the top of the screen, it should show `Platform is in User Mode`.  
To finally finish the setup, press `Exit` to reboot!

From now on, only your new Arch setup (or any EFI executable signed with your keys) will be able to boot on this PC unless you disable Secure Boot.

### d. Restart and set a BIOS password
This step is **very** important as not adding a password may not protect you in the case of an evil maid attack, consider this.

## 7. Post-install

### a. Backup your LUKS partition header
Just in case, I highly recommand you to backup your LUKS header so in the case of a corruption, you will be able to use this header to access your encrypted files:
```
cryptsetup luksHeaderBackup /dev/sda2 --header-backup-file luks_header_backup
```
Save this file on a safe, offline and encrypted device.

If you ever need to restore your LUKS header:
```
cryptsetup luksHeaderRestore /dev/sda2 --header-backup-file luks_header_backup
```

If you wish to open the encrypted partition without restoring the LUKS header:
```
cryptsetup open --header luks_header_backup --allow-discards /dev/sda2 cryptlvm
```
*Note:* remove the `--allow-discards` option if you are not using an SSD or if you do not wish to use TRIM for "security reasons".

### b. Automatically unlock your encrypted LUKS partition using your motherboard TPM
First of all, make sure that your PC has a TPM 2.0 chip.  
If you don't then sorry, this part of the guide is not for you.

Install the `tpm2-tools` package:
```
pacman -S tpm2-tools
```

Add the `tpm_tis` module in the `/etc/mkinitcpio.conf` file:
```
MODULES(... tpm_tis)
```

Also add the `encrypt-tpm` hook in the hooks list, this one needs to be before the `encrypt` hook:
```
HOOKS(... encrypt-tpm encrypt ...)
```

Get the `encrypt-tpm` hook (this is to automate decryption on boot) and move it into the `/etc/initcpio/hooks/encrypt-tpm` directory:
```
wget -O /etc/initcpio/hooks/encrypt-tpm https://raw.githubusercontent.com/pawitp/arch-luks-tpm/6eda788de04b206697b5175a34e2fc969c5a6a66/hooks/encrypt-tpm
sed -i 's/sha1/sha256/g' /etc/initcpio/hooks/encrypt-tpm
```

Same for the other `encrypt-tpm` file:
```
wget -O /etc/initcpio/install/encrypt-tpm https://raw.githubusercontent.com/pawitp/arch-luks-tpm/6eda788de04b206697b5175a34e2fc969c5a6a66/install/encrypt-tpm
```

Regenerate the initramfs and sign the EFI executable:
```
pacman -S linux
```

Restart your system:
```
reboot
```

Now generate a random key that we'll store in the `/root/tpm` directory:
```
mkdir /root/tpm && cd /root/tpm
dd if=/dev/random of=secret.bin bs=32 count=1
```

Add the key to the LUKS header:
```
cryptsetup luksAddKey /dev/sda2 secret.bin
```

Then create a policy to only deliver the secret when the PCR 0, 2, 4 and 7 are validated:
```
tpm2_createpolicy --policy-pcr -l sha1:0,2,4,7 -L policy.digest
tpm2_createprimary -C e -g sha1 -G rsa -c primary.context
tpm2_create -g sha256 -u obj.pub -r obj.priv -C primary.context -L policy.digest -a "noda|adminwithpolicy|fixedparent|fixedtpm" -i secret.bin
tpm2_load -C primary.context -u obj.pub -r obj.priv -c load.context
tpm2_evictcontrol -C o -c load.context 0x81000000
rm load.context obj.priv obj.pub policy.digest primary.context
```

Restart your system once again:
```
reboot
```

And as you may have noticed, your system didn't ask you for a password, if it did, you may have done something wrong.

After a kernel update, you will notice that the system doesn't automatically unlock the partition, that is because some PCRs are now unvalidated as the hash of the EFI executable has changed.

You will need to remove the secret that is stored in the TPM before trying to add it back:
```
tpm2_evictcontrol -C o -c 0x81000000
```
Now repeat the commands above to make the system automatically unlock again.

___

Now: if you are bored of doing this every time there is a kernel update, I made a small script that'll add the secret into the TPM at every system boot.  
To make it short, you'll only have to type your password only once on boot after a kernel update, the script add the secret back into the TPM on boot for you.

Download the service and the script from my GitHub repo:
```
wget -O /etc/systemd/system/add-secret-to-tpm.service https://raw.githubusercontent.com/theo546/my-arch-setup/master/tpm-secret-service/add-secret-to-tpm.service
wget -O /usr/bin/add-secret-to-tpm https://raw.githubusercontent.com/theo546/my-arch-setup/master/tpm-secret-service/add-secret-to-tpm
```

Activate the service and change the permissions of the script so it can be executed:
```
systemctl enable add-secret-to-tpm
chmod +x /usr/bin/add-secret-to-tpm
```

___

If you wish to automatically calculate the future PCR when updating your kernel, install the `tpm_futurepcr` package from AUR:
```
$ yay -S tpm_futurepcr
```

Make sure the add-secret-to-tpm service is configured to use the `tpm_futurepcr` package:
```
sed -i 's/FUTURE_PCR=false/FUTURE_PCR=true/g' /usr/bin/add-secret-to-tpm
```

Install the libalpm hook so it automatically update the PCR when the kernel is updated:
```
wget -O /usr/share/libalpm/hooks/97-futurepcr.hook https://raw.githubusercontent.com/theo546/my-arch-setup/main/tpm-secret-service/97-futurepcr.hook
```

You are now ready and good to go, try and update the system using:
```
pacman -S linux
```

Restart your system and it should automatically unlock the LUKS partition.

**Note:** tpm_futurepcr doesn't work in OVFM as its TPM implementation is not properly done and is preventing the script from accessing the `/sys/kernel/security/tpm0/binary_bios_measurements` special file.

# Self-notes

My WiFi USB dongle doesn't work without installing these drivers:
```
$ yay -S rtl8821cu-dkms-git
```

___

To enable and start the Syncthing user service:
```
$ systemctl enable --user --now syncthing
```

___

Fix MultiMC5 with Firejail:
```
echo 'ignore noexec ${HOME}' >> /etc/firejail/multimc5.local
```

___

Fix Firejail being too strict with Eye of GNOME (GNOME image viewer).
```
echo 'noblacklist ${HOME}' >> /etc/firejail/eo-common.local
```

___

[Set a blank password for GNOME Keyring so it doesn't ask for your user password when an app needs access to it.](https://wiki.archlinux.org/index.php/GNOME/Keyring#Manage_using_GUI)

```
flatpak run org.gnome.seahorse.Application
```
Then change the password of both keyrings to nothing.

___

All the libraries required for Lutris:
```
pacman -S lutris giflib lib32-giflib libpng lib32-libpng libldap lib32-libldap gnutls lib32-gnutls mpg123 lib32-mpg123 openal lib32-openal v4l-utils lib32-v4l-utils libpulse lib32-libpulse libgpg-error lib32-libgpg-error alsa-plugins lib32-alsa-plugins alsa-lib lib32-alsa-lib libjpeg-turbo lib32-libjpeg-turbo sqlite lib32-sqlite libxcomposite lib32-libxcomposite libxinerama lib32-libgcrypt libgcrypt lib32-libxinerama ncurses lib32-ncurses opencl-icd-loader lib32-opencl-icd-loader libxslt lib32-libxslt libva lib32-libva gtk3 lib32-gtk3 gst-plugins-base-libs lib32-gst-plugins-base-libs vulkan-icd-loader lib32-vulkan-icd-loader zenity
```

___

Make FIDO keys work in Firefox / Chrome:
```
pacman -S libfido2
```

___

To install Discord Canary using the Flatpak beta repo:
```
flatpak remote-add --if-not-exists flathub-beta https://flathub.org/beta-repo/flathub-beta.flatpakrepo
flatpak install flathub-beta com.discordapp.DiscordCanary
```

___

To use the virtual camera on OBS, install the `v4l2loopback-dkms` package:
```
pacman -S v4l2loopback-dkms
```

# Final words

Thank you for reading this guide, I hope you will find anything useful in here.  
Please open an issue if you need to suggest or fix something, I'll be glad to respond!