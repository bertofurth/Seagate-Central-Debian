# HOWTO Create Minimal Debian Image
This a procedure detailing how the Debian installation image for
Seagate Central was created. There is no need for anyone to perform this
procedure because prebuilt Seagate Central Debian installation images
are made available in the releases section of this project. This
procedure is documented here for the sake of completeness and openness 
in case someone wants to perform the process themselves.

This procedure should only be performed by someone who has a relatively
good understanding of what's happening, is fairly familiar with the
Linux installation process and is fairly familiar with u-boot and the
Linux bootup process on 32 bit "armel" style platforms. For this reason
the step by step details of how to perform every step is not included
because it's assumed that anyone with the required level of skill to
perform this procedure will already be familiar with things like how
to transfer files, how to mount filesystems, how to use a terminal
emulator and how to use basic Linux commands.

Most importantly, this procedure requires access to the serial console
of the Seagate Central. This serial console needs to be manually installed
by physically opening up the unit, soldering wires onto the circuit board,
and connecting a terminal system to the unit's serial console. Note that
unless this soldering process is done carefully, the unit may be physically
damaged and become inoperative!

## Warning
**Performing modifications of this kind on the Seagate Central is not without
risk. Performing the procedure documented in these instructions will likely 
void any warranty and in rare circumstances may lead to the device becoming
unusable or damaged.**

**Do not use the products of this project in a mission critical system or in
a system that people's health or safety depends on.**

**This procedure will overwrite any data or settings on the unit being worked 
on. Be sure to backup any important data before proceeding.**

**This project is not endorsed or supported by the original vendors or
manufacturers of the Seagate Central NAS.**

## Prerequisites
* A working serial console connection to the Seagate Central (Hard)
* Root access to a Debian based Linux distribution on another machine.
* A USB hard drive reader.

## Procedure
### Connect the Serial Console to the Seagate Central
Connecting the physical serial console to the Seagate Central involves,
disassembling the unit, removing the hard drive, removing the circuit
board, then soldering leads to the 4 circuit board pads next to the
"J2" or "J4" label on bottom side of the Seagate Central circuit board.

You may also need to drill some small holes through the bottom of the 
unit's plastic case and internal metal frame so that the leads can be
fed outside the unit when you put it back together. I normally
drill 4 small holes so that the leads can be kept seperate (see note
below about cross talk)

The following diagram is borrowed from

http://seagate-central.blogspot.com/2014/01/blog-post.html

Archive : https://archive.ph/ONi4l


    TX   RX
    []---[]
    |     |
    |     |  J4
    |     |
    []---[]
    VCC  GND
      
This shows which pads on the circuit board correspond to which TTL serial
lines.

TODO: Put some pictures of a Seagate Central circuit board with leads
soldered to it into a folder in the project.

You will most likely need to connect a TTL to RS232 converter to the 
leads coming from the Seagate Central in order to use a standard RSR232 
serial connection to your terminal/computer.

The serial connection parameters are 38400-8-N-1.

You can test the console connection by powering on the circuit board without
the hard drive connected. If the console is working then you should see
the unit start to boot up and you should be able to enter the u-boot
environment. 

If you find that **some** of the characters being transmitted from the
Seagate Central appear as "garbage" then it may be due to crosstalk between
the leads. This is where the electrical signal in one lead interferes with 
the signal in an adjacent lead. This may particularly be the case if you are
typing at the time when you see the garbage characters.

Try your best to keep the leads as physically separate as possible. I
accomplished this by having 4 separate holes in the bottom of the Seagate
Central and feeding a single lead through each hole.

If **all** of the characters coming back from the unit are garbage then you
may have selected the wrong baud rate. We need to use baud rate setting of
38400bps for the Seagate Central console, but most terminal emulation programs 
use a default of 9600 or 115200.

If the circuit board is not fitted into the chassis try to only keep the 
board powered on for short bursts of time because the CPU will not be connected
to it's heatsink.

Once you've confirmed that the serial console is working, reassemble the unit
and install the hard drive but there's no need to put the plastic cover back 
on for the moment because later in the procedure the hard drive will
need to be removed again.

### Obtain the 4K page Linux kernels for Seagate Central
Refer to the "Releases" section of the Seagate-Central-Slot-In-v5.x-Kernel project
to download the latest version of the 4K page size Linux kernels for Seagate Central

https://github.com/bertofurth/Seagate-Central-Slot-In-v5.x-Kernel/releases

Note that we will be downloading **two** kinds of kernel images. 

uImage.4k - 4k page kernel with smp enabled
uImage.4k.nosmp - 4k page kernel with smp disabled

As an example let's start working in a base directory on our Debian building
machine and download these files. Check the URL you are using to download.

    wget https://github.com/bertofurth/Seagate-Central-Slot-In-v5.x-Kernel/releases/download/v1.6/uImage.4k.nosmp
    wget https://github.com/bertofurth/Seagate-Central-Slot-In-v5.x-Kernel/releases/download/v1.6/uImage.4k
    
The reason we need to download two kernels is that our tests have shown that
under some circumstances the unit is not physically capable of using SMP
mode. This is because of heat dissipation issues. There is more discussion
about this issue in the main README.md file in this project under the
section entitled "Choice of Linux kernels (SMP / no SMP)".

We will set up the installation image to install the "nosmp" version of the kernel
by default and users can chose to manually change to the smp enabled version if
they want to take the risk of running it.

Another point worht noting is that the Seagate Central natively uses a 64K page
size kernel for the sake of disk performance however the 4K page size kernel is
significantly more memory efficient. Since the Debian operating system can be more
memory hungry than the native Seagate Central firmware it was judged necessary to
make this compromise between disk performance and memory efficiency.

If you wish, you can recompile the kernel according to your own desire and
specifications.

