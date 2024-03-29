# Arch Linux Install 

This is the process I followed to set up my personal Arch Linux installation. Your mileage may vary. When in doubt, consult the official Arch Wiki and related documentation. 

This process is mostly cobbled together from the [Arch Wiki](https://wiki.archlinux.org/title/Installation_guide), [Stephen's Tech Talks](https://www.youtube.com/@stephenstechtalks5377), [Learn Linux TV](https://www.youtube.com/@LearnLinuxTV), and [Dreams of Autonomy](https://www.youtube.com/@dreamsofautonomy). I owe a debt of gratitude to them all.

If you're going to use the package lists here, I highly recommend going through and removing whatever you don't need. For example, the fonts package list has about 3GB worth of files to download. You may want to skip that one altogether, if only to speed things along. You can always go back to the Arch repositories and the AUR to get what you need post-installation. 

As always, practice installation and configuration in a virtual environment (e.g., VirtualBox) before trying on actual hardware.

# Contents

1. [First Steps](#first-steps)
2. [Disks](#disks)
3. [Chroot into Target System](#chroot-into-target-system)
4. [Post-Install Configuration](#post-install-configuration)
5. [Tiling Window Manager](#tiling-window-manager)
6. [Hyprland](#hyprland)
7. [Additional Resources](#additional-resources)

<a href="#first-steps"></a>
## First Steps

Boot into the Live ISO. Verify that Internet connection exists. Use an Ethernet connection so that this is automatically established by the Live ISO.

```
root@archiso ~ # ip addr show
root@archiso ~ # ping archlinux.org
```

If Internet connection is not available via Ethernet, you can set up a connection using a Wi-Fi device.

```
root@archiso ~ # iwctl

[iwd] device list
                                    Devices                                   
--------------------------------------------------------------------------------
  Name                  Address               Powered     Adapter     Mode      
--------------------------------------------------------------------------------

[iwd] station <device> connect <network>
[iwd] exit
```

It may be more convenient to do setup remotely from an already functional computer. We can accomplish that using SSH. The `sshd` service daemon is probably already running. We can verify that by trying to start it.

```
root@archiso ~ # systemctl start sshd
```

To use SSH, we'll need to set a password on the root superuser.

```
root@archiso ~ # passwd 
```

We can use SSH on our other computer to connect to our target system.

```sh
$ ssh root@<ip address>  
```

To make things a little faster, we can update our `pacman` mirrorlist to use better mirrors. Remember that this is updating the mirrorlist on the Live ISO, not the target system.

```
root@archiso ~ # reflector --country US --latest 5 --sort rate --save /etc/pacman.d/mirrorlist
```

We can make things faster still by enabling parallel downloads for `pacman`.

```
root@archiso ~ # vim /etc/pacman.conf

# Misc options
#UseSyslog
Color
#NoProgressBar
CheckSpace
#VerbosePkgLists
ParallelDownloads = 10
```
If you need 32-bit support packages, uncomment the `multilib` repository include lines inside `/etc/pacman.conf`.

We can check the NTP status.

```
root@archiso ~ # timedatectl status
               Local time: Wed 2024-02-07 21:34:28 UTC
           Universal time: Wed 2024-02-07 21:34:28 UTC
                 RTC time: Wed 2024-02-07 21:34:28
                Time zone: UTC (UTC, +0000)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

If NTP is not active, we can activate it.

```
root@archiso ~ # timedatectl set-ntp true
```

We can synchronize our Live ISO's `pacman` package database.

```
root@archiso ~ # pacman -Syy
:: Synchronizing package databases...
 core
 extra
 community
```

<a href="#disks"></a>
## Disks

Identify disk(s) available to the system. In this case, I have two disks: `/dev/sda` and `/dev/sdb`.

```
root@archiso ~ # lsblk
NAME  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0   7:0    0 766.5M  1 loop /run/archiso/airootfs
sda     8:0    0    32G  0 disk
sdb     8:16   0    32G  0 disk
sr0    11:0    1 932.3M  0 rom  /run/archiso/bootmnt
```

### Partition the Disks

Set up partitions on one or more disks. Use whichever disk formatting utility you want. Make sure to use the full disk, not a single partition.

```
# Use the actual disk, not a partition!
root@archiso ~ # cfdisk /dev/sda          
```

Set up whatever partitions you want. You'll need at least an EFI partition and a root partition (for the root filesystem `/`). Unless you have good reason not to, set up your disk partitions with a GPT partition table.

Sample partition layout:

```sh
Device       Start      End  Sectors Size Type
/dev/sda1     2048  2099199  2097152   1G EFI System
/dev/sda2  2099200 67106815 65007616  31G Linux filesystem

Device     Start      End  Sectors Size Type
/dev/sdb1   2048 67106815 67104768  32G Linux filesystem
```

If you're using Linux LVM, set the filesystem type to "Linux LVM" instead of "Linux filesystem".

When you're happy with your disk layout, write the changes to disk. Repeat the partitioning process for however many disks you have.

You should now have a partitioned disk. For example, something like this:

```
root@archiso ~ # lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0 766.5M  1 loop /run/archiso/airootfs
sda      8:0    0    32G  0 disk
├─sda1   8:1    0     1G  0 part
└─sda2   8:2    0    31G  0 part
sdb      8:16   0    32G  0 disk
└─sdb1   8:17   0    32G  0 part
sr0     11:0    1 932.3M  0 rom  /run/archiso/bootmnt
```

### Format the Partitions

We need to format the EFI partition as `FAT32`.

```
root@archiso ~ # mkfs.fat -F32 /dev/sda1 
mkfs.fat 4.2 (2021-01-31)
```

We'll format the root partition as `btrfs`. You can format it as `ext4` or whatever else you prefer. If you're setting up LVM, be sure to configure LVM before formatting the partition.

```
root@archiso ~ # mkfs.btrfs /dev/sda2
btrfs-progs v6.7
See https://btrfs.readthedocs.io for more information.

NOTE: several default settings have changed in version 5.15, please make sure
      this does not affect your deployments:
      - DUP for metadata (-m dup)
      - enabled no-holes (-O no-holes)
      - enabled free-space-tree (-R free-space-tree)

Label:              (null)
UUID:               a5e81acd-feda-460c-a3ff-22891db1609b
Node size:          16384
Sector size:        4096
Filesystem size:    31.00GiB
Block group profiles:
  Data:             single            8.00MiB
  Metadata:         DUP             256.00MiB
  System:           DUP               8.00MiB
SSD detected:       no
Zoned device:       no
Incompat features:  extref, skinny-metadata, no-holes, free-space-tree
Runtime features:   free-space-tree
Checksum:           crc32c
Number of devices:  1
Devices:
   ID        SIZE  PATH
    1    31.00GiB  /dev/sda2
```

You can use `lsblk` to verify the filesystem type under the `FSTYPE` column.

```
root@archiso ~ # lsblk -f
NAME   FSTYPE   FSVER            LABEL       UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0  squashfs 4.0                                                                     0   100% /run/archiso/airootfs
sda
├─sda1 vfat     FAT32                        FE06-1AD0
└─sda2 btrfs                                 a5e81acd-feda-460c-a3ff-22891db1609b
sdb
└─sdb1 btrfs                                 0a207df4-7dc6-43fd-ad4e-04154c8e9e61
sr0    iso9660  Joliet Extension ARCH_202402 2024-02-01-12-07-52-00                     0   100% /run/archiso/bootmnt           
```

### BTRFS Subvolumes

To set up our btrfs subvolumes, we need to first mount our btrfs filesystem partition.

```
root@archiso ~ # mount /dev/sda2 /mnt
```

You can verify that it successfully mounted by running `lsblk`.

```
root@archiso ~ # lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0 766.5M  1 loop /run/archiso/airootfs
sda      8:0    0    32G  0 disk
├─sda1   8:1    0     1G  0 part
└─sda2   8:2    0    31G  0 part /mnt                     # Success!
sdb      8:16   0    32G  0 disk
└─sdb1   8:17   0    32G  0 part
sr0     11:0    1 932.3M  0 rom  /run/archiso/bootmnt
```

We can now create some btrfs subvolumes. Remember that you can create and mount (or unmount and delete) subvolumes at any time.

The Arch Wiki offers the following tips:


> - When taking a snapshot of `@` (mounted at the root `/`), other subvolumes are not included in the snapshot. Even if a subvolume is nested below `@`, a snapshot of `@` will not include it. Create snapper configurations for additional subvolumes besides `@` of which you want to keep snapshots.
> - Consider creating subvolumes for other directories that contain data you do not want to include in snapshots and rollbacks of the `@` subvolume, such as `/var/cache`, `/var/spool`, `/var/tmp`, `/var/lib/machines` (systemd-nspawn), `/var/lib/docker` (Docker), `/var/lib/postgres` (PostgreSQL), and other data directories under `/var/lib/`. It is up to you if you want to follow the flat layout or create nested subvolumes. On the other hand, the pacman database in `/var/lib/pacman` must stay on the root subvolume (`@`).
> - You can run Snapper on `@home` and any other subvolume to have separate snapshot and rollback capabilities for data.

If you do create subvolumes for directories under `/var/lib`, do *NOT* make a subvolume for `/var/lib/pacman`. It's important that the `pacman` directory stays in sync with whatever snapshot we're rolling back to (or from). If `pacman` gets out of sync, that could severely diminish your ability to effectively maintain installed packages.

```
root@archiso / # cd /mnt
root@archiso /mnt # btrfs subv create @                 # /
root@archiso /mnt # btrfs subv create @var_cache        # /var/cache
root@archiso /mnt # btrfs subv create @libvirt_images   # /var/lib/libvirt/images
root@archiso /mnt # btrfs subv create @var_log          # /var/log
root@archiso /mnt # btrfs subv create @var_tmp          # /var/tmp
root@archiso /mnt # btrfs subv create @snapshots        # /.snapshots
root@archiso /mnt # cd ..
root@archiso / # umount /mnt
root@archiso / # mount /dev/sdb1 /mnt                   # Different btrfs FS on different disk
root@archiso / # cd /mnt
root@archiso /mnt # btrfs subv create @home             # /home
root@archiso /mnt # btrfs subv create @snapshots        # /home/.snapshots
root@archiso /mnt # cd ..
root@archiso / # umount /mnt
```

Now that we have our subvolumes created, we can remount with the options that btrfs will expect.

In this first example, `@` is the subvolume that we want to use. The `@` subvolume is located in the filesystem on the `/dev/sda2` partition. We want to mount it to `/mnt`, which will be `/` (i.e., the root filesystem) on the target system.

```
root@archiso ~ # mount -o compress=zstd,noatime,subvol=@ /dev/sda2 /mnt
```

We need to create the subdirectories for our other subvolumes to mount.

```
root@archiso / # mkdir -p /mnt/{boot/efi,home/{.snapshots},.snapshots,var/{cache,log,tmp,lib/libvirt/images}}
```

We can use `ls -la` to verify that directories -- including the hidden `.snapshots` directory -- were created. You can `ls` any of these subdirectories to confirm whether content exists within them.

```
root@archiso ~ # ls -la /mnt
total 0
drwxr-xr-x 1 root root  42 Feb  7 21:52 .
drwxr-xr-x 1 root root 100 Feb  7 21:26 ..        
drwxr-xr-x 1 root root   0 Feb  7 21:52 .snapshots
drwxr-xr-x 1 root root   6 Feb  7 21:52 boot      
drwxr-xr-x 1 root root   0 Feb  7 21:52 home      
drwxr-xr-x 1 root root  22 Feb  7 21:52 var 
```

With those directories created, we can mount the rest of our subvolumes.

```
root@archiso / # mount -o compress=zstd,noatime,subvol=@var_cache /dev/sda2 /mnt/var/cache
root@archiso / # mount -o compress=zstd,noatime,subvol=@libvirt_images /dev/sda2 /mnt/var/lib/libvirt/images
root@archiso / # mount -o compress=zstd,noatime,subvol=@var_log /dev/sda2 /mnt/var/log
root@archiso / # mount -o compress=zstd,noatime,subvol=@var_tmp /dev/sda2 /mnt/var/tmp
root@archiso / # mount -o compress=zstd,noatime,subvol=@snapshots /dev/sda2 /mnt/.snapshots
root@archiso / # mount -o compress=zstd,noatime,subvol=@home /dev/sdb1 /mnt/home
root@archiso / # mount -o compress=zstd,noatime,subvol=@snapshots /dev/sdb1 /mnt/home/.snapshots
```

We can mount our EFI partition.

```
root@archiso / # mount /dev/sda1 /mnt/boot/efi
```

We can use `lsblk` to verify that everything so far has been mounted successfully.

```
root@archiso / # lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0 766.5M  1 loop /run/archiso/airootfs
sda      8:0    0    32G  0 disk
├─sda1   8:1    0     1G  0 part /mnt/boot/efi
└─sda2   8:2    0    31G  0 part /mnt/.snapshots
                                 /mnt/var/tmp
                                 /mnt/var/log
                                 /mnt/var/lib/libvirt/images
                                 /mnt/var/cache
                                 /mnt
sdb      8:16   0    32G  0 disk
└─sdb1   8:17   0    32G  0 part /mnt/home
                                 /mnt/home/.snapshots
sr0     11:0    1 932.3M  0 rom  /run/archiso/bootmnt
```

### Preparing the Target System

We need to use `pacstrap` to install the initial packages onto the target system. Remember that `/mnt` on the Live ISO will be `/` (i.e., the root filesystem) on the target machine.

```
root@archiso ~ # pacstrap -K /mnt base base-devel git vim reflector openssh rsync
```

We'll set up the `fstab` using `genfstab`. The `-U` option ensures that our `/etc/fstab` file on the target system will use UUIDs to refer to devices and partitions. The `-p` option is technically redundant.

Make sure you're appending to the `fstab` file on the target system (i.e., `/mnt/etc/sfstab`).

```
root@archiso ~ # genfstab -U -p /mnt >> /mnt/etc/fstab
```

<a href="#chroot-into-target-system"></a>
## Chroot into Target System

Now we can `chroot` into the in-progress installation on the target system. Note that we're now working on the target system from within the Live ISO. Notice that the prompt has changed.

```
root@archiso ~ # arch-chroot /mnt
[root@archiso /]#
```

Verify that the `fstab` generated correctly.

```
[root@archiso /]# vim /etc/fstab
```

If you're formatting an SSD with btrfs, it should automatically be detected and the `ssd` option automatically added to the mount options.

```sh
# Static information about the filesystems.
# See fstab(5) for details.

# <file system> <dir> <type> <options> <dump> <pass>
# /dev/sda2
UUID=a5e81acd-feda-460c-a3ff-22891db1609b       /               btrfs           rw,noatime,compress=zstd:3,space_cache=v2,subvolid=256,subvol=/@    0 0

# /dev/sda2
UUID=a5e81acd-feda-460c-a3ff-22891db1609b       /var/cache      btrfs           rw,noatime,compress=zstd:3,space_cache=v2,subvolid=257,subvol=/@var_cache   0 0

# /dev/sda2
UUID=a5e81acd-feda-460c-a3ff-22891db1609b       /var/lib/libvirt/images btrfs           rw,noatime,compress=zstd:3,space_cache=v2,subvolid=258,subvol=/@libvirt_images      0 0

# /dev/sda2
UUID=a5e81acd-feda-460c-a3ff-22891db1609b       /var/log        btrfs           rw,noatime,compress=zstd:3,space_cache=v2,subvolid=259,subvol=/@var_log     0 0

# /dev/sda2
UUID=a5e81acd-feda-460c-a3ff-22891db1609b       /var/tmp        btrfs           rw,noatime,compress=zstd:3,space_cache=v2,subvolid=261,subvol=/@var_tmp     0 0

# /dev/sda2
UUID=a5e81acd-feda-460c-a3ff-22891db1609b       /.snapshots     btrfs           rw,noatime,compress=zstd:3,space_cache=v2,subvolid=260,subvol=/@snapshots   0 0

# /dev/sdb1
UUID=0a207df4-7dc6-43fd-ad4e-04154c8e9e61       /home           btrfs           rw,noatime,compress=zstd:3,space_cache=v2,subvolid=256,subvol=/@home        0 0

# /dev/sdb1
UUID=0a207df4-7dc6-43fd-ad4e-04154c8e9e61       /home/.snapshots          btrfs           rw,noatime,compress=zstd:3,space_cache=v2,subvolid=257,subvol=/@snapshots        0 0

# /dev/sda1
UUID=FE06-1AD0          /boot/efi       vfat            rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro       0 2
```

Before we install any packages using `pacman`, we can make the same tweaks we did in the Live ISO environment. This updates the `pacman` mirrorlist on the target system. Make sure you have `rsync` installed. If you didn't install `rsync` via `pacstrap`, you can do so via `pacman -S rsync`.

```
[root@archiso /]# reflector --country US --latest 5 --sort rate --save /etc/pacman.d/mirrorlist
```

```
[root@archiso /]# vim /etc/pacman.conf

# Misc options
#UseSyslog
Color
#NoProgressBar
CheckSpace
#VerbosePkgLists
ParallelDownloads = 10
```

If you need to enable `multilib` or any of the other official repositories, do it now.

Now we can sync the local `pacman` package database with the mirrors.

```
[root@archiso /]# pacman -Syy
:: Synchronizing package databases...
```

Now that we're on the target system, we can use `pacman` to install our preferred initial packages. Be sure to choose your preferred provider when prompted for packages with multiple providers.

```
[root@archiso /]# mkdir GitPackageLists
[root@archiso /]# cd GitPackageLists
[root@archiso /GitPackageLists]# git clone https://github.com/mbeaver502/ArchLinux
[root@archiso /GitPackageLists]# cd ArchLinux

# Install from our package lists
# Use the --needed flag so we don't reinstall anything
[root@archiso ArchLinux]# pacman -S --needed - < 01-base.txt
[root@archiso ArchLinux]# pacman -S --needed - < 02-kernel.txt
[root@archiso ArchLinux]# pacman -S --needed - < 03-sysutils.txt
[root@archiso ArchLinux]# pacman -S --needed - < 04-drivers.txt
[root@archiso ArchLinux]# pacman -S --needed - < 05-network.txt
[root@archiso ArchLinux]# pacman -S --needed - < 06-fonts.txt
[root@archiso ArchLinux]# pacman -S --needed - < 07-printer.txt
[root@archiso ArchLinux]# pacman -S --needed - < 08-multimedia.txt
[root@archiso ArchLinux]# pacman -S --needed - < 09-de-wm.txt
[root@archiso ArchLinux]# pacman -S --needed - < 10-misc.txt  
```

Now we can set a password for the `root` superuser. If you don't specify a user for `passwd`, it will default to the `root` superuser.

```
[root@archiso /]# passwd
```

If you've installed `openssh`, now's a good time to start the `sshd` daemon.

```
[root@archiso /]# systemctl enable sshd
Created symlink /etc/systemd/system/multi-user.target.wants/sshd.service → /usr/lib/systemd/system/sshd.service.
```

Now we can create a user for ourselves and give them a password. This particular list of groups is probably overkill. Consult the Arch Wiki to see what's appropriate for you. In this case, we're specifying that this user will use the `zsh` shell (`-s /bin/zsh`).

```
[root@archiso /]# useradd -m -G adm,audio,ftp,games,log,lp,network,rfkill,scanner,storage,sys,users,video,wheel -s /bin/zsh <name>
[root@archiso /]# passwd <name>
[root@archiso /]# id <name>  # Inspect a user
```

We need to allow our user to execute commands via `sudo`. We can do this by editing the `sudoers` file.

```
[root@archiso /]# EDITOR=vim visudo  

## Uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL:ALL) ALL
```

### GRUB

We can modify our GRUB default config. This is especially necessary if you're using an encrypted filesystem and/or LVM. You may also want to specify the root filesystem's path.

```
[root@archiso /]# vim /etc/default/grub
```

Look for `GRUB_CMDLINE_LINUX_DEFAULT` and `GRUB_CMDLINE_LINUX` near the top of the file.

We need to create a place for our EFI partition to mount, if it doesn't already exist. Mount to that point, if not already mounted. If you set this up inside the Live ISO, you can skip this.

```
[root@archiso /]# mkdir /boot/efi
[root@archiso /]# mount /dev/sda1 /boot/efi  # Use the correct device and partition
```

We'll install the GRUB bootloader.

```
[root@archiso /]# grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id="Arch Linux" --recheck
Installing for x86_64-efi platform.
Installation finished. No error reported.
```

Optionally, set up the locale for GRUB.

```
[root@archiso /]# cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo
```

Now we can generate the GRUB config file.

```
[root@archiso /]# grub-mkconfig -o /boot/grub/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-linux-lts
Found initrd image: /boot/intel-ucode.img /boot/initramfs-linux-lts.img
Found fallback initrd image(s) in /boot:  intel-ucode.img initramfs-linux-lts-fallback.img
Found linux image: /boot/vmlinuz-linux
Found initrd image: /boot/intel-ucode.img /boot/initramfs-linux.img
Found fallback initrd image(s) in /boot:  intel-ucode.img initramfs-linux-fallback.img
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
Found memtest86+ image: /boot/memtest86+/memtest.bin
done
```

### `mkinitcpio`

We can add necessary values to our `mkinitcpio` config.

```
[root@archiso /]# vim /etc/mkinitcpio.conf
```

We're using `btrfs`, so we can add `btrfs` to the `MODULES` and `BINARIES` collections. If you're using LVM, add `lvm2` to the `HOOKS`. Also make sure you `pacman -S lvm2`. 

Once we're finished updating `mkinitcpio.conf`, we need to rebuild our kernel image. Repeat this command for whichever kernels you installed.

```
[root@archiso /]# mkinitcpio -p linux
[root@archiso /]# mkinitcpio -p linux-lts  
```

### Time and Locale

```
[root@archiso /]# ln -sf /usr/share/zoneinfo/US/Central /etc/localtime
[root@archiso /]# hwclock --systohc --utc
```

We can use `timesyncd` to sync time via NTP for us.

```
[root@archiso /]# vim /etc/systemd/timesyncd.conf  # Set the NTP and FallbackNTP 
[root@archiso /]# systemctl enable systemd-timesyncd.service
Created symlink /etc/systemd/system/dbus-org.freedesktop.timesync1.service → /usr/lib/systemd/system/systemd-timesyncd.service.        
Created symlink /etc/systemd/system/sysinit.target.wants/systemd-timesyncd.service → /usr/lib/systemd/system/systemd-timesyncd.service.
```

We can set one or more locales for our machine.

```
[root@archiso /]# vim /etc/locale.gen                           # Uncomment your UTF-8 locale and save
[root@archiso /]# locale-gen
Generating locales...
  en_US.UTF-8... done
Generation complete.
[root@archiso /]# echo "LANG=en_US.UTF-8" > /etc/locale.conf    # Use your own locale
```

We can set the machine's hostname by writing to `/etc/hostname`.

```
[root@archiso /]# echo "<hostname>" > /etc/hostname
```

Set up the `/etc/hosts` file for the system. Substitute the same `hostname` you wrote to `/etc/hostname`.

```
[root@archiso /]# vim /etc/hosts

# Static table lookup for hostnames.
# See hosts(5) for details.
127.0.0.1       localhost                   # IPv4
::1             localhost                   # IPv6
127.0.1.1       <hostname>.localdomain     <hostname>
```

### Enable System Services

We can enable various service daemons that systemd will control.

```
[root@archiso /]# systemctl enable sddm.service         # SDDM greeter
[root@archiso /]# systemctl enable NetworkManager       # Network management
[root@archiso /]# systemctl enable avahi-daemon         # Detect devices on network
[root@archiso /]# systemctl enable bluetooth.service    # Bluetooth support
[root@archiso /]# systemctl enable haveged              # RNG for some services
[root@archiso /]# systemctl enable cups.service         # Printing support
[root@archiso /]# systemctl enable firewalld.service    # Firewall daemon
[root@archiso /]# systemctl enable fstrim.timer         # FS trim timer (good for SSDs)
[root@archiso /]# systemctl enable reflector.timer      # `pacman` mirror updates
[root@archiso /]# systemctl enable sshd                 # SSH support
[root@archiso /]# systemctl enable upower               # Probably unnecessary
```

If you want numlock to be enabled when you boot into your system -- and you're using SDDM as your greeter -- add this to your configuration.

```
[root@archiso /]# vim /etc/sddm.conf

[General]
Numlock=on
```

### Exit Chroot

Once we're finished doing initial setup within the target system, we can `exit` the `chroot`.

```
[root@archiso /]# exit
```

We need to make sure that we unmount all the devices and partitions that we've mounted from inside the Live ISO.

```
root@archiso ~ # umount -R /mnt
root@archiso ~ # lsblk                      # Verify everything was unmounted
```

Finally, we can `reboot` to restart the system.

```
root@archiso ~ # reboot now
```

You can remove your install medium at this point.

<a href="#post-install-configuration"></a>
## Post-Install Configuration

Now that we have Arch Linux installed, we can do some configuration on our actual system. 

### AUR Helper

We'll use the `paru` AUR helper. The `paru` AUR package will also prompt you to install Rust. Alternatively, you can get a precompiled version from the `paru-bin` package. If you don't want to use `paru`, you can use `yay`, which is written in Go.

Clone `paru` from the AUR and make the package. This will take some time while `cargo` compiles the `paru` binary.

```sh
beaver@archbox ~ % git clone https://aur.archlinux.org/paru.git
beaver@archbox ~/paru % cd paru
beaver@archbox ~/paru % makepkg -si
```

Verify that `paru` works, and clean up whatever's left behind from the download and installation.

```sh
beaver@archbox ~/paru % paru
beaver@archbox ~/paru % cd
beaver@archbox ~ % rm -rf /paru
```

If you want to make the ZSH shell pretty, you can use `paru` (or `yay`) to install the powerlevel10k theme now. The maintainers of powerlevel10k [recommend](https://github.com/romkatv/powerlevel10k?tab=readme-ov-file#arch-linux) using the AUR version instead of the version available in the official Arch repository.

```sh
beaver@archbox ~ % paru -S zsh-theme-powerlevel10k-git
beaver@archbox ~ % echo 'source /usr/share/zsh-theme-powerlevel10k/powerlevel10k.zsh-theme' >> ~/.zshrc
beaver@archbox ~ % source ~/.zshrc
```

### Snapshots

We'll use `snapper` to manage our btrfs snapshots. We'll also install packages to automatically create pre-post snapshots for `pacman`/`paru` (`snap-pac`), as well as to show snapshots on the GRUB boot menu (`grub-btrfs`). We can do all this with the convenient AUR package `snapper-support`.

```sh
beaver@archbox ~ % paru -S snapper-support
```

`snapper` will try to create its own snapshots directory at `/.snapshots`, but we already have a subvolume set up there. [We can fix that.](https://wiki.archlinux.org/title/Snapper#Creating_a_new_configuration)

```sh
beaver@archbox ~ % sudo -s
root@archbox /home/beaver $ cd /
root@archbox / $ umount /.snapshots 
root@archbox / $ rm -r /.snapshots
```

Create the "root" config with `snapper`, targeting the root filesystem.

```
root@archbox / $ snapper -c root create-config /
```

`snapper` will create a new `.snapshots` subvolume that we need to delete.

```sh
root@archbox / $ btrfs subv list /
ID 256 gen 164 top level 5 path @
ID 257 gen 162 top level 5 path @cache
ID 258 gen 164 top level 5 path @home
ID 259 gen 8 top level 5 path @images
ID 260 gen 164 top level 5 path @var_log
ID 261 gen 8 top level 5 path @snapshots
ID 262 gen 12 top level 256 path var/lib/portables
ID 263 gen 12 top level 256 path var/lib/machines
ID 264 gen 164 top level 256 path .snapshots            # We want to delete this one

root@archbox / $ btrfs subv delete /.snapshots 
Delete subvolume 264 (no-commit): '//.snapshots'
```

Now we can re-create the directory and re-mount it. We can re-mount it by leveraging our already existing `/etc/fstab`.

```
root@archbox / $ mkdir /.snapshots
root@archbox / $ mount -a
```

While we're here, we can set up the default subvolume for our root filesystem (`/`).

```sh
root@archbox / $ btrfs subv get-default /
ID 5 (FS_TREE)

root@archbox / $ btrfs subv list /
ID 256 gen 166 top level 5 path @                       # We want ID 256 for @
ID 257 gen 162 top level 5 path @cache
ID 258 gen 164 top level 5 path @home
ID 259 gen 8 top level 5 path @images
ID 260 gen 166 top level 5 path @var_log
ID 261 gen 8 top level 5 path @snapshots
ID 262 gen 12 top level 256 path var/lib/portables
ID 263 gen 12 top level 256 path var/lib/machines

root@archbox / $ btrfs subv set-default 256 /
root@archbox / $ btrfs subv get-default /    
ID 256 gen 167 top level 5 path @                       # Success!
```

Don't forget to do this on any other btrfs filesystems you created (e.g., on different disks).

The Arch Wiki page notes that we should re-generate our GRUB config using `grub-install` any time we change the default subvolume on a system. This will ensure that the bootloader knows the correct root location. Feel free to do that if you want.

We can allow our user(s) to see and access the `snapper` snapshots directory by editing the appropriate configuration file.

```sh
root@archbox / $ vim /etc/snapper/configs/root      # For the "root" config we created

# users and groups allowed to work with config
ALLOW_USERS="beaver"
ALLOW_GROUPS="wheel"
```

Adjust `TIMELINE` values as desired. The [Archi Wiki](https://wiki.archlinux.org/title/Snapper#Set_snapshot_limits) recommends the following defaults for busy subvolumes like `/`:

```sh
TIMELINE_MIN_AGE="1800"
TIMELINE_LIMIT_HOURLY="5"     
TIMELINE_LIMIT_DAILY="7"     
TIMELINE_LIMIT_WEEKLY="0"    
TIMELINE_LIMIT_MONTHLY="0"
TIMELINE_LIMIT_YEARLY="0"
```

We also need to update the permissions for `/.snapshots`.

```
root@archbox / # chmod a+rx .snapshots
root@archbox / # chown -R :wheel .snapshots
```

We can enable `snapper`'s built-in timer service.

```
root@archbox / # systemctl status snapper-timeline.timer
```

We can verify that the `grub-btrfs` service was enabled and started. We also want to update our GRUB config to ensure our snapshots are available at the boot menu.

```
root@archbox / # systemctl enable --now grub-btrfsd.service
root@archbox / # systemctl start --now grub-btrfs-snapper.service
root@archbox / # grub-mkconfig -o /boot/grub/grub.cfg
root@archbox / # systemctl daemon-reload
```

We can create a snapshot of our current system configuration. Remember that `snapper` snapshots are read-only by default. We re-make our GRUB config to ensure that our manual snapshots are also available.

```
root@archbox / # snapper -c root create -d "***** SYSTEM INSTALL *****"
root@archbox / # grub-mkconfig -o /boot/grub/grub.cfg
```

We can verify that our snapshot was created by checking `snapper ls`.

```
root@archbox / # snapper ls
 # | Type   | Pre # | Date                            | User | Cleanup | Description                | Userdata
---+--------+-------+---------------------------------+------+---------+----------------------------+---------
0  | single |       |                                 | root |         | current                    |
1  | single |       | Wed 07 Feb 2024 05:57:56 PM CST | root |         | ***** SYSTEM INSTALL ***** |
```

Exit root whenever you're ready.

If you want a nice GUI frontend for managing your `snapper` setup, you can install [`btrfs-assistant`](https://aur.archlinux.org/packages/btrfs-assistant).

```sh
beaver@archbox ~ % paru -S btrfs-assistant
```

### RAM-based Swap

We can use `zram` to use our RAM for swap space, instead of using precious read/write cycles on our SSD.

```
beaver@archbox ~ % paru -S zram-generator
```

We need to configure `zram-generator`.

```
beaver@archbox ~ % sudo vim /etc/systemd/zram-generator.conf 
```

We can configure `zram-generator` like so:

```ini
[zram0]
zram-size = ram / 2
```

Now we need to reload our system daemons.

```
beaver@archbox ~ % sudo systemctl daemon-reload 
beaver@archbox ~ % sudo systemctl start /dev/zram0
```

We can check `zram` by using `zramctl`. You may need to reboot for `zram` to take effect.

```
beaver@archbox ~ % zramctl
beaver@archbox ~ % free -h
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       998Mi       2.1Gi       6.4Mi       989Mi       2.8Gi
Swap:          1.9Gi          0B       1.9Gi
```

<a href="#tiling-window-manager"></a>
## Tiling Window Manager

Currently the package lists are installing `awesome` and `qtile`. You can switch between them at the `sddm` greeting screen (or whichever greeter you're using). The Wayland version of `qtile` appears to be broken inside VirtualBox, loading into a black screen. Could be a VM-specific problem. The Xorg version of `qtile` loads fine.

<a href="#hyprland"></a>
## Hyprland

Note that officially Hyprland does not work inside a VM.

If you're feeling lazy, install [hyprdots](https://github.com/prasanthrangan/hyprdots). Additional dependencies for hyprdots that can be found in the AUR:
- grimblast
- oh-my-zsh
- rofi-lbonn-wayland
- swaylock-effects
- swww
- waybar-hyprland
- wlogout
- zsh-theme-powerlevel10k


<a href="#additional-resources"></a>
## Additional Resources

- [Arch Wiki generic installation guide](https://wiki.archlinux.org/title/Installation_guide)
- [Arch Wiki page on BTRFS](https://wiki.archlinux.org/title/Btrfs)
- [Arch Wiki page on user groups](https://wiki.archlinux.org/title/Users_and_groups#Group_list)
- [Arch Wiki page on AMD GPUs](https://wiki.archlinux.org/title/AMDGPU)
- [Arch Wiki page on `snapper`](https://wiki.archlinux.org/title/Snapper)

<!--

https://www.reddit.com/r/archlinux/comments/108hzb2/help_needed_with_btrfs_layout/
-->
