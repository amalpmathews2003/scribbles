# Getting Started with RDK-Broadband (RDKB)

RDK-Broadband (RDKB) is an open-source platform for broadband gateways, cable modems, and embedded devices. It provides a unified software stack, supports TR-181 data models, and can be built using the Yocto/OpenEmbedded build system. This guide helps you set up your build environment, configure the RDKB workspace, build images, and run them on Raspberry Pi 4 or QEMU.

---

## Prerequisites

### Host Machine Requirements
- **OS**: Ubuntu 20.04 LTS  
- **Arch**: 64-bit  
- **Free Disk Space**: Minimum 100 GB  
- **RAM**: 8–16 GB recommended  
- **Platform Targets**:  
  - Raspberry Pi 4B (official development kit)  
  - QEMU (also supported)

---

## Step 1: Install Required Packages

Update the system and install all necessary dependencies:

```bash
sudo apt-get -y update
sudo apt-get -y upgrade

sudo apt install -y gawk wget python3-git python3-jinja2 libegl1-mesa libsdl1.2-dev \
pylint3 xterm python3-subunit mesa-common-dev zstd liblz4-tool git diffstat unzip \
texinfo gcc build-essential chrpath socat cpio python python3 python3-pip \
python3-pexpect xz-utils debianutils iputils-ping make xsltproc docbook-utils \
fop dblatex xmlto lib32z1 libc6-i386 g++-multilib curl locales vim git-extras tree \
valgrind exuberant-ctags cscope dos2unix lcov libcunit1-dev libjson-perl members \
screen silversearcher-ag git-review nano
```

## Step 2: Install and Configure repo

RDKB uses Google's repo tool to manage Yocto layers.

```bash
mkdir ~/bin
export PATH=~/bin:$PATH
curl http://commondatastorage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

## Step 3: Set Up the Workspace

```bash
mkdir <workspace dir>
cd <workspace dir>
```

Initialize the repo using the latest RDKB manifest (example for 2025 Q2 release):
```bash
repo init -u https://code.rdkcentral.com/r/rdkcmf/manifests -m rdkb-extsrc.xml -b rdkb-2025q2-kirkstone
```
RDKB release lists are available here:
[Releases](https://wiki.rdkcentral.com/spaces/CMF/pages/7701138/RDK-B+Code+Releases)

Sync the layers:
```bash
repo sync --no-clone-bundle --no-tags
```

## Step 4: Configure the Build Environment
Each RDKB platform has its own setup script.
Example for Raspberry Pi 4 RDK-Broadband:

```bash
MACHINE=raspberrypi4-rdk-broadband source meta-cmf-raspberrypi/setup-environment
```
This sets up the Yocto build folder (e.g., build-raspberrypi4-64-rdk-broadband).

## Step 5: Build RDKB Image
Start the Yocto build:

```bash
bitbake rdk-generic-broadband-image
```

Images will be output under:
```
build-raspberrypi4-64-rdk-broadband/tmp/deploy/images/raspberrypi4-64-rdk-broadband/
```

## Step 6: Flashing RDKB Image to Raspberry Pi 4

### Linux

```bash
bzip2 -d rdk-generic-broadband-image-raspberrypi-rdk-broadband.wic.bz2
sudo -E bmaptool copy --nobmap \
  rdk-generic-broadband-image-raspberrypi-rdk-broadband.wic /dev/sdb
```
Replace /dev/sdb with your SD card.

### Windows (BalenaEtcher)

 1. Open Etcher
 2. Select the .wic or .wic.bz2 image
 3. Select SD card
 4. Click Flash

## Step 7: Run RDKB on QEMU (ARM / Raspberry Pi 4 Emulation)

Example QEMU run script:
```bash
#!/bin/bash

KERNEL_FILE="./Image-raspberrypi4-64-rdk-broadband.bin"
IMAGE_FILE="./rdk-generic-broadband-image-raspberrypi4-64-rdk-broadband.wic"
HOST_FWD="hostfwd=tcp::2224-:22,hostfwd=tcp::8082-:80"

qemu-system-aarch64 \
    -machine virt \
    -cpu cortex-a72 \
    -m 2G \
    -smp 4 \
    -nographic \
    -serial mon:stdio \
    -kernel "${KERNEL_FILE}" \
    -drive file="${IMAGE_FILE}",if=none,id=hd0,cache=writeback,format=raw \
    -device virtio-blk-device,drive=hd0 \
    -append "rw earlyprintk loglevel=8 console=ttyAMA0,115200 root=/dev/vda2 rootdelay=1" \
    -device "virtio-net-device,netdev=eth0" \
    -netdev "user,id=eth0,${HOST_FWD}"
```

This forwards:
 - SSH → localhost:2224
 - Web UI → localhost:8082


Conclusion
 - You now have a complete workflow for:
 - Installing dependencies
 - Setting up the Yocto repo
 - Building RDKB
 - Flashing images
 - Running RDKB in QEMU

This setup works for both real Raspberry Pi hardware and virtualized environments.

Happy building!