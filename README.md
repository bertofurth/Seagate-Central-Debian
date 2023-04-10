# Debian for Seagate Central NAS

WORK IN PROGRESS. NOT FINISHED

These instructions 

Note that these instructions only get you to the point of installing
a very basic Debian based Linux operating system with a working ssh
service. No other tools or utilities, such as samba file services or
a web management interface are installed. Not even the large Data
partition is mounted. All further customization is left to the user
to complete.

## Obtain the Debian upgrade image


See  Brief Image Creation Notes file for very broad steps on how this 
Debian image was created. 


## Install Debian Linux on a Seagate Central NAS

Just follow the normal Seagate Central web based procedure.


At the point where the image has been uploaded you'll get a message

    Rebooting in progress

    The device is rebooting after completing system updates and changes. Wait until the page refreshes and the Seagate Central application appears.

The page will never refresh because the unit will no longer be running a Debian operating system.



When the system boots up we need to fix a few things

/sbin/mkswap /dev/sda6

rm /rfs.squashfs



Need to change root and "sc" user passwords

Need to format and mount /Data partition. Potentially might set it up as "home".

/sbin/mkfs.ext4 -F -L Data -O none,has_journal,ext_attr,resize_inode,dir_index,filetype,extent,flex_bg,sparse_super,large_file,hug
e_file,uninit_bg,dir_nlink,extra_isize -m0 /dev/vg1/lv1



Notes

Note about webmin working but not particularly quickly

Note about performance being slower than native Seagate Central but obviously
it's more versatile

Try loading packages required to compile linux and see if you can get it to work. 