### Download and expand uInitrd
The Debian installation uInitrd is a special image that provides a basic
operating system and utilities to facilitate installing a full version
of the Debian operating system.

This section is based on instructions derived from the following URLs

* "How to view, modify and create initrd"
https://www.thegeekstuff.com/2009/07/how-to-view-modify-and-recreate-initrd-img/

* Extracting content from the file uInitrd
http://h-wrt.com/en/mini-how-to/uInitrd

The Seagate Central shares the same CPU architecture as the LaCie family
of NAS products which is supported on Debian. For this reason we make
use of the "lacie" Debian uInitrd installation image.

As of writing the latest version of this uInitrd image is available at

http://ftp.debian.org/debian/dists/stable/main/installer-armel/current/images/kirkwood/network-console/lacie/uInitrd

In order to manipulate this image make sure your Debian system has
the "u-boot-tools" package installed so that the "mkimage" utility is
available. 

    apt-get install u-boot-tools

Download and expand the uInitrd image as per the following example. Note that
some commands need to be executed using root privileges.

    # Download the appropriate Debian installation uInitrd image
    $ wget http://ftp.debian.org/debian/dists/stable/main/installer-armel/current/images/kirkwood/network-console/lacie/uInitrd

    # List properties of uInitrd
    $ mkimage -l ./uInitrd
    Image Name:   debian-installer ramdisk
    Created:      Thu Dec 15 11:11:52 2022
    Image Type:   ARM Linux RAMDisk Image (gzip compressed)
    Data Size:    10617039 Bytes = 10368.20 KiB = 10.13 MiB
    Load Address: 00000000
    Entry Point:  00000000

    # Remove u-boot header from uInitrd to leave gzipped ramdisk
    $ dd if=./uInitrd of=./initrd.gz skip=1 bs=64
    165891+1 records in
    165891+1 records out
    10617039 bytes (11 MB, 10 MiB) copied, 0.323688 s, 32.8 MB/s

    # Decompress the gzipped ramdisk
    $ gunzip initrd.gz
    
    # Create a new directory to expand and manipulate the ram disk
    $ mkdir new
    $ cd new

    # Expand the contents of the ram disk. Note the following "cpio" command
    # must be run with root privileges, hence the "sudo" prefix.
    $ sudo cpio -i -F ../initrd
    47801 blocks
    
    # Check the contents of the directory. The owner of all these
    # operating system directories should be "root"
    $ ls -l
    total 60
    drwxr-xr-x  2 root root 4096 Apr  9 01:18 bin
    drwxr-xr-x  2 root root 4096 Apr  9 01:18 dev
    drwxr-xr-x 11 root root 4096 Apr  9 01:18 etc
    -rwxr-xr-x  1 root root  456 Apr  9 01:18 init
    drwxr-xr-x  2 root root 4096 Apr  9 01:18 initrd
    drwxr-xr-x 14 root root 4096 Apr  9 01:18 lib
    drwxr-xr-x  2 root root 4096 Apr  9 01:18 media
    drwxr-xr-x  2 root root 4096 Apr  9 01:18 mnt
    drwxr-xr-x  2 root root 4096 Apr  9 01:18 proc
    drwxr-xr-x  3 root root 4096 Apr  9 01:18 run
    drwxr-xr-x  2 root root 4096 Apr  9 01:18 sbin
    drwxr-xr-x  2 root root 4096 Apr  9 01:18 sys
    drwxr-xr-x  2 root root 4096 Apr  9 01:18 tmp
    drwxr-xr-x  6 root root 4096 Apr  9 01:18 usr
    drwxr-xr-x  6 root root 4096 Apr  9 01:18 var
    
    # Change back to the base directory
    $ cd ..

### Check the version of "anna" included in uInitrd 
"anna" is the very simple Debian package installation tool used during 
installation of the Debian operating system

Older versions of "anna" are unable to install Linux on the Seagate Central 
and need to be replaced with an up to date version. See the section at
the end of this document entitled "Notes about anna" for more details. 

To detect whether an old version of anna is included run the following
command from the directory where the uInitrd has been expanded to

    cat new/var/lib/dpkg/info/anna.templates | sed -n '/no_kernel_modules/{n;p}'

If the output of the above command is

    Type: error

then you have "old" anna installed and you need to create and insert a new
version by following the steps below labelled with "(old anna)".

If the output is

    Type: boolean

then you have a good version of "anna" and can move past the steps labelled
with "(old anna)".

As of the time of writing (April 2023) there are no official versions of the
Debian installation uInitrd image with an appropriate version of "anna"
so you will probably have to proceed with the steps below that are labelled 
with "(old anna)".

### Download "anna" source and recompile "anna" (old anna)
If the uInitrd contains the old version of "anna" then a new version will
need to be cross compiled.

On a Debian Linux build machine as the root user we make sure the 
appropriate pre-requisites needed to build anna are installed.

    # Make sure an appropriate cross compiler is installed
    sudo apt install gcc-10-arm-linux-gnueabi
    
    # Install u-boot tools for "mkimage" utility
    sudo apt install u-boot-tools
    
    # Install the "git" utility to download the source code
    sudo apt install git
    
    # Allow this machine to download "armel" style components
    sudo dpkg --add-architecture armel
    sudo apt-get update
    
    # Install required development libraries
    sudo apt-get install libdebconfclient0-dev:armel
    sudo apt-get install libdebian-installer4-dev:armel

