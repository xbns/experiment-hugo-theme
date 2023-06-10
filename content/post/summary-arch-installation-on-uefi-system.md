---
title: Summary - Arch Linux Installation on UEFI system
date: 2020-12-30
tags: [arch, installation]
description: Uefi Arch installation in 10 steps
draft: false
---

### Step 1 Connect to internet from install disk or usb

Start `iwd.service` and connect your wifi

```shell
# systemctl start iwd.service
# systemctl status iwd.service
# iwctl --passphrase <you_wifi_password> station <name_of_station> connect <wifi_name>
# timeout 2 ping google.com
```

Verify the host specified by ping is reachable otherwise don't proceed until you fix it.
Since you won't be able to do anything without internet connection.

### Step 2 Create Filesystems and Mount Partitions

Assuming you have **created** the partitions using e.g `cfdisk <SSD>`

```shell
# cfdisk /dev/sda
```

**Boot**

```shell
# mkdir -p /mnt/boot/efi
# mkfs.fat -F32 /dev/sda1
# mount /dev/sda1 /mnt/boot/efi
```

**Var**

```shell
# mkdir -p /mnt/var
# mkfs.ext4 /dev/sda2
# e2label /dev/sda2 var
# mount /dev/sda2 /mnt/var
```

**Root**

```shell
# mkfs.ext4 /dev/sda3
# mount /dev/sda3 /mnt
```

**Home**

```shell
# mkdir -p /mnt/home
# mkfs.ext4 /dev/sda4
# e2label /dev/sda4 home
# mount /dev/sda4 /mnt/home
```

**Swapfile**

```shell
# fallocate -l 32GB /swapfile mkswap
# /swapfile swapon /swapfile
```

### Step 3 Install Base System and Grub Bootloader

**install base system**

```shell
# pacstrap /mnt base linux-lts linux-firmware nano dialog iw wpa_supplicant networkmanager
```

**install grub bootloader**

```shell
# pacman -S grub efibootmgr os-prober

# grub-install --target=x86_64-efi --bootloader-id=grub --efi-directory=/boot/efi

```
Generate grub config

```shell
# grub-mkconfig -o /boot/grub/grub.cfg
```

### Step 4 Generate fstab

```shell
# genfstab -U /mnt >> /mnt/etc/fstab
```

Sometimes you might get file errors that the system is readonly if you try to generate the fstab file.
Remount the system to fix it.

`# mount -o remount, rw /`

**Edit the fstab file**

```shell
# nano  /mnt/etc/fstab
```

**Enter the Swapfile to the FSTAB**

```shell
# nano /etc/fstab /swapfile none swap defaults 0  0
```

For the mounted partitions change the `UUID` to the `name of partition`
i.e inplace of `UUID=xxxxxx` change this to `/dev/sda2`.
Though not necessary,it make the system boot faster.

**nano commands**

-   _Ctrl + o_ : Write Edits
-   _Enter_ : Save Edits
-   _Ctrl + x_ : Exit nano
-   _Ctrl + l_ : Clear Screen

### Step 5 Clock and Timezone Settings

**Set timezone**
You need to `chroot` to set the timezone

```shell
# arch-chroot /mnt
```

Then

```shell
# ln -sf /usr/share/zoneinfo/Africa/Nairobi /etc/locatime
```

**Set hardware clock**

```shell
# hwclock --systohc --utc
```

**synchronize clock with timezone**
This is simply activating the `NTP service`

```shell
# timedatectl set-ntp true
```

### Step 6 Set Locale and Language Settings

**generate locale settings**
Using American English
Uncomment this part `#en_US.UTF-8 UTF-8` in this file `/etc/locale.gen`
i.e

```shell
# nano /etc/locale.gen
```

Then run `# locale-gen`
**set language & export it to a locale config file**

```shell
# echo LANG = en_US.UTF-8 > /etc/locale.conf
# export LANG = en_US.UTF-8
```
**set hostname**

change `your_hostname` below to what you would like to call your laptop

```shell
# echo your_hostname > /etc/hostname
```

Run this to verify everything is OK

```shell
# mkinitcpio -p linux-lts
```

### Step 7 Set Default Shell and Terminal Emulator

**Shell**
Am using `zsh`

```shell
# pacman -S zsh zsh-completions zsh-syntax-highlighting zsh-theme-powerlevel10k
```

Set it as default

```shell
# chsh -S /bin/zsh
```

**Terminal Emulator**
Install terminology terminal emulator

```shell
# pacman -S terminology
```

### Step 8 : Create a Sudo User and change the default password of Root

**Root**

```shell
# passwd
```

