---
layout: post
title:  "Compiling U-Boot for Raspberry Pi 3"
date:   2019-04-16 07:41:10 -0300
categories: jekyll update
---

This post is an improvement of [this one][original-post]. I changed some bits and pieces that I found a bit confusing on the original post.

# Introduction

U-Boot is a multi-platform, open-source, universal boot-loader with comprehensive support for loading and managing boot images, such as the Linux kernel. This article is a quick start up guide on porting U-Boot for Raspberry Pi 3 (RPi3) board using an SD card.

This tutorial is intended for getting u-boot up and running on a raspberry pi 3 model b. Hopefully I'll be posting another tutorial showing how to boot a kernel image using u-boot.


# Requirements

For this tutorial, you'll need a raspberry pi 3 model B (and its power supply), an SD card, and a serial-to-usb cable adapter. Keep in mind to leave your RPi3 off until you're told to apply power to it.

## RPi3 Boot process

### Stage 1

To reduce cost, the Raspberry Pi (Model A & B) omits any on-board non-volatile memory used to store the boot loaders, Linux Kernels and file systems as seen in more traditional embedded systems. Rather, a SD/MMC card slot is provided for this purpose. The Raspberry PI Compute Module has 4GB eMMC Flash on-board.

stage 1 boot is in the on-chip ROM. Loads Stage 2 in the L2 cache. The Raspberry Pi's Broadcom BCM2835 system on a chip (SoC) powers up with its ARM1176JZF-S 700 MHz processor held in reset. The VideoCore IV GPU core is responsible for booting the system. It loads the first stage boot loader from a ROM embedded within the SoC. The first stage bootloader is designed to load the second stage bootloader (bootcode.bin) from a FAT32 or FAT16 file system on the SD Card.

### Stage 2

The second stage bootloader - **bootcode.bin** - is executed on the VideoCore GPU and loads the third stage bootloader - **start.elf**. Historically, yet another bootloader called loader.bin was loaded at this stage, but has since been phased out.

### Stage 3

The third stage bootloader - **start.elf** - is where all the action happens. It starts by reading **config.txt**, a text file containing configuration parameters for both the VideoCore (Video/HDMI modes, memory, console frame buffers etc) and loading of the Linux Kernel (load addresses, device tree, uart/console baud rates etc).

### Stage 4

We will use **config.txt** to tell **start.elf** to load a kernel image. We'll use our compiled **u-boot.bin** to pass as kernel image. This is the first thing that runs on the ARM processor. Our config.txt will look as:

    kernel = u-boot.bin

After completing above steps Booting is completed and RPI board is ready to take a commands from user.

# Formatting the SD card

Stick the SD card on your computer and make sure it is not mounted before continuing steps below.


    $ sudo fdisk /dev/mmcblk0


You'll want to remove any other partitions that may previously existed in your card. After deleting partitions, it's time to create a brand new one using the entire SD card. Once the new partition is created, mark it as FAT32 and later turn on the boot flag. All that can be accomplished by running the following commands within *fdisk*:

     - d # repeat that untill all partitions are removed
     - n # start creating a new partition
        - p # mark new partition as primary
        - [enter] # keep the defaults for all options
     - a # turn on boot flag for the partition just created
     - t # change partition type, select the option that contains FAT32 type
     - w # writes all changes to SD card

After creating the new partition, it's time to create a FAT32 file system on the SD card:

    $ sudo mkfs.vfat -F 32 -n boot /dev/mmcblk0p1

**NOTE: in this tutorial, I'm using Debian 9, which recognized the SD card as */dev/mmcblk0* and the partition as */dev/mmcblk0p1*, which may be different on your set up.**

Once formatting is done, mount the SD card in some folder or just unplug and plug it back on so that it will be automatically mounted to a folder.


# Compiling

We'll need a compiler and a cross compiler in order to get u-boot compiled to run on a raspberry.

    $ sudo apt install gcc-arm-linux-gnueabi

Get the source code by cloning the U-Boot git repository :

    $ git clone -depth 1 -branch v2017.11 git://git.denx.de/u-boot.git v2017.11
    $ echo -e '\n#define CONFIG_CMD_BOOTZ' >> v2017.11/include/configs/rpi.h # enables bootz command for u-boot
    $ sudo make -C v2017.11/ CROSS_COMPILE=aarch64-linux-gnu- rpi_3_defconfig
    $ sudo make -C v2017.11/ CROSS_COMPILE=aarch64-linux-gnu-

Many new files must have been generated. The important one to look for is **u-boot.bin**. Copy that file into the SD card. Don't use *dd* command, just regular *cp* is enough.


# Retrieving raspberry pi boot files

You will need two files to be copied into the SD card to complement the boot process of the RPi3. Go to [raspberry pi firmware page][rpi-firmware-github] and download **bootcode.bin** and **start.elf**. Create a copy of those two files in the SD card. 

    $ wget https://github.com/raspberrypi/firmware/raw/master/boot/bootcode.bin
    $ wget https://github.com/raspberrypi/firmware/raw/master/boot/start.elf

