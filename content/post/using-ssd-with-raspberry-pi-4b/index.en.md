---
title: "How to use SSD with Raspberry Pi 4B"
description: "Configuring, using, and benchmarking SSD with Raspberry Pi 4B connected through USB."
date: "2022-05-01"
lastmod: "2022-05-01"
toc: true
slug: using-ssd-with-raspberry-pi-4b
tags:
  - Raspberry-Pi
  - RPi
  - SSD
  - Linux
links: []

---

Current needs of my band for web storage turned us to hosting Nextcloud
instance. I wanted to self-host it on my RPi4, my current home server. So, I
configured it to use my SD card as storage, but it didn't work out. SD card was
too slow for usage as web storage. Then, I moved to B2 as a storage. And as you
can see from the title, it didn't work out as well. B2 turned out to be too
clanky and unresponsive as primary storage for the Nextcloud instance.

That was the moment of my decision to buy an SSD. I bought a cheap 250GB Crucial
2.5" SSD and a USB 3.0 to SATA 3 adapter (Unitek Y-1096). Whole purchase was
under $50. There is no need for an external power outlet for the SSD. So, once I
got home, I plugged it right away into the RPi4. 

## Configuration

At the start I confirmed that the SSD is visible at the OS level.

```bash {linenos=false}
$ sudo fdisk -l

Disk /dev/sda: 223.57 GiB, 240057409536 bytes, 468862128 sectors
Disk model: 00SSD1
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 33553920 bytes
Disklabel type: dos
Disk identifier: 0xeb8a9682

# The rest of the output is removed by me, because it was too long.
```

Usually, after connecting new disk to the Linux device, it is visible as a file
in `/dev/`. The `/dev` directory is special place for devices and device
files in Linux systems. Disk devices usually get a name like `sda`, `sdb`,
`sdc`, etc. As a funny example of how it works: If you have a USB connected
speakers, you can send an file directly to a device file of your speakers.
It will, of course, play some sound.

### Prepare partition table

To format the disk before using, it has to get a partition table. I'm used to
use `fdisk` tool for that purpose.

```bash {linenos=false}
$ sudo fdisk /dev/sda
```

While inside the program you can use `m` command to display list of available
commands. If there's a partition on a disk, you can use `d` command to delete
it. You can check it by using `p` command.

```bash {linenos=false}
Command (m for help): p
Disk /dev/sda: 223.57 GiB, 240057409536 bytes, 468862128 sectors
Disk model: 00SSD1
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 33553920 bytes
Disklabel type: dos
Disk identifier: 0xeb8a9682

Device     Boot Start       End   Sectors   Size Id Type
/dev/sda1       65535 468862127 468796593 223.5G 83 Linux
```

After removing the partition, you can use `n` command to create a new one. At
this case I needed only one partition, so I just took all the defaults from the
command prompts and entered `w` to write the changes. `w` command writes the
changes to the disk permanently. Our new partition is now visible as a
`/dev/sda1`.

### Format the partition
Then, the partition is
ready to be formatted to the `ext4` file system.

```bash {linenos=false}
sudo mkfs -t ext4 /dev/sda1
```

### Create mount point

I'm used to mount the disks to the `/mnt/` directory in the Linux file system.
It's a generic mount point for mounting disks. First, I need to create a mount
point directory.

```bash {linenos=false}
$ sudo mkdir /mnt/ssd
```

Then, I can mount the partition to the directory. But, I have to make sure that
after the reboot my mounted partition remain mounted. For that, I'm going to use
`/etc/fstab` configuration file. Before changes it looks like this:

```bash {linenos=false}
$ sudo cat /etc/fstab
# <file system>        <dir>          <type>    <options>             <dump> <pass>
LABEL=writable         /              ext4      defaults              0      0
LABEL=system-boot      /boot/firmware vfat      defaults              0      1
```

Format of this file is very simple. More information about it can be found in
here: [wiki.debian.org/fstab](https://wiki.debian.org/fstab). In short 

After adding my entry to mount the `/dev/sda1` partition to the `/mnt/ssd`
directory, it looks like this:

```bash {linenos=false}
# <file system>        <dir>            <type>  <options>   <dump> <pass>
LABEL=writable         /                ext4    defaults    0      0
LABEL=system-boot      /boot/firmware   vfat    defaults    0      1
/dev/sda1              /mnt/ssd         ext4    defaults    0      0
```

Finally, I can mount the partition. The `mount` command will mount the partition to
the directory configured in the `/etc/fstab`.

```bash {linenos=false}
$ sudo mount /mnt/ssd
```

It can be verified by `df`:

```bash {linenos=false} 
$ df | grep /dev/sda1
/dev/sda1      229669796  1021996 216911504   1% /mnt/ssd
```

## Benchmark

I'm not a benchmarking expert yet, but I'm going to use `dd` and `hdparm` to
show you the difference between SD card and SSD.

### Reads

SD Card:
```bash {linenos=false}
$ sudo hdparm -Tt /dev/mmcblk0

/dev/mmcblk0:
 Timing cached reads:   1664 MB in  2.00 seconds = 833.99 MB/sec
 Timing buffered disk reads: 126 MB in  3.02 seconds =  41.74 MB/sec
```

SSD:
```bash {linenos=false}
sudo hdparm -Tt /dev/sda

/dev/sda:
 Timing cached reads:   1764 MB in  2.00 seconds = 883.68 MB/sec
 Timing buffered disk reads: 764 MB in  3.11 seconds = 245.39 MB/sec
```

### Writes

Writes are tested by writing a 2GB file to the disk.

SD Card:
```bash {linenos=false}
$ dd if=/dev/zero of=/tmp/output.img bs=8k count=256k
262144+0 records in
262144+0 records out
2147483648 bytes (2.1 GB, 2.0 GiB) copied, 95.2907 s, 22.5 MB/s
```

SSD:
```bash {linenos=false}
$ dd if=/dev/zero of=/mnt/ssd/output.img bs=8k count=256k
262144+0 records in
262144+0 records out
2147483648 bytes (2.1 GB, 2.0 GiB) copied, 11.2758 s, 190 MB/s
```

### Commentary

That's not a surprise that SD card is slower than SSD. But, those numbers are
interesting. The SSD outperforms the SD card in writes by a factor of 10.
Reading from the SD card can be 6 times slower than reading from the SSD.
Honestly I expected much better performance from the SSD. But, it's not a much
expensive benchmark of some Samsung EVO SSD.

## Usage

At the moment, the Ubuntu system on my RPi is still runnning from the SD card. I
moved MySQL and Redis to use storage on the SSD, and mounted `/var/www/html`
catalog from my Nextcloud container to the `/mnt/ssd/nextcloud/html` directory.
I resolved all of the performance problems I had with my Nextcloud instance. No
more timeouts or slow processing of files. All of the users, including me are
happy now.

## Final thoughts

Looking back in time, I now wonder why I haven't used the SSD from the start.
It's a real game changer for old PCs, so what stops us from speeding up our
Raspberry-Pies? Next thing I want to do, is to move the OS to entirely to the
SSD, but that's a lot of work at the moment.
