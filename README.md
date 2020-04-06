# LVM on LUKS with systemd-boot
Encrypts all drives with a single master password, prompted at boot, and supports arbitrary number of disks.
All partitions are LVs, including swap, so everything is guaranteed to be encrypted.

For performance, probably a good idea to have different VGs on drives that are very large and slow vs small and fast.
For reasons of hibernate/sleep resume speed, a swap partition is used and should be located on the fastest available drive.  While it's possible to use a swap file, it's much more complicated to do so and the implementation is brittle.

systemd-boot is incredibly fast, but requires UEFI boot to make use of it.  While it's possible to dual-boot in a UEFI system, it requires running a command before restart to set the next partition to boot.  Additionally, the steps given here are focused on a single-boot system rather than a multi-boot system.  The common "pause with a menu" style of boot (e.g. GRUB) introduces significant boot delay, complexity, and requires entering the disk decryption password twice during boot, so it is avoided.

These instructions are easiest to perform from a livedisk UEFI boot of Arch Linux.  They will require network access, so if your WiFi module isn't automatically detected when booting into the livedisk then you should plan on leaving the system connected to an Ethernet cable until installation is complete.
Zen Arch Linux Installer is a pretty easy livedisk for Arch that supports UEFI.  While the actual GUI-based installation isn't used, the scripts that it uses to perform the installation are used as a guideline in developing these instructions, and the livedisk is very useful for providing a terminal with standard tools (pamac for AUR, web browser, etc) during setup.  

While it's possible to use BTRFS or ZFS instead of LVM for some things, neither plays very well with this use case.  BTRFS for example doesn't support swap partitions.

## References
The following were used as reference for parts of this configuration, layout, and instructions.

https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS

https://wiki.archlinux.org/index.php/Systemd-boot

https://gmpreussner.com/reference/fully-encrypted-archlinux-with-secure-boot-on-yoga-920

https://github.com/spookykidmm/zen_installer/blob/master/zif

https://wiki.archlinux.org/index.php/installation_guide

https://www.tecmint.com/manage-and-create-lvm-parition-using-vgcreate-lvcreate-and-lvextend/

https://wiki.archlinux.org/index.php/Power_management/Suspend_and_hibernate

## Configure your livedisk

If you're using the Zen Arch Installer live image, there are some steps that will need to be done within the live instance.

### Update the livedisk system clock
You must have a reasonably accurate system clock within the livedisk you're running or you won't be able to verify connections to servers.

```
timedatectl set-ntp true
```
Then confirm it is working
```
timedatectl status
```

### Add LVM support
Zen Arch Installer doesn't include the LVM tools, so add them.

```
pacman -Sy lvm2
```
You can ignore the errors that occur at the end stating it's unable to update the mkinitcpio, bootloader, and linux since we aren't expecting the installation to survive a reboot.

# Partitioning disks

## Answer some planning questions
1. Is your boot partition going to be on a USB stick?
This adds some security since you can take it with you even if the computer itself is left somewhere.  You will want to make a backup of the stick once installation is complete, otherwise corruption or loss of the USB stick will prevent you from ever booting the computer.
While you'll only need the USB stick for boot and resume, you'll have to ensure it's attached and mounted properly in the file system whenever you do a system update, or your system will become permanently unbootable.
If you do use a boot USB stick, you'll likely be carrying it around a lot (otherwise what's the point).  Most USB sticks aren't very durable and are apt to get corrupted or ruined.  The Samsung Bar sticks are extremely robust and durable though.  You won't need a very large one, you're only putting a 512MB-1GB partition on it, so they're also pretty cheap. 


2. Are you going to use a detached LUKS header stored in the boot partition?
The encrypted drive(s) will be using LUKS2, which has a header containing some parameters of the encryption and hashes for the passwords necessary to decrypt the contents of the LUKS volume.  While normally included at the start of the encrypted volume, they can also be detached and stored elsewhere.  Without the header the volume cannot be decrypted, so it is a candidate for including on a boot USB stick.  The instructions here will include the directions for installing with a detached header that gets used automatically from a boot USB, and will leave space in the encrypted volume so the header can be re-attached later.
Obtaining a copy of the LUKS header doesn't provide much benefit in decrypting the drive without the password, but does reveal some basic information about the type of encryption used and the number of passwords you have setup to allow decryption of the drive (you can have more than one, but we'll only be setting one).

3. What drives are you going to encrypt, and how are you going to split VGs vs LVs?
Whatever your theoretical partitioning scheme is can be adapted to VGs and LVs.  Usually you don't want to include disks that are very different speeds in a single VG because you will get VG performance that varies between the speed of the faster and speed of the slower devices depending on which it decides to use when storing data.
The simplest, if you're using a single drive, is to have a single VG that includes a swap and root LV.  If you want to separate your home folder or other folders, you'll need to decide how you split the VGs and LVs for them accordingly.
You should also be aware of the performance of your disks when making this decision.  You can check the disk performance with `hdparam -Tt --direct /dev/mybasedrivedevice`.  You should use an entire disk device when calling this, not a partition. 