This next part of the procedure can be executed as a normal user on
the Debian build machine to download and compile anna. Make sure
to execute the following commands in a different directory than
that which the expanded uInitrd is in. In this example we place
the new source code in the "anna" subdirectory of the user's
home directory.

    # Download the latest "anna" source (v1.84 or later)
    # to the base directory
    git clone https://salsa.debian.org/installer-team/anna
    cd anna
    
    # Build anna with the arm cross compiler
    CC=arm-linux-gnueabi-gcc-10 make
    
    # Remove debugging symbols to save disk space and memory
    arm-linux-gnueabi-strip anna
    
    # Confirm that "anna" has been built for ARM
    file anna
    
    # Change back to the base directory
    cd ..
    
These commands should have built the "anna" binary for "armel" platform.
We confirm this by running the "file" command on "anna" and make sure that
it has been built for the "ARM" architecture and NOT for the build machine's
architecture (probably x86-64 or aarch64) as per the following example.

    # file anna
    anna: ELF 32-bit LSB pie executable, ARM, EABI5 version 1 (SYSV), dynamically 
    linked, interpreter /lib/ld-linux.so.3, for GNU/Linux 3.2.0, stripped

The source tree will also contain the file "debian/anna.templates" that will be
needed to be added to the uInitrd image that will be created in the next part
of the process.

### Copy the updated anna into uInitrd (old anna)
At this stage we need to copy the newly built "anna" tool into the expanded
uInitrd file system. These commands must be executed with root privileges 
from the base folder. 

    # Copy "anna" and required data files over the top of the existing ones
    sudo cp anna/anna new/bin/anna
    sudo cp anna/debian/anna.templates new/var/lib/dpkg/info/anna.templates

### Modify and rebuild uInitrd (old anna)
Here we rebuild a uInitrd format file containing the Linux operating
system. Note that some of these commands must be executed with root
privileges.

    # Change into the new directory
    cd new
    
    # Create a disk image based of the updated contents of this folder 
    sudo find . | cpio --create --format='newc' > ../newinitrd
    
    # gzip the disk image
    cd ..
    gzip newinitrd
    
    # Create the uInitrd image with appropriate properties
    mkimage -A arm -T ramdisk -C gzip  -d ./newinitrd.gz -n "debian-installer ramdisk" ./uInitrd.new
    
    # Confirm that the uinitrd.new file has been created.
    ls -l uInitrd.new

You will now have a "uInitrd.new" file that contains the updated "anna"
utility. It should be roughly 10MB in size.

### Upload kernel and uInitrd ramdisk to the boot partition on the Seagate Central
The goal of this section is to copy the new 4K kernels and the new uInitrd to the
appropriate boot partition on the Seagate Central.

The easiest way to accomplish this is to upload these files to the Seagate Central,
then ssh to the unit and login as root, mount the relevant boot partition and
then copy the files onto that partition.

I would suggest placing the files on the primary/first boot partition, /dev/sda1,
and to leaving the secondary partition, /dev/sdX2, with the native Seagate Central kernel.
The examples in this document will be given on this basis.

Note that even though we will be copying both types of kernel images to the unit,
the kernel image we wish to use during installation must be copied to the relevant
boot partition using the name "uImage" so as to overwrite the native "uImage" file
on that partition. In the example below we chose to use the "nosmp" kernel for the
sake of system stability. The uInitrd image name can be anything, but you must take a
note of the name. We use "uInitrd.new" in this document.

The following shows an example session on a Seagate Central that has had the
relevant files already transferred to it. (N.B. You must be logged in as root)

    # Check that the files have been uploaded to the Seagate Central
    root@NAS-X:/home/admin# ls -l
    total 18628
    -rw-r--r-- 1 admin 1000  4316328 Apr 17 22:15 uImage.4k
    -rw-r--r-- 1 admin 1000  4125320 Apr 17 22:15 uImage.4k.nosmp
    -rw-r--r-- 1 admin 1000 10628646 Apr 17 22:15 uInitrd.new
    
    # Mount the primary boot partition
    root@NAS-X:/home/admin# mount /dev/sda1 /mnt/sda1
    root@NAS-X:/home/admin# ls -l /mnt/sda1
    total 2945
    drwx------ 2 root root   12288 Apr  9 17:06 lost+found
    -rw-r--r-- 1 root root 2989612 Apr  9 17:11 uImage
    
    # Copy the kernels and uInitrd to the boot partition
    # Note that here we set the nosmp image to be the
    # "uImage" that will be booted.
    root@NAS-X:/home/admin# cp uImage.4k.nosmp /mnt/sda1/uImage
    root@NAS-X:/home/admin# cp uImage.4k /mnt/sda1/
    
    # Copy the uInitrd.new image to the boot partition
    root@NAS-X:/home/admin# cp uInitrd.new /mnt/sda1/
    
    # Check that the images are present on the boot partition
    root@NAS-X:/home/admin# ls -l /mnt/sda1
    total 18714
    drwx------ 2 root root    12288 Apr 17 15:00 lost+found
    -rw-r--r-- 1 root root  4125320 Apr 17 22:18 uImage
    -rw-r--r-- 1 root root  4316328 Apr 17 22:18 uImage.4k
    -rw-r--r-- 1 root root 10628646 Apr 17 22:20 uInitrd.new

### Prepare to boot the installer via the serial console
Reboot the unit with the serial console in place (use the "reboot" command).
As the unit boots up, output similar to the following will appear on the console.


    U-Boot 2008.10-mpcore (Apr  2 2013 - 14:41:52)
    Cirrus model:(SENTINEL) release v1.3
    
    CPU: Cavium Networks CNS3000
    ID Code: 410fb024 (Part number: 0xB02, Revision number: 4)
    CPU ID: 900
    Chip Version: c
    
    DRAM:  256 MB
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
    Hit any key to stop autoboot:  5

When the "Hit any key to stop autoboot" countdown message appears you should 
hit a key to interrupt the boot process and you should see the "u-boot"
command prompt which in the Seagate Central says "Whitney #".

