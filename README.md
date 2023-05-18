# Debian for Seagate Central NAS
This is a procedure for installing a Debian Linux system on a
Seagate Central NAS.

These instructions are designed to get the unit to the point of having
a very basic Debian based Linux operating system with a working
ssh service installed. Existing usernames and settings will not
be automatically migrated over.

It is assumed that users embarking on this upgrade process have some
familiarity with the basics of operating and maintaining a Debian based
Linux system.

Some brief guidelines for setting up a simple samba file sharing service
are given in the file HOWTO-Samba-File-Sharing.md. Installing other services
that the native Seagate Central firmware provided such as a DLNA media
server, and a web management interface are left to the user.

## Warning 
**Performing modifications of this kind on the Seagate Central is not 
without risk. Making the changes suggested in these instructions will
likely void any warranty and in rare circumstances may lead to the
device becoming unusable or damaged.**

**Do not use the products of this project in a mission critical system
or in a system that people's health or safety depends on.**

**This procedure may overwrite any data or settings on the unit being
worked on. Be sure to backup any important data before proceeding.**

**This project is not endorsed or supported by the original vendors or
manufacturers of the Seagate Central NAS.**

## TLDNR
* Obtain a Debian for Seagate Central upgrade image (see Releases)
* Install image using the Seagate Central Web Management page
* Establish an ssh connection to the unit (username "sc", default pw "SCDebian2022")
* Elevate to root with the "su -" command (root default pw "SCDebian2022")
* Perform system customization (passwords, hostname, timezone etc)
* Optional - Format and mount the large Data partition
* Cleanup
* Perform any other desired customizations

## Procedure
### Obtain the Debian upgrade image
Download a Debian for Seagate Central installation image to a machine
with connectivity to the Seagate Central being upgraded. Upgrade images
can be found in the releases section of this project at

https://github.com/bertofurth/Seagate-Central-Debian/releases

The image name will be something like 
"Debian-Bullseye-Seagate-Central-vX.Y.img"

As of writing (April 2023) the image will install Debian version 11,
"Bullseye".

For interested parties, this upgrade image was created using the procedure
documented in this project in the file HOWTO-create-debian-image.md

### Install the Debian image using the Seagate Central Web Management page
Login to the Seagate Central's web based management tool as an admin
level user.

Make sure you know the unit's IP address as it will be needed later in
the procedure. You can find the unit's IP address by navigating to the
"Settings" tab, opening the "Advanced" folder and selecting "Lan".
The unit's IP address should be shown. 

To start upgrading the unit navigate to the "Settings" tab, open the
"Advanced" folder and select "Firmware Update".

Next to the "Install From File" option, click on the "Choose File" button
and in the file selection dialog, select the Debian for Seagate Central 
firmware update image that you obtained in the last step. Next, click on
the "Install" button. 

The following warning may appear.

    Your Seagate Central must be on for the duration of the
    update. This can take from 5 to 10 minutes. Do you want 
    to continue?
    
 Click OK and make sure to keep the Seagate Central turned on!

After a few minutes, at the point where the image has been uploaded to the
unit, the following message will be displayed.

    Rebooting in progress

    The device is rebooting after completing system updates and 
    changes. Wait until the page refreshes and the Seagate Central
    application appears.

Note that this page may **never** refresh because the unit will no longer 
be running a Seagate Central native operating system. Instead, wait for about
two minutes for the status LED on the unit to transition from flashing red,
then solid amber, then flashing green then finally solid green which indicates
that the unit has successfully rebooted and move on to the next section.

### Establish an ssh connection to the unit
#### Finding the unit's new IP address
After the unit has rebooted you will need to establish an ssh connection
to the IP address of the unit. 

In some cases, depending on your local DHCP server configuration, the unit may
come back up with the same DHCP assigned IP address it had while running Seagate
Central native firmware. If this is the case then a few minutes after the upgrade
is complete you should see a message appear in your browser indicating that the
unit has upgraded to Debian.

If no such status message appears then the unit's IP address may have changed.
The first thing to try is to type the following URL in your browser's address
bar.

     http://SC-Debian
     
If your DHCP service is integrated with your local DNS server then the system
might be intelligent enough to be able to connect to the NAS this way. You
should see a message indicating that the NAS is now running Debian and what
the unit's IP address is.
     
