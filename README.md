**WORK IN PROGRESS. NOT FINISHED**

# Debian for Seagate Central NAS
This is a procedure for installing a basic Debian Linux system 
on a Seagate Central NAS.

## Warning 
**Performing modifications of this kind on the Seagate Central is not 
without risk. Making the changes suggested in these instructions will
likely void any warranty and in rare circumstances may lead to the
device becoming unusable or damaged.**

**Do not use the products of this project in a mission critical system
or in a system that people's health or safety depends on.**

**This procedure will overwrite any data or settings on the unit being
worked on. Be sure to backup any important data before proceeding.**

Note that these instructions only get the unit to the point of having a
a very basic Debian based Linux operating system with a working ssh
service. No other services, such as a samba file sharing service or
a web management interface are installed. Installing these types of
services and other customizations are left to the user. 

## TLDNR
TODO


## Procedure
### Obtain the Debian upgrade image
Download a Debian for Seagate Central installation image to a machine
with connectivity to the Seagate Central being upgraded. Upgrade images
can be found in the releases section of this project at

https://github.com/bertofurth/Seagate-Central-Debian/releases

The image name will be something like 
"Debian-Bullseye-Seagate-Central-v1.0.img"

As of writing the image will install Debian version 11, "Bullseye".

For interested parties, these images were created using the procedure
documented in this project by HOWTO-create-debian-image.md

### Install the Debian image using the Seagate Central Web Management page
Login to the Seagate Central's web based management tool as an admin
level user.

If you don't already know the unit's IP address then take a note of it
as it will be needed later in the procedure. You can find the unit's IP
address by navigating to the "Settings" tab, opening the "Advanced" folder
and selecting "Lan". The unit's IP address should be shown. 

To start upgrading the unit navigate to the "Settings" tab, open the
"Advanced" folder and select "Firmware Update".

Next to the "Install From File" option, click on the "Choose File" button
and in the file selection dialog, select the Debian for Seagate Central 
firmware update image that you obtained in the last step. Next, click on
the "Install" button. 

Click OK a the warning saying "Your Seagate Central must be on for the 
duration of the update. This can take from 5 to 10 minutes. Do you want 
to continue?"

After a few minutes, at the point where the image has been uploaded to the
unit the web page will display a message

    Rebooting in progress

    The device is rebooting after completing system updates and changes. Wait until the page refreshes and the Seagate Central application appears.

Note that this page will **never** refresh because the unit will no longer 
be running a Seagate Central native operating system. Instead, wait for about
two minutes for the status LED on the unit to transition from flashing red,
then solid amber, then flashing green then finally solid green which indicates
that the unit has succesfully rebooted. 

### Establish an ssh connection to the unit
After the unit has rebooted you will need to establish an ssh connection
to the IP address of the unit. The default username when using the firmware
images published in the Releases section of this project is "sc" and the
password is "SCdebian2022". You should then elevate to the root user
by issuing the "su" command using the default root password which is
also "SCdebian2022" as per the following example.

    Linux SC-debian 5.15.86-sc #2 SMP Fri Jan 6 22:43:06 AEDT 2023 armv6l

    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.
    sc@SC-debian:~$ su
    Password: SCdebian2022
    root@SC-debian:/home/sc#

In most cases the unit should come back up with the same DHCP assigned IP address
it had while running Seagate Central native firmware. If this is the case and you've 
managed to establish an ssh connection then at this point you can skip forward 
to the next section. 

If, however, your Seagate Central was configured with a static IP address then
the unit will be likely to have acquired a different DHCP assigned IP address 
from the one it used to have.

For this reason you will have to determine the IP address of the new system
before connecting to it and there isn't a straight forward way to do this.
The easiest way is to use a "network scanner" or "ip scanner" style tool or app
such as "Advanced IP scanner" or "Angry IP scanner" on a computer connected to
the same network as the Seagate Central to find it's IP address.

If you have a Linux system then another alternative is to use the "nmap"
utility to search for hosts on the local network that are offering an ssh service
on port 22.

In the following example the local network segment has address 192.168.1.X so we
use the following form of nmap command.

    nmap -p 22 192.168.1.0/24

This command will report the IP and MAC addresses of all devices on the
local network segment with an ssh service available. You must use your
own local IP subnet in the command.

Scroll through the results. One of the listed hosts should look similar 
to the following and will indicate the IP address of the Seagate Central

    Nmap scan report for 192.168.1.58
    Host is up (0.00014s latency).

    PORT   STATE SERVICE
    22/tcp open  ssh
    MAC Address: 00:10:75:XX:XX:XX (Segate Technology)

