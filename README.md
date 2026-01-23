# my-archlinux-installation-guide
![xkcd: Cautionary](https://imgs.xkcd.com/comics/cautionary.png)

motto: [RTFM](https://wiki.archlinux.org/index.php?title=Arch_terminology&oldid=849549#RTFM)

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


###### considerations:
 - drive preparation (wiping)
 - sector size (drive + encryption + file settings)
 - TRIM (pros and cons; consider encryption again) probably
 - encryption LUKS2
 - file system btrfs
 - swap file vs swap partition
 - drive allignment
 - swap size

###### What's TRIM?
Here is a good resource to learn about SSD pages, blocks, Garbage Collection, TRIM and the difference between the last two:

https://www.thessdreview.com/daily-news/latest-buzz/garbage-collection-and-trim-in-ssds-explained-an-ssd-primer/
