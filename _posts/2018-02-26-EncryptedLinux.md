---
layout: post
section-type: post
title: Dual Boot Win10 & Linux with Full Disk Encryption 
category: Linux
tags: [ 'Encryption' ]
---

This details the set-up I performed to dual boot windows and Linux with full disk encryption (LUKS encrypted **root/home/swap**, 
though not boot - [Evil Maids](https://www.schneier.com/blog/archives/2009/10/evil_maid_attac.html) don't scare me!)

I had initially planned to install Debian, though settled for Linux Mint as Debian still didn't support Secure Boot, and after
some thought, I decided to go ahead with Mint as it's been a straight forward setup whenever I've used it in the past.

## Setting up the encrypted volumes

The first set I took was to run GParted from a Live USB stick and add the partitions I required, four in total: boot, root,
swap, home.  At this stage all unencrypted:

![GParted](/img/2018/20180226_GParted.jpg)

As the boot partition remains unencrypted, I formatted only the partitions that were to be encrypted as follows (as
superuser):

```shellsession
# cryptsetup luksFormat /dev/sda6
# cryptsetup luksFormat /dev/sda7
# cryptsetup luksFormat /dev/sda8
```

I then opened these encrypted volumes as follows:

```shellsession
$ cryptsetup luksOpen /dev/sda6 root
$ cryptsetup luksOpen /dev/sda7 swap
$ cryptsetup luksOpen /dev/sda7 home
```

This creates unencrypted entries with corresponding names in **/dev/mapper/**.  It's these that should be mounted, should there
ever be an issue and the data needs to be retrieved.  With the devices unencrypted, I then created their filesystems

```
mkfs.ext4 /dev/mapper/root
mkfs.ext4 /dev/mapper/home
mkswap /dev/mapper/swap
```

## Installing Linux

I then proceeded to install Linux using the install option in the live USB.  When the Installation Type screen appears,
**"Something else"** should be selected and the appropriate unencrypted volume used (/dev/mapper/....) to install home, root and 
swap mount points:

Select `/dev/mapper/root`, click Change select **ext4** and mount point **/**  
Select `/dev/mapper/home`, click Change select **ext4** and mount point **/home** 
Select `/dev/mapper/swap`, click Change select **swaparea**  
Select `/dev/sda5`, click Change select **ext2** and mount point **/boot**  

Then continue with the install process.  Once complete, choose **"Continue Testing"** rather than Restart

### Further Changes Required

The following directories need to be mounted, using **chroot** as we're running from our Live USB:

```
cd /mnt
mkdir root
mount /dev/mapper/root root
mount /dev/sda5 root/boot

chroot root
mount -t proc proc /proc
mount -t sysfs sys /sys
```

The `crypttab` file needs to be created, to tell the system which volumes are encrypted.  

I used the UUIDs to identify the partitions to be.  A `sudo blkid` command was used to view the UUIDs of each volume.  Note 
that the entries I was interested in all stated **"crypto_LUKS"** which  corresponded to the devices we encrypted (sda6, sda7, 
sda8 in my case).  Gottcha: Don't make the mistake of using the UUID for the line that states Swap (without the crypto_LUKS bit, 
it's the sda7 line that was needed)

The crypttab file can be created using `vi /etc/crypttab` and should be edited as follows (Using UUIDs from above):

```
root UUID=f0f7443a-2530-68b36158793f none luks
root UUID=50f7443a-249c-e7b36158353a none luks,swap
home UUID=a757443a-4163-c2376154793d none luks
```

The additional **swap** entry at the end of the swap line means mkswap is ran after boot, so nothing in the swap can survive
a re-boot.

### Regenerate intramfs

The initramfs is a boot filesystem image, used to mount root (Or something like that). I found that I had to regenerate locale as otherwise I got an error relating to no support for en_US.utf8

```
locale-gen --purge --no-archive
update-initramfs -u
```

### Backup LUKS Headers

If the headers in the LUKS volumes become corrupted, then the partition will become unusable.  It is a good idea to have a backup 
to restore from.  This was done as a root user as follows:

```
cryptsetup luksHeaderBackup /dev/sda6 --header-backup-file /root/root.img
cryptsetup luksHeaderBackup /dev/sda7 --header-backup-file /root/home.img
```


### Should Just Work, right?

Um, well, no.  On re-booting, I was taken straight to the Windows boot manager instead of grub and then Windows started.  On looking
in the Bios (Or whatever it's called these days :smiley: )  there was no Mint entry to be seen!

The first step to fix was to boot using the USB Live image, and run  `efibootmgr -v` which showed entries for Mint and
Ubuntu (I think it uses Ubuntu, as that has the signed key for secure boot).  The Ubuntu entry was above the Windows Boot
Manager, so it should have worked....

After much head scratching, I found that the BIOS had feature to **select a UEFI file as trusted**.  Once I selected the ubuntu
file as trusted, it then appeared in the Bios.  I could then ensure this was ahead of the Windows Boot Manager in the boot
order - which worked!

This behaviour appeared to be specific to my make of laptop (Acer), with many people complaining about their buggy BIOS.

### Swap Password and Other Issues

Finally I could then boot into my encrypted Linux, though was forced to enter identical passwords for both root and swap.  The home
partition was automatically decrypted with the root. The next task to to update the setup so as only one password was
required.

To do this, I initially added a keyfile to the Swap crypttab entry. I  later found out that the swap wasn't working as viewing the
partition in GParted showed it as **unknown** rather than **luks_crypt**.  Executing `cat /proc/swaps` showed nothing and
doing a `top` confirmed my swap was was 0.

I think what had went wrong was that on re-creating the swap, the UUID had got trashed.  I booted into the Live usb again and
recreated the swap area manually:

```
cryptsetup luksFormat /dev/sda7
cryptsetup luksOpen /dev/sda7 swap
mkswap /dev/mapper/swap 
```

I them re-performed the mappping/chroot operations and edited the swap line in the crypttab file to use the new UUID and regenerate 
a password:

```text
root UUID=68e7423a-a3f2-e7b36158353a /dev/urandom offset=2048,cipher=aex-xts-plain64,size=256,swap
```

This uses a random password, which is discarded at shutdown, so no need to enter a password manually.  

As I suspect the UUID was previously being trashed when the swap was re-created.  The **offset=2048** means there is a 1Mb 
offset used when re-creating the swap and thus the original UUID/LABEL is maintained rather than wiped since they live in the 
initial 1Mb header.  As the UUID will remain unchanged, the crypttab entry is always valid and the swap should be correctly mounted.

After the above steps, I again updated the locale and ran `update-initramfs -u`.  On rebooting, it appeared to work :shipit:


