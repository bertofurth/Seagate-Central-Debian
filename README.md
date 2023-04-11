# Debian for Seagate Central NAS
This is a procedure for installing an up to date (2023) version of
Debian Linux distribution on a Seagate Central NAS.

This procedure will overwrite any data and settings stored on the
unit so be sure to make a backup of any important data before
proceeding.

**WORK IN PROGRESS. NOT FINISHED**

Note that these instructions only get the unit to the point of having a
a very basic Debian based Linux operating system with a working ssh
service. No other services, such as a samba file sharing service or
a web management interface are installed. Installing these types of
services and other customizations are left to the user to complete.


## TLDNR




## Obtain the Debian upgrade image
Download a Debian for Seagate Central installation image to a machine
with connectivity to the Seagate Central being upgraded. Upgrade images
can be found in the releases section of this project at

https://github.com/bertofurth/Seagate-Central-Debian/releases

The image name will be something like "Debian-for-SC-XXXX-XX-XX.img"

For interested parties, these images were created using the procedure
documented in this project by HOWTO-create-debian-image.md

## Install Debian Linux using the Seagate Central Web Management page
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

This page will never refresh because the unit will no longer be running a
Seagate Central native operating system. Instead wait for the status LED on
the unit to come back to solid green to indicate that it has succesfully 
rebooted. 

### Establish an ssh connection to the unit
After the unit has rebooted you will need to establish an ssh connection
to the IP address of the unit. The default username when using the firmware
images in the Releases section of this project is "sc" and the password
is "SCdebian2022". The default root password is also "SCdebian2022".

Unfortunately the unit will be likely to have acquired a different DHCP assigned
IP address from the one it used to have while running the Seagate Central 
native firmware. This is because Debian uses a different DHCP client identifier 
than the old operating system so your DHCP server will have assigned it a different
IP address. For this reason you will have to determine the IP address
of the new system before connecting to it and there isn't a straight forward
way to do this.

The easiest way is to use a "network scanner" tool to search for the 
Seagate Central. Another alternative is to use the linux "nmap"
utility to search for hosts on the local network that are offering an
ssh service on port 22. In the following example the local network segment
has address 192.168.1.X so we use the following form of nmap command.

    nmap -p 22 192.168.1.0/24

This command will report the IP and MAC addresses of all devices on the
local network segment (use your own local IP subnet in the command)

One of the results should look similar to the following and would indicate
the IP address of the Seagate Central

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
    New password: NewRootPassword
    Retype new password: NewRootPassword
    passwd: password updated successfully
    root@SC-debian:~# passwd sc
    New password: NewUserPassword
    Retype new password: NewUserPassword
    passwd: password updated successfully

#### Change the system hostname
The default system hostname is "SC-debian" but you will most likely
wish to change this to something more meaningful. This can be
done with the "hostnamectl set-hostname <new-name>" command. The
/etc/hosts file also needs to be modified with the "nano"
or "vi" editor to remove any references to "SC-debian" and replace
them with the new hostname.

    hostnamectl set-hostname NAS-X
    nano /etc/hosts

#### Set the local timezone

    # Show current timezone settings
    timedatectl
    
    # Show available timezones
    timedatectl list-timezones
    
    # Note your timezone and change to it
    timedatectl set-timezone Europe/Berlin
    
    # Check the time 
    timedatectl
    
    # If necessary set the current time
    timedatectl set-time "2022-01-30 10:30:00"
    
Optional but suggested. Set up the system to sync to an NTP server for the time.
    
    apt install systemd-timesyncd
    timedatectl set-ntp yes
    


When the system boots up we need to fix a few things

/sbin/mkswap /dev/sda6

NOT YET rm /rfs.squashfs



Need to change root and "sc" user passwords

Need to format and mount /Data partition. Potentially might set it up as "home".

/sbin/mkfs.ext4 -F -L Data -O none,has_journal,ext_attr,resize_inode,dir_index,filetype,extent,flex_bg,sparse_super,large_file,hug
e_file,uninit_bg,dir_nlink,extra_isize -m0 /dev/vg1/lv1


Before doing any customization need to install debian to the alternate partitions.

Use 

unsquashfs -d $BASE/squashfs-root -n $BASE/rfs.squashfs 

Reboot into other system and peform the same customizations.



OPTIONAL : Optimize disk layout.

One problem with using the native Seagate Central disk layout is that the root
partition, which stores most of the Linux operating system's files, is not
very big (only 1GB).

Below is a table showing the native disk partition layout for the Seagate Central.
The Seagate Central has a primary and secondary boot and root partition that can
be switched to if the working partition becomes unbootable. It also has a "Config"
and "Update" partition that are used for Seagate Central specific purposes
that don't need to be present when running Debian.

### Native Segate Central disk layout

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

It is possible to modify this layout to grant the root partition up to 3GiB by 
removing some of the Seagate Central specific partitions as follows.

### Optimized Debian layout

| Partition       |  Size  |  Type   |         Label        | Description
|-----------------|--------|---------|----------------------|-------------------------------|
| sda1            |  50MB  |  ext2   |  Kernel_1            |  Emergency Kernel             |
| sda2            |  50MB  |  ext2   |  Kernel_2            |  Primary Kernel               |
| sda3            |  1GB   |  ext4   |  Root_File_System_1  |  Emergency Root File System   |
| sda4            |  3GB   |  ext4   |  Root_File_System_2  |  Primary Root File System     |
| sda5            |  1GB   |  swap   |  Swap                |  Linux Swap                   |
| sda8            |  ...   |  lvm    |  Data                |  Large Data Partition         |
---------------------------------------------------------------------------------------------

Another approach is to remove the hard drive from the unit and use the
procedure at 

https://github.com/bertofurth/Seagate-Central-Tips/blob/main/Unbrick-Replace-Reset-Hard-Drive.md

to repartition the drive so that the partitions are sized appropriately. I would
suggest 

sda1 :  50M


End layout







Get rid of "config" and "update" partitions.



Notes

Note about webmin working but not particularly quickly

Note about performance being slower than native Seagate Central but obviously
it's more versatile

Try loading packages required to compile linux and see if you can get it to work. 