## Start Partitioning

### Create your boot partition(s)

Arch Linux installation instructions are incredibly unclear about partitioning requirements for booting, likely because there are so many options and it varies based on how the rest of your system is setup.  Our setup is much simpler to configure since we're using a direct UEFI boot into a fully encrypted LVM system, and we're using the systemd-boot.

Since you may be using either a dedicated boot USB stick or an internal drive, the boot device will hereafter be referred to as _devboot_.  If you're using a USB stick and it shows up as `/dev/sdc`, then this is what it refers to.  If you're putting your boot partition on your internal hard drive, it might be `/dev/sda`.

To get an overview of the current state of your disks, run `lsblk`.  This will list a tree-like view of device names, partitions, any LVMs, any mounted encrypted volumes, and the sizes of all of the above.

1. Wipe out all existing content in an unrecoverable manner
```
shred -n 1 -v /dev/devboot
```

2. Create a new GPT partition table on the device
```
cgdisk /dev/devboot
```
You should be prompted to create a new partition table on the device (`shred` made sure of that).  If you aren't prompted, don't worry about it, it may also occur when you create the first partition.

3. Ensure there are no partitions present.
There should be no partitions on the drive since you just shredded it.  If by happenstance there are, double check to make sure you are looking at the correct device, then select and delete all of them.

4. Create the boot partition
* Select the single large free-space entry, and choose New.
* Leave the default starting offset.
* Set the size to either `512MB` or `1G`.  
It's unlikely you will need 1G, 512MB is probably enough, but it doesn't hurt to go bigger if you expect to have a lot of different kernel versions installed.
Do NOT set this to use the full USB stick size, even if you end up with a lot of leftover space on the stick.  If you set this to 16GB, 32GB, or something else very large, it will greatly increase your boot and resume time.
* Set the Partition Type to `ef00` which is `EFI System`.
* Set the name to something, probably "ESP" (EFI System Partition).

5. If your boot partition is not on a USB stick, create a partition for the rest of the space on the drive.
If your boot partition is on a USB stick, do NOT use the USB stick for data storage.  That increases the likelihood of it gettting corrupted, and eliminates most of the benefits of having it on a separate USB stick by allowing other systems to gain access to the device.

