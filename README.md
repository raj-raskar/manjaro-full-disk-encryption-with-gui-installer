
# Install Manjaro with Full-Disk Encryption (Using default Calamares Installer)
## Encryptd Boot, LVM on Luks
Manjaro comes with Calamares GUI Installer and Architect CLI installer. The Calamares is easy to use but the partition layout are not very customizable. So in this tutorial we are going to install Manjaro with GUI installer with custom partition layout.
### 1)Booting into Manjaro Live Disk
Create Live Manjaro USB and boot into the live system.
### 2)Edit `/lib/calamares/modules/mount/main.py`
After booting into live system, run following command as sudo:

    sudo nano /lib/calamares/modules/mount/main.py

Append the following code in `/lib/calamares/modules/mount/main.py`

Code:

    #-------------------------------------------------
        with open("/tmp/calamares_block", 'w') as fp:
            pass
    
        while True:
            if os.path.exists("/tmp/calamares_block"):
                pass
            else:
                break
    #-------------------------------------------------

![edit_calamares](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/ba5b3640-edd5-47aa-b50f-70aa59cf2544)
#### (Note: Don't forget to use proper Indentations.)
This code stops the installer after mount step, then we can create custom mountpoints and partition layout.
### 3)Start the GUI Installer
 Launch GUI Installer.

![launch_installer](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/215dea73-4004-4524-b0dd-948e9ee824e9)

Select Location & Keyboard layout.

At Partitions step select manual partitioning, create 256MB ext4 dummy partation and mount at /

![manual_partitioning](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/86361b82-40bf-4c84-bec1-7bf11febff84)

![dummy_partition](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/31c6060e-b1d1-4209-8804-cc8dcb314e7d)

Click on Next & Ignore 'No EFI partation'.

![ignore_no_efi_part](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/78b6ff48-574b-4dc9-b211-025dbcc4a49f)

Continue upto Summary & Click Install.

![summary](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/dd5b2d8c-5976-4ddf-b915-5a0c43cc25ca)

Installation will stop at 'Mounting Partitions'

![calamares_block](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/f17833bf-5192-474d-a777-7c036f5e5047)

Now check wether `/tmp/calamares_block` file exists.

Run:

    ls /tmp/calamares_block

Now Installation stopped at this stage, you can perform manual partitioning & mounting.
#### (Note: Don't close the installer.)
## 4)Unmount Dummy Partation
List `/tmp` directory, there will be `/tmp/calamares-root-*` directory.

![list_tmp](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/ef00d5e2-9f51-49a7-a429-578f360d42c3)

Run:

    sudo umount /tmp/calamares-root-*/{sys/firmware/efi/efivars,sys,run/udev,run,proc,dev}
    sudo umount /tmp/calamares-root-*

This will unmount the dummy root partition.
#### (Note: Don't delete `/tmp/calamares-root-*` directory, it is needed for future mount point.)

## 5)Create Partitions
For creating partitions we are going to use 'GParted', it comes pre-installed with iso.

Launch GParted

![gparted_launch](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/0ce4b788-4d5a-46f5-8d7b-ca20d2ba4a8d)

#### Delete 256MB ext4 Dummy partition
#### Create partitions as below : 
- 256MB fat32 `EFI System Partation`
- 1024MB `Unformated Partation`
- 3rd partation from remaining space `Unformated Partation`

![gparted_layout](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/1abac7dc-5dcd-420b-8c1f-d78340cee4b9)

#### (Note: Don,t forget to set `boot,esp` flags on efi partation.)
## 6)Formatting Partation
Mine disk name is `/dev/vda`, your disk name may be diffrent. Modify commands accrodingly.

List Partations.

Run:

    lsblk

![lsblk_1](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/cf044dff-b188-4337-9054-45ef450d113d)

Encrypt 2nd i.e. `boot` partition with lusk1. Currently grub dose not support luks2 encryption on boot partition.

Run:

    sudo cryptsetup luksFormat --type luks1 /dev/vda2

Encrypt 3rd partition with lusk2.