If you hit a key too late and the Linux kernel starts booting then just power
off the unit and try again!

At ths point we need to configure u-boot to make sure to attempt to boot the
primary partition by issuing the following commands at the "Whitney#" u-boot
prompt.

    setenv num_boot_tries 0
    setenv current_kernel kernel1
    saveenv

Here is an example showing these commands being executed.

    Whitney # setenv num_boot_tries 0
    Whitney # setenv current_kernel kernel1
    Whitney # saveenv
    Saving Environment to dataflash...
    
    Serial Flash Sector 56 Erase OK!
    Serial Flash Sector 57 Erase OK!
    Serial Flash Sector 58 Erase OK!
    Serial Flash Sector 59 Erase OK!
    Serial Flash Sector 60 Erase OK!
    Serial Flash Sector 61 Erase OK!
    Serial Flash Sector 62 Erase OK!
    Serial Flash Sector 63 Erase OK!
    0x00008000
    
    Serial Flash Sector 48 Erase OK!
    Serial Flash Sector 49 Erase OK!
    Serial Flash Sector 50 Erase OK!
    Serial Flash Sector 51 Erase OK!
    Serial Flash Sector 52 Erase OK!
    Serial Flash Sector 53 Erase OK!
    Serial Flash Sector 54 Erase OK!
    Serial Flash Sector 55 Erase OK!
    0x00008000

### Boot the Debian installation uInitrd from the serial console
To load the required kernel uImage and Debian installation ramdisk
from u-boot issue the following commands.

    # Paste these commands in one at a time as some of them can take
    # up to 20 seconds to execute
    #
    setenv bootargs 'console=ttyS0,38400 mem=256M root=/dev/sda3 rw lowmem=1'
    
    # Might need to execute this "scsi init" twice if any errors appear
    scsi init
    
    # Preload the kernel and uInitrd into memory
    ext2load scsi 0:1 0x24000000 uImage
    ext2load scsi 0:1 0x24A00000 uInitrd.new

    # Boot the kernel and uInitrd
    bootm 0x24000000 0x24A00000

Here is an example output of issuing the above commands. After issuing these commands
you should see the Linux kernel booting as per the example below.

    Whitney # setenv bootargs 'console=ttyS0,38400 mem=256M root=/dev/sda3 rw lowmem=1'
    Whitney # scsi init
    
    Initialize SCSI
    AHCI 0001.0100 32 slots 2 ports 3 Gbps 0x3 impl SATA mode
    flags: ncq stag pm led clo only pmp pio slum part
    scanning bus for devices...
    Supprt LBA48 addressing.
      Device 0: (0:0) Vendor: ATA Prod.: ST4000DM000-1F21 Rev: CC52
                Type: Hard Disk
                Supports 48-bit addressing
                Capacity: 3815447.8 MB = 3726.0 GB (7814037168 x 512)
    Whitney # ext2load scsi 0:1 0x24000000 uImage
    
    4316328 bytes read
    Whitney # ext2load scsi 0:1 0x24A00000 uInitrd.new

    10625068 bytes read
    Whitney # bootm 0x24000000 0x24A00000
    enter do_eth_down!!!
    ## Booting kernel from Legacy Image at 24000000 ...
       Image Name:   Linux-5.15.86-sc
       Created:      2023-01-06  11:43:31 UTC
       Image Type:   ARM Linux Kernel Image (uncompressed)
       Data Size:    4316264 Bytes =  4.1 MB
       Load Address: 22000000
       Entry Point:  22000000
    ## Loading init Ramdisk from Legacy Image at 24a00000 ...
       Image Name:   debian-installer ramdisk
       Created:      2023-04-09   7:48:38 UTC
       Image Type:   ARM Linux RAMDisk Image (gzip compressed)
       Data Size:    10625004 Bytes = 10.1 MB
       Load Address: 00000000
       Entry Point:  00000000
       Loading Kernel Image ... OK
    OK

    Starting kernel ...

    Uncompressing Linux...  done, booting the kernel.
    . . . .

Once the system has booted up you should be greeted with a prompt warning
you that "Low memory mode" has been engaged. This is expected and necessary
on the Seagate Central as this unit only has limited memory resources. Click
on "continue" to proceed to the Debian installation process.

### Debian installation
The Debian installation process involves navigating a number of text based
menus.

In this section we are not going to detail every step. In general with the Debian
installation you should just proceed according to the menus that appear. Usually
selecting "continue" will be fine.

The whole installation can be done just using the serial console, or you can
ssh into the unit after the network has been configured.

GNU screen is running the entire time so you can scroll between windows
to view system status, logs and a command line by using standard GNU screen
keys (CTRL-A space or CTRL-A backspace to move forward/backwards)

If at any stage you see any error messages such as

    Title: Install the base system
    Description: Warning: Failure while unpacking required packages

then just click on "Continue".

If any stage of the installation fails with a message saying 

     Title: Installation step failed
     Description: An installation step failed. You can try to run the failing item again from the menu, or skip it and choose something else. 

then just select "Continue". This should take you back to the Debian 
installer main menu. Scroll to the step that failed and try it again.

Here are some brief notes describing notable parts of the installation 
process and some suggested answers.

    Title: Low memory Mode
    Description: Entering low memory mode
    Answer: Continue

    Title: Configure the network
    Description: Please enter the hostname for this system.
    Suggested Answer: SC-debian   (or whatever you like)

    Title: Configure the network
    Description: Domain Name
    Suggested Answer: lan

    Title: Continue installation remotely using SSH
    Description: You need to set a password for remote access to the Debian installer.
    Suggested Answer: SCdebian2022

    Title: Continue installation remotely using SSH
    Description: Please enter the same remote installation password again
    Suggested Answer: SCdebian2022

    Title: Continue installation remotely using SSH
    Description: To continue the installation, please use an SSH client

