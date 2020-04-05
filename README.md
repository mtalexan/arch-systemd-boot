# LVM on LUKS with systemd-boot
Encrypts all drives with a single master password, prompted at boot, and supports arbitrary number of disks.
All partitions are LVs, including swap, so everything is guaranteed to be encrypted.

For performance, probably a good idea to have different VGs on drives that are very large and slow vs small and fast.
For reasons of hibernate/sleep resume speed, a swap partition is used and should be located on the fastest available drive.

systemd-boot is incredibly fast, but requires UEFI boot to make use of it.  While it's possible to dual-boot in a UEFI system, it requires running a command before restart to set the next partition to boot.  Additionally, the steps given here are focused on a single-boot system rather than a multi-boot system.  The common "pause with a menu" style of boot (e.g. GRUB) introduces significant boot delay, complexity, and requires entering the disk decryption password twice during boot, so it is avoided.

These instructions are easiest to perform from a livedisk UEFI boot of Arch Linux.  They will require network access, so if your WiFi module isn't automatically detected when booting into the livedisk then you should plan on leaving the system connected to an Ethernet cable until installation is complete.
Zen Arch Linux Installer is a pretty easy livedisk for Arch that supports UEFI.  While the actual GUI-based installation isn't used, the scripts that it uses to perform the installation are used as a guideline in developing these instructions, and the livedisk is very useful for providing a terminal with standard tools (pamac for AUR, web browser, etc) during setup.  

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

3. Create LUKS volume(s)

If you're going to have a detached header(s), create a file for each detached header (one per device probably).  Pick detached header file names that are distinctive so you can keep them straight later.  Create them as blank binary files to start.
```
truncate -s 2M ~/myheader
```

On each disk that will be used, format it.  Be sure to use the same password for each volume, or you'll be required to enter one for each device at boot rather than one that gets reused to unlock them all.  If you're not going to use detached headers, leave off the `align-payload` and `header` options, as well as their arguments.
```
cryptsetup luksFormat /dev/devsys --align-payload 4096 --header ~/myheader
```
The `align-payload` is used here to leave space at the start of the disk so we have the option of later reattaching the headers.  

