# Installing [ArchLinuxARM](https://archlinuxarm.org/) on Khadas VIM Pro device

I am not working for Khadas Support team! This is Community written HowTo!

**Disclaimer**: Do this at your own risk. Me or anyone else mentioned in this HowTo is not responsible for what you do on your device.

>**Goal:** run ArchLinuxARM with mainline Kernel on Khadas Vim Pro. Please note this is going to be only "server" installation all other stuff can be installed after, following the [ArchWiki](https://wiki.archlinux.org/).

>**Why Arch Linux:**  because it is a lightweight and flexible Linux® distribution that tries to Keep It Simple.

[Arch Linux != ArchLinuxARM](https://wiki.archlinux.org/index.php/Category:ARM_architecture)

This assumes you have already some Linux knowledge and that you have Arch Linux running on your host machine (this could be done on any other Linux distribution as well).

## Requirements
* Khadas Vim Pro (I have only Pro version)
* Linux u-boot (from an Khadas Image release) as the Android u-boot does currently not support ext4load.
* USB to Serial TTL cable (this is needed to set boot parameters, for instruction how to test and setup this please see [Khadas docs](http://docs.khadas.com/develop/SetupSerialTool/) page)
* micro SD card (tested with 2GB)
* Arch Linux installation on host device (other Linux flavor should work to)
* Internet connection (we are downloading around 400MB during this installation)

>Do everything as root (not via sudo)!

# Hardware information
* Ethernet uses an internal MII link with an internal 10/100 Ethernet PHY (meson8b-dwmac driver). Quick summary how Ethernet works (VERY simplified version): there's an Ethernet controller which is responsible for transferring data over the line and there's an Ethernet PHY which is responsible for negotiating speeds (and all other connection parameters)
* Wireless AP6255-based SDIO WiFi (RPi3 uses the same brcmfmac driver with other SDIO host driver and there are also lot of complaints about poor Wifi performance.)


## Preparation
Plug micro SD card to your machine where you have Linux running and find which device it is.
```bash
# dmesg | tail -n 10
……
[  170.097821] usb 2-3: new high-speed USB device number 3 using ehci-pci
[  177.828076] sd 6:0:0:0: [sdb] 3921920 512-byte logical blocks: (2.01 GB/1.87 GiB)
[  177.841750]  sdb: sdb1
#
```
In this log above it is an `sdb` device and it will be used in whole HowTo. Replace sd**X** in the following instructions with the device name for the micro SD card as it appears on your computer.

>**Disclaimer:** double check that you really use your micro SD device otherwise you could destroy your data on main computer or any other connected device (better unplug all not needed USB devices).

1. Zero the beginning of the micro SD card:
```bash
# dd if=/dev/zero of=/dev/sdb bs=1M count=8
```
2. Start fdisk to partition the micro SD card:
`# fdisk /dev/sdb`
3. At the fdisk prompt, create the new partitions:
  * Type **o**. This will clear out any partitions on the drive.
  * Type **p** to list partitions. There should be no partitions left.
  * Type **n**, then **p** for primary, **1** for the first partition on the drive, and **enter** twice to accept the default starting and ending sectors.
  * Write the partition table and exit by typing **w**.

  Check if the partition created and that it have Linux ID type **83**:
```bash
# fdisk -l /dev/sdb
.....
Device     Boot Start     End Sectors  Size Id Type
/dev/sdb1        2048 3921919 3919872  1,9G 83 Linux
#
```
4. Create the ext4 file system:
  * For e2fsprogs < 1.43: `# mkfs.ext4 /dev/sdb1`
  * For e2fsprogs >= 1.43: `# mkfs.ext4 -O ^metadata_csum,^64bit /dev/sdb1`
5. Mount the file system. I am going to do everything in **/opt** . Make sure to have at least 1GB of free space available on your host machine under **/opt**.
```bash
# mkdir -p /opt/root
# mount /dev/sdb1 /opt/root
```
6. Download and extract the root file system:
```bash
# cd /opt
# wget http://archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
# wget http://de5.mirror.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz.md5
# wget http://de5.mirror.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz.sig
```
  verify checksum and signature of downloaded file (pacman-key only works on Arch Linux)
```bash
# md5sum -c ArchLinuxARM-aarch64-latest.tar.gz.md5
# pacman-key -v ArchLinuxARM-aarch64-latest.tar.gz.sig
# bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C root/
# sync
```
7. Now that we have rootfs on our micro SD card, lets download latest mainline Kernel. In time of writing this HowTo that is 4.12-rc2
```bash
# wget http://ftp.halifax.rwth-aachen.de/archlinux-arm/aarch64/core/linux-aarch64-rc-4.12.rc2-1-aarch64.pkg.tar.xz
# wget http://ftp.halifax.rwth-aachen.de/archlinux-arm/aarch64/core/linux-aarch64-rc-4.12.rc2-1-aarch64.pkg.tar.xz.sig
# pacman-key -v linux-aarch64-rc-4.12.rc2-1-aarch64.pkg.tar.xz.sig
```
8. We will need to force install Kernel, once booted into ArchLinuxARM, to have all files in right place. So move to micro SD card for later install:
```bash
# cp linux-aarch64-rc-4.12.rc2-1-aarch64.pkg.tar.xz root/opt/
```
9. for booting Khadas VIM we need these files on micro SD boot directory:
```bash
# mkdir kernel
# bsdtar -xpf linux-aarch64-rc-4.12.rc2-1-aarch64.pkg.tar.xz -C kernel/
# cp -p kernel/boot/Image* root/boot/
# cp -R kernel/boot/dtbs/* root/boot/dtbs/
# sync
```
10. on host machine install uboot-tools and create uImage (includes the OS type and loader information) which is needed by u-boot:
```shell
# pacman -S uboot-tools
# mkimage -A arm64 -O linux -T kernel -C none -a 0x1080000 -e 0x1080000 -n "Arch Linux ARM kernel" -d root/boot/Image root/boot/uImage
```
output should be like this
```session
Image Name:   Arch Linux ARM kernel
Created:      Thu May 25 19:14:48 2017
Image Type:   AArch64 Linux Kernel Image (uncompressed)
Data Size:    21561856 Bytes = 21056.50 KiB = 20.56 MiB
Load Address: 01080000
Entry Point:  01080000
```
11. Unmount the partition:
` # umount /opt/root`

## On Khadas VIM Pro
1. Insert the micro SD card in Khadas VIM device. On your host machine start some client which can communicate over Serial connection type (minicom, putty, etc.) For this part an USB to Serial TTL cable is needed as we need to stop autoboot process, to configure Khadas VIM Pro to boot from prepared micro SD card. So almost immediately after plugging USB-C plug to host machine or wall adapter you need to hit Enter or space or Ctrl+C key to stop autoboot, you will get u-boot prompt like this:
```session
..
read emmc dtb
gpio: pin GPIOAO_2 (gpio 102) value is 1
get_cpu_id flag_12bit=1
Product checking: Khadas VIM.
Net:   dwmac.c9410000
Hit Enter or space or Ctrl+C key to stop autoboot -- :  0
kvim#
kvim#
```
2. Now in u-boot run the following commands:
```bash
kvim# setenv bootargs "console=ttyAML0,115200 root=/dev/mmcblk0p1 rootwait=1 rootdelay=2 rw init=/usr/bin/init"
kvim# ext4load mmc 0:1 ${loadaddr} /boot/uImage
kvim# ext4load mmc 0:1 $dtb_mem_addr /boot/dtbs/amlogic/meson-gxl-s905x-khadas-vim.dtb
```
  if everything successful then boot the ArchLinuxARM system on microSD card
```bash
kvim# bootm ${loadaddr} - $dtb_mem_addr
```
3. Login into your newly installed ArchLinuxARM  with user: **root** and password: **root**
```session
Arch Linux 4.12.0-rc2-1-ARCH (ttyAML0)
alarm login: root
Password:
[root@alarm ~]#
```
Check if we have any network device
```bash
[root@alarm ~]# ls /sys/class/net
lo
[root@alarm ~]#
```
4. As long as ArchLinuxARM still ships Kernel v4.10.13-1 by default we need to do following to have newest Kernel installed:
```bash
[root@alarm ~]# pacman -U --force /opt/linux-aarch64-rc-4.12.rc2-1-aarch64.pkg.tar.xz
```
  When asked to remove linux-aarch64 Kernel and install linux-aarch64-rc press **y**.

  Reboot the system but please note you need to stop autoboot and to re-enter again all u-boot commands from above
`[root@alarm ~]# systemctl reboot`
5. As soon as you see u-boot stuff hit Enter or space or Ctrl+C key to stop autoboot.
  Now in u-boot run again the following commands:
```bash  
kvim# setenv bootargs "console=ttyAML0,115200 root=/dev/mmcblk0p1 rootwait=1 rootdelay=2 rw init=/usr/bin/init"
kvim# ext4load mmc 0:1 ${loadaddr} /boot/uImage
kvim# ext4load mmc 0:1 $dtb_mem_addr /boot/dtbs/amlogic/meson-gxl-s905x-khadas-vim.dtb
```
  if everything successful then boot the ArchLinuxARM system on microSD card
```bash
kvim# bootm ${loadaddr} - $dtb_mem_addr
```
6. Login again into your newly installed ArchLinuxARM and check if you see now any network device
```session
Arch Linux 4.12.0-rc2-1-ARCH (ttyAML0)
alarm login: root
Password:
[root@alarm ~]# ls /sys/class/net
eth0  lo
[root@alarm ~]#
```
7. As you can see we have eth0 now. For this time I will use dhcp to get IP. Don’t forget to connect network cable to Khadas VIM Pro device.
```bash
[root@alarm ~]# systemctl enable dhcpcd@eth0.service
Created symlink /etc/systemd/system/multi-user.target.wants/dhcpcd@eth0.service -> /usr/lib/systemd/system/dhcpcd@.service.
[root@alarm ~]# systemctl start dhcpcd@eth0.service
[root@alarm ~]# ip addr
```
  With last command you will get IP of your device.

8. Now install openssh and enable root SSH login with password(this is security issue and should be only used when testing) and try to connect to your device.
```bash
[root@alarm ~]# pacman -Sy openssh vim lshw
[root@alarm ~]# sed -i -e 's/\#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
[root@alarm ~]# systemctl start sshd
```

**Now you have successful installed ArchLinuxARM on your Khadas VIM device. This is initial HowTo commit and with the Time I will add/change/remove some stuff as ArchLinuxARM or Kernel evolves.**

## Some information from ArchLinuxARM wiki:
The default configuration of the userspace is:
- Packages in the base group, Kernel firmware and utilities, openssh, and haveged
- Default root password is root
- A normal user account named alarm is set up, with the password alarm <- true but we removed this user
- sshd started (note: system key generation may take a few moments on first boot) <- actually not true we need to install and enabled it
- haveged started to provide entropy
- systemd-networkd DHCP configurations for eth0 and en* ethernet devices
- systemd-resolved management of resolv.conf
- systemd-timesyncd NTP management

## Basic system setup
Set device hostname, timezone, and NTP
```bash
[root@alarm ~]# hostnamectl set-hostname khadasvimpro
[root@alarm ~]# timedatectl set-ntp true
[root@alarm ~]# timedatectl list-timezones | sed 's/\/.*$//' | uniq
[root@alarm ~]# timedatectl list-timezones | grep Europe | sed 's/^.*\///'
[root@alarm ~]# timedatectl set-timezone Europe/Vienna
```
Set the NTP Server (there is already default Server built-in in packages)
```bash
[root@alarm ~]# vim /etc/systemd/timesyncd.conf
[root@alarm ~]# cat /etc/systemd/timesyncd.conf
.....
[Time]
NTP=0.at.pool.ntp.org 1.at.pool.ntp.org 2.at.pool.ntp.org
FallbackNTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org
```
To check the service status, use timedatectl status:
```bash
[root@alarm ~]# timedatectl status
      Local time: Sat 2016-05-14 00:26:43 CEST
  Universal time: Fri 2016-05-13 22:26:43 UTC
        RTC time: n/a
       Time zone: Europe/Vienna (CEST, +0200)
 Network time on: yes
NTP synchronized: yes
 RTC in local TZ: no
```
Check if time was synced with remote NTP Server
```
[root@alarm ~]# systemctl status systemd-timesyncd
......
May 25 19:57:23 alarm systemd-timesyncd[303]: Synchronized to time server 81.16.38.161:123 (2.arch.pool.ntp.org).
```

delete the alarm user (-r = remove home directory and mail spool), we are going to add our own user later on:
```bash
[root@alarm ~]# userdel -r alarm
```
language and locale settings /etc/locale.gen and /etc/locale.conf
```bash
[root@alarm ~]# vim /etc/locale.gen
[root@alarm ~]# grep -v "^#" /etc/locale.gen
en_DK.UTF-8 UTF-8
en_US.UTF-8 UTF-8
```
```bash
[root@alarm ~]# locale-gen
Generating locales...
  en_DK.UTF-8... done
  en_US.UTF-8... done
Generation complete.
```
```bash
[root@alarm ~]# vim /etc/locale.conf
[root@alarm ~]# grep -v "^#" /etc/locale.conf
LANG=en_US.UTF-8
LC_COLLATE=C
LC_DATE=en_DK.utf8
LC_NUMERIC=en_DK.utf8
LC_TIME=en_DK.utf8
```

## ToDo:
- [ ] u-boot command store
- [ ] WiFi Network setup
- [ ] write ArchLinuxArm to eMMC Storage of Khadas VIM device

## Know Problems:
- [ ] USB is not supported on the mainline Linux Kernel yet, see [linux-meson](http://linux-meson.com/doku.php#wip) (Khadas VIM uses the S905X SoC, also called GXL -> USB is still work-in-progress)
- [x] WiFi problem reported, solved with Heiner Kallweit patch, more about can be be found [here](http://lists.infradead.org/pipermail/linux-amlogic/2017-June/003864.html).
- [ ] Ethernet problems: sometimes detected only as 10Mbps and with 4.12-rc4 download will stall and SSH session would be disconnected. Reported [here](http://lists.infradead.org/pipermail/linux-amlogic/2017-June/003939.html)

## Authors and docummentation:
  * Martin Blumenstingl - Biggest part of this instruction is written by him (developer who is contributing code so we have our device working with mainline Kernel).
  * ArchLinuxArm wiki - some part are from ArchLinuxARM installation page
  * Me (vrabac) - and of course some stuff are written from me. I tested this and would like to give it to community so everyone can prepare ArchLinuxARM working on Khadas Vim device (this HowTo should work with some adjustment with any other arm device)

## Change log:
- 20170611: added hardware information section, moved know problems to task list
- 20170608: requirements (u-boot); know problems with WiFI and Ethernet
- 20170607: language and locale settings
- 20170606: retyped in GitHub Markdown
- 20170527: added Authors topic
- 20170526: added basic system setup
- 20170525: initial HowTo commit