If this does not work then the next step is to use a "network scanner" or
"ip scanner" style tool or app such as "Advanced IP scanner" or 
"Angry IP scanner" on a computer connected to the same network as the
Seagate Central to find it's IP address. The tool should indicate that there
is a device with a "Seagate" MAC address and this is most likely the 
Seagate Central.

There is a command line tool called "nmap" that can also be used to search for
hosts on the local network that are offering an ssh service on port 22.

In the following example the local network segment has address 192.168.1.X so we
use the following form of nmap command. Substitute your own local IP subnet
address.

    nmap -p 22 192.168.1.0/24

This command will report the IP and MAC addresses of all devices on the
local network segment with an ssh service available. One of the listed
hosts should look similar to the following and will indicate the IP
address of the Seagate Central

    Nmap scan report for 192.168.1.58
    Host is up (0.00014s latency).

    PORT   STATE SERVICE
    22/tcp open  ssh
    MAC Address: 00:10:75:XX:XX:XX (Segate Technology)

In some rare cases, the unit's ethernet interface may not work after rebooting
just after an upgrade. Try power cycling the unit, wait for the status LED
to turn green again, wait a few minutes then try to find the unit's IP address
again.

If the unit becomes completely unreachable then go to the section at the end
of this document titled "Revert to original firmware".

#### ssh into the unit
The default username when using the firmware images published in the Releases
section of this project is "sc" and the password is "SCdebian2022". You should
then elevate to the root user by issuing the "su -" command using the default
root password which is also "SCdebian2022" as per the following example.

    Linux SC-debian X.XX.XX-sc #1 SMP Fri Jan 6 22:43:06 AEDT 2023 armv6l

    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.
    sc@SC-debian:~$ su -
    Password: SCdebian2022
    root@SC-debian:/home/sc#
    
    
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

#### Change the system hostname and regenerate keys
The default system hostname is set to "SC-debian", however you should
change this to something more meaningful. If you wish to change the
system hostname, it is strongly suggested that you change it before
installing any more tools or utilities. You should also regenerate
the ssh security keys for your system.

