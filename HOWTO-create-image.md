# HOWTO Create Minimal Debian Image
Very brief and not very detailed overview of how the Debian installation 
image was created using a Seagate Central with a working serial console
connection.

These instructions are only brief notes describing how I created the
Debian installation images. There is no need for anyone to go through
this procedure. This is only documented

This something that should only be done by someone who has a relatively
good understanding of what's happening, is fairly familiar with the
Linux installation process and is fairly familiar with u-boot and the
Linux bootup process on 32 bit "armel" style platforms.


Recompile "anna"

Expand and rebuild uInitrd

Boot with new uInitrd and uImage

Debian installation

Post Debian installation steps

Creating the image


## Prerequisites

Debian based build host

Pre-compiled up to date Linux kernel for Seagate Central using a 4K page size as
per the Seagate-Central-Slot-In-v5.x-Kernel project at

https://github.com/bertofurth/Seagate-Central-Slot-In-v5.x-Kernel

Serial console connection on Seagate Central




## Expand uInitrd
uInitrd is a u-boot style "ram disk" that may be loaded just after the kernel is 
activated in order to perform basic system initialization functions. In normal
operation a Seagate Central does not use uInitrd but the Debian installation
process makes use of this feature.

BERTO: TODO: MIGHT BE POSSIBLE TO DO THIS WITHOUT SERIAL CONSOLE BY PUTTING uInitrd
into a disk image and loading it on a root_File_system.

This section is based on instructions derived from the following URLs

* "How to view, modify and create initrd"
https://www.thegeekstuff.com/2009/07/how-to-view-modify-and-recreate-initrd-img/

* Extracting content from the file uInitrd
http://h-wrt.com/en/mini-how-to/uInitrd

First download an existing Debian installation uInitrd from a similar "armel"
style platform to the build host.

The uInitrd image we've used in building the Debian installation here is based on the
"lacie" platform which is similar (but not the same as) the Seagate Central
in terms of its architecture. This was the latest at the time of writing
(Nov 2022).

http://ftp.debian.org/debian/dists/stable/main/installer-armel/20210731+deb11u5/images/kirkwood/network-console/lacie/uInitrd

It may be that there are later images available. Check the date on the uImage 
you're downloading. If it's later than the writing of this document (Nov 2022)
then it may be that it already contains the updated "anna" tool. 

Expand the downloaded uImage on the build host as follows. You must execute
these commands as the root user.

    # List properties of uInitrd
    mkimage -l ./uInitrd
    
    # Remove 64 byte u-boot header to leave gzipped ramdisk
    dd if=./uInitrd of=./initrd.gz skip=1 bs=64

    # Expand gzipped ramdisk
    gunzip initrd.gz
    
    # Expand filesystem image into a new directory
    mkdir new
    cd new
    cpio -i -F ../initrd

At this point you should be sitting in the expanded initrd filesystem. Run
the "ls" command as per the following example to check. You should see
all the typical directories seen in a Linux root (bin, var, lib, and so forth)

    root@Debian:~/new# ls
    bin  dev  etc  init  initrd  lib  media  mnt  proc  run  sbin  sys  tmp  usr  var

## Check the version of "anna" in use
"anna" is the very simple Debian package installation tool used during 
Debian installation.

We found that during the netconsole Debian installation process we were
getting an error message saying

    No kernel modules found                                                              
    
    No kernel modules were found. This probably is due to a mismatch
    between the kernel used by this version of the installer and the
    kernel version available in the archive.
    
    You should make sure that your installation image is up-to-date,
    or - if that's the case - try a different mirror, preferably
    deb.debian.org.

The installation would terminate and go no further.

This message would occur because the kernel being used on the Seagate
Central is a custom kernel that isn't going to be found in the Debian 
repository.

It seems that we need an updated version of the "anna" tool that can
let you skip this but this updated version (v1.84) is not yet included
in the installer as of the time of writing of this document (Nov 2022).

The problem is described by Debian bug 998668 as seen at

https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=998668

At this point we need to check whether the version of "anna" included in
the downloaded uImage is v1.84 or later. This can be done by searching
for a particular string as follows from the expanded file system's
root.

    grep -A3 "no_kernel_modules" ./var/lib/dpkg/info/anna.templates
    
If following the "Template: anna/no_kernel_modules" line the output contains 
the statement "Type: error" then we have the old version of "anna" that must
be updated. This is shown in the following example.

    root@Debian:~/new# grep -A3 "no_kernel_modules" ./var/lib/dpkg/info/anna.templates
    Template: anna/no_kernel_modules
    Type: error
    Description: No kernel modules found
     No kernel modules were found. This probably is due to a mismatch between

If, however, the output contains the statement "Type: boolean" then version v1.84
or later of "anna" in included. See the following example showing this to be the
case.

    root@Debian:~/new# grep -A3 "no_kernel_modules" ./var/lib/dpkg/info/anna.templates
    Template: anna/no_kernel_modules
    Type: boolean
    Default: false
    # :sl2:

If the uInitrd image already contains a recent enough version of anna (v1.84 or later) then
there is no need to modify uInitrd and the downloaded version can be used without
further modification.

## Recompile "anna"
If the uInitrd contains the old version of "anna" then "anna" a new version will
need to be cross compiled.