At this point you can elect to ssh into the unit using the indicated IP address, 
or you can select "Continue" to proceed with installation using the serial console.
I'd suggest just continuing to use the serial console as this saves memory, but using
an ssh session should be fine. You can move between GNU screen windows by issuing the
keystroke sequence "CTRL-A SPACE". to get access to a shell or the logs.
    
If you wish to ssh into the unit then use username "installer" with password 
"SCdebian2022" (as configured above)

    Title: Choose a mirror of the Debian archive
    Description: Debian archive mirror country
    Suggested Answer: United States (or pick the country closest to you)

    Title: Choose a mirror of the Debian archive
    Description: Debian archive mirror
    Suggested Answer: deb.debian.org

    Title: Choose a mirror of the Debian archive
    Description: HTTP proxy information
    Suggested Answer: <Leave blank>

    Title: Download installer components
    Suggested Answer: Yes

The installer program and parameters will be downloaded. This may take a
minute or so. Next we configure the region for the unit and the
root password. Note that all of these parameters can be changed
by the user once the unit actually comes up.

    Title: Select your location
    Description: Continent or region
    Suggested Answer: North America (or closest region)

    Title: Select your location
    Description: Country, territory or area
    Suggested Answer: United States (or your country)

    Title: Set up users and passwords
    Description: Root password
    Suggested Answer: SCdebian2022  (or whatever you like)

    Title: Set up users and passwords
    Description: Re-enter password to verify
    Suggested Answer: SCdebian2022

Note that in the following section where we create the first
user, the username cannot be set to "admin" as this is a reserved
username in Debian. In this procedure we create a user called "Seagate Central"
(Username : "sc") but you can specify a different username if you
wish. This will be the username you log in via ssh with once the
system is up and running. Naturally, you can create other usernames
once the system is up and running.
    
    Title: Set up users and passwords
    Description: Full name for the new user
    Answer: Seagate Central

    Description: Username for your account
    Answer: sc

    Description: Choose a password for the new user
    Answer: SCdebian2022

    Description: Re-enter password to verify
    Suggested Answer: SCdebian2022

Next select the time zone.

    Title: Configure the clock
    Description: Select your time zone
    Answer: Pacific (or whatever your timezone is)
    
The disk partitioning section is the most complicated part of the
installation. We need to manually configure the partitioning.

    Title: Partition disks
    Description: Partitioning method
    Answer: Manual (Scroll down to the bottom option)

Scroll down to the partitions that need to be modified. Note
that we do not setup the large "Data" partition. This has to
be done once Debian has booted up on the unit. You may need
to scroll up and down in the text display to see all the
available options.

First, configure the root file system partition.    

    Title: Partition disks

    Under "SCSI1 (0,0,0) (sda)"
    Select Partition #3 "Root_File_System_1"

    Use as: "Ext4 journaling file system"
    Format the partition: "yes, format it"
    Mount point: "/ - the root file system"

    Scroll down to and select "Done setting up the partition"

Next, configure the swap partition.

    Under "SCSI1 (0,0,0) (sda)"
    Select Partition #6 "Swap"

    Use as: "swap area"
    
    Scroll down to and select "Done setting up the partition"

After configuring these two partitions scroll right down to the bottom 
of the menu and select 

    Finish partitioning and write changes to disk

You will be prompted as follows.

    Title: Partition disks
    Description: If you continue, the changes listed below will be written to the disks.
    Answer: Yes

The installer will now write the changes to the disk then move on to
downloading and installing the base system which will take in the order
of 20 minutes.

After the "Installing the base system" progress is at about 92%, the following 
message will appear complaining about the kernel. This message appears because
the Seagate Central kernel is a custom kernel that is not available on the
Debian servers. Make sure to select "Yes" to continue the installation.

    Title : Install the base system
    Description: 
    No installable kernel was found in the defined APT sources.                                                                                                

    You may try to continue without a kernel, and manually install 
    your own kernel later. This is only recommended for experts,
    otherwise you will likely end up with a machine that doesn't boot.                                                                                                                       
    Continue without installing a kernel?

    Answer: Yes (Make sure to answer Yes!!)

A few minutes later after installing more software, you will be asked to
participate in a package usage survey. I'd suggest answering no. Users
can opt in to it later if they wish to.

    Select and install software

    Title: Configuring popularity-contest
    Description: Participate in the package usage survey?
    Answer: "No"

After approximately 10 minutes or so you will be presented with a screen 
titled "Software selection". You'll be presented with a list packages
and software to install. In order to minimize the size of the 
installed software, make sure that nothing but the "SSH server" is
selected. If "Standard System Utilities" or any other options are selected 
then de-select them. Users can install whatever other software they 
like once the system is up and running but the goal here is to keep the
image as small as possible. You may need to scroll down to deselect
"Standard Software Utilities".

The following prompts will appear at the end of the installation process.

    Title: Continue without boot loader
    Description: No boot loader installed
    Answer: Continue
        
    Title: Finish the installation
    Description: Installation complete
    
    Installation is complete, so it is time to boot into your new system.
    
    Answer: Continue

Go back to the serial console session and you'll see the unit rebooting.
After a minute or so you should be presented with the Debian login prompt
on the console where you can log into the system. Log in as root using the
credentials supplied during installation.

    Debian GNU/Linux 11 SC-debian ttyS0

    SC-debian login: root
    Password: SCdebian2022
      
## Post Debian installation steps
A few Seagate Central specific modifications need to be made to the
Debian system before it can be compiled into an image.

### Copy the kernel images to the root partition
By default the "nosmp" kernel will be installed however advanced users 
are given the alternative of installing the higher performing but less
stable "smp" capable kernel.