* Select the free-space entry at the bottom and choose New
* Leave the default starting offset.
* Leave the default size (all that's left)
* Set the Partition Type to `ef08` which is `Linux LUKS`.
* Give it some name, like "Encrypted".

6. Write out the changes.
The changes you've made up til this point are not on the disk until you write it out.  Select Write at the bottom to make it permanent.  Then you can select Quit.

7. Format the boot partition
The boot partition is required to be FAT32 to be loadable by most systems.
```
mkfs.fat -F32 /dev/devboot
```

### Create your LUKS system partitions

1. Shred any other drives

If your boot partition is on an internal hard drive, you created a partition to eventually hold your LUKS volume and already shredded any data on it.  If you have any additional hard drives you're going to include in your system and/or if you do not have your boot partition on an internal drive, make sure you shred the existing contents of the drives.

```
shred -n 1 -v /dev/mydevicetoshred
```

2. Write random data to drives (optional)

Because the primary system device that will contain the main LVM on LUKS might be a full disk if the boot partition is on a USB stick, or might be a partition on a disk, if the boot partition is on an internal drive, it will hereafter be referred to as _devsys_.  If it's a partition, be sure to use the device file corresponding to the partition and not to the main device.  If it is not a partition, this should be the main device without any partition numbers. 

For additional security, you can write random data to the device(s) that are going to contain encrypted volumes.  This provides some benefit but not much, and has little-to-no effect on some types of disks.  This will take a while and depends on the size of the disk.  Repeat for each device that will be included in the final system.
```
dd if=/dev/urandom of=/dev/devsys bs=10M status=progress
```

4. Create LUKS volume(s)

If you're going to have a detached header(s), create a file for each detached header (one per device).  Pick detached header file names that are distinctive so you can keep them straight later.  Create them as blank binary files to start.  We're temporarily creating them somewhere convenient, and will by moving them into the boot partition later.  We _myheader_ as a placeholder for the filename here, but this will be different for each LUKS volume and will need to be matched with the LUKS volume you're working with in each later step.  
**Warning** If you reboot your livedisk before these header files get moved to the boot partition, you will need to restart 
```
truncate -s 2M ~/myheader
```

On each disk that will be used, format it.  Be sure to use the same password for each volume, or you'll be required to enter one for each device at boot rather than one that gets reused to unlock them all.  If you're not going to use detached headers, leave off the `align-payload` and `header` options, as well as their arguments.  If you are using detached headers, be sure you use the correct filename for _myheader_ that corresponds to the _devsys_ you're formatting.
**Warning** If you reboot your livedisk after this point and before the header files get moved to the boot partition, you will need to start over at this step since you will have lost your headers that are the only mechanism of decrypting the LUKS volumes.
```
cryptsetup luksFormat /dev/devsys --align-payload 4096 --header ~/myheader
```
The `align-payload` is used here to leave space at the start of the disk so we have the option of later reattaching the headers.  

5. Open LUKS volume(s)
To work with the encrypted volumes, they need to be opened.  The name of the volume can be almost anything you want, but should be something you can differentiate from other encrypted volumes (if you have more than one).  Here's it's called _cryptroot_.
If you have detached headers, you have to specify where the detached header is.  If not, then leave off the `header` option and argument.  Make sure you're using the correct header for the drive you're working with too.
```
cryptsetup open --header ~/myheader /dev/devsys cryptroot
```
Remember, _devsys_ is the name of the LUKS volume you're working with, which may be either whole disk or a partition on a disk (if you also have unencrypted content on the same disk). If you're encrypting multiple disks, this device name will be different for each disk.
_myheader_ is the header file you previously created that is specific to the _devsys_ device.
The _cryptroot_ name must be different for each device and sets the mount point of the decrypted LUKS volume.

### Create Your LVM Instance

The instructions are the basics of how to initialize Physical Volumes (PVs), create Volume Groups (VGs), and create Logical Volumes (LVs).  Any instruction manual will suffice.

1. Initialize your PV(s)
It's easiest to perform this step for each LVM encrypted volume you intend to include in your system all at once, regardless of what VG it will end up in.  You can specify a list of devices to the `pvcreate` command. Make sure you use the open LUKS device for this, and not the device that LUKS has encrypted.
```
pvcreate /dev/mapper/cryptroot
```

2. Create your VG(s)
If you want to have multiple PVs in a VG, list all of them when creating the VG.  You can add them later, but it's more complicated to do so and requires a `vgchange`call.  The name you give the VG determines the folder the LVs appear in and will be permanently associated with the VG even in the final installed system.  Here the name used is _archdisk_.
```
vgcreate archdisk /dev/mapper/cryptroot
```

3. Create your swap partition
Ideally your swap disk should be between 0.5 and 1.5 the size of your RAM.  If you have a lot of RAM it might be better to allocate smaller amounts of swap space since you're less likely to use it in the running system other than for hibernate/sleep.  If you have lesser amounts of RAM, you may want your swap space to be larger since it's more likely to get used during regular system usage.
The example here was on a system with 8GB of RAM that doesn't usually encounter extremely RAM heavy tasks.
Keep in mind that the name you give your LV, `swap` in this case, is a permanent part of the VG even once you boot into the final system installation.
```
lvcreate -n swap -L 8G archdisk
```

4. Format and enable your swap partition
The folder _archdisk_ given here is the name of your VG.
```
mkswap /dev/archdisk/swap
swapon /dev/archdisk/swap
```

5. Create your root partition
These steps can be used for creating additional LVs on different VGs.  The main two formats are as a percentage of space left, `-l 100%FREE` or as a fixed size, `-L 150G`.  While it's possible to change the size of these LVs, if you intend to use part of your VG as a LVM thinclient later (allows over provisioning as long as you don't actually overuse), you should leave some free space in the VG.
Keep in mind that the name you give your LV, `root` in this case, is a permanent part of the VG even once you boot into the final system installation.
```
lvcreate -L 400G -n root archdisk
```

6. Format your LV(s)
It's generally not a good idea to put a file system type like ZFS or BTRFS on top of an LVM because ZFS/BTRFS try to perform many of the same activities as LVM.  You're therefore paying the performance costs of ZFS and BTRFS without gaining any of the benefits.  EXT4 is probably your best choice, unless you have a lot of large files that you're reading and writing, and then you may be a candidate for XFS.
```
mkfs.ext4 /dev/archdisk/root
```

### Mount Your partitions
The `/mnt` directory is going to be used as a mountpoint for the root file system that we're going to do an installation into.

1. Mount the rootfs
```
mount /dev/archdisk/root /mnt
```

2. Mount the boot partition
It's possible to mount it elsewhere, but it's much easier if we just mount our EFI boot partition to what will eventually become `/boot`.  In the Arch wiki, this `/boot` is frequently called _esp_.
```
mkdir -p /mnt/boot
mount /dev/devboot
```

3. Mount any other partitions
For any other partitions you've created that you want mounted into your rootfs, create the folder under /mnt and mount it manually.

# Install Arch
The following will do the very initial installation and configuration, but will not make the system bootable yet.

## Rank mirrors for your specific location
The following is a list of possible countries recognized by locale, you will need to pick the one that corresponds to you.
```
all AU AT BD BY BE BA BR BG CA CL CN CO HR CZ DE DK EE ES FR GB HU IE IL IN IT JP KR KZ LK LU LV MK NL NO NZ PT RO RS RU SU SG SK TR TW UA US UZ VN ZA
```