From this output we can determine the IP address of the unit. In the example
above it is 192.168.1.58.

In some rare cases the unit's ethernet interface may not work after rebooting.
Try power cycling the unit, waiting for the status LED to turn green and then
trying to connect to the unit again.

If the unit becomes completely unreachable then go to the section at the end
of this document titled "Revert to original firmware".

### Customize the system
#### Change the root and "sc" user passwords
The first, and most important order of business is to change the default
passwords for the root and "sc" users. Issue the "passwd root" and "passwd su"
commands to set new passwords as per the following example

    root@SC-debian:~# passwd root
    New password: YourNewRootPassword
    Retype new password: YourNewRootPassword
    passwd: password updated successfully
    root@SC-debian:~# passwd sc
    New password: YourNewUserPassword
    Retype new password: YourNewUserPassword
    passwd: password updated successfully

#### Change the system hostname
The default system hostname is set to "SC-debian", however you may
wish to change it to something more meaningful. If you
wish to change the system hostname, it is strongly suggested that you
change it before installing any more tools or utilities. 

This can be done with the following commands issued as root

    hostnamectl set-hostname YourNewHostName
    sed -i 's/SC-debian/YourNewHostName/g' /etc/hosts
    sed -i 's/SC-debian/YourNewHostName/g' /etc/ssh/*.pub

You will need to log out and log back in to your ssh session before
you see the hostname changed in your command prompt.

### Configure the local timezone and time
By default the system will be set to the North America/Pacific timezone.
Change this as per the following example using your own timezone.

    # Show current timezone settings
    timedatectl
    
    # Show available timezones. Take note of which one
    # represents your local timezone
    timedatectl list-timezones
    
    # Note your timezone and change to it. Substitute
    # your timezone as per the listing from the last
    # command
    timedatectl set-timezone Europe/Berlin
    
    # Check the system time 
    timedatectl
    
    # If necessary set the current time using
    # a date and time style as per this example
    timedatectl set-time "2023-03-28 10:35:00"
    
    # Optional but suggested. Set up the system to sync 
    # to an NTP server for accurate time.    
    apt-get -y install systemd-timesyncd
    timedatectl set-ntp yes
    
### Make sure the swap partition is working
By default the swap partition on a Seagate Central is formatted to work with the
native 64K page kernel. This is not compatible with the 4K page kernel used in 
Debian. 

To reconfigure the swap space issue the following commands.

    /sbin/mkswap /dev/sda6
    /sbin/swapon /dev/sda6

Issue the "free" command to confirm that the swap space is now in use as
per the following example

    root@SC-debian:~# free
                   total        used        free      shared  buff/cache   available
    Mem:          247108       24972       91924         940      130212      213252
    Swap:        1048572           0     1048572
    
### Format and mount the large Data partition
**Warning: As stated in the introduction of this document, this step will delete
all files and data from the large Data partition.**

The Data partition as used by the native Seagate Central firmware makes
use of a non-standard 64K page size. This means that it cannot be easily 
accessed by the Debian system which uses a standard 4K page size.

The Data partition needs to be formatted using a 4K page size with
the following command.

    /sbin/mkfs.ext4 -F -L Data -m0 /dev/vg1/lv1

After the partition has been formatted it can be mounted as /Data by
creating this directory and adding an appropriate entry to the
/etc/fstab file as follows

    mkdir /Data
    echo "/dev/vg1/lv1    /Data           auto    errors=remount-ro      0      2" >> /etc/fstab
    mount /Data
  
Another alternative for advanced users might be to repartition the
large space at the end of the drive into multiple partitions. Perhaps
one for Data, another for home directories and so on.

### Optional - Install Debian on backup partitions
At this stage of the process the active Seagate Central parititons have
Debian installed but the backup partitions have the native Seagate Central
firmware installed.

If you would like to completely remove the Seagate Central native firmware 
on the backup partition and replace it with a basic "emergency" Debian system
then it can be done at this point in the installation process as follows.

First, identify which set of partitions is operational. Issue the following
command to see whether "kernel1" or "kernel2" is active.

    fw_printenv | grep current_kernel

If you get a result saying "current_kernel=kernel1" then the primary set of
partitions are active and we need to install debian to the secondary set. If
you get a result saying "current_kernel=kernel2" then the secondary set
of partitions are active and we need to install debian to the primary set.
Follow the set of instructions below that are relevant to your situation

#### current_kernel=kernel1
In this case we need to copy data from the primary partitions to the
secondary partitions. Issue the following commands to do so.

     # Create directories to mount the filesystems
     mkdir /tmp/sda1
     mkdir /tmp/sda2
     mkdir /tmp/sda4
     
     # Mount the boot partitions
     mount /dev/sda1 /tmp/sda1
     mount /dev/sda2 /tmp/sda2
     
     # Copy the kernel image from the primary boot partititon
     # to the secondary boot partition
     cp /tmp/sda1/uImage /tmp/sda2/uImage

     # Format the secondary root partititon and mount it
     /sbin/mkfs.ext4 -F -L Root_File_System -O none,has_journal,ext_attr,resize_inode,dir_index,filetype,extent,flex_bg,sparse_super,lar
ge_file,huge_file,uninit_bg,dir_nlink,extra_isize /dev/sda4
     mount /dev/sda4 /tmp/sda4
     
     # Install squashfs-tools package
     apt-get -y install squashfs-tools
     
     # Transfer the basic operating system to the secondary root partititon
     unsquashfs -p 1 -f -d /tmp/sda4/ /rfs.squashfs
     
     # Copy the /etc configuration directory of the working system 
     # to the backup system.
     cp -r /etc/* /tmp/sda4/etc/

#### current_kernel=kernel2
In this case we need to copy data from the secondary partitions to the
primary partitions. Issue the following commands to do so.

     # Create directories to mount the filesystems
     mkdir /tmp/sda1
     mkdir /tmp/sda2
     mkdir /tmp/sda3
     
     # Mount the boot partitions
     mount /dev/sda1 /tmp/sda1
     mount /dev/sda2 /tmp/sda2
     
     # Copy the kernel image from the secondary boot partititon
     # to the primary boot partition
     cp /tmp/sda2/uImage /tmp/sda1/uImage

     # Format the primary root partititon and mount it
     /sbin/mkfs.ext4 -F -L Root_File_System -O none,has_journal,ext_attr,resize_inode,dir_index,filetype,extent,flex_bg,sparse_super,lar
ge_file,huge_file,uninit_bg,dir_nlink,extra_isize /dev/sda3
     mount /dev/sda3 /tmp/sda3
     
     # Install squashfs-tools package
     apt-get -y install squashfs-tools
     
     # Transfer the basic operating system to the primary root partititon
     unsquashfs -p 1 -f -d /tmp/sda3/ /rfs.squashfs
     
     # Copy the /etc configuration directory of the working system 
     # to the backup system.
     cp -r /etc/* /tmp/sda3/etc/


### Cleanup
After initial system setup is complete then the installation files left
over from the Seagate Central upgrade process can be removed with
the following commands

    rm /rfs.squashfs
    
At this point the system is ready for customization.

## Notes and Discussion
This section contains some discussion about advanced topics that are only for
interested readers. These sections do not need to be followed to get a basic
system working.

### Optimize disk layout.
One problem with using the native Seagate Central native disk layout is that it is
not optimized for Debian. Most notably, the root partition, which stores most of
the Linux operating system's files, is only 1GB which is not very big.

Below is a table showing the native disk partition layout for the Seagate Central.
The Seagate Central has a primary and secondary boot and root partition that can
be switched to if the working partition becomes unbootable. It also has a "Config"
and "Update" partition that are used for Seagate Central specific purposes
that don't need to be present when running Debian.

#### Native Segate Central disk layout

| Partition       |  Size  |  Type   |         Label        | Description
|-----------------|--------|---------|----------------------|-------------------------------|
| sda1            |  50MB  |  ext2   |  Kernel_1            |  Primary Kernel               |
| sda2            |  50MB  |  ext2   |  Kernel_2            |  Secondary Kernel             |
| sda3            |  1GB   |  ext4   |  Root_File_System_1  |  Primary Root File System     |
| sda4            |  1GB   |  ext4   |  Root_File_System_2  |  Secondary Root File System   |
| sda5            |  1GB   |  ext4   |  Config              |  Seagate Central config       |
| sda6            |  1GB   |  swap   |  Swap                |  Linux Swap                   |
| sda7            |  1GB   |  ext4   |  Update              |  Seagate Central Update       |
| sda8            |  ...   |  lvm    |  Data                |  Large Data Partition         |
---------------------------------------------------------------------------------------------

Expert users might be inclined to modify this layout to grant the root partition more space
(say 3GB) by removing some of the Seagate Central specific partitions and merging them
into other partitions as follows.

#### Alternative Optimized Debian layout

| Partition       |  Size  |  Type   |         Label        | Description
|-----------------|--------|---------|----------------------|-------------------------------|
| sda1            |  50MB  |  ext2   |  Kernel_1            |  Emergency Kernel             |
| sda2            |  50MB  |  ext2   |  Kernel_2            |  Primary Kernel               |
| sda3            |  1GB   |  ext4   |  Root_File_System_1  |  Emergency Root File System   |
| sda4            |  3GB   |  ext4   |  Root_File_System_2  |  Primary Root File System     |
| sda5            |  1GB   |  swap   |  Swap                |  Linux Swap                   |
| sda6            |  ...   |  lvm    |  Data                |  Large Data Partition         |
---------------------------------------------------------------------------------------------

A "basic/emergency" version of Debian could be installed on Kernel_1 / Root_file_System_1 and 
the main primary version could be installed on Kernel_2 / Root_File_System_2. If the primary
version of Debian became unbootable then after 4 failed bootup attempts the system should
automatically switch to the "emergency" partitions and load them.

Another approach is to remove the hard drive from the unit and use the
procedure at 

https://github.com/bertofurth/Seagate-Central-Tips/blob/main/Unbrick-Replace-Reset-Hard-Drive.md

to repartition the drive so that the partitions are sized appropriately.

### Webmin
I tried installing the "webmin" web based management tool on Debian running on the
Seagate Central. It works but very, very slowly and not 100% reliably.

Here are some brief notes on installing as per the documentation at https://www.webmin.com/deb.html

As root, install the required packages:

    apt-get install wget perl libnet-ssleay-perl openssl libauthen-pam-perl libpam-runtime libio-pty-perl apt-show-versions python unzip shared-mime-info 

Add this line to /etc/apt/sources.list
    
    deb https://download.webmin.com/download/repository sarge contrib

To enable ipv6 for webmin also install the following package

    apt-get libsocket6-perl

Install webmin

    wget https://download.webmin.com/jcameron-key.asc
    apt-key add jcameron-key.asc
    apt-get install apt-transport-https
    apt-get update
    apt-get install webmin

Since it takes so long for the slow Seagate Central to activate webmin, we need
to modify the "TimeoutSec" line in /lib/systemd/system/webmin.service as follows

    TimeoutSec=30s

Reboot the unit and login via the webmin web interface. Let me know
if you manage to somehow tweak webmin to be less resource hungry or to 
work better on the Seagate Central.

### Performance 
Debian on Seagate Central will perform less efficiently than the native Seagate 
Central firmware, however it is obviously far more versatile.

I would suggest only installing and running the bare minimum of daemons and services to
save CPU and memory resources.

The main reason why Debian will perform less efficiently than native firmware
is because the native firmware used a 64K page size. This makes disk
operations quicker at the expense of system memory efficiency.

It might be possible to load a 64K page size kernel with the Debian distribution 
but my brief experience shows that the Seagate Central simply doesn't have 
enough memory for it to work properly. In addition most Debian stock tools
assume that the system is using a standard 4K page size.

### CPU Cooling issues and random Segmentation Faults
I have had some problems where occasionally the unit will fail under heavy load with
"Segmentation Fault" style error messages. I have come to the conclusion that these are likely
due to the CPU overheating. It seems that when the Seagate Central was manufactured, the
heat dissipation characterstics of the unit were designed assuming that it would not be
under sustained heavy CPU load.

Unfortunatly the only remedy I can offer, other than manually modifying your unit to include
a better heat sink, is to recompile the Linux kernel to NOT use SMP mode. That is, by 
forcing the unit to use only one of the two CPU cores the unit does not seem to generate as
much heat and hence becomes more reliable but obviously it will run more slowly.

### Revert to original firmware
The most common issue that may occur after an upgrade is that the unit no longer has network
connectivity. If this happens then the first thing you should try is manually power cycling 
the unit. That is, disconnect the power supply and then reconnect it.

There have been reports that sometimes after an upgrade the unit needs to have the power supply
disconnected then reconnected for the Ethernet to work.

If this does not help then it may be neccessary to revert back to the original Seagate
Central firmware which should be on the backup partitions of the unit. This can be done
by following the steps below

1) Power down then power up the Seagate Central by disconnecting and reconnecting the power.
2) Wait about 30 seconds to a minute for the LED status light on top of the unit to turn from solid amber to flashing green.
3) As soon as the LED flashes green execute the first 2 steps three more times in a row.
4) On the 4th bootup let the unit fully boot. It should now load the original version of firmware.

Make sure that step 2 is followed correctly. That is, power off the unit 
as soon as the LED status light starts flashing green. Don't let it proceed
to the solid green state.

If after following these steps the unit is still not reachable then it may
be necessary to perform a complete system recovery as per the procedure
at

https://github.com/bertofurth/Seagate-Central-Tips/blob/main/Unbrick-Replace-Reset-Hard-Drive.md



