# HOWTO Samba File Sharing

**WORK IN PROGRESS - NOT FINISHED**

This document describes how to install and configure a simple
samba file sharing service on the Debian for Seagate Central system.

The example used will share a Public folder (as on the native 
Seagate Central), share user's password protected home directories
and publish another seperate folder for the sc user.

All commands in this document are executed as the root user or with the
sudo prefix on the Seagate Central running Debian. 

## Installation of samba software
Install the samba file sharing software as follows

    apt-get install samba
    
The installation of this package and it's dependancies will consume in the
order of about 200 MB of space on the root partition.

## Preparation of directories and users   
Create the "Public" and "sc-backup" directories that are going to be shared. 
In the example below we assume that the large Data partition is mounted at
/Data . The Public directory will be accessible by any user, including anonymous
guests but the "sc-backup" directory will only be accessible by user "sc".

    mkdir /Data/Public
    chown nobody:nogroup /Data/Public
    chmod 755 /Data/Public
    
    # Assign the "sc" user owner of the backup directory
    mkdir /Data/sc-backup
    chown sc:sc /Data/sc-backup/
    
The root user needs to create a samba account for any users that wish
to have a samba file share by using the "smbpasswd -a username" command.
This would normally be done just after a new unix user is created with
the "adduser" command or similar.

In the example below we create a samba account for existing Unix user "sc".
This needs to be done in future for any other users that wish to share
folders with samba.

    root@SC-Debian:~# smbpasswd -a sc
    New SMB password: MyFileSharePassword_sc
    Retype new SMB password: MyFileSharePassword_sc

Users can use the "smbpasswd" command to change their own samba file
sharing password in the same way that they would use the "passwd" command 
to change their unix login password. The samba file sharing passwords are
not automatically synchronized with unix user passwords. This means that
the file sharing and unix login passwords can be different, but if the
user wishes they can set the same as each other.

## samba configuration
The samba configuration is stored in the /etc/samba/smb.conf file.
Backup the original configuration file and create a new file contents
as seen below. The parameters used are based on those in the original
Seagate Central firmware, so hopefully the file sharing service should
behave in a similar way.

    cp /etc/samba/smb.conf /etc/samba/smb.conf.orig
    cat << EOF > /etc/samba/smb.conf
    # Samba configuration file
    
    [global]
        workgroup = WORKGROUP
        deadtime = 15
        load printers = No
        log file = /var/log/samba/log.%m
        logging = file
        map to guest = Bad User
        max log size = 1000
        obey pam restrictions = Yes
        pam password change = Yes
        panic action = /usr/share/samba/panic-action %d
        passwd chat = *Enter\snew\s*\spassword:* %n\n *Retype\snew\s*\spassword:* %n\n *password\supdated\ssuccessfully* .
        passwd program = /usr/bin/passwd %u
        server role = standalone server
        server string = Seagate Central Debian Shared Storage
        unix password sync = Yes
        usershare allow guests = Yes
        wins support = Yes
        fruit:model = RackMac
        fruit:time machine = yes
        idmap config * : backend = tdb
        delete veto files = Yes
        hide files = /.*/Network Trash Folder/Temporary Items/
        use sendfile = Yes
        veto files = /.AppleDesktop/.AppleDouble/.bashrc/.profile/
        vfs objects = catia fruit streams_xattr

    [Public]
        comment = Public Folder. Anyone can read, write and delete files here.
        guest ok = yes
        force group = nogroup
        force user = nobody
        create mask = 0666
        directory mask = 0777
        path = /Data/Public
        browsable = yes
        read only = no
        
    # This section makes user's home directories 
    # accessible via \\server-name\username 
    # with a password. 
    [homes]
        comment = Home Directories
        browseable = No
        create mask = 0700
        directory mask = 0700
        valid users = %S
        read only = no
        
    [sc-backup]
        comment = User sc network backup folder
        browseable = Yes
        create mask = 0700
        directory mask = 0700
        valid users = sc
        path = /Data/sc-backup
        read only = no
    EOF   

After any changes are made to the /etc/samba/smb.conf configuration file
issue the "testparm" command to sanity check the new configuration, then
restart the samba service as follows.

    testparm
    systemctl restart smbd nmbd

The server is now serving the following folders

\\server-name\Public - Public folder accessible by anyone
\\server-name\sc - The "sc" user's home directory (password protected)
\\server-name\sc-backup - The "sc" user's backup storage directory (password protected)

These file shares can be accessed by specifying either the name
of the server or the IP address of the server. For example
"\\server-name\Public"  or "\\192.168.1.58\Public" 

#### Optional - samba not discoverable by Windows Explorer
Although the samba file sharing service is now accessable, it may not be
automatically discoverable by Windows explorer. This is a consequence of
the less secure SMBv1 being disabled in by default in Windows 10 and
in the version of samba in Debian. (Seagate Central native firmware
still uses SMBv1.)

If you'd like your server to be automatically discoverable using the
Windows explorer "Network" window then you may need to install a tool
called "wsdd" (Web Services Dynamic Discovery) on the Seagate Central.
This tool is not natively available in the Debian Bullseye software 
repository at the time of writing but can be installed as per the
documentation at the tool's homepage.

https://github.com/christgau/wsdd

First, install the "wget" package if it hasn't already been installed.

     apt-get install wget

Install the wsdd tool as follows. The "gpg" tool and most of the other
dependancies of wssd should should already be installed as part of
installing samba.

    wget -O- https://pkg.ltec.ch/public/conf/ltec-ag.gpg.key | gpg --dearmour > /usr/share/keyrings/wsdd.gpg
    source /etc/os-release
    echo "deb [signed-by=/usr/share/keyrings/wsdd.gpg] https://pkg.ltec.ch/public/ ${UBUNTU_CODENAME:-${VERSION_CODENAME:-UNKNOWN}} main" > /etc/apt/sources.list.d/wsdd.list
    apt update
    apt install wsdd
    
Activate the wsdd tool as follows

    systemctl start wsdd
     
After activating wsdd the server should be automatically detectable in the
Windows Explorer Network view.

## TODO (but probably not)
Add instructions for creating "usershares" using the "sambashare" group

https://wiki.archlinux.org/title/samba

These allow users to publish and create their own shares without relying on
the system administrator to reconfigure smb.conf


Configure Avahi service announcement so that Mac clients can automatically discover
a samba share.