Run the mirror ranking and put it into a file, so it gets used during the installation and once the new system is running.
Replace the value for `COUNTRY`.
```
export COUNTRY=US
curl -s "https://www.archlinux.org/mirrorlist/?country="${COUNTRY}"&protocol=https&use_mirror_status=on" | sed -e 's/^#Server/Server/' -e '/^#/d' | rankmirrors -n 10 - | tee mirrors.txt
cp mirrors.txt /etc/pacman.d/mirrorlist
pacman -Syy
```

## Install base packages
This specifies the tools that will be available in your installed system already.  Feel free to modify by adding or removing relevant packages.
```
pacstrap /mnt base base-devel bash nano vim linux-firmware efibootmgr cryptsetup e2fsprogs findutils coreutils binutils glibc file git ripgrep tar less diffutils grep sed util-linux gawk pciutils procps-ng sysfsutils usbutils lvm2 iproute2 man-db man-pages texinfo zsh xz bzip ed python python2 lua ruby perl libsecret subversion tk gitk lzop lz4 crda 
```

If you're on an Intel system:
```
pacstrap /mnt intel-ucode
```
Otherwise, if you're on an AMD system:
```
pacstrap /mnt amd-ucode
```

## Install Linux
You need to pick a Linux version to follow.  The common choices are the default (`linux`), LTS (`linux-lts`), hardened (`linux-hardened`), or zen (`linux-zen`).  Be aware that the hardened kernel does not allow hibernate/suspend without custom kernel rebuilding.  Zen is considered a "balanced" kernel configuration optimized for "most desktop and laptop users".  Replace `linux-zen` in the package names below with the relevant variant.

```
pacstrap /mnt linux-zen linux-zen-headers
```

## Configure the system

### Setup fstab
1. Generate it
This fstab will get modified, but is a good starting point to work from.
```
genfstab -Up /mnt >>/mnt/etc/fstab
```

2. Make /tmp use RAM
Change /mnt/etc/fstab to the following line for /tmp, adding it if it's not present.
```
tmpfs     /tmp         tmpfs  defaults,noatime,mode=1777  0 0
```

3. Disable automatic mounting of boot USB (optional)
If you are using a USB stick to contain your boot partition, you may not want it to automatically mount in your system.  This allows you to remove the USB stick as soon as the system has booted without first having to manually unmount it.
If you choose for the boot partition to not auto-mount, you will need to be sure to manually mount it before you do system updates.

Add the `noauto` to the list of parameters for the fstab entry associated with `/boot`.

4. Specify disk partition by-id for USB boot
If you're boot partition is on a USB stick, you can't rely on the automatic device name assignment to be consistent.  You need to specify it by drive ID instead.

* Determine the device name (e.g. /dev/sda1)
* Find the matching disk by-id name
Look for the disk by-id name that is a symlink pointing ot the matching device.
```
ls -l /dev/disk/by-id/* | grep "sda1"
```
Disk by-id is a way to guarantee you're always referencing a specific device.  Linux will automatically create the same name in the disk by-id no matter the version of Linux for a given device.
* Replace the lettered device (e.g. `/dev/sda1`) with the by-id name (e.g. `/dev/disk/by-id/usb-Samsung_Flash_Drive_0373119040006771-0:0-part1`) in the fstab entry for `/boot`.

The resulting line in the fstab should look something like the following
```
# systemd-boot /boot
/dev/disk/by-id/usb-Samsung_Flash_Drive_0373119040006771-0:0-part1             /boot            vfat            rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8,errors=remount-ro       0 2
```

5. Fix LV device references
For each of the LVs, there are multiple ways to reference the device.  The only consistently guaranteed one is directly within the `/dev/mapper` folder.
In the case of the swap partition, the following should be used, where the device node name is always comprised of the VG name and the LV name.
```
/dev/mapper/archdisk-swap
```

6. Encryption password timeout configuration
Mounting of drives listed in fstab has a default timeout.  If the LUKS volume isn't decrypted within that time, the mounting will fail and won't retry.
While the actual encryption password timeout is set elsewhere (discussed later), the timeout in the fstab must be equal or greater than the password timeout.  The easiest is to set the timeout on mounting to infinite time, and control any password timeout with the password entry timeout.

For each partition that is mounted from a LUKS volume, add the following parameter to the mount command `x-systemd.device-timeout=0`.  For the swap and root partitions, this should look something like the following:
```
/dev/mapper/archdisk-root      /               ext4            rw,relatime,x-systemd.device-timeout=0  0 1

# /dev/mapper/archdisk-swap
/dev/mapper/archdisk-swap      none            swap            defaults,x-systemd.device-timeout=0             0 0
```

### Timezone, localization, hostname, and root password

* Enable what locales are available
Edit the /etc/locale.gen file on your livedisk and uncomment the locales you might want to use.  The minimum is probably `en_US.UTF-8 UTF-8`.

