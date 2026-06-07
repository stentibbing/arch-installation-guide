# Archlinux Installation Guide

[Official Arch Linux guide](https://wiki.archlinux.org/title/Installation_guide). 
[Modern Arch linux installation guide by mjkstra](https://gist.github.com/mjkstra/96ce7a5689d753e7a6bdd92cdc169bae)

## Preparation

- List all keyboard layouts to find the right one
    ```
        localectl list-keymaps | less
    ```

- Load the correct keyboard layout. 
    ```
        loadkeys et
    ```

- Set bigger font if text is too small. 
    ```
        setfont ter-i28b
    ```

- Verify UEFI boot mode (the output has to be 32 or 64)
    ```
        cat /sys/firmware/efi/fw_platform_size
    ```

- Check for wifi not being blocked by rfkill
    ```
        rfkill list
    ```

- If wifi is blocked, unblock by running
    ```
        rfkill unblock wlan
    ```

- Start DHCP client
    ```
        dhcpcd && dhcpcd wlan0
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

    - List networks 
        ```
           station wlan0 get-networks
        ```
    
    - Connect to the network specified
        ```
           station wlan0 connect "Network SSID"
        ```

    - Exit iwctl
        ```
          CTRL-C
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

### Preparing Disk

- First find the right device, mine is /dev/nvme0n1
    ```
        fdisk -l
    ```

- Start fdisk on the device
    ```
        fdisk /dev/nvme0n1
    ```

- For clean installation, delete any existing partitions. Do not do this, if you plan on dualbooting.
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

- Create LVM partition, taking up all the remaining space 
    ```
       n
    ```

- Select Linux LVM as type
    ```
        t 
        44
    ```

- Write changes and quit
    ```
        w
        q
    ```

- Wipe the disk clean of any previous data
    ```
        dd if=/dev/urandom of=/dev/nvme0n1p2 bs=1M status=progress
    ```

- Create LUKS encrypted contianer
    ```
        cryptsetup luksFormat /dev/nvme0n1p2
    ```
 

- Open the LUKS encrypted contianer
    ```
        cryptsetup open /dev/nvme0n1p2 cryptlvm
    ```

- Create physical volume into the containerˇ 
    ```
        pvcreate /dev/mapper/cryptlvm
    ```

- Create volume group
    ```
        vgcreate Encrypted /dev/mapper/cryptlvm
    ```

- Create logical volumes into the volume group
    ```
        lvcreate -L 4G MyVolGroup -n swap
        lvcreate -L 32G MyVolGroup -n root
        lvcreate -l 100%FREE MyVolGroup -n home
    ```

- Reduce home size to leave space for e2scrub which scans for volume corruptions in the background
    ```
        lvreduce -L -256M MyVolGroup/home
    ```

- Format logical volumes 
    ```
        mkfs.ext4 /dev/MyVolGroup/root
        mkfs.ext4 /dev/MyVolGroup/home
        mkswap /dev/MyVolGroup/swap
    ```

- Format EFI partition
    ```
        mkfs.fat -F32 /dev/nvme0n1p1
    ```

- Mount filesystems
    ```
        mount /dev/MyVolGroup/root /mnt
        mount --mkdir /dev/MyVolGroup/home /mnt/home
        mount --mkdir /dev/nvme0n1p1 /mnt/boot
        swapon /dev/MyVolGroup/swap
    ```

### Installing packages 

- Now we need to install the kernel, base pacakges, drivers and utils we might need right away after rooting to 
  our new system (which is the next step) and after first reboot.
  
  #### Packages we're installing:

  - base base-devel linux linux-firmware amd-ucode
    Kernel, firmware, system packages and packages necessary to build.

  - fwupd
    Firmware update utility

  - amdvlk
    AMD maintained Vulkan 3D drivers

  - networkmanager, bluez, bluez-utils
    Network manager for managing wired and wireless connections and bluetooth drivers and utils

  - pipewire, pipewire-alsa, pipewire-jack, pipewire-pulse
    Audio drivers and utils

  - plymouth
    Boot splash screen

  - man, sudo, openssh, git, wget, curl, nvim, wl-clipboard, terminus-font
    Basic utilities

  - xdg-utils
    For defining default apps etc.

    ```
        pacstrap -K /mnt base base-devel linux linux-firmware fwupd amd-ucode amdgpu vulkan-radeon networkmanager bluez bluez-utils pipewire xdg-utils pipewire-alsa pipewire-pulse pipewire-jack wireplumber plymouth man sudo openssh git curl wget vim wl-clipboard terminus-font
    ```

### Configuring the system

- We need to generate filesystem table for the system to know what partitions should be mounted every time.  
  The table is saved to the systems /etc/fstab. This is generated based on the currently mounted partitions.
    ```
        genfstab -U /mnt >> /mnt/etc/fstab
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

- Sync hardware clock to system time
    ```
        hwclock --systohc
    ```

- Uncomment the locales (system languages) you wish to be available. This has to be done before generating the 
  locale file.
    ```
        nvim /etc/locale.gen
    ```

- Generate locale file
    ```
       locale-gen 
    ```

- Set preferred locale to use
    ```
        echo LANG=en_US.UTF-8 >> /etc/locale.conf
    ```

- Set persistent keyboard layout for TTY
    ```
        echo KEYMAP=et >> /etc/vconsole.conf
    ```

- Set persistently larger TTY font. This is personal preference and depends on your screen. Different font sizes
  are available like ter-124, ter-122, ter-116 etc. To use bold font, add b at the end, n for normal. The font we use
  is previously installed terminus-font.
    ```
        echo FONT=ter-128b >> /etc/vconsole.conf
    ```

- Set your hostname, I use ArchPad
    ```
        echo ArchPad >> /etc/hostname
    ```

- Set localhost in the hosts file by entering the following into `/etc/hosts`
    ```
        127.0.0.1 localhost
        ::1 localhost
        127.0.1.1 ArchPad
    ```

### Set up root and users

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
        EDITOR=nvim visudo 
    ```
    Uncomment this line: %wheel ALL=(ALL:ALL) ALL


### Setting up boot

- Configure mkinitcpio
    ```
        HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)
    ```

- Generate initramfs
    ```
        mkinitcpio -p linux
    ```

- Install bootloader
    ```
        bootctl install
    ```

- Go to loader directory
    ```
        cd /boot/loader 
    ```

- Edit the loader.conf file to look like this. With the default arch-* line, we're stating that the default boot 
  should be the entry named arch.
    ```
        default arch-*
    ```

- Lastly we need to create the entry, go to entries directory
    ```
        cd entries 
    ```

- Create file with the name stated in the loader.conf file (default arch-*)
    ```
        touch arch.conf
    ```

- Edit the arch.conf (nvim arch.conf) and have the file looking like this. Incase you installed different kernel,
  you'll see the filenames of linux and initrd by listing /boot directory contents. Options should point to encrypted partition.
    ```
        title	Arch Linux
        linux	/vmlinuz-linux
        initrd	/initramfs-linux.img
        options rd.luks.name=device-UUID=cryptlvm root=/dev/MyVolGroup/root rw rootfstype=ext4 rd.shell=0 rd.emergency=reboot
    ```

### Rebooting
- Enable GDM service so Gnome Display Manager would run after reboot
    ```
        systemctl enable gdm.service
    ```

- Enable audio system service 
    ```
        sudo systemctl enable --global pipewire-pulse
    ```

- Enable network manager as service to connect to internet after reboot
    ```
        systemctl enable NetworkManager
    ```

- Enable bluetooth service 
    ```
        systemctl enable bluetooth.service
    ```

- Start Gnome Keyring service so ssh-agent would be automatically started
    ```
        systemctl --user enable gcr-ssh-agent.socket
    ```

- Exit back to installation iso root
    ```
        exit
    ```

- Unmount mounted volumes 
    ```
        umount -R /mnt
    ```

- Reboot 
    ```
        sudo reboot
    ```

