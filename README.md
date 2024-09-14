# Archlinux Installation Guide

There are a lot of good guides out there and as with this one, they are somewhat oppionionated based on the authors
hardware and preferences. A really good start, although a bit more complicated for a beginner is the 
[official installation guide](https://wiki.archlinux.org/title/Installation_guide). 

This guide is for using UEFI with systemd-boot, AMD cpu & gpu && Gnome desktop. I've selected systemd-boot instead of 
grub because I want to have flicker-free boot splash, it's easier to install and I'm running only Arch anyway 
on my machine.

Great guides helping you through:
[Modern Arch linux installation guide by mjkstra](https://gist.github.com/mjkstra/96ce7a5689d753e7a6bdd92cdc169bae)

## Preparation

Preparing is about internet connection and correct time. The rest is more about making installation a bit more 
convinient.

- List all keyboard layouts to find the right one
    ```
        localectl list-keymaps | less
    ```

- Load the correct keyboard layout. 
    ```
        loadkeys et
    ```

- Set bigger font if text is too small
    ```
        setfont ter-i32b
    ```

- Check if in UEFI mode (the output has to be 32 or 64)
    ```
        cat /sys/firmware/efi/fw_platform_size
    ```

- If you don't need to connect to internet via wifi, skip this section

    - Launch iwctl
        ```
            iwctl
        ```
    
    - List devices 
        ```
            device list
        ```
    
    - Scan for networks (my device was wlan0)
        ```
           station wlan0 scan 
        ```
    
    - Connect to the network specified
        ```
           station wlan0 connect "Network SSID"
        ```

    - Quit iwctl 
        ```
          quit 
        ```

- Check that you have working internet connection
    ```
        ping -c 5 archlinux.org 
    ```

- Check if local time is correct and network time protocol active for syncing
    ```
        timedatectl    
    ```

- If NTP is not active, activate it.
    ```
        timedatectl set-ntp true
    ```

## Installation

### Creating partitions

For partitioning we're using fdisk. The following section does not include default selections with <ENTER>

- First find the right device, mine is /dev/nvme0n1
    ```
        fdisk -l
    ```

- Start fdisk on the device
    ```
        fdisk nvme0n1
    ```

- Delete any existing partitionsi (incase you're not planning on dual booting)
    ```
       d 
    ```

- Use GPT partition table
    ```
       g
    ```

- Create EFI partition, recommended size by Arch is 1 GB
    ```
       n
       +1G
    ```

- Select EFI as type
    ```
        t 
        1
    ```

- Whether or not to use swap is up to you. I use it for improved stability. 
    ```
        n
        this section needs update
    ```

- Select swap as type
    ```
        this section needs update
    ```

- Create Linux EXT4 filesystem
    ```
        n
        p
    ```

- Write changes and quit
    ```
        w
        q
    ```

### Formating partitions

- First find out partitions for formating
    ```
        fdisk -l
    ```

- Format EFI partition with FAT32 system
    ```
        mkfs.fat -F 32 /dev/efi_system_partition
    ```

- Format swap partition
    ```
        mkswap /dev/swap_partition
    ```

- Format system partition
    ```
        mkfs.ext4 /dev/root_partition
    ```

### Mounting disks 

- Mount previously created EFI partition so we could install bootloader onto it. Remember the bootloaders directory
  for later installation. In this case its /mnt/boot
    ```
        mount --mkdir /dev/efi_system_partition /mnt/boot
    ```

- Mount prevously created system EXT4 partition
    ```
        mount /dev/root_partition /mnt
    ``` 

- If you created swap volume, enable it
    ```
        swapon /dev/swap_partition 
    ``` 

### Installing essential packages

- Now install essential packages to be able to boot into the new installation and connect to the internet. 
    ```
        pacstrap -K /mnt base linux linux-firmware amd-ucode networkmanager pipewire pipewire-alsa 
        pipewire-pulse pipewire-jack wireplumber man sudo
    ``` 

### Configuring the system

- Generate instructions for the system on how to mount the disks automatically
    ```
        pacstrap -K /mnt base linux linux-firmware amd-ucode networkmanager pipewire pipewire-alsa 
        pipewire-pulse pipewire-jack wireplumber man sudo neovim
    ``` 

- Change root into the new system
    ```
        arch-chroot /mnt
    ``` 

- List timezones to find yours (if you do not know it already)
    ```
        timedatectl list-timezones
    ```

- Set correct timezone by creating symlink to localtime
    ```
        ln -sf /usr/share/zoneinfo/Europe/Tallinn /etc/localtime
    ```

- Sync system time
    ```
        hwclock --systohc
    ```

- Generate locale file
    ```
        hwclock --systohc
    ```

- Uncomment the locales (system languages) you wish to be available
    ```
        nvim /etc/locale.gen
    ```

- Set preferred locale to use
    ```
        echo LANG=en_US.UTF-8 >> /etc/locale.conf
    ```


- Set persistent keyboard layout
    ```
        echo KEYMAP=et >> /etc/vconsole.conf
    ```

- Set your hostname
    ```
        echo ArchPad >> /etc/hostname
    ```

- Set localhost in the hosts file by entering the following into `/etc/hosts`
    ```
        127.0.0.1 localhost
        ::1 localhost
        127.0.1.1 ArchPad
    ```

## Setting up bootloader

- Arch has systemd already installed, installing systemd-boot bootloader is straight forward if EFI boot
  partition is mounted to either /efi or /boot
    ```
        bootctl install
    ```

## Setting up root and users

- Set root users password
    ```
        passwd
    ```

- Create new user and set password
    ```
        useradd -mG wheel [username]
        passwd [username]
    ```

- Set neovim as visudo editor and enable wheel group as sudoers
    ```
        EDITOR=neovim visudo 
    ```
    Uncomment this line: %wheel ALL=(ALL:ALL) ALL

## Rebooting

- Enable network manager as service to connect to internet after reboot
    ```
        systemctl enable NetworkManager
    ```

- Unmount mounted volumes 
    ```
        umount -R /mnt
    ```

- Reboot 
    ```
        sudo reboot
    ```

## After reboot

- Install Wayland
    ```
        sudo pacman -S --needed wayland
    ```

- Install Gnome Display Manager
    ```
        pacman -S gdm
    ```

- Enable GDM service so we'll be booted into GDM

    ```
        sudo systemctl enable gdm
    ```

- Install various Gnome packages
    ```
        pacman -S --needed xorg-xwayland xorg-xlsclients glfw-wayland
    ```