Create a special directory in the root of the device containing the
two kernels with the following commands issued as the root user.

    mkdir /tmp/sda1
    mount /dev/sda1 /tmp/sda1
    mkdir /kernel
    cp /tmp/sda1/uImage /kernel/uImage.4k.nosmp
    cp /tmp/sda1/uImage.4k /kernel/uImage.4k
    cat << "EOF" > /kernel/README.txt
    This directory contains two Linux kernel images that can be used
    for Debian on the Seagate Central.
    
    uImage.4k.nosmp - A kernel that uses just 1 of the 2 CPU cores on
    the unit. This kernel is more stable. This is the default kernel.
    
    uImage.4k - A kernel that uses both CPU cores via SMP mode. This
    kernel has higher performance but due to the poor heat dissipation 
    characteristics of the Seagate Central, it may result in some system
    instability (Segmentation Faults) if the unit is under sustained 
    high CPU load. Advanced users may wish to try this kernel.
    
    If you wish to change the kernel image the system uses then copy
    the desired kernel image to the boot partition using the filename 
    "uImage" to overwrite the old kernel image and reboot the unit to
    activate the new kernel.
    
    Identify the boot partition by running the shell command
    
        fw_printenv | grep current_kernel
    
    If "current_kernel=kernel1" then the boot partition is /dev/sda1
    
    If "current_kernel=kernel2" then the boot partition is /dev/sda2
    
    Visit the following projects for more details.
    
    https://github.com/bertofurth/Seagate-Central-Debian
    https://github.com/bertofurth/Seagate-Central-Slot-In-v5.x-Kernel
    https://github.com/bertofurth/Seagate-Central-Tips
    
    EOF

### Install extra required Debian packages
We need to install the "u-boot-tools" and "dbus" Debian packages on
the Seagate Central. These packages provide functionality that we'll depend
on later. Use the following command on the Seagate Central to install
these packages.

    apt-get -y install u-boot-tools dbus

The important "fw_printenv" and "fw_setenv" utilities which manipulate the u-boot
environment variables should now been installed. We need to create the configuration file
for these tools (/etc/fw_env.config) by issuing the following commands on the Seagate
Central as root.

    cat << "EOF" > /etc/fw_env.config 
    # MTD device name       Device offset   Env. size       Flash sector size       Number of sectors
    /dev/mtd1               0x0000          0x8000          0x1000
    /dev/mtd2               0x0000          0x8000          0x1000
    EOF

After creating this fw_env.config file, check that the "fw_printenv" command
correctly displays the u-boot environment variables as per the following example.
Note that the values you see might be slightly different.

    root@SC-debian:~# fw_printenv
    baudrate=38400
    boardtest_state_memory=none
    bootargs=console=ttyS0,38400 mem=256M root=/dev/sda3 rw
    bootcmd=scsi init;ext2load scsi 0:1 0x4000000 uImage;bootm
    bootdelay=5
    . . . .
    . . . .
    
### Create Seagate Central specific boot scripts    
Next, we need to create some boot scripts that perform Seagate Central specific
tasks. 

Most notably we need to reset the "num_boot_tries" u-boot variable back to zero
on each boot. This variable is incremented by u-boot each time the 
unit tries to boot up. If it reaches "4" then u-boot assumes that the kernel on
a particular partition was not able to boot and it will switch to trying to boot
up the kernel and operating systems on the backup set of partitions.

The script also commands the LED status light to turn solid green and sets the
network interrupt CPU affinity to CPU 1, which our testing showed will slightly
improve networking performance if the SMP kernel is active.

Create these boot scripts and associated systemd service files with the following
commands issued as root on the Seagate Central.

    # Create the startup script
    cat << "EOF" > /usr/sbin/sc-bootup.sh
    #!/bin/bash
    echo Performing Seagate Central Specific Startup
    echo Setting status LED to solid green
    echo 1 > /proc/cns3xxx/leds
    echo Resetting u-boot environment variable num_boot_tries to 0
    fw_setenv num_boot_tries 0 &> /dev/null
    echo Set network CPU affinity to CPU 1
    # If the smp kernel is not active these will fail, but no matter.
    echo 2 > /proc/irq/49/smp_affinity
    echo 2 > /proc/irq/51/smp_affinity
    EOF
    chmod u+x /usr/sbin/sc-bootup.sh
    
    # Create the shutdown script
    cat << "EOF" > /usr/sbin/sc-shutdown.sh
    #!/bin/bash
    echo Performing Seagate Central Specific Shutdown
    echo Setting status LED to flashing red
    echo 5 > /proc/cns3xxx/leds
    EOF
    chmod u+x /usr/sbin/sc-shutdown.sh
    
    # Create the systemd service file for startup
    cat << "EOF" > /etc/systemd/system/sc-bootup.service
    [Unit]
    Description=Seagate Central specific bootup
    After=multi-user.target

    [Service]
    Type=oneshot
    ExecStart=-/bin/bash /usr/sbin/sc-bootup.sh
    
    [Install]
    WantedBy=multi-user.target
    EOF

    # Create the systemd service file for shutdown
    cat << "EOF" > /etc/systemd/system/sc-shutdown.service
    [Unit]
    Description=Seagate Central specific shutdown
    DefaultDependencies=no
    Before=halt.target shutdown.target reboot.target

    [Service]
    ExecStart=/bin/bash /usr/sbin/sc-shutdown.sh
    
    [Install]
    WantedBy=halt.target shutdown.target reboot.target
    EOF
    
    # Enable the new services in systemd
    systemctl start sc-bootup
    systemctl enable sc-bootup
    systemctl enable sc-shutdown
    
From now on when the unit has successfully booted up the status LED should turn
from blinking green to solid green and when the unit is commanded to shutdown 
or reboot the status LED should start blinking red.