* Generate the locales
This makes them available to be used from your livedisk, and therefore possible to include in your target system.
```
locale-gen
```

* Configure the new system locale and timezone
While the Arch wiki explains how to do these one at a time, it's actually preferred to use the tool explicitly for this purpose `systemd-firstboot` for most of the locale configuration.

```
systemd-firstboot --root=/mnt --prompt
```
This will prompt you to pick a locale from a list, and a timezone from a list, as well as to set a hostname.  

* Set the hosts file
Edit the /mnt/etc/hosts file and add the following, replacing _myhostname_ with the name you picked during the `systemd-firstboot` prompt.
```
127.0.0.1	localhost
::1		localhost
127.0.1.1	myhostname.localdomain	myhostname
```

* Switch into the new target system via chroot
```
arch-chroot /mnt /bin/bash
```

* Set the root password
```
passwd
```

* Set the hardware clock
```
hwclock -l --systohc
```
Note that this will set the hardware clock to UTC.  Using UTC for the hardware clock is strongly preferred under Linux, but is not compatible with Windows.  Since we aren't discussing how to dual-boot with Windows though, it's merely something to be aware of.

* Exit the chroot
```
exit
```

## Install more system packages
Some basic system packages need the locale and and fstab configured first.  The following are best installed after.

This includes desktop, sound, and video drivers.

```
pacstrap /mnt mesa xorg-server xorg-apps xorg-xinit xorg-twm xterm xorg-drivers alsa-utils pulseaudio pulseaudio-alsa xf86-input-synaptics xf86-input-keyboard xf86-input-mouse xf86-input-libinput b43-fwcutter polkit-gnome ttf-dejavu gnome-keyring xdg-user-dirs gvfs
```

### Install a desktop and window manager

The following are pretty standard sets of package options for various desktops and window managers.
```
gnome
gnome gnome-extra
plasma
plasma kde-applications
xfce4
xfce4 xfce4-goodies
lxde
lxqt
mate
mate mate-extra
budgie-desktop
budgie-desktop gnome
cinnamon
deepin
deepin deepin-extra
enlightenment
jwm
i3-wm i3status i3lock
i3-gaps i3status i3lock
openbox tint2
```
For example:
```
pacstrap /mnt xfce4 xfce4-goodies
```

The default association of display manager for each desktop is:
 
| :------------: | :------------------------: |
| gnome          | gdm |
| budgie-desktop | lightdm |
| lxde           | lxdm |
| plasma         | sddm |
| Others         | lightdm |

### Install a display manager
The following are pretty standard display managers used to handle user login.

#### lightdm
* Install it.
```
pacstrap /mnt lightdm lightdm-gtk-greeter-settings lightdm-gtk-greeter
```
If using budgie-desktop, the following additional packages are also added.
```
pacstrap /mnt gnome-control-center gnome-backgrounds
```

* Enable it
```
arch-chroot /mnt /bin/bash -c "systemctl enable lightdm.service"
```

#### lxdm
* Install it.
```
pacstrap /mnt lxdm-gtk3
```
* Enable it
```
arch-chroot /mnt /bin/bash -c "systemctl enable lxdm.service"
```

#### sddm
* Install it.
```
pacstrap /mnt sddm
```
* Enable it
```
arch-chroot /mnt /bin/bash -c "systemctl enable sddm.service"
```
#### gdm
* Install it
```
pacstrap /mnt gdm
```

* Enable it
```
arch-chroot /mnt /bin/bash -c "systemctl enable gdm.service"
```

### cups support
* Install it.
```
pacstrap /mnt ghostscript gsfonts system-config-printer gtk3-print-backends cups cups-pdf cups-filters
```

* Enable It
```
arch-chroot /mnt /bin/bash -c "systemctl enable org.cups.cupsd.service"
```

### Network manager
* Install it.
```
pacstrap /mnt networkmanager nm-connection-editor network-manager-applet
```

* Enable it.
```
arch-chroot /mnt /bin/bash -c "systemctl enable NetworkManager"
```

### Add revenge-repo to AUR list
Adds lines from the command-line directly to the end of the existing file, then updates the repo list.
```
cat >> /mnt/etc/pacman.conf <<EOF

[revenge_repo]
SigLevel = Optional TrustAll
Server = https://gitlab.com/spookykidmm/revenge_repo/raw/master/x86_64
Server = https://downloads.sourceforge.net/project/revenge-repo/revenge_repo/x86_64
EOF

arch-chroot /mnt /bin/bash -c "pacman -Syy"
```

### Add spooky-aur to AUR list 
Adds lines from the command-line directly to the end of the existing file, then updates the repo list.
```
cat >> /mnt/etc/pacman.conf <<EOF

[spooky_aur]
SigLevel = Optional TrustAll
Server = https://raw.github.com/spookykidmm/spooky_aur/master/x86_64
EOF

arch-chroot /mnt /bin/bash -c "pacman -Syy"
```

