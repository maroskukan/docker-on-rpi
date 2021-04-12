# Running Docker on Raspberry Pi

- [Running Docker on Raspberry Pi](#running-docker-on-raspberry-pi)
  - [Introduction](#introduction)
  - [Documentation](#documentation)
  - [OS Installation](#os-installation)
    - [Prepare SD Card](#prepare-sd-card)
    - [Boot the Image](#boot-the-image)

## Introduction

The following document describes the installation and configuration of Docker on RPI running Raspberry Pi OS Lite.


## Documentation

[Installing Image using Linux](https://www.raspberrypi.org/documentation/installation/installing-images/linux.md)
[Installing Image using Mac](https://www.raspberrypi.org/documentation/installation/installing-images/mac.md)
[Setting up a wireless LAN via the CLI](https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md)

## OS Installation

### Prepare SD Card

Start by downloading latest available Raspberry Pi OS Lite archive available from [raspberrypi.org](https://www.raspberrypi.org/software/operating-systems/).

Example from Mac OS X below.

```bash
# Download archive
wget https://downloads.raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2021-03-25/2021-03-04-raspios-buster-armhf-lite.zip

# Download checksum
wget https://downloads.raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2021-03-25/2021-03-04-raspios-buster-armhf-lite.zip.sha256

# Verify checksum
sha256sum -c 2021-03-04-raspios-buster-armhf-lite.zip.sha256
2021-03-04-raspios-buster-armhf-lite.zip: OK
```

Identify the block device (SD Card) and write the image.

```bash
# Identify the SD Card
diskutil list
...
/dev/disk2 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *8.0 GB     disk2
   1:             Windows_FAT_32 boot                    46.0 MB    disk2s1
   2:                      Linux                         7.9 GB     disk2s2
...

# Unmount the disk
diskutil unmountDisk /dev/disk2

# Extract image
unzip 2021-03-04-raspios-buster-armhf-lite.zip

# Erite image to SD Card
sudo dd bs=1m if=2021-03-04-raspios-buster-armhf-lite.img of=/dev/rdisk2; sync
1780+0 records in
1780+0 records out
1866465280 bytes transferred in 215.911277 secs (8644594 bytes/sec)

# Remove archite, checksum and image
rm 2021-03-04-raspios-buster-armhf-lite.zip
rm 2021-03-04-raspios-buster-armhf-lite.zip.sha256
rm 2021-03-04-raspios-buster-armhf-lite.img
```

Create a new configuration file for wireless network. Updated `country`, `ssid` and `psk` accordingly. Save it on `boot` partition.

```bash
sudo mkdir /Volumes/Rpi
sudo mount -t msdos  /dev/disk2s1 /Volumes/Rpi
cd /Volumes/Rpi
cat << EOF >> wpa_supplicant.conf
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=<your_ISO-3166-1_two-letter_country_code>
network={
     ssid="<your_SSID>"
     psk="<your_PSK>"
     key_mgmt=WPA-PSK
}
EOF
```

Enable ssh by creating an empty `ssh` file. 

```bash
touch ssh
```

Finally, eject the card.

```bash
cd /
sudo diskutil unmount /Volumes/Rpi
sudo rm -rf /Volumes/Rpi
sudo diskutil eject /dev/rdisk2
```

### Boot the Image

Insert the SD Card into the RPI and power on. After initial boot sequence, the RPI should be up and running. Default username is `pi` and default password is `raspberry`.

```bash
ssh pi@10.0.2.118

# Verify model
cat /proc/device-tree/model
Raspberry Pi 3 Model B Rev 1.2

# Verify release
cat /etc/os-release
PRETTY_NAME="Raspbian GNU/Linux 10 (buster)"
NAME="Raspbian GNU/Linux"
VERSION_ID="10"
VERSION="10 (buster)"
VERSION_CODENAME=buster
ID=raspbian
ID_LIKE=debian
HOME_URL="http://www.raspbian.org/"
SUPPORT_URL="http://www.raspbian.org/RaspbianForums"
BUG_REPORT_URL="http://www.raspbian.org/RaspbianBugs"

# Verify memory usage
free -m
              total        used        free      shared  buff/cache   available
Mem:            924          39         666          11         218         820
Swap:            99           0          99

# Verify storage usage
df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       7.1G  1.3G  5.5G  19% /
devtmpfs        430M     0  430M   0% /dev
tmpfs           463M     0  463M   0% /dev/shm
tmpfs           463M   12M  451M   3% /run
tmpfs           5.0M  4.0K  5.0M   1% /run/lock
tmpfs           463M     0  463M   0% /sys/fs/cgroup
/dev/mmcblk0p1  253M   49M  204M  20% /boot
tmpfs            93M     0   93M   0% /run/user/1000
```