On a Debian Linux build machine as the root user we make sure the 
appropriate pre-requisites needed to build anna are installed.

    # Make sure an appropriate cross compiler is installed
    apt install gcc-10-arm-linux-gnueabi
    
    # Install u-boot tools for "mkimage" utility
    apt install u-boot-tools
    
    # Allow this machine to download "armel" style components
    dpkg --add-architecture armel
    apt-get update
    
    # Install required libraries
    apt-get install libdebconfclient0-dev:armel
    apt-get install libdebian-installer4-dev:armel

This next part of the procedure can be executed as a normal user on
the Debian build machine

    # Download the latest "anna" source (v1.84 or later)
    git clone https://salsa.debian.org/installer-team/anna
    cd anna
    
    # Make sure to use the cross compiler during the build
    CC=arm-linux-gnueabi-gcc-10 make
    
    # Remove debugging symbols to save disk space and memory
    arm-linux-gnueabi-strip anna
    
These commands should have built the "anna" binary for "armel" platform.
Confirm this by running the "file" command on "anna" and make sure that
it has been built for the "ARM" architecture and NOT for the build machine's
architecture as per the following example.

    user@Debian:~/anna# file anna
    anna: ELF 32-bit LSB pie executable, ARM, EABI5 version 1 (SYSV), dynamically 
    linked, interpreter /lib/ld-linux.so.3, for GNU/Linux 3.2.0, stripped

The source tree will also contain the file "debian/anna.templates" that will be
needed to be added to the uInitrd image that will be created in the next part
of the process.

## Modify and rebuild uInitrd
At this stage we need to copy the newly built "anna" tool into the expanded
uInitrd file system. These commands must be executed as the root user from the
root folder of the expanded uInitrd. Note that these instructions assume that
the cross compiled "anna" tool is located at ~user/anna

    # Copy the anna and data files over the top of the existing ones
    cp ~user/anna/anna ./bin/anna
    cp ~user/anna/debian/anna.templates ./var/lib/dpkg/info/anna.templates

    # Create a disk image based on the updated contents of this folder 
    find . | cpio --create --format='newc' > ../newinitrd
    
    # gzip the disk image
    cd ..
    gzip newinitrd
    
    # Create the uInitrd image with appropriate properties
    mkimage -A arm -T ramdisk -C gzip  -d ./newinitrd.gz -n "debian-installer ramdisk" ./uInitrd.new

You will now have a "uInitrd.new" file that contains

# Upload kernel and ramdisk to boot partition on Seagate Central
Copy the new 4K kernel and the new uInitrd to the appropriate boot partition on
the Seagate Central.

The easiest way to do this would be to upload these files to a Seagate Central running
stock Seagate Central firmware, ssh to the unit, mount the relevant boot partition
and then simply copy the files onto the partition.



# Boot uInitrd from serial console
To load specific uImage and uInitrd from uboot u-boot from the Seagate Central console
boot the unit and make sure to hit a key within the first few seconds of the
unit booting up to get to the u-boot prompt as per the following example

    U-Boot 2008.10-mpcore (Nov 27 2012 - 02:15:26)
    Cirrus model:(SENTINEL) release v1.3

    CPU: Cavium Networks CNS3000
    ID Code: XXXXXXXX (Part number: 0xB02, Revision number: 4)
    CPU ID: 900
    Chip Version: c

    DRAM:  512 MB
    Parallel Flash:  0 kB
    Flash Manufacturer: MX
    Serial Flash: 512 kB
    Serial Flash:
    Bank # 1:  Nb pages: 2048  Page Size: 256
      Size:    524288 bytes,  Logical address: 0x60000000
      Area 0: 60000000 to 60FFFFFF      SPI flash
    In:    serial
    Out:   serial
    Err:   serial
    CPU works at 700 MHz (700/1/1)
    DDR2 Speed is 400 MHz
    Restoring RTC
    Hit any key to stop autoboot:  2



test

    # Note the use of "lowmem". It just seems to work better.
    
    # Note, substitute the name of the uInitrd on your /dev/sda1 device
    # For example, it might be called "uInitrd.new"
    # 
    # The uImage being used should be installed as "uImage" on the
    # /dev/sda1 partition as this is the image the u-boot bootloader will
    # try to load when the unit is reset.
    #
    # 
    # Paste these commands in one at a time as some of them can take
    # 10-20 seconds to execute
    #
    setenv current_kernel kernel1
    setenv num_boot_tries 0
    saveenv
    setenv bootargs 'console=ttyS0,38400 mem=256M root=/dev/sda3 rw lowmem=1'
    # Might need to do "scsi init" twice in a row if there are errors
    scsi init
    ext2load scsi 0:1 0x4000000 uImage
    ext2load scsi 0:1 0x4A00000 uInitrd

    bootm 0x4000000 0x4A00000


# After this you should see the Linux kernel booting



setenv current_kernel kernel2
setenv num_boot_tries 0
saveenv
setenv bootargs 'console=ttyS0,38400 mem=256M root=/dev/sda4 rw lowmem=1'
scsi init
ext2load scsi 0:2 0x4000000 uImage
ext2load scsi 0:2 0x4A00000 uInitrd

bootm 0x4000000 0x4A00000