### Install pamac-aur and yay
Requires the spooky-aur to be added already

```
arch-chroot /mnt /bin/bash -c "pacman -S --noconfirm pamac-aur"
arch-chroot /mnt /bin/bash -c "pacman -S --noconfirm yay"
```

### Add multilib to AUR list
Adds lines from the command-line directly to the end of the existing file, then updates the repo list.
```
cat >> /mnt/etc/pacman.conf <<EOF

[multilib]
Include = /etc/pacman.d/mirrorlist
EOF

arch-chroot /mnt /bin/bash -c "pacman -Syy"
```

### Install different shells
Any/all of the following sets of shells can be installed and frequently include the additional listed packages.
```
zsh zsh-syntax-highlighting zsh-completions grml-zsh-config
bash bash-completions
fish
```

### Configure sudo access
The wheel user is usually allowed to perform sudo.
```
echo "%wheel ALL=(ALL) ALL" >> /mnt/etc/sudoers
```

## Install extra packages
For example, if you need the Broadcom STA wireless driver for your WiFi chip, you can add `broadcom-wl-dkms` via a pacstrap command.

```
pacstrap /mnt broadcom-wl-dkms
```

This is the point where you might want to pick a desktop environment to use, and install other software.


# Setup Initramfs
The initramfs is loaded right at kernel start and is used to bootstrap the rest of the system.  The initramfs has the boot partition mounted, but is primarily controlled through the /etc/mkinitcpio.conf file.

## Save LUKS headers
If you're using detached headers for your LUKS volumes, copy them into /mnt/boot.

## mkinitcpio
Each of the following sections has you making changes to the /mnt/etc/mkinitcpio.conf file.

### Add modules 
Modify the `MODULES` variable to be the following, adding to anything that's already there.  Each module should be space separated in the list.  Leave off the `i915` if you're not on an Intel system.
```
MODULES=(i915 loop)
```

### Add files
Modify the `FILES` variable to list all the detached LUKS headers you've coped to /mnt/boot.  Space separate fully-pathed file names, where the file path should always start with `/boot/` since that will be the path to them once we boot into the new system.
```
FILES=(/boot/myheader)
```

### Add hooks
The `HOOKS` variable references packaged hooks used during startup.  Order of these matters, so pay attention.  They should be added to or moved within the list of hooks, and may require removing another hook they conflict with.

A final result of this section might look like the following.
```
HOOKS=(base systemd sd-plymouth autodetect keyboard sd-vconsole modconf block sd-encrypt sd-lvm2 uresume filesystems fsck)
```

#### systemd
`systemd`
Should be second, right after `base`, and must be before `autodetect`.
Remove `udev` if it is present.

This performs device detection and allows all the `sd-*` hooks to function.

#### keyboard
`keyboard`
Should be right after `autodetect`, and before `sd-vconsole`.

#### sd-vconsole
`sd-vconsole`
Should be after `keyboard`, and before `sd-encrypt`.
Remove `keymap` and `consolefont` if either is present.

Must come after the `keyboard` since it configures the keyboard, and before the `sd-encrypt` since it's needed to be able to enter the LUKS password.

vconsole requires some extra configuration.  The file was created with the keymap specification when `systemd-firstboot` was called, but the font to use was not set.  
The font should be picked from the `/mnt/usr/share/kbd/consolefonts/` folder from among the *.psf.gz and *.psfu.gz files.  
The font is specified by adding lines like the following to the file `/mnt/etc/vconsole.conf`.
```
FONT=Lat2-Terminus16
```
This example corresponds to the `Lat2-Terminus16.psfu.gz` file.

#### sd-encrypt
`sd-encrypt`
For our LVM on LUKS, must come before `sd-lvm2`, and after `systemd`.
Remove `encrypt`.

Comes before the `sd-lvm2` because the devices must be decrypted before the VGs can be used.

#### sd-lvm2
`sd-lvm2`
For our LVM on LUKS, must come after `sd-encrypt` and before `filesystems`.
Remove `lvm2`.

Comes after the `sd-encrypt` hook because the VGs are on devices that are encrypted and therefore must be decrypted first.  Must come before the filessystems because the LVs must be available before the root file system can be mounted.

#### uresume
`uresume`
Must come after `sd-lvm2` and `sd-encrypt`, but before `filesystems`.

This is used if hibernate/sleep support is enabled.  It will do nothing on the linux-hardened kernel that doesn't support hibernate/suspend.  It must come after the `sd-lvm2` and `sd-encrypt` because the swap partition where the resume data is stored must first be decrypted and then the LV mounted.

Must come before the filesystem since it sets the state for the running system.

### Regenerate mkinitcpio
The following regenerates the initramfs for all installed kernel presets.

```
mkinitcpio -P
```

If you need to specify only a single preset configuration, you can do so with the following, specifying the linux type you selected to install.

