# my-archlinux-installation-guide
![xkcd: Cautionary](https://imgs.xkcd.com/comics/cautionary.png)

motto: [RTFM](https://wiki.archlinux.org/index.php?title=Arch_terminology&oldid=849549#RTFM) & [KISS](https://wiki.archlinux.org/index.php?title=Arch_terminology&oldid=849549#KISS)

After rocking arch for 5+ years as a sole os I'm taking my ssd upgrade as an opportunity to have some fun with a fresh install.

Starting point: https://wiki.archlinux.org/title/Installation_guide

One of the greatest assets of the archlinux community is the extensive wiki. One of the coolest features of the wiki is every page has a history of updates(with diffs too). Assuming changes down the line I'll be linking my wiki references with the latest versions as of the day I write this.

I'm following [this version of the installation guide.](https://wiki.archlinux.org/index.php?title=General_recommendations&oldid=859535)

## 1 Pre-installation

Apparently there's an "official" way to create an Arch Linux Installer USB drive on Android. That's what I'll do. Sounds fun and since I have no other machine laying around or os that's what I'll do.

### 1.1 Acquire an installation image

Visit the [Download](https://archlinux.org/download/) page. I am going with the recommended BitTorrent Download using [uTorrent](https://play.google.com/store/apps/details?id=com.utorrent.client&hl=en)(mainly for historical reasons, despite the ads).

### 1.2 Verify signature

This step is recommended plus it's fun to do it on your phone.

I'm using the BLAKE2b checksum verification. [Termux](https://play.google.com/store/apps/details?id=com.termux&hl=en) is the terminal emulator of choice + I find [Coding Keyboard](https://play.google.com/store/apps/details?id=com.ajay.prokeyboard&hl=en) useful.

Download the iso and b2sum.txt(see the link above). The b2sum.txt file contains the sum and the file name. Actually it contains 4 pairs of sums and filenames so expect 3 errors.

Run

```
pkg update && pkg upgrade
```

inside Termux for good measure.

Then find the downloaded iso and b2sum.txt. Click open with and choose Termux so they appear in Termux. The two files appeared in ~/downloads for me.

```
cd downloads
```
Both the iso and b2sum.txt should be in the same dir.

Finally check the checksum itself. "OK" is expected on the iso version downloaded and 3 "No such file or directory" errors.

```
b2sum -c b2sum.txt
```

### 1.3 Prepare an installation medium
[EtchDroid](https://play.google.com/store/apps/details?id=eu.depau.etchdroid&hl=en) is officially acknowledged in the wiki as an OS image flasher for Android. After checking the sum I used it to create my USB.

https://wiki.archlinux.org/index.php?title=USB_flash_installation_medium&oldid=859741#Using_EtchDroid

### 1.4 Boot the live environment

Plug the USB and pray your hardware works.

### 1.5 Set the console keyboard layout and font

I guess optional. 

List available layouts:

```
localectl list-keymaps
```
I assume bg_bds-utf8 is mine. Load it:

```
loadkeys bg_bds-utf8
```

### 1.6 Verify the boot mode

Check the UEFI bitness:

```
cat /sys/firmware/efi/fw_platform_size
```
Expect 64. This confirms the system is booted in 64-bit x64 EUFI.

### 1.7 Connect to the internet

I'll use iwctl to connect to my Wi-Fi

To get an interactive prompt:

```
iwctl
```

List all Wi-Fi devices:

```
device list
```

My device is named wlan0.

Initiate a scan for networks:

```
station wlan0 scan
```

List all available networks:

```
station wlan0 get-networks
```

To connect:

```
station wlan0 connect SSID
```

where SSID is your network name. You'll be prompted for the password.

To exit the iwctl interactive prompt you can go with Ctrl-d, Ctrl-c or just type exit

```
exit
```

To check the connection:

```
ping ping.archlinux.org
```

Again you can stop it with Ctrl-C.

More info on iwctl:

https://wiki.archlinux.org/index.php?title=Iwd&oldid=847035#iwctl

### 1.8 Update the system clock

```
timedatectl
```

### 1.9 Partition the disk
#### now the fun part

partition -> format -> mount 

plan: full system encryption on rollback capable install with swap partition

by:   LVM on LUKS + btrfs + snapper + grub

why LVM?     I want swap partition

why LUKS?    I want encryption

why btrfs?   I want rollbacks when I brake stuff

why snapper? ?manager for btrfs I guess, not there yet


##### considerations:
 - drive preparation (wiping)
 - sector size (drive + encryption + file settings)
 - TRIM (pros and cons; consider encryption again) probably
 - encryption LUKS2
 - file system btrfs
 - swap file vs swap partition
 - drive allignment
 - swap size

##### What's TRIM?
Here is a good resource to learn about SSD pages, blocks, Garbage Collection, TRIM and the difference between the last two:

https://www.thessdreview.com/daily-news/latest-buzz/garbage-collection-and-trim-in-ssds-explained-an-ssd-primer/

##### nvme ssd prep

###### optional drive info and stats

List all the NVMe SSDs attached with name, serial number, size, LBA format and serial:

```
nvme list
```

Output the NVMe SMART log page for health status, temp, endurance, and more:

```
nvme smart-log /dev/nvme0
```

See more at:

https://wiki.archlinux.org/index.php?title=Solid_state_drive/NVMe&oldid=853641#Management

###### Memory cell clearing

You can't just overwrite with 0s and call it a day... 

NVME SSDs usually support two commands - format and sanitize.

In order to verify what is supported by your drive, use the Identify Controller command:

```
nvme id-ctrl /dev/nvme0 -H | grep -E 'Format |Crypto Erase|Sanitize'
```

Example output:

```
  [1:1] : 0x1	Format NVM Supported
  [29:29] : 0	No-Deallocate After Sanitize bit in Sanitize command Supported
    [2:2] : 0	Overwrite Sanitize Operation Not Supported
    [1:1] : 0x1	Block Erase Sanitize Operation Supported
    [0:0] : 0x1	Crypto Erase Sanitize Operation Supported
  [2:2] : 0x1	Crypto Erase Supported as part of Secure Erase
  [1:1] : 0	Crypto Erase Applies to Single Namespace(s)
  [0:0] : 0	Format Applies to Single Namespace(s)
```

My ssd supports only format so I'll just format all the namespaces with crypto erase:

```
nvme format /dev/nvme0 -s 2 -n 0xffffffff
```

See more at:

https://wiki.archlinux.org/index.php?title=Solid_state_drive/Memory_cell_clearing&oldid=853889#NVMe_drive

###### Sector size

To check the formatted logical block address size:

```
nvme id-ns -H /dev/nvme0n1 | grep "Relative Performance"
```

Example output:

```
LBA Format  0 : Metadata Size: 0   bytes - Data Size: 512 bytes - Relative Performance: 0x2 Good (in use)
LBA Format  1 : Metadata Size: 0   bytes - Data Size: 4096 bytes - Relative Performance: 0x1 Better
```

My ssd supports only 512 B so I'll leave it as is.

See more at:
https://wiki.archlinux.org/index.php?title=Advanced_Format&oldid=857461#NVMe_solid_state_drives

###### Allegedly dm-crypt wipe on an empty device or partition
Do this if you have an HDD.

I tried this on my SSD. Didn't see any 0s with hexdump(see bonus) during wiping. Seems futile and I can't be bothered to give it any more attention...

Create a temporary encrypted container:

```
cryptsetup open --type plain --key-file /dev/urandom --sector-size 4096 /dev/nvme0n1 to_be_wiped
```

You can verify that it exists:
```
lsblk
NAME          MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda             8:0    0  1.8T  0 disk
└─to_be_wiped 252:0    0  1.8T  0 crypt
```

Wipe the container with zeros:

```
dd if=/dev/zero of=/dev/mapper/to_be_wiped status=progress bs=1M
```

Close the container:

```
cryptsetup close to_be_wiped
```

See:

https://wiki.archlinux.org/index.php?title=Dm-crypt/Drive_preparation&oldid=839285#dm-crypt_wipe_on_an_empty_device_or_partition

###### Partition allignment

A typical practice for personal computers is to have **each partition's start and size aligned to 1 MiB (1 048 576 bytes)** marks, as it is divisible by all commonly used  sector sizes—1 MiB, 512 KiB, 128 KiB, 4 KiB, and 512 B.

See more at:

https://wiki.archlinux.org/index.php?title=Advanced_Format&oldid=857461#Partition_alignment

##### What's Data-at-rest encryption?

For general overview of all encryption topics see:

https://wiki.archlinux.org/index.php?title=Data-at-rest_encryption&oldid=857790

## Bonus

### check battery level
```
cat /sys/class/power_supply/BAT0/capacity
```

### see whats writen on the ssd
```
hexdump -C /dev/nvme0n1 | less
```

