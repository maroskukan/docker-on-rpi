# Running Docker on Raspberry Pi

- [Running Docker on Raspberry Pi](#running-docker-on-raspberry-pi)
  - [Introduction](#introduction)
  - [Documentation](#documentation)
  - [Automated Installation](#automated-installation)
    - [Prepare SD Card](#prepare-sd-card)
    - [First Boot](#first-boot)
    - [Configuration](#configuration)
  - [Manual Installation](#manual-installation)
    - [Prepare SD Card](#prepare-sd-card-1)
    - [First Boot](#first-boot-1)
    - [Configuration](#configuration-1)
  - [Docker](#docker)
    - [Installation](#installation)
    - [Systemctl Verify](#systemctl-verify)
    - [Upgrade](#upgrade)
    - [Uninstall](#uninstall)
  - [Docker-compose](#docker-compose)
    - [Installation](#installation-1)
    - [Verification](#verification)
  - [Containers](#containers)
    - [Pi-hole](#pi-hole)
  - [Tips](#tips)
    - [Customize using systemd-nspawn](#customize-using-systemd-nspawn)
      - [Customize the SD card](#customize-the-sd-card)
      - [Access target environment](#access-target-environment)

## Introduction

The following document describes the different ways how to install and configure of Docker on RPI running Raspberry Pi OS Lite.


## Documentation

- [Installing Image using Linux](https://www.raspberrypi.org/documentation/installation/installing-images/linux.md)
- [Installing Image using Mac](https://www.raspberrypi.org/documentation/installation/installing-images/mac.md)
- [Setting up a wireless LAN via the CLI](https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md)
- [Micro-SD Cards Benchmark](http://www.pidramble.com/wiki/benchmarks/microsd-cards)
- [RPi3 Card Reader Overclocking](https://www.jeffgeerling.com/blog/2016/how-overclock-microsd-card-reader-on-raspberry-pi-3)
- [Dockerhub ARM Images](https://registry.hub.docker.com/search?q=&type=image&architecture=arm%2Carm64)
- [Systemd-nspawn](https://blog.oddbit.com/post/2016-02-07-systemd-nspawn-for-fun-and-wel/)
- [Custom image](https://forums.raspberrypi.com/viewtopic.php?f=63&t=231762&p=1462118#p1462118)

## Automated Installation

### Prepare SD Card

The automated option includes installation of [Raspberry PI Imager tool](https://www.raspberrypi.com/software/). With this tool you are able to define the following configuration settings before flashing the SD Card.

For the demo build I have selected the following general values in the wizard:

| Key                                  | Value                         |
| ------------------------------------ | ----------------------------- |
| Operating System                     | Raspberry Pi OS Lite (64-bit) |
| Hostname                             | rpi01                         |
| Enable SSH                           | True                          |
| Allow public-key authentication only | True                          |
| Set authorized_keys for 'ansible'    | <your-public-key>             |
| Username                             | ansible                       |
| Password                             | <your-password>               |
| Configure wireless LAN               | True                          |
| SSID                                 | <your-ssid>                   |
| Set locale settings                  | True                          |
| Time zone                            | Europe/Bratislava             |
| Keyboard layout                      | us                            |

### First Boot

Once the flashing process is finished insert SD card and power on. The PI should be available on your local network for further configuration via Ansible.

### Configuration

In order to leverage Ansible for configuration management you need to ensure that the following prerequsities are met:
- Python runtime - [virtual environment recommended](https://github.com/pyenv/pyenv), tested with version 3.10.2
- Python modules - described in `requirements-dev.txt` this will also install `ansible-core`
- Ansible Galaxy collections - described in `requirements.yml`

After these prerequisities are met, optionally inspect the `default.config.yml` to under stand which configuration values will be applied and which packages will be installed.

Once you are happy with the definitions execute the playbook.

```bash
ansible-playbook main.yml
```

Once completed the desired packages including docker will be installed and lifecycle for sample `hello-world` container will be validated.


## Manual Installation

The manual option includes SD card layout preparation, custom wireless configuration, user credetials change after first boot and Docker installation and image lifecycle validation.

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

### First Boot

Insert the SD Card into the RPI and power on. After initial boot sequence, the RPI should be up and running. Default username is `pi` and default password is `raspberry`.

*By default when RPI connects to Wireless network, it will receive a dynamic IP address from DHCP service running on local router. In order to make the address assignment more deterministic, you can configure a static DHCP address binding on your home router and use name instead of IP address.*

```bash
ssh pi@rpi01.home

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

# Poor man's benchmark for slow SD card
sudo dd if=/dev/zero of=/home/pi/test bs=8k count=50k conv=fsync; sudo rm -f /home/pi/test
51200+0 records in
51200+0 records out
419430400 bytes (419 MB, 400 MiB) copied, 43.8039 s, 9.6 MB/s
```

### Configuration

Retrieve the latest list of packages and update the system.

```bash
sudo apt update && sudo apt upgrade -y
```

Change default password for `pi` user.

```bash
# Change default password
passwd <your-new-password>
```

Update system name.

```bash
sudo hostnamectl set-hostname rpi01.home
```

Apply the changes.

```bash
sudo reboot
```

After reboot update the timezone.

```bash
sudo timedatectl set-timezone Europe/Bratislava
```

Copy your public key from local machine to RPI.

```bash
# Copy RSA Public Key
ssh-copy-id -i ~/.ssh/home/id_rsa.pub pi@rpi01.home

# Verify Access
ssh pi@rpi01.home
Linux rpi01 5.10.17-v7+ #1403 SMP Mon Feb 22 11:29:51 GMT 2021 armv7l
```

## Docker

This procedure only needs to take place in case you selected manual installation described above. Automated one will also install and verify docker installation.

### Installation

Download the installation script.

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
```

Execute the script.

```bash
sudo sh get-docker.sh
```

Add `pi` user to Docker group

```bash
sudo usermod -aG docker pi
```

Logout and login and Verify version and info.

```bash
docker version
Client: Docker Engine - Community
 Version:           20.10.5
 API version:       1.41
 Go version:        go1.13.15
 Git commit:        55c4c88
 Built:             Tue Mar  2 20:18:46 2021
 OS/Arch:           linux/arm
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.5
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.13.15
  Git commit:       363e9a8
  Built:            Tue Mar  2 20:16:18 2021
  OS/Arch:          linux/arm
  Experimental:     false
 containerd:
  Version:          1.4.4
  GitCommit:        05f951a3781f4f2c1911b05e61c160e9c30eaa8e
 runc:
  Version:          1.0.0-rc93
  GitCommit:        12644e614e25b05da6fd08a38ffa0cfe1903fdec
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

```bash
docker info
Client:
 Context:    default
 Debug Mode: false
 Plugins:
  app: Docker App (Docker Inc., v0.9.1-beta3)
  buildx: Build with BuildKit (Docker Inc., v0.5.1-docker)

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 0
 Server Version: 20.10.5
 Storage Driver: overlay2
  Backing Filesystem: extfs
  Supports d_type: true
  Native Overlay Diff: true
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Cgroup Version: 1
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: io.containerd.runc.v2 io.containerd.runtime.v1.linux runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 05f951a3781f4f2c1911b05e61c160e9c30eaa8e
 runc version: 12644e614e25b05da6fd08a38ffa0cfe1903fdec
 init version: de40ad0
 Security Options:
  seccomp
   Profile: default
 Kernel Version: 5.10.17-v7+
 Operating System: Raspbian GNU/Linux 10 (buster)
 OSType: linux
 Architecture: armv7l
 CPUs: 4
 Total Memory: 924.2MiB
 Name: rpi01
 ID: DQUK:L7MR:VKNB:O46Y:E77W:KLVU:WK3O:SWJS:JJZQ:WSQQ:DJLY:E7EV
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false
```

Run a test container

```bash
docker run --rm hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
4ee5c797bcd7: Pull complete
Digest: sha256:308866a43596e83578c7dfa15e27a73011bdd402185a84c5cd7f32a88b501a24
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (arm32v7)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

### Systemctl Verify

Verify that `docker.service` is set to start after reboot.

```bash
sudo systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2021-04-12 12:16:10 CEST; 17min ago
     Docs: https://docs.docker.com
 Main PID: 1689 (dockerd)
    Tasks: 14
   CGroup: /system.slice/docker.service
           └─1689 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```
### Upgrade

Upgrade using package manager

```bash
sudo apt-get upgrade
```

### Uninstall

Uninstall docker using package manager.

```bash
# Uninstall binaries
sudo apt-get purge docker-ce

# Clean images, containers, volumes and other data
sudo rm -rf /var/lib/docker
```


## Docker-compose

### Installation

Install docker-compose using package manager.

```bash
# Install depdendencies
sudo apt-get install -y \
             libffi-dev \
             libssl-dev \
             python3 \
             python3-pip

# Install Docker compose
sudo pip3 -v install docker-compose
```

### Verification

```bash
docker-compose -v
docker-compose version 1.29.0, build unknown
```

## Containers

### Pi-hole

```bash
# Download the compose file
wget -O docker-compose.yml \
    https://raw.githubusercontent.com/maroskukan/docker-on-rpi/main/Dockerfiles/pi-hole/docker-compose.yml

# Build and run
docker-compose up --detach
Creating network "pi-hole_default" with the default driver
Pulling pihole (pihole/pihole:latest)...
latest: Pulling from pihole/pihole
9159b6bb9431: Pull complete
010046f0b40e: Extracting [==================================================>]  1.901kB/1.901kB
956067455ce8: Download complete
b26a0a9087a9: Download complete
bc2a1c51f34f: Download complete

# Verify
docker-compose ps
 Name    Command       State                                                   Ports
---------------------------------------------------------------------------------------------------------------------------------
pihole   /s6-init   Up (healthy)   0.0.0.0:443->443/tcp, 0.0.0.0:53->53/tcp, 0.0.0.0:53->53/udp, 0.0.0.0:67->67/udp,
                                   0.0.0.0:80->80/tcp

# Retrieve Admin password
docker logs pihole | grep random
Assigning random password: aso4aA
+ pihole -a -p aso4aA aso4aA
```


## Tips

### Customize using systemd-nspawn

#### Customize the SD card

Start by downloading latest available Raspberry Pi OS Lite archive available from [raspberrypi.org](https://www.raspberrypi.org/software/operating-systems/).

Example from Ubuntu below.

```bash
# Download archive
wget https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2024-03-15/2024-03-15-raspios-bookworm-arm64-lite.img.xz

# Download checksum
wget https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2024-03-15/2024-03-15-raspios-bookworm-arm64-lite.img.xz.sha256

# Verify checksum
sha256sum -c 2024-03-15-raspios-bookworm-arm64-lite.img.xz.sha256
2024-03-15-raspios-bookworm-arm64-lite.img.xz: OK
```

Identify the block device (SD Card) and write the image.

```bash
# Search for disk device with size smaller than 64G
lsblk -d -o NAME,TYPE,SIZE | awk '$2 == "disk" && $3 ~ /[0-9.]+G/ && $3+0 < 64 {print $1}'

mmcblk0

# Unmount the disk and or existing partitions
umount /dev/mmcblk0p1

# Extract image from archive
unxz 2024-03-15-raspios-bookworm-arm64-lite.img.xz

# Write image to SD Card
sudo dd if=2024-03-15-raspios-bookworm-arm64-lite.img of=/dev/mmcblk0 bs=4M status=progress; sync
2336227328 bytes (2,3 GB, 2,2 GiB) copied, 5 s, 466 MB/s
660+0 records in
660+0 records out
2768240640 bytes (2,8 GB, 2,6 GiB) copied, 72,7164 s, 38,1 MB/s

# Verify newly created partitions
lsblk /dev/mmcblk0

NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
mmcblk0     179:0    0 58,2G  0 disk 
├─mmcblk0p1 179:1    0  512M  0 part 
└─mmcblk0p2 179:2    0  2,1G  0 part 

sudo fdisk -l /dev/mmcblk0                                                                       
Disk /dev/mmcblk0: 58,24 GiB, 62534975488 bytes, 122138624 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xfb33757d

Device         Boot   Start     End Sectors  Size Id Type
/dev/mmcblk0p1         8192 1056767 1048576  512M  c W95 FAT32 (LBA)
/dev/mmcblk0p2      1056768 5406719 4349952  2,1G 83 Linux
```

As you can see in the above output, the image only uses 2.1G out of 58.2G for the root partition. In order to fully utilize the available space on the SD card we need to resize this paritition and extend the filesystem.

```bash
# Resize the second partition
sudo parted /dev/mmcblk0 resizepart 2 100%

# Verify the second partition
lsblk /dev/mmcblk0

NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
mmcblk0     179:0    0 58,2G  0 disk 
├─mmcblk0p1 179:1    0  512M  0 part 
└─mmcblk0p2 179:2    0 57,7G  0 part 

# Extend the filesystem on the second partition
sudo resize2fs /dev/mmcblk0p2
resize2fs 1.47.0 (5-Feb-2023)
Resizing the filesystem on /dev/mmcblk0p2 to 15135232 (4k) blocks.
The filesystem on /dev/mmcblk0p2 is now 15135232 (4k) blocks long.

# Verify the filesystem on the second partition
sudo fdisk -l /dev/mmcblk0
Disk /dev/mmcblk0: 58,24 GiB, 62534975488 bytes, 122138624 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xfb33757d

Device         Boot   Start       End   Sectors  Size Id Type
/dev/mmcblk0p1         8192   1056767   1048576  512M  c W95 FAT32 (LBA)
/dev/mmcblk0p2      1056768 122138623 121081856 57,7G 83 Linux
```

Once happy with the layout, remove the source artifacts.

```bash
# Remove image and checksum files 
rm 2024-03-15-raspios-bookworm-arm64-lite.img
rm 2024-03-15-raspios-bookworm-arm64-lite.img.xy.sha256
```

#### Access target environment

```bash
sudo apt-get install -y qemu-user-static
```

```bash
# Mount root directory
sudo mkdir /mnt/rootfs
sudo mount /dev/mmcblk0p2 /mnt/rootfs

# Mount boot directory
sudo mount /dev/mmcblk0p1 /mnt/rootfs/boot
```

```bash
# Run a shell inside the rpi root filesystem using a container
sudo systemd-nspawn --hostname raspberrypi --directory=/mnt/rootfs

Spawning container rpiroot on /mnt/rootfs.
Press ^] three times within 1s to kill container.
```

Now that we are in the container lets start by some simple verification.

```bash
# Print the current architecture
dpkg --print-architecture
arm64
```


```bash
# Set password for root user
passwd

# Set password for pi user
passwd pi 

# Enable ssh
touch /boot/ssh

# Update pakacage list and upgrade all packages
apt update && sudo apt upgrade -y

#### Cleanup
```bash
^^^
umount /mnt/rootfs/boot
umount /mnt/rootfs
```


1. powerup