```
mkinitcpio -p linux-zen
```

## Crypttab.initramfs
Start by copying `/mnt/etc/crypttab` to `/mnt/etc/crypttab.initramfs`.

The crypttab.initramfs specifies decryption options for disks that need to be decrypted in the initramfs as part of the `sd-encrypt` mkinitcpio hook.
The entries in the file correspond to the arguments that are passed to a `cryptsetup open` command.

Much like the fstab, the device used should always be from /dev/disk/by-id.

The resulting lines should look something like the following.
```
cryptroot       /dev/disk/by-id/ata-TS256GMTS400_C887531295-part1       none                  header=/boot/myheader,timeout=0
```

### Add the cryptroot
Only the LUKS volumes necessary for minimum functionality of a system need to be specified here.  Primarily this is the LUKS volume(s) used in the VG you have the root LV and swap LV in.  If the VG that contains these LVs is spread across multiple PVs, you need to make sure all of the LUKS encrypted PVs are listed in this file.

* Pick a name for the decrypted volume.  Here it is _cryptroot_.
This is the name of the decrypted LUKS device created in /dev/mapper.  Make sure you use the same name as what you used when creating the VGs for consistency.  It's unclear whether this might cause errors if you use different names.

* Look up the disk/by-id/
If you created the root file system in a LUKS volume on /dev/sda
```
ls -l /dev/disk/by-id/* | grep "sda"
```
If you created it on /dev/sda2, grep for `sda2` instead.

* Set the password to `none`, which corresponds to prompting the user.
If you have a keyfile, list it with an absolute path instead.  This is the equivalent of having your password in a text file, but if you aren't worried about malicious access to your boot USB stick you can generate a keyfile (not covered here) and put it on the boot partition, then use it during boot instead of requiring manual password entry.

* If you have a detached header file, include it in the options column of comma-separated arguments as `header=/boot/myheader`.
If you don't have a detached header, leave the options blank or unchanged.

### Set the password entry timeout
The default when the password field is set to `none` is to prompt the user.  The prompt has a default timeout of 120 seconds, but can be configured to a different timeout or disabled entirely.

To disable it entirely, add `timeout=0` to the options column of comma-separated arguments.

The timeout can be set in seconds `s`, minutes `m`, or hours `h` like `timeout=600s` or `timeout=5m`.

### Regenerate mkinitcpio after cryptab.initramfs modifications
After the crypttab.initramfs is modified, the initramfs needs to be regenerated.

```
mkinitcpio -P
```

# Bootloader
Start by chrooting into the system
```
arch-chroot /mnt /bin/bash
```

## Install systemd-boot
```
bootctl --path=/boot install
```
This installs the identical binaries, `/boot/EFI/systemd/systemd-bootx64.efi` and `/boot/EFi/BOOT/BOOTX64.EFI` (on a 64-bit system), and transfers them to the `/boot/ESP`.  It then sets systemd-boot as the default EFI application (default boot entry) loaded by the EFI Boot Manager.

## Enable Automatic update of systemd-boot
While any working version of the systemd-boot can be used, it's usually better to keep it updated.  Without this, even if the systemd package gets updated then the systemd-boot won't be updated.

Be aware that an AUR capable installer is needed.
```
yay -Sy systemd-boot-pacman-hook
```
This effectively adds the following pacman hook to `/etc/pacman.d/hooks/100-systemd-boot.hook`
```
[Trigger]
Type = Package
Operation = Upgrade
Target = systemd

[Action]
Description = Updating systemd-boot
When = PostTransaction
Exec = /usr/bin/bootctl update
```

## Configure loader
The loader is used to configure the properties for a grub-like boot menu based on UEFI.  It is configured in file `/boot/loader/loader.conf`.

The following are good default options, where commented out lines are already the defaults.
```
timeout 0
#console-mode keep
default arch*
editor no
#auto-entries 1
#auto-firmware 1
```
The `timeout` sets the number of seconds the menu is visible before the default selection is made.  Pressing a key in the loader halts the timeout, so even with a 0 timeout a user can rapidly press a key (e.g. up or down) to get the menu to stop and allow selection.

The `console-mode` sets the resolution of the menu.  It has a few fixed formats, or some automatic ones.  `auto` picks the most suitable, `max` picks the highest available, and `keep` (the default) picks the one the firmware selects.

The `default` sets the default selection from in the menu.  The argument can be a wildcard or a fixed value, and picks the first match from the `/boot/loader/entries/` folder (more about adding entries later).

The `editor` sets whether kernel parameters can be edited before being run.  Allowing this can be useful for debugging, but is a major security vulnerability.  Setting it to `no` disables this option, or `yes` to enable it.

The `auto-entries` sets whether automatic entries for Windows, EFI Shell, and Default Loader are shown. `1` (default) to show, `0` to hide.

The `auto-firmware` sets whether entries should be presented for rebooting into UEFI firmware settings.  `1` (default) to show, `0` to hide.

