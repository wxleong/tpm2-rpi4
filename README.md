# Introduction

Enable OPTIGA™ TPM 2.0 on Raspberry Pi 4.

# Table of Contents

- **[Prerequisites](#prerequisites)**
- **[Install Raspberry Pi OS (using Ubuntu)](#install-raspberry-pi-os-using-ubuntu)**
- **[Install Raspberry Pi OS (using Windows)](#install-raspberry-pi-os-using-windows)**
- **[Set Up Network & SSH without Having a Monitor](#set-up-network--ssh-without-having-a-monitor)**
- **[Enable SPI TPM 2.0](#enable-spi-tpm-20)**
- **[Enable I2C TPM 2.0](#enable-i2c-tpm-20)**
    - **[Rebuild Raspberry Pi 4 Kernel](#rebuild-raspberry-pi-4-kernel)**
    - **[Device Tree](#device-tree)**
    - **[Verify TPM](#verify-tpm)**
- **[Enable I2C TPM 2.0 (Linux-6.1)](#enable-i2c-tpm-20-linux-61)**
- **[Set Up TSS and Tools](#set-up-tss-and-tools)**
    - **[Install Dependencies](#install-dependencies)**
    - **[Install tpm2-tss](#install-tpm2-tss)**
    - **[Install tpm2-tools](#install-tpm2-tools)**
    - **[Install tpm2-tss-engine](#install-tpm2-tss-engine)**
    - **[Debug](#debug)**
- **[What's Next](#whats-next)**
- **[References](#references)**
- **[License](#license)**

# Prerequisites

- Raspberry Pi 4 Model B with the following board mounted:
    - TPM 2.0 (SPI): Iridium 9670 TPM 2.0 board [[1]](#1), OPTIGA™ TPM SLB 9672 RPI evaluation board [[7]](#7)
    - TPM 2.0 (I2C): OPTIGA™ TPM SLB 9673 [[8]](#8)
- Host machine:
    ```
    $ lsb_release -a
    No LSB modules are available.
    Distributor ID:	Ubuntu
    Description:	Ubuntu 20.04.2 LTS
    Release:	20.04
    Codename:	focal

    $ uname -a
    Linux ubuntu 5.8.0-59-generic #66~20.04.1-Ubuntu SMP Thu Jun 17 11:14:10 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
    ```

# Install Raspberry Pi OS (using Ubuntu)

This section is intended for Ubuntu users.

First, install dependencies.
```
$ sudo apt update
$ sudo apt -u install \
  curl
```

Download the *Raspberry Pi OS with desktop* image (Linux 5.15.32-v7l+) onto your Ubuntu machine.
```
$ curl https://downloads.raspberrypi.org/raspios_armhf/images/raspios_armhf-2022-04-07/2022-04-04-raspios-bullseye-armhf.img.xz --output ~/2022-04-04-raspios-bullseye-armhf.img.xz
$ cd ~
$ unxz 2022-04-04-raspios-bullseye-armhf.img.xz
```

Connect your microSD card to the Ubuntu machine. Execute command `sudo fdisk -l` to find your microSD (e.g., `/dev/sdc`). Be cautious and strongly advise you to confirm the path `/dev/???` by either checking the total storage size (e.g., 7.3 GiB), or remove the microSD and check if it is still visible on the fdisk utility.
```
$ sudo fdisk -l
Disk /dev/???: 7.3 GiB, 7876902912 bytes, 15384576 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x1fa31320

Device     Boot  Start      End  Sectors  Size Id Type
/dev/???1         8192   532479   524288  256M  c W95 FAT32 (LBA)
/dev/???2       532480 15384575 14852096  7.1G 83 Linux
```

Unmount all partitions.
```
$ sudo umount /dev/???1
$ sudo umount /dev/???2
```

**!!! Warning, following command will write to the path `/dev/???`, selecting the wrong path will most certainly result in data loss or killing your operating system !!!**
```
$ sudo dd if=~/2022-04-04-raspios-bullseye-armhf.img of=/dev/??? bs=100M status=progress oflag=sync
```

# Install Raspberry Pi OS (using Windows)

This section is intended for Windows users.

1) Download the [Raspberry Pi Imager](https://downloads.raspberrypi.org/imager/imager_latest.exe) for Windows.
2) Put the SD card you'll use with your Raspberry Pi into the reader and run Raspberry Pi Imager.
3) Select the Operating System
    <p align="center">
        <img src="https://github.com/wxleong/tpm2-rpi4/blob/master/media/rpi-imager-select-os-1.jpg" width="50%">
        <img src="https://github.com/wxleong/tpm2-rpi4/blob/master/media/rpi-imager-select-os-2.jpg" width="50%">
    </p>
4) Select a microSD
    <p align="center">
        <img src="https://github.com/wxleong/tpm2-rpi4/blob/master/media/rpi-imager-select-sd-1.jpg" width="50%">
        <img src="https://github.com/wxleong/tpm2-rpi4/blob/master/media/rpi-imager-select-sd-2.jpg" width="50%">
    </p>
5) Write the OS
    <p align="center">
        <img src="https://github.com/wxleong/tpm2-rpi4/blob/master/media/rpi-imager-select-write.jpg" width="50%">
        <img src="https://github.com/wxleong/tpm2-rpi4/blob/master/media/rpi-imager-writing.jpg" width="50%">
    </p>

# Set Up Network & SSH without Having a Monitor

Connect the microSD card to your machine (Windows/Ubuntu) and add the following files to the boot partition:
- Create an empty file `ssh` with no file extension. When Raspberry Pi OS sees the file, it will enable SSH.
- Create a file `wpa_supplicant.conf` with the following content. This will tell the Raspberry Pi OS which network to connect to. **Remember to update the country code.**
    ```
    ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
    update_config=1
    country=SG

    network={
        scan_ssid=1
        ssid="wifi-ssid"
        psk="wifi-password"
    }
    ```

# Enable SPI TPM 2.0

Enable the TPM driver by using device tree overlay. Set the overlay in the bootloader config file. Run the following command after booting into Raspberry Pi OS:

```
$ sudo su -c "echo 'dtoverlay=tpm-slb9670' >> /boot/config.txt"
```

Power down your Raspberry Pi and attach the [Iridium 9670 TPM 2.0 board](https://www.infineon.com/cms/en/product/evaluation-boards/iridium9670-tpm2.0-linux/) according to the following image. 

<p align="center">
    <img src="https://github.com/wxleong/tpm2-rpi4/blob/master/media/raspberry-with-tpm.jpg" width="50%">
</p>

Power up your Raspberry Pi and check if the TPM is enabled by looking for the device nodes.

```
$ ls /dev | grep tpm
/dev/tpm0
/dev/tpmrm0
```

# Enable I2C TPM 2.0

<!--
**Test not completed yet. To-do: check the device tree config `tpm-pirq = <&gpio 24 GPIO_ACTIVE_HIGH>;`**
-->

## Rebuild Raspberry Pi 4 Kernel

On your host machine.

Install dependencies:
```
$ sudo apt update
$ sudo apt install git bc bison flex libssl-dev make libc6-dev libncurses5-dev
```

Install 32-bit toolchain:
```
$ sudo apt install crossbuild-essential-armhf
```

Download kernel source:
```
$ git clone https://github.com/raspberrypi/linux ~/linux
$ cd ~/linux
$ git checkout 1.20220331
$ make kernelversion
5.15.32
```

Patch the kernel (the changes are taken from [[4]](#4)[[5]](#5)):
```
$ git clone https://github.com/wxleong/tpm2-rpi4 ~/tpm2-rpi4
$ cd ~/linux
$ git am ~/tpm2-rpi4/patch/0001-tpm-Remove-read16-read32-write32-calls-from-tpm_tis_.patch
$ git am ~/tpm2-rpi4/patch/0002-Tpm-i2c-driver-patch-pre-release-test.patch
```

Build:
```
$ KERNEL=kernel7l
$ make -j$(nproc) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2711_defconfig
$ make -j$(nproc) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
    Device Drivers  --->
        Character devices  --->
            <M> TPM Hardware Support  --->
                <M> TPM Interface Specification 1.3 Interface / TPM 2.0 FIFO Interface - (I2C - generic)
$ make -j$(nproc) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage modules dtbs
```

Copy to microSD card:
```
$ mkdir mnt
$ mkdir mnt/fat32
$ mkdir mnt/ext4
$ sudo umount /dev/sd?1
$ sudo umount /dev/sd?2
$ sudo mount /dev/sd?1 mnt/fat32
$ sudo mount /dev/sd?2 mnt/ext4
$ sudo env PATH=$PATH make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=mnt/ext4 modules_install
$ sudo cp mnt/fat32/$KERNEL.img mnt/fat32/$KERNEL-backup.img
$ sudo cp arch/arm/boot/zImage mnt/fat32/$KERNEL.img
$ sudo cp arch/arm/boot/dts/*.dtb mnt/fat32/
$ sudo cp arch/arm/boot/dts/overlays/*.dtb* mnt/fat32/overlays/
$ sudo cp arch/arm/boot/dts/overlays/README mnt/fat32/overlays/
$ sudo umount mnt/fat32
$ sudo umount mnt/ext4
```

## Device Tree

Install device tree compiler on your host machine:
```
$ sudo snap install device-tree-compiler 
```

<!--
Decompile Raspberry Pi 4 device tree blob `~/linux/arch/arm/boot/dts/bcm2711-rpi-4-b.dtb`:
```
$ dtc -I dtb -O dts -o ~/bcm2711-rpi-4-b.dts ~/linux/arch/arm/boot/dts/bcm2711-rpi-4-b.dtb 
```

Boot into Raspberry Pi OS and scan the i2c bus for the address of TPM:
```
$ sudo i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- 2e --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- --
```
0x2e is the TPM address.
-->

Build the device tree blob overlay:
```
$ git clone https://github.com/wxleong/tpm2-rpi4 ~/tpm2-rpi4
$ dtc -@ -I dts -O dtb -o tpm-tis-i2c.dtbo ~/tpm2-rpi4/dts/tpm-tis-i2c.dts
```

Copy the `tpm-tis-i2c.dtbo` to `/boot/overlays/` and add the following line to the file `/boot/config.txt`:
```
dtoverlay=tpm-tis-i2c
```

## Verify TPM

Power up your Raspberry Pi and check if the TPM is enabled by looking for the device nodes.

```
$ ls /dev | grep tpm
/dev/tpm0
/dev/tpmrm0
```

# Enable I2C TPM 2.0 (Linux-6.1)

As of March 13th, 2023, the latest release of Raspberry Pi OS comes with Kernel v5.15. However, it's worth noting that the TPM 2.0 I2C driver (`TCG_TIS_I2C`) is only available starting from Kernel version v6.0. Fortunately, it is possible to manually update to version v6.1 of Raspberry Pi OS to gain access to the `TCG_TIS_I2C` driver:

```
$ sudo apt update
$ sudo rpi-update next
$ reboot

$ uname -a
Linux raspberrypi 6.1.10-v8+ #1628 SMP PREEMPT Mon Feb  6 19:22:54 GMT 2023 aarch64 GNU/Linux
```

## Device Tree

The `TCG_TIS_I2C` driver module is enabled by default on the `bcm2711_defconfig` build, so there is no need to rebuild the kernel. However, we do need to enable the driver on the device tree blob (DTB). To do this, you should first install the device tree compiler on your host machine:
```
$ sudo snap install device-tree-compiler
```

Build the device tree blob overlay:
```
$ git clone https://github.com/wxleong/tpm2-rpi4 ~/tpm2-rpi4
$ dtc -@ -I dts -O dtb -o tpm-tis-i2c.dtbo ~/tpm2-rpi4/dts/tpm-tis-i2c-2.dts
```

Copy the `tpm-tis-i2c.dtbo` to `/boot/overlays/` and add the following line to the file `/boot/config.txt`:
```
dtoverlay=tpm-tis-i2c
```

## Verify TPM

Power up your Raspberry Pi and check if the TPM is enabled by looking for the device nodes.

```
$ ls /dev | grep tpm
/dev/tpm0
/dev/tpmrm0
```

# Set Up TSS and Tools

## Install Dependencies

Update the package list:
```
$ sudo apt update
```

Install dependencies for tpm2-tss:
```
$ sudo apt -y install \
  autoconf-archive \
  libcmocka0 \
  libcmocka-dev \
  procps \
  iproute2 \
  build-essential \
  git \
  pkg-config \
  gcc \
  libtool \
  automake \
  libssl-dev \
  uthash-dev \
  autoconf \
  doxygen \
  libjson-c-dev \
  libini-config-dev \
  libcurl4-openssl-dev
```

Additional dependencies for tpm2-tools:
```
$ sudo apt -y install \
  uuid-dev \
  pandoc
```

## Install tpm2-tss

Download tpm2-tss:
```
$ git clone https://github.com/tpm2-software/tpm2-tss ~/tpm2-tss
$ cd ~/tpm2-tss
$ git checkout 3.2.0
```

Build tpm2-tss:
```
$ ./bootstrap
$ ./configure
$ make -j$(nproc)
```

Install tpm2-tss:
```
$ sudo make install
$ sudo ldconfig
```

Check installation:
```
$ ls /usr/local/lib/
```

## Install tpm2-tools

Download tpm2-tools:
```
$ git clone https://github.com/tpm2-software/tpm2-tools ~/tpm2-tools
$ cd ~/tpm2-tools
$ git checkout 5.2
```

Build tpm2-tools:
```
$ ./bootstrap
$ ./configure
$ make -j$(nproc)
```

Install tpm2-tools:
```
$ sudo make install
$ sudo ldconfig
```

Check installation:
```
$ ls /usr/local/bin/
```

Grant access permission to TPM device nodes:
```
$ sudo chmod a+rw /dev/tpm0
$ sudo chmod a+rw /dev/tpmrm0
```

Execute any `tpm2_` command, e.g.,
```
$ tpm2_getrandom --hex 16
```

## Install tpm2-tss-engine

Download tpm2-tss-engine:
```
$ git clone https://github.com/tpm2-software/tpm2-tss-engine ~/tpm2-tss-engine
$ cd ~/tpm2-tss-engine
$ git checkout v1.1.0
```

Build tpm2-tss-engine:
```
$ ./bootstrap
$ ./configure
$ make -j$(nproc)
```

Install tpm2-tss-engine:
```
$ sudo make install
$ sudo ldconfig
```

Check installation:
```
$ ls /usr/lib/arm-linux-gnueabihf/engines-1.1/
```

## Debug

To debug tpm2-tss [[6]](#6):
```
# possible levels are: NONE, ERROR, WARNING, INFO, DEBUG, TRACE
$ export TSS2_LOG=all+TRACE
```

# What's Next

More examples of tpm2-tools on [[2]](#2).

# References

<a id="1">[1] https://www.infineon.com/cms/en/product/evaluation-boards/iridium9670-tpm2.0-linux/</a><br>
<a id="2">[2] https://github.com/wxleong/tpm2-cmd-ref</a><br>
<a id="3">[3] https://forums.raspberrypi.com/viewtopic.php?t=236915</a><br>
<a id="4">[4] https://patchwork.kernel.org/project/linux-integrity/list/?series=628665</a><br>
<a id="5">[5] https://git.kernel.org/pub/scm/linux/kernel/git/jarkko/linux-tpmdd.git/commit/?id=0961f3b0457388111b92a8306d3718e0ad3932c8</a><br>
<a id="6">[6] https://github.com/tpm2-software/tpm2-tss/blob/master/doc/logging.md</a><br>
<a id="7">[7] https://www.infineon.com/cms/en/product/evaluation-boards/optiga-tpm-9672-rpi-eval/</a><br>
<a id="8">[8] https://www.infineon.com/cms/en/product/security-smart-card-solutions/optiga-embedded-security-solutions/optiga-tpm/optiga-tpm-slb-9673-fw26/</a><br>

# License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
