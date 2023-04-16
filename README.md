**WORK IN PROGRESS. NOT FINISHED**


TODO

rootfs support in kernel? Make a 4k config file


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
services and other customizations are left to the user to complete.

## TLDNR
TODO


## Procedure
### Obtain the Debian upgrade image
Download a Debian for Seagate Central installation image to a machine
with connectivity to the Seagate Central being upgraded. Upgrade images
can be found in the releases section of this project at

https://github.com/bertofurth/Seagate-Central-Debian/releases

The image name will be something like "Debian-for-SC-XXXX-XX-XX.img"

As of writing the image will install Debian version 11, "Bullseye".

For interested parties, these images were created using the procedure
documented in this project by HOWTO-create-debian-image.md

### Install the Debian image using the Seagate Central Web Management page
Login to the Seagate Central's web based management tool as an admin
level user.

Navigate to the "Settings" tab, open the "Advanced" folder and
select "Firmware Update".

Next to the "Install From File" option, click on the "Chose File" button
and select the Debian for Seagate Central firmware update image. Next,
click on the "Install" button.

After a few minutes, at the point where the image has been uploaded to the
unit the web page will display a message

    Rebooting in progress

    The device is rebooting after completing system updates and changes. Wait until the page refreshes and the Seagate Central application appears.

Note that this page will **never** refresh because the unit will no longer 
be running a Seagate Central native operating system. Instead wait for the
status LED on the unit to come back to solid green to indicate that it has
succesfully rebooted. 

### Establish an ssh connection to the unit
After the unit has rebooted you will need to establish an ssh connection
to the IP address of the unit. The default username when using the firmware
images published in the Releases section of this project is "sc" and the
password is "SCdebian2022". The default root password is also "SCdebian2022".

In most cases the unit should come back up with the same DHCP assigned IP address
it had while running Seagate Central native firmware. However, if your Seagate
Central was configured with a static IP address then the unit will be likely to
have acquired a different DHCP assigned IP address from the one it used to have.

For this reason you will have to determine the IP address of the new system
before connecting to it and there isn't a straight forward way to do this.

The easiest way is to use a "network scanner" or "ip scanner" style tool or app
on a computer connected to the same network as the Seagate Central to find it's
IP address.

If you have a Linux system then another alternative is to use the Linux "nmap"
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

When you have found the IP address of the unit establish an ssh connection
using the default username of "sc" and password "SCdebian2022".

After logging in via ssh, elevate to the root user by issuing the "su"
command and using the default root password "SCdebian2022". All the 
following commands need to be issued as the root user.

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

    hostnamectl set-hostname NewHostName
    sed -i 's/SC-debian/NewHostName/g' /etc/hosts
    sed -i 's/SC-debian/NewHostName/g' /etc/ssh/*.pub




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
    timedatectl set-time "2022-01-30 10:35:00"
    
    # Optional but suggested. Set up the system to sync 
    # to an NTP server for the time.    
    apt-get -y install systemd-timesyncd
    timedatectl set-ntp yes
    
### Make sure the swap partition is working
By default the swap partition on a Seagate Central is formatted to work with the
native 64K page kernel. This is not compatible with the 4K page kernel used in 
Debian. 

To reconfigure the swap space issue the following commands.

    /sbin/mkswap /dev/sda6
    swapon /dev/sda6

Issue the "free" command to confirm that the swap space is now in use as
per the following example

    root@SC-debian:~# free
                   total        used        free      shared  buff/cache   available
    Mem:          247108       24972       91924         940      130212      213252
    Swap:        1048572           0     1048572
    
### Format and mount the large Data partition

**Warning: As stated in the introduction of this document, this step will delete
all files and data from the large Data partition.**

Note that executing these commands will delete all files currently on the
Data partition. If you have 

format

/sbin/mkfs.ext4 -F -L Data -O none,has_journal,ext_attr,resize_inode,dir_index,filetype,extent,flex_bg,sparse_super,large_file,hug
e_file,uninit_bg,dir_nlink,extra_isize -m0 /dev/vg1/lv1




Maybe just mount as "Data"

Optionally move "home" to this directory


    

NOT YET rm /rfs.squashfs






### Optional. Install Debian on backup partitions

unsquashfs and then copy /etc directory



Before doing any customization need to install debian to the alternate partitions.

Use 

unsquashfs -d $BASE/squashfs-root -n $BASE/rfs.squashfs 

Reboot into other system and peform the same customizations.



## Notes and Discussion
This section contains some ideas and notes that interested readers
might like to peruse.

### Optimize disk layout.
One problem with using the native Seagate Central native disk layout is that it is
not optimized for Debian. Most notably, the root partition, which stores most of
the Linux operating system's files, is not very big (only 1GB).

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
(say 3GB) by removing some of the Seagate Central specific partitions as follows.

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

Modify the "TimeoutSec" line in /lib/systemd/system/webmin.service to

    TimeoutSec=30s

Reboot the unit and login via the webmin web interface. Let me know
if you manage to somehow tweak webmin to be less resource hungry.

### Performance
Debian on Seagate Central will perform less efficiently than the native Seagate Central firmware,
however it is obviously far more versatile.

I would suggest only installing and running the bare minimum of daemons and services to
save CPU and memory resources.

Finally, I have had some problems where occasionally the unit will fail under heavy load with
"Segmentation Fault" style error messages. I have come to the conclusion that these are likely
due to the CPU overheating. It seems that when the Seagate Central was manufactured, the
heat dissipation characterstics of the unit were designed assuming that it would not be
under sustained heavy CPU load.

Unfortunatly the only remedy I can offer, other than manually modifying your unit to include
a better heat sink, is to recompile the Linux kernel to NOT use SMP mode. That is, by 
forcing the unit to use only one of the two CPU cores the unit does not seem to generate as
much heat and hence becomes more reliable but obviously it will run more slowly.