Additional arguments are available and can be found by reading `man 8 loader.conf`.

## Add a loader entry
In order to have something to boot, you need to manually create an entry for the bootloader.

Create a conf file in the `/boot/loader/entries` file.  
The name you give the file is used by the `default` argument in the `/boot/loader/loader.conf` file described above.
A decent name would be `arch-zen.conf` for a linux-zen install for example.

The loader entry file contains some bare minimum required settings.

An example of this file is the following.
```
title Arch Linux
linux /vmlinuz-linux-zen
initrd /intel-ucode.img
initrd /initramfs-linux-zen.img
options root=/dev/archdisk/root rootflags=x-systemd.device-timeout=0 resume=/dev/archdisk/swap apparmor=1 security=apparmor audit=1 no_console_suspend i915.enable_rc6=1 i915.enable_fbc=1 i915.lvds_downclock=1 i915.semaphores=1 quiet splash loglevel=3 rd.systemd.show_status=auto rd.udev.log_priority=3 vt.global_cursor_default=0 rw
```

The `title` is required, and is the text to show in the menu.

The `linux` is a nicer alternative to `efi` that can be used with `initrd`.  It specifies the vmlinuz image from the /boot folder to load.  These vmlinuz images are created by the kernel installer (usually by pacman).

The `initrd` can appear multiple times, and lists images to load from /boot for the initramfs.  The `intel-ucode.img` or `amd-ucode.img` should be specified first, then the initramfs file that matches the vmlinuz selected by the `linux` argument.

The `options` is used to specify the arguments to the kernel.

While it's possible to manually create a custom efi image from the kernel that combines the vmlinuz, initrd, and the boot arguments and then use them via the `efi` setting, it's not covered here.

### Boot arguments
The kernel command-line arguments are frequently referenced as `GRUB_LINUX_CMDLINE`, and the format is the same.

Some of these arguments are required for basic functionality given our configuration.

#### Rootfs arguments
```
root=/dev/archdisk/root rootflags=x-systemd.device-timeout=0 rw
```
These arguments set the device to boot as the root file system.  Because the rootfs is an LV, we can guarantee the name of the VG-created device node and use it here.
As described in the fstab section above, we want to eliminate any timeout on mounting the rootfs so we can independently configure the timeout for the decryption password without having to worry about the mounting of the device timing out before we've even reached the password timeout.
We would like the rootfs to be writeable.

#### Resume support
```
resume=/dev/archdisk/swap
```
Hibernate/suspend/sleep support requires the mkinitcpio hook `uresume` and makes use of the `resume` command-line argument.  It should point to the LV swap partition.

#### Security
```
apparmor=1 security=apparmor audit=1
```
Enables apparmor support.  One option for a security setting.

#### Limited visible logging
```
no_console_suspend quiet splash loglevel=3 rd.systemd.show_status=auto rd.udev.log_priority=3 vt.global_cursor_default=0
```
If you choose to use an `sd-plymouth` mkinitcpio hook, which uses an interactive boot splash screen, you'll want to limit extraneous visible logging.

#### Faster boot speed
```
i915.enable_rc6=1 i915.enable_fbc=1 i915.lvds_downclock=1 i915.semaphores=1
```
If you're using an Intel processor, the following arguments usually reduce boot time.  It requires the `i915` mkinitcpio module.

# Crypttab
Encrypted LUKS volumes that aren't necessary for boot should have entries added to the `/mnt/etc/crypttab`.  This will ensure they are decrypted before fstab is run.

If they contain a VG, the VG will be detected automatically and the fstab only needs to mount the LVs.

The format is the same as the Crypttab.initramfs file described previously.

# Add user
* Enter the chroot
```
arch-chroot /mnt /bin/bash
```

* Create the username with group associations
Set _myusername_ to the user you'd like to setup, and replace `/usr/bin/zsh` with the default shell you'd like for the user (e.g. `/bin/bash`).
```
useradd -m -g users -G adm,lp,wheel,power,audio,video -s /usr/bin/zsh myusername
```
Be aware that default installation of Arch Linux adds all users to the `users` group rather than creating a unique group for each user that matches their username (e.g. `-g myusername`).

* Set the user password
```
passwd myusername
```

# Configure sleep/hibernate
It's handled by systemd, acpid, laptop-tools, and tlp.
However it can also be configured for much faster support if you configure uswsusp.

You'll have to boot into the new system before you can configure it.
```
yay -Sy uswsusp-git
```

Edit `/etc/suspend.conf`, common settings are:
```
snapshot device = /dev/snapshot
resume device = /dev/archdisk/swap
#image size = 350000000
#suspend loglevel = 2
compute checksum = y
#compress = y
threads = y
#encrypt = y
early writeout = y
#splash = y
```
The important values are the `resume device`, which needs to be the swap LV, `threads` which should be set to `y` to use multiple threads to suspend and resume faster, `early writeout` to `y`.