Run:

    sudo cryptsetup luksFormat /dev/vda3

![encrypt_part](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/07b2bc16-5796-4c94-b76f-96aef39b8fa5)

You can use same or diffrent passphrase.

Now open luks encrypted partitions, 2nd as `boot` & 3rd as `cryptlvm`.

Run:

    sudo cryptsetup open /dev/vda2 boot

Run:

    sudo cryptsetup open /dev/vda3 cryptlvm

New partition layout will look like.

![lsblk_2](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/188e677f-2804-45de-8a44-8cb2dbd8c83e)

Create a physical volume on top of `cryptlvm`.

Run: 

    sudo pvcreate /dev/mapper/cryptlvm

Create a `vg0` volume group on previously created physical volume.

Run:

    sudo vgcreate vg0 /dev/mapper/cryptlvm

Create all your logical volumes on the volume group `vg0`.

Run:

    sudo lvcreate -L 2G vg0 -n swap
    sudo lvcreate -l 100%FREE vg0 -n root

I am creating 2GB `swap` and remaining `root` subvolumes on `vg0`.

![subvol_created](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/72f8cbb5-9314-45b5-bdef-73c75a740f31)

#### Format the partitions accrodingly
- `/dev/mapper/vg0-root` ext4
- `/dev/mapper/boot` ext4
- `/dev/mapper/vg0-swap` swap

Run:

    sudo mkfs.ext4 /dev/mapper/vg0-root
    sudo mkfs.ext4 /dev/mapper/boot
    sudo mkswap /dev/mapper/vg0-swap

#### Mount the partitions
- `/dev/vda1` esp
- `/dev/mapper/vg0-root` root
- `/dev/mapper/boot` boot
- `/dev/mapper/vg0-swap` swap

#### Before running following code replace `export ROOT=/tmp/calamares-root-xxxxxxxx` with actual directory name.

Run:

    export ROOT=/tmp/calamares-root-xxxxxxxx

Run:

    sudo mount /dev/mapper/vg0-root $ROOT
    sudo mkdir $ROOT/boot
    sudo mount /dev/mapper/boot $ROOT/boot
    sudo mkdir $ROOT/boot/efi
    sudo mount /dev/vda1 $ROOT/boot/efi
    sudo swapon /dev/mapper/vg0-swap
    sudo mkdir -p $ROOT/{dev,proc,run,sys}
    sudo mount -t devtmpfs dev $ROOT/dev
    sudo mount -t proc proc $ROOT/proc
    sudo mount -t tmpfs tmpfs $ROOT/run
    sudo mount -t sysfs sys $ROOT/sys
    sudo mkdir -p $ROOT/{run/udev,sys/firmware/efi/efivars}
    sudo mount -t tmpfs run -o rw,nosuid,nodev,mode=755 $ROOT/run/udev
    sudo mount -t efivarfs efivarfs $ROOT/sys/firmware/efi/efivars

After mounting all partitions layout will look like

![after_mount_lsblk](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/2eb2fb2e-8c23-4778-beed-0970b49aa4d2)
## 7)Continue the GUI Installer
After all partitions mounted sucessfully, we can continue the installation.

To continue install just delete `/tmp/calamares_block`.

Run:

    sudo rm /tmp/calamares_block

![all_done](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/97a4eb5b-0841-4fad-858c-d7727802d464)

#### After Complition of installation don't reboot. We need to configure system.

## 8)Final Configuration
For final system configuration we need arch-install-script. I reccomend you to manually download & install the package

Download Link: https://archlinux.org/packages/extra/any/arch-install-scripts/download/

To install run:

    cd ~/Downloads
    sudo pacman -U arch-install-scripts*.tar.zst

Remount partitions.
#### Before running following code replace `export ROOT=/tmp/calamares-root-xxxxxxxx` with actual directory name.

Run:

    export ROOT=/tmp/calamares-root-xxxxxxxx

Run:

    sudo mount /dev/mapper/vg0-root $ROOT
    sudo mount /dev/mapper/boot $ROOT/boot
    sudo mount /dev/vda1 $ROOT/boot/efi
    sudo swapon /dev/mapper/vg0-swap