When prompted type and confirm the new password for root
**Sudo User**

```shell
# useradd -m -G wheel -s /bin/zsh jay
# passwd jay
```

When prompted type and confirm the password for sudo user above

Uncomment the `wheel` group

```shell
# EDITOR=nano visudo
```

**Exit chroot**

```shell
# exit
```

**Umount all partitions**

```shell
# umount -R /mnt
```

**Reboot the system**

```shell
# reboot
```

_NOTE_
While they system is rebooting you may now remove the USB installer disk.
From now onwards you will be working with your fresh arch install using your
`sudo user` instead of `root user`.

### Step 9 Reconnect to Wifi

Since we removed the USB installer disk we have lost the `iwd.service` that comes with it to
connect to wi-fi.But since we installed these wireless packages;**wpa_supplicant** and **networkmanager**
in [Step 3]({{<relref "#step-3-install-base-system">}})
We only need to enable,start them.Then connect to our wifi network and continue with the installation process.

Note that the shell prompt has changed from `#` to `%`.

**Enable & Start wpa_supplicant and networkmanager services**

```shell
% sudo systemctl is-enable wpa_supplicant.service
% sudo systemctl enable wpa_supplicant.service
% sudo systemctl start wpa_supplicant.service
% sudo systemctl status wpa_supplicant.service
```

```shell
% sudo systemctl is-enable networkmanager.service
% sudo systemctl enable networkmanager.service
% sudo systemctl start networkmanager.service
% sudo systemctl status networkmanager.service
```

If the status of the 2 services above is "running".Then we can connect our wifi again

`nmcli` is the command line client for networkmanager

**List nearby wireless networks**

```shell
% nmcli device wifi list
```

If your network is listed among the above;then
**connect to a wireless network**

```shell
% nmcli device wifi <SSID> password <SSID_PASSWORD>
```

where SSID -> your wifi name and SSID_PASSWORD -> your wifi password

Sometimes your wifi could be hidden hence it won't show up in the list.
If that is the case then

```shell
% nmcli device wifi connect <SSID> password <SSID_PASSWORD> hidden yes

```

Am assuming if you are able to get through [Step 1]({{<relref "#step-1-connect-to-internet-from-install-disk-or-usb">}})
then the above is easier.

### Step 10 : Configure your installation

Now that you have reconnected to the internet(ping to verify),you can continue configuring you installation.

**Edit file hosts**

```shell
% sudo nano /etc/hosts
```

Change the above file to

```shell
127.0.0.1 localhost
::        localhost
127.0.0.1 <your_hostname>.localdomain   <your_hostname>

```

Replace anything within the angle brackets inclusive.

**Add multilib Repo**

```shell
% sudo nano /etc/pacman.conf
```

Uncomment the `multilib` repo.

**sync repos**

```shell
# sudo pacman -Syy
```

You should also see the multilib repo syncing just like it peers `Core`,`Extra` etc

**Optimize the mirrors**

-   Am skipping this one till we have login out desktop environment;xfce

**Install GUI Server**

```shell
% sudo pacman -S xorg-server xorg-xinit xorg-xrandr
```

**Install some important packages**

```shell
% sudo pacman -S gvfs dkms xdg-user-dirs fuse2 haveged git ntfs-3g
```

**Install the Display Manager & Desktop Environment**

```shell
% sudo pacman lightdm lighdm-gtk-greeter xfce4 xfc4-goodies
```

**Configure Display Manager(lightdm)**

```shell
% sudo nano /etc/lightdm/lightdm.conf
```

Change these settings

```shell
greeter-session=lightdm-gtk-greeter
user-session=xfce4-session
```

**Enable the LightDM Service**

```shell
% sudo systemctl is-enabled lightdm.service
% sudo systemctl enable lightdm.service
```

**Verify that Graphical Target is the Default**
If not then set it to be the default one

```shell
% sudo systemctl get-default
```

You can skip the below step if the above command return graphical target

```shell
% sudo systemctl set-default graphical.target
```

**Verify the enabled service**

```shell
% sudo systemctl list-unit-files | grep enabled
```

**Reboot your system and login using your sudo user credentials**

```shell
% reboot
```

**Set Terminology Emulator as Default in xfce**

-   Applications-->Settings--->Default Applications --->Utilities--->Terminal Emulator
-   Select 'Terminology' so that whenever you press `ctrl + alt + t`,its the one that pops up.

**NOTE**

-   I have skipped the installation for audio drivers and others,this could be done once desktop environment is up and running.

**Reference**
[UEFI Process]({{<relref "2020-07-08-arch-linux-installation-on-uefi-gpt-system/index.md">}}).