# Creating the configuration file

Now you need to create a configuration file that will tell the RPi3 how which kernel image do load and set up the serial connection:

    cat > config.txt <<EOM
    enable_uart=1
    arm_control=0x200
    kernel=u-boot.bin
    dtparam=i2c_arm=on
    dtparam=spi=on
    EOM

This will create **config.txt**, just move it to the SD card along with the other files copied in steps above.

You are done retrieving files to get u-boot started. Safely unmount the SD card and plug it into your RPi3.


# Connecting the RPi3 to the serial port

Now you need to get hacky on the hardware part of the tutorial. The two images below illustrates the schematics for a regular usb-to-serial cable adapter and the pins of a normal RPi3 model B.

{% include image.html
  path="usb_serial_scheme.png"
  caption="Figure 1: USB-to-Serial cable adapter"
  css="height: 333px" %}

{% include image.html
  path="raspberry-pi-15b.jpg"
  caption="Figure 2: Raspberry Pi 3 pins"
  css="height: 500px" %}

- Connect the **black** wire to **pin #6** (ground).
- Connect the **white** wire to **pin #8** (RXD to TXD)
- Connect the **green** wire to **pin #10** (TXD to RXD)

**NOTE: Be careful when wiring because any bad connection will prevent serial communication from happening**


# Communicating with RPi3 through serial port

You're done connecting the wires, now we need to make sure the serial connection was successfully made. Check it by running the following command then **connect the USB cable to the computer**.

    $ sudo dmesg -w

The command above simply opens a dynamic system log viewer. Anything happening in the kernel will show up there in real time! When you connected the usb to your computer, a message like this one below should appear:

    [142054.216068] usb 4-1.2: new full-speed USB device number 8 using ehci-pci
    [142054.324715] usb 4-1.2: New USB device found, idVendor=067b, idProduct=2303
    [142054.324719] usb 4-1.2: New USB device strings: Mfr=1, Product=2, SerialNumber=0
    [142054.324721] usb 4-1.2: Product: USB-Serial Controller
    [142054.324724] usb 4-1.2: Manufacturer: Prolific Technology Inc.
    [142054.325177] pl2303 4-1.2:1.0: pl2303 converter detected
    [142054.326876] usb 4-1.2: pl2303 converter now attached to ttyUSB0

If you didn't see a message like that after you plugged the usb, something's wrong. Re-wire everything and make sure all connections and wires are good.

Next, we need a way to communicate with RPi3 via serial protocol. There are plenty ways of doing so, but we'll use **minicom** in this tutorial, just because. Get it installed by running:

    $ sudo apt install minicom
    $ sudo minicom -s

The second part of the command will start minicom's setup. It should show something like this

    +-----[configuration]------+
    | Filenames and paths      |
    | File transfer protocols  |
    | Serial port setup        |
    | Modem and dialing        |
    | Screen and keyboard      |
    | Save setup as dfl        |
    | Save setup as..          |
    | Exit                     |
    | Exit from Minicom        |
    +--------------------------+

Scroll to *Serial port setup* then make sure settings are as follows:

    +-----------------------------------------------------------------------+
    | A -    Serial Device      : /dev/ttyUSB0                              |
    | B - Lockfile Location     : /var/lock                                 |
    | C -   Callin Program      :                                           |
    | D -  Callout Program      :                                           |
    | E -    Bps/Par/Bits       : 115200 8N1                                |
    | F - Hardware Flow Control : No                                        |
    | G - Software Flow Control : No                                        |
    |                                                                       |
    |    Change which setting?                                              |
    +-----------------------------------------------------------------------+

The **Serial Device** name should be the same as the one you found output by dmesg command. Press escape or enter to leave the setup menu then select *Exit*

After exiting the setup menu, you should see something like:

    Welcome to minicom 2.7

    OPTIONS: I18n                                                                
    Compiled on Apr 22 2017, 09:14:19.                                           
    Port /dev/ttyUSB0, 07:47:40                                                  
                                                                                 
    Press CTRL-A Z for help on special keys                                      

Finally, apply power to RPi3 and u-boot should start off with a message like this one below:

    U-Boot 2019.01-dirty (Apr 15 2019 - 11:40:07 -0300)                          
                                                                                 
    DRAM:  128 MiB                                                               
    RPI 3 Model B (0xa02082)                
    MMC:   mmc@7e202000: 0, sdhci@7e300000: 1
    Loading Environment from FAT... *** Warning - bad CRC, using default environment
                                            
    In:    serial                           
    Out:   vidconsole                       
    Err:   vidconsole
    Net:   No ethernet found.
    starting USB...

And you are done!

[original-post]: https://pranavlavhate761157457.wordpress.com/2018/05/14/porting-of-u-boot-on-raspberrypi-3/
[rpi-firmware-github]: https://github.com/raspberrypi/firmware/tree/master/boot