![remount](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/bd355166-24e0-4b73-b4da-d1a28131c397)

chroot in new installation.

Run:

    sudo arch-chroot $ROOT

#### *From now all commands are executed in chroot unless stated. Also there is no need of `sudo` to execute command, by default you are root user in chroot.

HOOKS=(base **udev** autodetect modconf kms **keyboard keymap consolefont** block **encrypt lvm2** filesystems fsck)

Add these hooks in `/etc/mkinitcpio.conf`

Run:

    nano /etc/mkinitcpio.conf

![hooks_mkinitcpio](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/e675c9db-26c7-464f-baed-487aa4e65ac4)

#### Avoiding having to enter the passphrase twice

Run:

    dd bs=512 count=4 if=/dev/random of=/root/cryptlvm.keyfile iflag=fullblock
    chmod 000 /root/cryptlvm.keyfile
    cryptsetup -v luksAddKey /dev/vda3 /root/cryptlvm.keyfile

![create_keyfile](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/426a9c46-3f21-4f5b-8145-6822c493f819)

#### Also add key for boot partition.

Run:

    cryptsetup -v luksAddKey /dev/vda2 /root/cryptlvm.keyfile

![boot_addkey](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/600b8638-d2c9-4a5c-8842-caf14b9144f9)

#### Add the keyfile to the initramfs image.

**FILES=(/root/cryptlvm.keyfile)**

Run:

    nano /etc/mkinitcpio.conf

![initramfs_add_file](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/8851e2a7-3a1c-4f34-9016-d8f2629d0ca7)

#### Edit `/etc/crypttab`
Add the following line.

**boot    UUID=UUID-identifier    /root/cryptlvm.keyfile**

Run:

    nano /etc/crypttab

![crypttab](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/d240708f-bdb7-4dc9-91b0-f3370a1d7981)

#### Configure GRUB bootloader.

To edit GRUB default config run:

    nano /etc/default/grub

Edit following lines.

**GRUB_ENABLE_CRYPTODISK=y**

**GRUB_CMDLINE_LINUX="... cryptdevice=UUID=device-UUID:cryptlvm cryptkey=rootfs:/root/cryptlvm.keyfile root=/dev/mapper/vg0-root"**

To find UUID run:

    blkid | grep crypto_LUKS

![grub_config](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/a84ecd8e-8a09-4ab9-a646-08ac62ef4484)

**Don't  forget to comment out `GRUB_THEME="/usr/share/grub/themes/manjaro/theme.txt"`.**

This is important because the default manjaro GRUB theme is localted in luks2 encrypted root partition, this causes to GRUB trying to unlock luks2 partition, fails & show false 'Enter Passphrase' prompt for root partition.

![grub_unset_theme](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/ed041255-8fe2-452b-a990-ca1fb7c3de63)

#### Genrate initramfs
Recreate the initramfs image and secure the embedded keyfile:

Run:

    mkinitcpio -P
    chmod 600 /boot/initramfs-*

#### Install GRUB & genrate config

Run:

    grub-install --target=x86_64-efi --bootloader-id=Manjaro --efi-directory=/boot/efi
    grub-mkconfig -o /boot/grub/grub.cfg

![grub_install_success](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/5eb7a9e4-1daf-4ab0-8e3f-89e88a7b33a2)

#### Exit the charoot

Run:

    exit

#### *Now we exited from chroot

#### Genrate `/etc/fstab`

#### Before running following code replace `export ROOT=/tmp/calamares-root-xxxxxxxx` with actual directory name.

Run:

    export ROOT=/tmp/calamares-root-xxxxxxxx

On host run:

    sudo su -c "genfstab -U $ROOT > $ROOT/etc/fstab"

![fstab](https://github.com/raj-raskar/manjaro-full-system-encryption/assets/134035279/78b4d928-804b-4cc6-a6ac-1b7d503f58a7)

#### The final command

Run:

    sudo umount -a
    reboot
