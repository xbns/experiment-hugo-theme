---
title: Arch Linux Installation on UEFI/GPT system
date: 2020-07-08
tags: [arch,installation]
description: Arch installation on UEFI system
draft: false
---

## Installing Arch Linux for UEFI/GPT System

For BIOS/MBR Legacy system refer to [here]({{<relref "2020-07-08-arch-linux-installation-on-bios-mbr-system/index.md">}}) 

### Installation process over a Wi-Fi

Download the .iso file from [Arch Linux](https://www.archlinux.org/download/) Official Site.

### Step 1: Make a Live USB Disk and boot from it.

Am assuming you know how to make live USB drives using a program like [balenaEtcher](https://balena.io) or an equivalent.

**Boot from Live USB**
example:
Press ESC --- > Boot Menu (F9) --> UEFI Generic Mass Storage ....

You should see prompt like:
```bash
[root@archiso ~]$
```
### Step 2: Start Wireless Service
Try to issue this command
```bash
[root@archiso ~]$ timeout 3 ping google.com
```
Most probably you will get this response `ping: google.com: Name or service not known`.
Meaning you are not connected to the internet.

Now start the INet Wireless Daemon(iwd).This wireless package comes pre-installed on intel based hp laptops.Not sure if it's in other laptop
models.

The installed `iwd` package provides the following:
1. iwctl - a client program
2. iwd - the daemon (the service)
3. iwon - Wi-Fi monitoring tool

**Enable the `iwd.service`**
```
$ systemctl enable iwd.service
```

**Start the service**
```shell
$ systemctl start iwd.service
```

**Verify it has started**
```shell
$ systemctl status iwd.service
```

### Step 3: Connect to a Wi-Fi network
After verifying the iwd service is running in the previous step, type `iwctl`  to get an interactive prompt.

The prompt will be displayed as `[iwd]#`.Here you are interacting with the client program 'iwctl'.

**3.1 Get you device name**

list all wifi devices nearby
```shell
[iwd]# device list
```

```
                    Devices
-------------------------------------------------------------
Name           Address            Powered    Adapter  Mode
-------------------------------------------------------------
wlan0          b8:81:98:75:b6:79    on        phy0    station
```

Note: wireless devices usually start with letter 'w'. e.g `wlan0` or `wlp3s0`

**3.2 List all available networks**
```shell
[iwd]# station wlan0 get-networks
```
```
                  Available networks
------------------------------------------------------------
Network name            Security          Signal
-------------------------------------------------------------
James2                     psk             ****
the_guy_next_door          psk             **** 
````
   
Your `Network Name` is also referred to as `SSID` technically.

**3.3 Connect to your network**

If you can see your network name, then type
```
[iwd]# station wlan0 connect <your_network_name>
```
substitute the above command with your network  name, angle brackets excluded to connect to it.

When you type the above command, for instance `station wlan0 connect guy_next_door`

you will be prompted for passphrase.This is nothing but you network password.Type it and press enter.

Exit the iwctl client by typing `exit`

**3.4 Ping to verify the connection**
Now back to prompt.Ping google servers for 3 seconds.
```bash
[root@archiso ~]$ timeout 3 ping google.com
```
You should see see response different from the one stated in step 2.

To use `iwctl` directly without entering into the prompt

```shell
root@archiso ~#iwctl --passphrase <your_network_password> station <your_interface> connect <your_network_name>

e.g

root@archiso ~#iwctl --passphrase m455fgfH station wlan0 connect the_guy_next_door
```

### Step 4: Check that you have booted in the correct mode.
Am assuming you are in UEFI  mode, not legacy mode.Because commands are quite difference for both modes.
You can type
```shell
$ls /sys/firmware/efi/efivars
```
If the above commands displays output,then you are in UEFI mode.Clear the screen `Ctrl + l`.

Otherwise if it displays `ls:cannot access 'sys/firmware/efi/vars':No such file or directory`
then you have legacy system

Otherwise you can execute the following command to verify
```bash
# [ -d /sys/firmware/efi/efivars ] && echo "UEFI" || echo "Legacy"

```

Then you can refer to the [Installation for Legacy/BIOS/MBR Legacy system process]({{<relref "2020-07-08-arch-linux-installation-on-bios-mbr-system/index.md">}}) 

### Step 5: Check clock is set correctly
```shell
$ timedatectl set-ntp true
```

### Step 6: Disk Partition

You can refer to [this article]({{<relref "2020-07-08-why-this-partitioning-scheme/index.md">}}) of how my partition scheme looks like.


### Step 7: Install the Base System.
You have finished partitioning and formating the disk in Step 6 as per your expectation.It's time for installing base system.
Pacstrap!

#### 7.1 Mount partitions so that you can install software on them.

1. create mount dir
2. create new file system
3. label drive
4. mount drive

**make boot EFI System directory,create a filesystem for it & mount it to its respective partition**
```shell
$ mkdir -p /mnt/boot/efi
$ mkfs.fat -F32 /dev/sda1
$ e2label /dev/sda1 Boot
$ mount /dev/sda1 /mnt/boot/efi
```

**make var directory & mount it to its respective partition**
```shell
$ mkdir -p /mnt/var
$ mkfs.ext4 /dev/sda2
$ e2label /dev/sda2 var
$ mount /dev/sda2 /mnt/var
```

**create a filesystem & mount it to its root partition**
```shell
$ mkfs.ex4 /dev/sda3
$ mount /dev/sda3 /mnt
```
-p flag here means create the previous directory i.e boot if it does not exist.


**make home directory & mount it to its respective partition**
```shell
$ mkdir -p /mnt/home
$ mkfs.ext4 /dev/sda4
$ e2label /dev/sda4 arch-home
$ mount /dev/sda4 /mnt/home
```
**make swap and turn it on**
```bash
$ mkswap /dev/sda5
$ swapon /dev/sda5
```

**install the base system**
```
$pacstrap /mnt base base-devel linux-lts linux-firmware nano dialog iw wpa_supplicant networkmanager

```

### Step 8: Generate fstab file
This file has info about your partitions.It picks up partitions that have already been mounted.
So we expect this file to contain partitions info.

- boot
- root
- var
- home

So the command to generate fstab file
```
$genfstab -U /mnt >> /mnt/etc/fstab
```
To verify this  we can type
```
$cat /mnt/etc/fstab
```

### Step 9: More Configurations and Set Up on root

Run the following commands as `root`

```
$ arch-chroot /mnt

[root@archiso]
```

We need to change to root of the system that is being installed so that we can:
1. Install Bootloader.
2. Set Up Time Zone.
3. Generate Hardware Clock Settings.
4. Set Up Language Settings(Locale)
5. Add Super User.
6. Verify everthing is OK, upto now.

#### 9.1 Set Up Timezone


Create a soft link to your timezone
```
# ln -sf /usr/share/zoneinfo/Africa/Nairobi  /etc/localtime
```

#### 9.2 Generate hardware clock settings
Ensure that your clock is set to UTC
```
# hwclock --systohc --utc
```


#### 9.3 Set Up Language Settings
```
# echo en_US.UTF-8 UTF-8 >> /etc/locale.gen
```
Then generate
```
# locale-gen
```

```
# echo LANG=en_US.UTF-8 > /etc/locale.conf
```

```
# export LANG=en_US.UTF-8
```

Set hostname mine is:hp-450g3-i7
```
# echo hp-450g3-i7 > /etc/hostname
```

Check everything has been installed properly.
```
# mkinitcpio -p linux-lts
```

### 9.4 Add Super user (Sudo)
mine is josphat
```
# useradd -m -G sys,wheel,users,adm,log -s /bin/bash josphat
```
we have added user `josphat` to these groups
- sys
- wheel
- users
- adm
- log

create a password for your user
```
#passwd
```
Enter the password again when prompted again.

To delete a user
```
$ userdel <username>
```

### 9.5 Edit file /etc/hosts

```
# nano /etc/hosts
```
The file is empty, add the following content

Substitute `hostname` for yours.

```
127.0.0.1 localhost
::        localhost
127.0.1.1  hp-450g3-i7.localdomain   hp-450g3-i7
```

- Ctrl + o  --> write changes
- Enter  --> to save/confirm changes 
- Ctrl + x  --> Exit
- Ctrl + l  ---> Clear Screen

### 9.6 Set Up Sudo User Using Nano
```
# EDITOR=nano visudo
```
Find the following line in that file, uncomment(remove #) and save

`#% wheel ALL = (ALL) ALL`

### 9.7 Add Multilib Repo and Sync the Repos
```
#nano /etc/pacman.conf
```
Find the following line in that file, uncomment and save.

uncomment `multilib` not `multilib-testing`

Now sync the repos
```
# pacman -Syy
```

### 9.8 Optimize the mirrors

Go to [this site](https://www.archlinux.org/mirrorlist) and generate your nearest mirrors.I'm in Kenya

So here we go

```
# nano /etc/pacman.d/mirrorlist
```

I will populate this file with those for Kenya and South Africa, in that order.

```shell
##
## Arch Linux repository mirrorlist
## Generated on 2020-06-26
##

## Kenya
Server = http://archlinux.mirror.liquidtelecom.com/$repo/os/$arch
Server = https://archlinux.mirror.liquidtelecom.com/$repo/os/$arch

## South Africa - 3074 km
Server = http://archlinux.za.mirror.allworldit.com/archlinux/$repo/os/$arch
Server = https://archlinux.za.mirror.allworldit.com/archlinux/$repo/os/$arch
Server = http://za.mirror.archlinux-br.org/$repo/os/$arch
Server = http://mirror.is.co.za/mirror/archlinux.org/$repo/os/$arch
Server = http://mirrors.urbanwave.co.za/archlinux/$repo/os/$arch
Server = https://mirrors.urbanwave.co.za/archlinux/$repo/os/$arch

## Israel - 3469 km
Server = http://mirror.isoc.org.il/pub/archlinux/$repo/os/$arch
Server = https://archlinux.mivzam.net/$repo/os/$arch

```
### 9.9 Install GRUB bootloader for UEFI system
```
# pacman -S grub efibootmgr os-prober
```

### 9.10 Create a GRUB file for your UEFI system
```
# grub-install --target=x86_64-efi --bootloader-id=grub --efi-directory=/boot/efi
```

### 9.11 Generate GRUB Configuration file

```
# grub-mkconfig -o /boot/grub/grub.cfg
```

### 9.12 Install various important packages
```shell
# pacman -S xorg xorg-apps xorg-server xorg-drivers xrandr terminator mesa linux-headers xdg-user-dirs fuse2 ntfs-3g exfat-utils pulseaudio pulseaudio-bluetooth bluez-utils pavucontrol  gvfs dkms haveged git unrar unzip htop feh lsb-release  firefox lightdm lightdm-gtk-greeter  
``` 

Install vlc and youtube-dl, then download a video clip(mp4 format(18)) from youtube for testing the audio & video
```
# pacman -S vlc youtube-dl

# youtube-dl -f 18 https://www.youtube.com/watch?v=aIINncHHko   /home/josphat/Music

```
### 9.13 Install enlightenment Windows Manager 
This is personal preference,you can install i3, cinnamon,lxde desktop/windows  environment etc.
More about the [Enlightenment](https://www.enlightenment.org/docs/distros/archlinux-start.md)

```
# curl https://download.enlightenment.org/distros/arch/archlinux/arch/repo.txt -o - l | tee -a /etc/pacman.conf
```
Then to install the packages for the first time
```
# pacman -Sy && pacman -S efl-git enlightenment-git terminology-git rage-git
```

### 9.14 Install fonts
```
# pacman -S  freetype2 terminus-font ttf-bitstream-vera
               ttf-dejavu ttf-droid ttf-fira-mono
               ttf-fira-sans ttf-freefont ttf-inconsolata ttf-liberation
               ttf-linux-libertine ttf-ubuntu-font-family xorg-xfontsel
```

### (Optional) Applications to install
```
# pacman -S xed gnome-screenshot gimp code cinnamon lightdm-webkit2-greeter 

```

You can use `sudo update-alternatives --config lightdm-greeter` to toggle the greeters.

### 9.15 Enable the vital services

```
# systemctl enable lightdm.service
```
```
# systemctl enable NetworkManager
```
```
# systemctl enable wpa_supplicant.service
```
```
# systemctl --user enable pulseaudio

```

**Exit Root**

```
# exit
```

```
$ shutdown now
```

Remove live USB

### 10. Start the vital services after you login
```
$ sudo systemctl start lightdm.service
```
```
$ sudo systemctl start NetworkManager
```
```
$ sudo systemctl start wpa_supplicant.service
```
```
$ sudo systemctl start pulseaudio

```

### 11. Set Up Default Light Display Manager Greeter

```
# nano /etc/lightdm/lightdm.conf
```

```
greeter-session=lightdm-gtk-greeter

user-session=enlightenment

```
### Post Power Up
**ping to verify we have a network**
```
$timeout 3 ping google.com 
```
**check your interface**
```
$iw dev
```
Above command retrieves wireless device name

**List nearby wireless networks**
```
$nmcli device wifi list
```
**connect to wireless network**
```
$nmcli device <SSID> password <SSID_PASSWORD>

```

SSID - Network Name, SSID_PASSWORD - Network Password


**You can also use the GUI**
```
$nmtui
```

### Troubleshooting

#### 1. Reset Root Password
plug in the live USB
```
[root@archiso]#mount /dev/sdaX /mnt
```
```
[root@archiso]#passwd --root /mnt user_name
```
- New Password
- Retype Password

Then unmount
```
[root@archiso]#umount /dev/sdaX
```

X - for root partition number

#### 2. [FAILED] Failed to start Light Display Manager
plug in the live USB
```
[root@archiso]#mount /dev/sdaX /mnt
```

```
[root@archiso]#sudo systemctl start lightdm.service
```
```
[root@archiso]#reboot
```

#### 3. No Space Left

`error: failed retrieving file 'linux-firmware' from archlinux.: Resolving timed out after 10520 milliseconds`

```
$mount -o remount,size=4G /run/archiso/cowspace
```

### Notes to Self
Important key things to remember

1. Don't forget to mount all partitions before you install any software.

2. Don't forget to enable vital service before you reboot.


### References
- https://blog.oshim.net/2011/10/how-to-move-var-folder-to-new-partition/
- [So You Want to Install Arch? Let's Do It!](https://www.youtube.com/watch?v=XmCuFJgxzL8)
- https://www.systutorials.com/docs/linux/man/8-grub-install/
- https://gnu.huihoo.org/grub-0.90/html_node/grub_11.html