The system hostname can be changed from "SC-debian" to a new host name
with the following commands issued as root.

    hostnamectl set-hostname YourNewHostName
    sed -i 's/SC-debian/YourNewHostName/g' /etc/hosts 
    rm /etc/ssh/*key*
    dpkg-reconfigure openssh-server
    
You will need to log out and log back in to your ssh session before
you see the hostname changed in your command prompt. Your ssh client
may complain that the hosts's keys have changed. You should accept
the new keys from the host because they have indeed changed.

#### Configure the local timezone and time
By default, the system will be set to the North America/Pacific timezone.
Change this timezone as per the following example.

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
    
### Optional - Install Debian on backup partitions
At this stage of the process the active Seagate Central partitions have
Debian installed but the backup partitions have the native Seagate Central
firmware installed.

If you would like to completely remove the Seagate Central native firmware 
on the backup partition then it is possible to replace it with a basic
"emergency" Debian system that can be booted if something goes wrong with
the active system. 

The logins and passwords on the "emergency" system will be as per the
running system's state at this point in time. That is, if you install a
service, change a password, or add a user on the working system after this 
point, that change will not be reflected on the "emergency" system. For
that reason, take a careful note of the current "sc" and "root" passwords.

If you would like to proceed with installing Debian on the alternate 
partition you first need to identify which set of partitions is currently 
operational. Issue the following command.

    fw_printenv | grep current_kernel

If you get a result saying "current_kernel=kernel1" then the primary set of
partitions are active and we need to install Debian on the secondary set. If
you get a result saying "current_kernel=kernel2" then the secondary set
of partitions are active and we need to install Debian on the primary set.
Follow the set of instructions below that are relevant to your situation.

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
     
     # Copy the kernel image from the primary boot partition
     # to the secondary boot partition
     cp /tmp/sda1/uImage /tmp/sda2/uImage

     # Format the secondary root partition and mount it
     /sbin/mkfs.ext4 -F -L Root_File_System -O none,has_journal,ext_attr,resize_inode,dir_index,filetype,extent,flex_bg,sparse_super,large_file,huge_file,uninit_bg,dir_nlink,extra_isize /dev/sda4
     mount /dev/sda4 /tmp/sda4
     
     # Install squashfs-tools package
     apt-get -y install squashfs-tools
     
     # Transfer the basic operating system to the secondary root partition
     unsquashfs -p 1 -f -d /tmp/sda4/ /rfs.squashfs
     
     # Copy the /etc configuration directory of the working system 
     # to the backup system.
     cp -r /etc/* /tmp/sda4/etc/
     
     # Optional. Remove the squashfs-tools package
     apt -y remove squashfs-tools

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
     
     # Copy the kernel image from the secondary boot partition
     # to the primary boot partition
     cp /tmp/sda2/uImage /tmp/sda1/uImage

     # Format the primary root partition and mount it
     /sbin/mkfs.ext4 -F -L Root_File_System -O none,has_journal,ext_attr,resize_inode,dir_index,filetype,extent,flex_bg,sparse_super,large_file,huge_file,uninit_bg,dir_nlink,extra_isize /dev/sda3
     mount /dev/sda3 /tmp/sda3
     
     # Install squashfs-tools package
     apt-get -y install squashfs-tools
     
     # Transfer the basic operating system to the primary root partition
     unsquashfs -p 1 -f -d /tmp/sda3/ /rfs.squashfs
     
     # Copy the /etc configuration directory of the working system 
     # to the backup system.
     cp -r /etc/* /tmp/sda3/etc/
     
     # Optional. Remove the squashfs-tools package
     apt -y remove squashfs-tools
     
### Cleanup
After initial system setup is complete then the installation files left
over from the Seagate Central upgrade process can be removed with
the following commands. This will free up over 100MB of disk space so
this is an important step.

    rm /rfs.squashfs
    
At this point the basic Debian system is ready. Customizations may
now be performed including adding users, installing services and
so forth.

If you plan on installing a web server or web management system on the unit
then you'll need to disable the temporary status display web server which is
currently running on http port 80.

    systemctl disable sc-statuspage

## Install Other Services (Optional)
Here are some brief notes about installing some other services on the unit.

## Samba file sharing
There is a seperate document called HOWTO-Samba-File-Sharing.md in this
project that discusses setting up a simple samba file sharing service.

### Webmin - Simple web based administration tool
It is possible to install the "webmin" web based management tool with Debian
running on the Seagate Central. Our testing showed that it works quite well but
it is very slow and it doesn't always seem to be 100% reliable. 

Since the installation takes somewhere around 500MB then I'd strongly suggest
repartitioning the drive so that the root partition is much bigger than the
default of 1GB. See the section below entitled "Optimize disk layout".

Here are some brief notes on installing webmin on the Seagate Central running
Debian. This is based on the documentation at https://www.webmin.com/deb.html

As root, install the required packages:

    apt-get install wget perl libnet-ssleay-perl openssl libauthen-pam-perl libpam-runtime libio-pty-perl apt-show-versions python unzip shared-mime-info gnupg apt-transport-https

To enable ipv6 for webmin also install the following package

    apt-get install libsocket6-perl
    
In order to access the webmin repository, add a line to /etc/apt/sources.list as follows
    
    echo "deb https://download.webmin.com/download/repository sarge contrib" >> /etc/apt/sources.list.d/webmin.list

Download and install webmin with the following commands

    wget https://download.webmin.com/jcameron-key.asc
    apt-key add jcameron-key.asc
    apt-get update
    apt-get install webmin

Since it takes so long for the slow Seagate Central to activate webmin, we need
to modify the statup service file /lib/systemd/system/webmin.service and 
change "TimeoutSec=15s" to "TimeoutSec=30s". This allows the webmin service enough time
to start. Use the "nano" or "vi" editors to accomplish this.

I would also suggest disabling ssl mode for the web server if you are running
the Seagate Central on a trusted/home network. This can be done by editing
/etc/webmin/miniserv.conf and changing "ssl=1" to "ssl=0". Even though 
it's less secure it's much less strain on the system CPU. If you're using your
Seagate Central on a public or untrusted network then leave ssl enabled.

Reboot the unit with the "reboot" command and login via the webmin web interface 
on port 10000 using a URL similar to http://192.168.1.99:10000 . If your browser complains 
about any security issues then just ignore them for the moment. When prompted, login using
the root username and password.

Please raise an "issue" in the project if you manage to somehow tweak webin or another
web based management tool to be less resource hungry or more reliable on the Seagate
Central.

## Notes and Discussion (Advanced)
This section contains some discussion about advanced topics that are only for
interested readers. These sections do not need to be followed to get a basic
system working.
     
### Choice of Linux kernels (64K / 4K Pagesize)
The default Linux kernel that is active in the Debian for Seagate Central
upgrade image uses a 64K page size but the option of using a more standard
4K page size is offered.

The basic difference is that a 4K page size is more memory efficient and
will work better for the case where many different services are running on
a unit. 64K mode however, will allow file serving operations to run more
quickly and efficiently and would probably be a better choice if the unit
is acting as a dedicated NAS/file server and did little else. In addition 
using a 64K kernel means that users are able to maintain access to the
existing data on the large Data partition as created by the Seagate Cental
native firmware. This is because this large Data partition can only be 
accessed by using a 64K page kernel. Users who choose the 4K kernel will
have to reformat the large Data partition using a 4K page size in order to
make use of it.

If you wish to change to the 4K kernel, simply overwrite the current
"uImage" file in the boot partition with the desired kernel image.

Identify the current boot partition by running the shell command

    fw_printenv | grep current_kernel

If "current_kernel=kernel1" then the boot partition is /dev/sda1

If "current_kernel=kernel2" then the boot partition is /dev/sda2 

Mount the relevant boot partition, copy the desired kernel image to it,
then reboot, as per the following example

    mount /dev/sda2 /boot
    cp /kernel/uImage-sc.vX.X.X.4k /boot/uImage
    reboot

#### 4K Kernel - Activate the swap partition
By default, the swap partition on a Seagate Central is formatted to work with the
native 64K page kernel. This is not compatible with the 4K page kernel used in 
Debian. 

To reconfigure and activate the swap partition, issue the following commands after
rebooting with the 4K kernel.

    /sbin/mkswap /dev/sda6
    /sbin/swapon /dev/sda6

Issue the "free" command to confirm that the swap space is now in use as
per the following example

    root@SC-debian:~# free
                   total        used        free      shared  buff/cache   available
    Mem:          247108       24972       91924         940      130212      213252
    Swap:        1048572           0     1048572
    
### 4K Kernel - Optional but recommended - Format and mount the large Data partition
**Warning: This step will delete all files and data from the large Data partition.**

The Data partition as used by the native Seagate Central firmware makes
use of a non-standard 64K page size. This means that it cannot be easily 
accessed by a Debian system which uses a standard 4K page size kernel.

The Data partition needs to be formatted using a 4K page size with
the following command once the unit has rebooted using a 4K kernel.

    /sbin/mkfs.ext4 -F -L Data -m0 /dev/vg1/lv1

After the partition has been formatted it can be mounted as /Data as
follows

    mount /dev/vg1/lv1 /Data
  
It is theoretically possible to access the data on the native 64K page
formatted Data partition by using the "fuse2fs" utility which is part of the
"e2fsprogs" Debian package which can be installed on the system. Note however
that using this utility has much slower performance than a natively mounted
partition.

### CPU overheating issues 
Our tests showed that when both CPU cores were running at full load for
a period of a few minutes, and the ambient temperature was significantly
more than 25C / 77F, spurious errors may be generated causing processes
to fail. In particular, errors such as "Segmentation Fault" or "Illegal 
Instruction" were reported.

Most modern Linux platforms can cope with momentary high CPU temperatures by
self regulating their CPU frequency and load, however the Seagate Central is not
that sophisticated! 

If reducing the ambient temperature is not an option, and if the unit is operating
under high CPU load for extended periods of time then it is possible to slightly 
reduce the CPU frequency of the unit in order to reduce the heat produced and
make the unit more reliable.

This can be done by modifying the /sbin/sc-bootup.sh startup script to select a
new CPU frequency index (see comments in the script). The highest index (12)
corresponds to a CPU frequency of 700MHz. This is the highest frequency and 
is the default. The lowest supported index is 3 and corresponds to a CPU
frequency of 400MHz. 

The CPU frequency is modified by using the /proc/cns3xxx/pm_cpu_freq special
file. Issue "cat /proc/cns3xxx/pm_cpu_freq " for more details.

### Optimize disk layout.
One problem with using the native Seagate Central native disk layout is that it is
not optimized for Debian. Most notably, the root partition, which stores most of
the Linux operating system's files, is only 1GB . This is not very big!

Below is a table showing the native disk partition layout for the Seagate Central.
The Seagate Central has a primary and secondary boot and root partition that can
be switched to if the working partition becomes unbootable. It also has a "Config"
and "Update" partition that are used for Seagate Central specific purposes
that don't need to be present when running Debian.

#### Native Seagate Central disk layout

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
| sda6            |  1GB   |  swap   |  Swap                |  Linux Swap                   |
| sda8            |  ...   |  lvm    |  Data                |  Large Data Partition         |
---------------------------------------------------------------------------------------------

A "basic/emergency" version of Debian could be installed on Kernel_1 / Root_file_System_1 and 
the main primary version could be installed on Kernel_2 / Root_File_System_2. If the primary
version of Debian became unbootable then after 4 failed bootup attempts the system should
automatically switch to the "emergency" partitions and load them. 

Here is a sample command session on the Seagate Central running Debian where the above
optimized partitioning system could be constructed starting with the default Seagate 
Central. We delete partitions 4 through 6 and then reconstruct them.

    # View the current state of the swap partition
    root@SC-debian:~# /sbin/swapon -s
    Filename                                Type            Size    Used    Priority
    /dev/sda6                               partition       1048572 0       -2
    
    # Temporarily disable the swap partition on /dev/sda6
    root@SC-debian:~# /sbin/swapoff /dev/sda6
    
    # View the current state of the disk partitions.
    # Take careful note of the Start and End sectors. 
    root#SC-debian:~# /sbin/fdisk -l /dev/sda
    Disk /dev/sda: 3.64 TiB, 4000787030016 bytes, 7814037168 sectors
    Disk model: ST4000DM000-1F21
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 4096 bytes
    I/O size (minimum/optimal): 4096 bytes / 4096 bytes
    Disklabel type: gpt
    Disk identifier: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX

    Device        Start        End    Sectors  Size Type
    /dev/sda1      2048      43007      40960   20M Microsoft basic data
    /dev/sda2     43008      83967      40960   20M Microsoft basic data
    /dev/sda3     83968    2181119    2097152    1G Microsoft basic data
    /dev/sda4   2181120    4278271    2097152    1G Microsoft basic data
    /dev/sda5   4278272    6375423    2097152    1G Microsoft basic data
    /dev/sda6   6375424    8472575    2097152    1G Linux swap
    /dev/sda7   8472576   10569727    2097152    1G Microsoft basic data
    /dev/sda8  10569728 7814037134 7803467407  3.6T Linux LVM

    # Enter the fdisk utility to modify the partition table
    root@SC-debian:~# /sbin/fdisk /dev/sda

    Welcome to fdisk (util-linux 2.36.1).
    Changes will remain in memory only, until you decide to write them.
    Be careful before using the write command.

    # Delete partitions 4 through 7
    Command (m for help): d
    Partition number (1-8, default 8): 4

    Partition 4 has been deleted.

    Command (m for help): d
    Partition number (1-3,5-8, default 8): 5

    Partition 5 has been deleted.

    Command (m for help): d
    Partition number (1-3,6-8, default 8): 6

    Partition 6 has been deleted.

    Command (m for help): d
    Partition number (1-3,7,8, default 8): 7

    Partition 7 has been deleted.

    # Recreate partition 4 (the root file system) to take up the
    # space previously occupied by partitions 4,5 and 6.
    Command (m for help): n
    Partition number (4-7,9-128, default 4): 4
    First sector (2181120-10569727, default 2181120): 2181120
    Last sector, +/-sectors or +/-size{K,M,G,T,P} (2181120-10569727, default 10569727): 8472575

    Created a new partition 4 of type 'Linux filesystem' and of size 3 GiB.
    Partition #4 contains a ext4 signature.

    Do you want to remove the signature? [Y]es/[N]o: N

    # Recreate partition 6 (the swap partition) to take up the
    # space previously occupied by partition 7.
    Command (m for help): n
    Partition number (5-7,9-128, default 5): 6
    First sector (8472576-10569727, default 8472576): 8472576
    Last sector, +/-sectors or +/-size{K,M,G,T,P} (8472576-10569727, default 10569727): 10569727

    Created a new partition 6 of type 'Linux filesystem' and of size 1 GiB.
    Partition #6 contains a ext4 signature.

    Do you want to remove the signature? [Y]es/[N]o: Y

    The signature will be removed by a write command.

    # Save and apply the changes
    Command (m for help): w
    The partition table has been altered.
    Syncing disks.

    # Format partition 6 as a swap partition and activate it
    root@SC-debian:~# /sbin/mkswap /dev/sda6
    Setting up swapspace version 1, size = 1024 MiB (1073737728 bytes)
    no label, UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX

    root@SC-debian:~# /sbin/swapon /dev/sda6
    
    # Allow partition 4 to expand to consume all 3GB of it's new space
    root@SC-debian:~# /sbin/resize2fs /dev/sda4
    resize2fs 1.46.2 (28-Feb-2021)
    Filesystem at /dev/sda4 is mounted on /; on-line resizing required
    old_desc_blocks = 1, new_desc_blocks = 1
    The filesystem on /dev/sda4 is now 786432 (4k) blocks long.

    # View the new partition layout. Note that partititon 4 is
    # now 3GB in size.
    root@SC-debian:~# /sbin/fdisk -l /dev/sda
    Disk /dev/sda: 3.64 TiB, 4000787030016 bytes, 7814037168 sectors
    Disk model: ST4000DM000-1F21
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 4096 bytes
    I/O size (minimum/optimal): 4096 bytes / 4096 bytes
    Disklabel type: gpt
    Disk identifier: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX

    Device        Start        End    Sectors  Size Type
    /dev/sda1      2048      43007      40960   20M Microsoft basic data
    /dev/sda2     43008      83967      40960   20M Microsoft basic data
    /dev/sda3     83968    2181119    2097152    1G Microsoft basic data
    /dev/sda4   2181120    8472575    6291456    3G Linux filesystem
    /dev/sda6   8472576   10569727    2097152    1G Linux filesystem
    /dev/sda8  10569728 7814037134 7803467407  3.6T Linux LVM
    
    # Check which set of partitions is currently active
    root@NAS-R-Debian:~# fw_printenv | grep current_kernel
    current_kernel=kernel1
    
    # If the result above says "current_kernel=kernel1" then we need
    # to switch to using "kernel2" and then reboot into the primary
    # Debian system as per the following commands.
    root@NAS-R-Debian:~# fw_setenv current_kernel kernel2
    root@NAS-R-Debian:~# reboot

Another approach would be to also delete the large Data partition, increase
the root partition to an even larger size, and then rebuild the Data partition.

### Performance 
Debian on Seagate Central will perform less efficiently than the native Seagate 
Central firmware, however it is obviously far more versatile.

I would suggest only installing and running the bare minimum of daemons and services to
save CPU and memory resources or at the very least only making use of services
one at a time.

As a benchmark for the system's performance, we managed to recompile the Linux
kernel for Seagate Central on a Seagate Central running Debian. This process
takes 45 minutes on a Raspberry Pi 4B. On the Seagate Central running Debian
it takes about 4 hours in SMP mode (-j2).

### Revert to original firmware
The most common issue that may occur after an upgrade is that the unit no longer has network
connectivity. If this happens then the first thing you should try is manually power cycling 
the unit. That is, disconnect the power supply and then reconnect it.

There have been reports that sometimes after an upgrade the unit needs to have the power supply
disconnected then reconnected for the Ethernet to work.

If this does not help then it may be necessary to revert back to the original Seagate
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

## TODO (but probably not)
### IP address change
The most flawed part of the upgrade procedure is that in some cases, the unit's
IP address can potentially change after the switch from Seagate Central native 
firmware to Debian. This means that the user must go to the effort of rediscovering
the new IP address of the unit before they can reconnect to it.

Although a brief guide is given on how to locate the unit's IP address
after the upgrade, it would be ideal if there was some automatic method.

One option is to have an upgrade image that contains a pre-installed samba
service, along with service advertisement daemons so that the server will
automatically appear in client File Manager programs however this would
add considerable size to the upgrade image.

### Factory default button
The factory default/reset button on the bottom of the unit does nothing in 
Debian. Maybe we could get it to switch between the primary and secondary
boot partitions.

### LED status light
The LED status light could be programmed to somehow indicate that Debian
has booted as opposed to the native firmware. Maybe a red/green flash for 
5 seconds before the status LED goes to solid green could indicate that 
Debian has successfully booted.