### Customize filesystem configuration in /etc/fstab
Modify the /etc/fstab file which governs what filesystems are mounted
on boot. In the original fstab file, the Debian installer makes use of UUIDs
to specify partitions but this is not helpful in our case because the UUIDs
on the target system will not match the ones on the system we're creating this
image with. We need to specify partition names instead.

    cat << "EOF" > /etc/fstab
    # /etc/fstab: static file system information.
    #
    # <file system> <mount point>   <type>  <options>             <dump>  <pass>
    /dev/root       /               auto    errors=remount-ro      0      1
    /dev/sda6       none            swap    sw                     0      0
    EOF

Once the unit has booted properly then users can perform further customization
of the fstab file to include the large Data partition and possibly the
"boot" and other partitions. 

### Customize dhcp client configuration in /etc/dhcp/dhclient.conf
Here we modify the default dhcp client configuration to make it more
likely that the unit gets the same DHCP assigned IP address as when
it was running Seagate Central native firmware. This makes it easier
for the user to reconnect to the unit after the upgrade.

The change made by sets the dhcp-client-identifier to the same
value as in Seagate Central native firmware. Make this modification 
with the following commands

    cp /etc/dhcp/dhclient.conf /etc/dhcp/dhclient.conf.orig
    echo "send dhcp-client-identifier = hardware;" >> /etc/dhcp/dhclient.conf

Unfortunately, depending on the DHCP server software in a user's network, the
unit may still end up with a different IP address after it boots up
the Debian operating system. In addition, if a user has configured their 
Seagate Central to use a static IP address, then the unit will almost certainly
come up with a different DHCP assigned IP address. This may make it
difficult for the user to reconnect to the unit after it is upgraded.

### System status webpage
This step sets up the unit to display a very simple system status web page
when someone tries to connect to the unit with a web browser. This helps
users to see that the unit is no longer running Seagate Central native 
firmware but that it is running Debian instead.

We use the netcat tool to generate a web page rather than a full blown web
server as this only uses a small amount of disk and cpu resources. The webpage
will simple state that Debian has been installed, what the IP 
address of the unit is, and that the unit needs to be configured via ssh. Note
that this web service is not very robust. Only one client can connect
every 30 seconds!

Install the netcat tool and create the required scripts with the following commands.

    apt-get -y install netcat

    cat << "EOF" > /usr/sbin/sc-generate-status.sh
    #!/bin/bash
    # This script generates a simple status page continually
    # and pipes it into file /tmp/status/Click-For-Status.html
    mkdir -p  /tmp/status
    echo -e "<!doctype html>
    <HTML>
    <HEAD>
    <TITLE>Debian for Seagate Central</TITLE>
    </HEAD>
    <BODY>
    <H1>
    Debian for Seagate Central NAS is running on this device</H1>
    <H2>
    Please connect to the unit's IP address $(hostname -I | cut -d ' ' -f1) via ssh to configure. </H2>
    <tt>
    Unit Serial Number: $(fw_printenv -n serialNo) <br>
    Unit MAC Address: $(cat /sys/class/net/eth0/address)  <br>
    Hostname: $(hostname) <br>
    All IP addresses: $(hostname -I)  <br>
    System Time: $(date) <br>
    System Uptime:  $(uptime -p) <br>
    Root Partition Disk Usage: <br>
    $(df -h / | grep Filesystem) <br>
    $(df -h / | grep -v Filesystem) <br>
    </tt> <br>
    <a href='https://github.com/bertofurth/Seagate-Central-Debian'>
    Debian for Seagate Central project homepage</a>
    </BODY>
    </HTML>
    " > /tmp/status/View-System-Status.html;
    EOF
    
    chmod u+x /usr/sbin/sc-generate-status.sh
    
    cat << "EOF" > /usr/sbin/sc-statuspage.sh
    #!/bin/bash
    # This script serves the file /tmp/status/Click-For-Status.html
    # as a web page via the netcat utility. Only one user at a time!
    echo -e "HTTP/1.1 200 OK
    Content-Type: text/html; charset=UTF-8
    Server: netcat

    $(cat /tmp/status/View-System-Status.html)" | nc -l 80 > /dev/null;
    EOF

    chmod u+x /usr/sbin/sc-statuspage.sh
    
    # Create the systemd service files
    cat << "EOF" > /etc/systemd/system/sc-statuspage.service
    [Unit]
    Description=Simple status webpage
    Requires=network-online.target
    After=network-online.target

    [Service]
    ExecStart=/bin/bash /usr/sbin/sc-statuspage.sh
    Restart=always
    RuntimeMaxSec=30s

    [Install]
    WantedBy=multi-user.target
    EOF
    
    cat << "EOF" > /etc/systemd/system/sc-generate-status.service
    [Unit]
    Description=Generate status webpage

    [Service]
    Type=oneshot
    ExecStart=/bin/bash /usr/sbin/sc-generate-status.sh
    EOF
    
    cat << "EOF" > /etc/systemd/system/sc-generate-status.timer
    Description=Generate status webpage timer
    Requires=network-online.target
    After=network-online.target

    [Timer]
    OnUnitActiveSec=60s
    OnBootSec=10s

    [Install]
    WantedBy=timers.target
    EOF
    
    # Enable the new services in systemd
    systemctl start sc-generate-status.timer
    systemctl start sc-statuspage
    systemctl enable sc-generate-status.timer
    systemctl enable sc-statuspage
    
Users should disable the "sc-statuspage" service if they plan on installing 
a proper web server on the unit.

### Reboot the unit    
At this point reboot the unit with the "reboot" command and make sure
it comes up as before with no significant errors in the console
bootup logs. Once the login prompt reappears, log back in as the root
user.

### Optional - Configure samba and other services
If you wish you may add or configure any other services at this point.

