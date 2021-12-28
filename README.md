# Introduction

Enable OPTIGA™ TPM 2.0 on Raspberry Pi 4.

# Table of Contents

- **[Prerequisites](#prerequisites)**
- **[Install Raspberry Pi OS (using Ubuntu)](#install-raspberry-pi-os-using-ubuntu)**
- **[Install Raspberry Pi OS (using Windows)](#install-raspberry-pi-os-using-windows)**
- **[Enable SPI TPM 2.0](#enable-spi-tpm-20)**
- **[Enable I2C TPM 2.0](#enable-i2c-tpm-20)**
    - **[Rebuild Raspberry Pi 4 Kernel](#rebuild-raspberry-pi-4-kernel)**
    - **[Device Tree](#device-tree)**
- **[References](#references)**
- **[License](#license)**

# Prerequisites

- Raspberry Pi 4 Model B
- TPM 2.0 board (SPI): Iridium 9670 TPM 2.0 board [[1]](#1)
- TPM 2.0 board (I2C): tbd
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

Download the *Raspberry Pi OS with desktop* image (~1.2GB) onto your Ubuntu machine.
```
$ curl https://downloads.raspberrypi.org/raspios_armhf/images/raspios_armhf-2021-11-08/2021-10-30-raspios-bullseye-armhf.zip --output ~/2021-10-30-raspios-bullseye-armhf.zip
$ cd ~
$ unzip 2021-10-30-raspios-bullseye-armhf.zip
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
$ sudo dd if=~/2021-10-30-raspios-bullseye-armhf.img of=/dev/??? bs=100M status=progress oflag=sync
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

Now you may proceed to the next step [[2]](#2).

# Enable I2C TPM 2.0

**This section is not tested yet!**

## Rebuild Raspberry Pi 4 Kernel

On your host machine.

Install dependencies:
```
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
$ git checkout 1.20210928
$ make kernelversion
5.10.63
```

Build:
```
$ KERNEL=kernel7l
$ make -j$(nproc) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2711_defconfig
$ make -j$(nproc) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
    Device Drivers  --->
        Character devices  --->
            <M> TPM Hardware Support  --->
                <M> TPM Interface Specification 1.2 Interface (I2C - Infineon)
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

Enable i2c by adding the following line to the file `/boot/config.txt`:
```
dtparam=i2c_arm=on
```
For more details about the parameter `i2c_arm`, please refer to [[3]](#3).

## Device Tree

Install device tree compiler on your host machine:
```
$ sudo snap install device-tree-compiler 
```

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

Build the i2c TPM device tree blob overlay:
```
$ git clone https://github.com/wxleong/tpm2-rpi4 ~/tpm2-rpi4
$ dtc -@ -I dts -O dtb -o tpm-i2c-infineon.dtbo ~/tpm2-rpi4/dts/tpm-i2c-infineon.dts
```

Copy the `tpm-i2c-infineon.dtbo` to `/boot/overlays/` and add the following line to the file `/boot/config.txt`:
```
dtoverlay=tpm-i2c-infineon
```

# References

<a id="1">[1] https://www.infineon.com/cms/en/product/evaluation-boards/iridium9670-tpm2.0-linux/</a><br>
<a id="2">[2] https://github.com/wxleong/tpm2-cmd-ref</a><br>
<a id="3">[3] https://forums.raspberrypi.com/viewtopic.php?t=236915</a><br>

# License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.