You could add a working samba service to the image by following the
instructions in the HOWTO-Samba-File-Sharing.md document at this point.
When following that procedure bear in mind that the large data volume
is yet to be mounted.

### Clean up files and power off the Seagate Central 
After confirming that the unit has successfully rebooted, run the following 
commands to clear the disk of any cached Debian repository files and logs
which can consume a large amount of space. We don't need to include these
in the upgrade image.
    
    apt-get clean
    rm -rf /var/log/*

Disable the dhcp client and delete it's lease so that it does not try to ask
for that IP address again. Note that these commands will disconnect any network or
ssh sessions to the unit so should be issued on the serial console.

    /sbin/dhclient -r -4 -v -i -pf /run/dhclient.eth0.pid -lf /var/lib/dhcp/dhclient.eth0.leases -I -df /var/lib/dhcp/dhclient6.eth0.leases eth0
    rm /var/lib/dhcp/*.leases
    
Shutdown the unit in preparation to take a snapshot of the disk image.

    shutdown -h now
    
After a minute or so once the unit has completely shutdown, power off the unit.

## Creating the Seagate Central "upgrade" image
After powering off the Seagate Central, remove the hard drive and
connect it to an external Linux system using something like a
USB hard drive reader.

Make sure this external Linux system has the "squashfs-tools" or
equivalent package installed so that the "mksquashfs" utility
is available.

All the commands shown from this point in this section should be
executed with root privileges (i.e. logged in as root or with 
the "sudo" prefix)

Check the identity of the drive connected using the hard drive reader
with the "lsblk" command. In the following example we see that the device
"sda" which has 8 partitions is clearly the Seagate Central hard drive.

    # lsblk
    NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
    sda           8:0    0   3.6T  0 disk
      sda1        8:1    0    20M  0 part
      sda2        8:2    0    20M  0 part
      sda3        8:3    0     1G  0 part
      sda4        8:4    0     1G  0 part
      sda5        8:5    0     1G  0 part
      sda6        8:6    0     1G  0 part
      sda7        8:7    0     1G  0 part
      sda8        8:8    0   3.6T  0 part
    mmcblk0     179:0    0 119.1G  0 disk
      mmcblk0p1 179:1    0    64M  0 part /boot/efi
      mmcblk0p2 179:2    0   500M  0 part [SWAP]
      mmcblk0p3 179:3    0 118.5G  0 part /

Mount the Seagate Central's primary boot partition and
root partition. Note that in the following examples we use
"sdX" for the drive name. You will need to substitute 
the name of the drive on your device (e.g. "sda")

    mkdir debian-boot
    mount /dev/sdX1 debian-boot
    mkdir debian-root
    mount /dev/sdX3 debian-root

Next, copy the uImage Linux Kernel to the local machine.

    cp debian-boot/uImage uImage
    
Create a squashfs image of the debian root partition. Note that we
use a low level of compression (1) so that the Seagate Central will not
have difficulty decompressing the image during installation.

    mksquashfs debian-root rfs.squashfs -noappend -Xcompression-level 1

Create a "config.ser" file which is used by the Seagate Central upgrade
process to validate the contents of the upgrade image.

    echo "version=$(date +%Y.%m%d.%H%M%S-SCD)" > config.ser
    echo "device=cirrus_v1" >> config.ser
    echo "from=all" >> config.ser
    echo "release_date=$(date +%d-%m-%Y)" >> config.ser
    echo "kernel=$(md5sum /tmp/uImage | cut -c1-32)" >> config.ser
    echo "rfs=$(md5sum /tmp/rfs.squashfs | cut -c1-32)" >> config.ser
    echo "uboot=a7d30b1fa163c12c9fe0abf030746629" >> config.ser
    
The contents of this file should look something like this example.

    # cat config.ser
    version=2023.0410.144332-SCD
    device=cirrus_v1
    from=all
    release_date=10-04-2023
    kernel=7321311ea92e024d464fba8e2222979b
    rfs=1b3bf577cba7078a3de78cfec09ebcd0
    uboot=a7d30b1fa163c12c9fe0abf030746629

Finally, create the Seagate Central upgrade image with the following commands.

    tar -czvf Debian-Bullseye-Seagate-Central.img rfs.squashfs uImage config.ser

The resultant "Debian-Bullseye-Seagate-Central.img" file should be in the order
of about 130MB in size.

This image can now be used to install Debian on a Seagate Central, in conjunction
with the instructions in the README.md file in this project.

## Technical Notes
### Notes about "anna"
"anna" is the very simple Debian package installation tool used during installation
of the Debian operating system.

During the development of this procedure, we found that when using the included version
of "anna", during the netconsole Debian installation process on Seagate
Central we were getting an error message similar to the following

    No kernel modules found                                                              

    No kernel modules were found. This probably is due to a mismatch between 
    the kernel used by this version of the installer and the kernel version 
    available in the archive.                                                                                                                        

    You should make sure that your installation image is up-to-date, or 
    - if that's the case - try a different mirror, preferably deb.debian.org.

This message occurs because the kernel on the Seagate Central is a custom kernel 
that isn't going to be found in the Debian repository.

It seems that we need an updated version of the "anna" tool that can
let you skip this but this updated version (v1.84) is not yet included
in the installer as of the time of writing of this document (April 2023).

The problem is described by Debian bug 998668 as seen at

https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=998668

Hopefully in the future as the installer image is updated the "anna" tool will
also be updated.

## Troubleshooting
In some rare cases after a major upgrade the ethernet interface can fail
to operate properly unless the unit is power cycled. If network connectivity
is lost at any stage after a reboot or after the Debian installation process
then shutdown with the "shutdown -h now" command via the serial console and
then physically power cycle the unit.

