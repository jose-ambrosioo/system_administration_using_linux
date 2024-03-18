# system_administration_using_linux
(LINUX (RED HAT ENTERPRISE, CENTOS, UBUNTU) | WINDOWS)

Implemented a network infrastructure that was easily scalable based on business volume.  Configured the following services: IP, FTP, YUM, HTTP, DNS, NIS, DHCP, NFSv4, Samba, and openLAD. 

Introduction
In this project I implemented a 100% Open Source Linux network infrastructure for a company.

Note: the texts in green are commands executed in the terminal as root and the texts
in blue are changes to files.

Note: The “iptables” firewall was disabled on the servers and then activated:
#service iptables off
#iptables –F

“kwanza” server
kwanza server partition table
![image](https://github.com/jose-ambrosioo/system_administration_using_linux/assets/59221796/13b87c51-c7b2-4700-b609-5860e1c0a1c8)

Static hostname/IP
Machine name “kwanza.bestsoft.com”
Machine IP “192.168.10.1”

YUM/FTP
FOR RHEL 6 REPOSITORY
The DVD Drive was mounted where the RHEL 6 image was located
[root@kwanza ~]# mount /dev/cdrom /media

We went to the directory where the DVD Drive with the RHEL6 image is mounted and to the Packages directory and installed the following packages

[root@kwanza ~]# cd /media/RHEL-6.6\ Server.x86_64/Packages/
[root@kwanza ~]# rpm -ivh vsftpd*
[root@kwanza ~]# rpm -ivh deltarpm*
[root@kwanza ~]# rpm -ivh python-deltarpm*
[root@kwanza ~]# rpm -ivh createrepo*

After installing FTP, the repository folder “/var/ftp/pub/repo” was automatically created.

The repository was created using the /var/ftp/pub/repo/Packages directory as its database
[root@kwanza ~]# createrepo --database /var/ftp/pub/repo/Packages/

The file containing the YUM settings has been edited.

[root@kwanza ~]# nano /etc/yum.repos.d/rhel-source.repo
[rhel-source]
name=Red Hat Enterprise Linux $releasever - $basearch - Source
baseurl=file:///var/ftp/pub/repo
enabled=1
gpgcheck=0
#gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

FTP
The vsftpd service was enabled to start when starting the machine

[root@kwanza ~]# chkconf vsftpd on

Start the service
[root@kwanza ~]# service vsftpd start

View FTP access
[root@kwanza ~]# getsebool -a | grep ftp
allow_ftpd_anon_write --> off
allow_ftpd_full_access --> off
allow_ftpd_use_cifs --> off
allow_ftpd_use_nfs --> off
ftp_home_dir --> off
ftpd_connect_db --> off
ftpd_use_passive_mode --> off
httpd_enable_ftp_server --> off
tftp_anon_write --> off

Allow access to FTP content
[root@kwanza ~]# setsebool -P allow_ftpd_full_access on

HTTP
HTTP packages have been installed
[root@kwanza ~]# yum install httpd*

An index.html page was created in /var/www/html/

HTTP was activated at the right levels, and the service started
[root@kwanza ~]# chkconfig --list httpd
httpd 0:off 1:off 2:off 3:off 4:off 5:off 6:off
[root@kwanza ~]# chkconfig httpd on
[root@kwanza ~]# service httpd start

3 – Configure httpd, adding the following configuration
[root@kwanza ~]# nano /etc/httpd/conf/httpd.conf
<VirtualHost *:80>
 ServerAdmin root@kwanza.bestsoft.com
 DocumentRoot /var/www/html
 ServerName kwanza.bestsoft.com
 ErrorLog logs/kwanza.bestsoft.com-error_log
 CustomLog logs/kwanza.bestsoft.com-access_log common
</VirtualHost>

DNS
DNS packages have been installed
[root@kwanza ~]# yum install bind*

Edited the configuration file and placed the main areas for translation
[root@kwanza ~]# nano /etc/named.conf
options {
listen-on port 53 { 127.0.0.1; 192.168.10.1; };
 #listen-on-v6 port 53 { ::1; };
 directory "/var/named";
 dump-file "/var/named/data/cache_dump.db";
 statistics-file "/var/named/data/named_stats.txt";
 memstatistics-file "/var/named/data/named_mem_stats.txt";
 allow-query { 192.168.9.0/24; };
 recursion yes;
 dnssec-enable yes;
 dnssec-validation yes;
 dnssec-lookaside auto;
 /* Path to ISC DLV key */
 bindkeys-file "/etc/named.iscdlv.key";
 managed-keys-directory "/var/named/dynamic";
};

The file “/etc/named.rfc1912.zones” was configured
zone "bestsoft.com" IN {
 type master;
 file "forward.zone";
 allow-update { none; };
};
zone "10.168.192.in-addr.arpa" IN {
 type master;
 file "reverse.zone";
 allow-update { none; };
};

Zone files were created and edited, translating the machines that need translation into the
identified zones
[root@kwanza]# cp /var/named/named.localhost /var/named/foward.zone
[root@kwanza]# cp / var/named/named.loopback /var/named/reverse.zone
[root@kwanza named]# nano /var/named/forward.zone
$TTL 1D
@ IN SOA bestsoft.com. root.bestsoft.com. (
0 ; serial
1D ; refresh
6
1H ; retry
1W ; expire
3H ) ; minimum
@ IN NS kwanza.bestsoft.com.
@ IN NS lombe.bestsoft.com.
@ IN NS gazela.bestsoft.com.
@ IN A 192.168.10.1
@ IN A 192.168.10.2
@ IN A 192.168.10.10
kwanza A 192.168.9.1
lombe A 192.168.9.2
gazela A 192.168.9.10

The steps are the same as those done in “forward.zone” except the last 5 lines, the “1”,
“2” and “3” represent the last octets represented in IP addresses
[root@kwanza]# nano /etc/reverse.zone
1 IN PTR kwanza. bestsoft.com.
2 IN PTR lombe. bestsoft.com.
3 IN PTR gazela.bestsoft.com.
1 IN PTR bestsoft.com.
1 IN PTR www. bestsoft.com.

The names of the main machines with fixed IP were added, for address translation and
Even if the DNS server fails there will still be communication with these machines.
Note: if you forget this, the server will not be able to translate the IPs and will not find the machines by the name.
[root@kwanza]# nano /etc/hosts
192.168.10.1 kwanza.bestsoft.com www.bestsoft.com
192.168.10.2 lombe.bestsoft.com
192.168.10.10 gazela.bestsoft.com

The server DNS has been configured
[root@kwanza]# nano /etc/sysconf/network-scripts/ifcfg-eth0
DNS1=192.168.10.1
DOMAIN=bestsoft.com

DNS has been activated at the right levels, the service has been started and verified
[root@kwanza]# chkconfig dnsmasq on
[root@kwanza]# service dnsmasq start
[root@kwanza]# nslookup kwanza

NIS Primary

Installed the packages for the NIS server and client
[root@kwanza]# yum install yp* -y

The NIS server was configured by adding the following lines
[root@kwanza]# nano /etc/yp.conf
domain nisbestsoft.com server kwanza.bestsoft.com
ypserver kwanza

The NIS domain in which the NIS server and clients will belong has been defined, all servers
and NIS clients must belong to the same domain
[root@kwanza]# nano /etc/sysconfig/network
NETWORKING=yes
HOSTNAME=kwanza.nisbestsoft.com
NISDOMAIN= nisbestsoft.com

Configured the domainname and ypdomainname
[root@kwanza /]# domainname nisbestsoft.com
[root@kwanza /]# ypdomainname nisbestsoft.com

Securenets has been configured
[root@kwanza /]# nano /var/yp/securenets
host 127.0.0.1
255.255.255.0 192.168.10.0

Portmap services and NIS server started
[root@kwanza /]# service rpcbind start
[root@kwanza /]# service ypserv start

Check ipserv
[root@kwanza /]# rpcinfo -u localhost ypserv
program 100004 version 1 ready and waiting
program 100004 version 2 ready and waiting

The NIS server database was created and the NIS map was initialized, where “ –m ” indicates that this is the NIS Master Server
[root@kwanza]# /usr/lib64/yp/ypinit –m

The ypbind service has started
[root@kwanza /]# service ypbind start

The service that allows NIS users to change their passwords has started
[root@kwanza /]# service yppasswdd start

NIS database transfer daemon started
[root@kwanza /]# service ypxfrd start

NIS services were configured to start when booting the machine
[root@kwanza /]# chkconfig rpcbind on
[root@kwanza /]# chkconfig ypserv on
[root@kwanza /]# chkconfig ypbind on
[root@kwanza /]# chkconfig yppasswdd on
[root@kwanza /]# chkconfig ypxfrd on
The “/nishome” directory was created where the home of all NIS users will be located

[root@kwanza /]# mkdir /nishome
[root@kwanza /]# chcon --reference /home /nishome -R

Added NIS user accounts
[root@kwanza /]# useradd –p joao –d /nishome/joao -c 'Joao' joao
[root@kwanza /]# useradd –p ana –d /nishome/ana -c 'Ana' ana
[root@kwanza /]# useradd –p vasco –d /nishome/vasco -c 'Vasco' vasco
[root@kwanza /]# useradd –p maria –d /nishome/maria maria -c 'Maria' maria
[root@kwanza /]# useradd –p teresa –d /nishome/teresa -c 'Teresa' teresa
[root@kwanza /]# useradd –p admin –d /nishome/admin admin -c 'Admin' admin
[root@kwanza /]# useradd –p admin_intern –d /nishome/admin_intern admin_
intern -c 'Admin_intern' admin_intern

NIS user passwords have been configured
[root@kwanza]# passwd usuario

Configured 3 NIS user accounts to expire on 2018-06-20
[root@kwanza]# chage -E 2018-06-20 ana
[root@kwanza]# chage -l ana
Account expires : Jun 20, 2018
[root@kwanza]# chage -E 2018-06-20 maria
[root@ kwanza]# chage -E 2018-06-20 teresa

So that NIS users' home directories can be mounted on NIS client machines
we export /nishome via NFS
[root@kwanza]# nano /etc/exports
/nishome *(rw)

To allow the primary NIS server database to be transferred to the NIS
Slaves:
[root@kwanza]# nano /var/yp/Makefile
NOPUSH=false

The database has been updated
[root@kwanza]# make -C /var/yp

NOTE: Whenever a user is added, we must update the server database
NIS doing:
[root@kwanza]# make -C /var/yp

NOTE2: When resetting the password of a user created without a password, delete the encrypted password in the
/etc/shadow file on NIS server

Security

To allow only the admin user to invoke the “su” command on any machine or
server, on all machines the “wheel” group was defined as the owner of the su command:
[root@kwanza /]# chgrp wheel /bin/su

Changed the permissions of /bin/su
[root@kwanza /]# chmod u=rws,g=rwx,o= /bin/su

The user “admin” was added to the “wheel” group
[root@kwanza /]# usermod –G wheel admin

So that the user “admin_intern” only has permission to restart Apache Web
Server (httpd restart) on the kwanza server and for users admin and admin_intern
have additional privileges, the following was added to the file:
[root@kwanza /]# visudo
admin_intern kwanza.bestsoft.com=/sbin/service httpd restart
admin ALL=(ALL) ALL
admin_intern ALL=(ALL) ALL

So that only the two administrator accounts admin and admin_intern and root can
login to the kwanza and lombe servers, the /etc/ssh/sshd_config file was edited in the
mentioned servers and the following lines were added:
[root@kwanza /]# nano /etc/ssh/sshd_config
AllowUsers admin admin_intern root
DenyUsers joao ana vasco maria teresa
PermitRootLogin yes

“lombe” server configuration
lombe server partition table
![image](https://github.com/jose-ambrosioo/system_administration_using_linux/assets/59221796/36421297-6a9c-4dde-817a-716188325630)

Static hostname/IP
Nome da máquina “lombe.bestsoft.com”
IP da máquina “192.168.10.2”

YUM/FTP
FOR RHEL 6 REPOSITORY

The DVD Drive was mounted where the RHEL 6 image was located
[root@lombe ~]# mount /dev/cdrom /media

We went to the directory where the DVD Drive with the RHEL6 image was mounted and to the
Packages directory and installed the following packages
[root@lombe ~]# cd /media/RHEL-6.6\ Server.x86_64/Packages/
[root@lombe ~]# rpm -ivh vsftpd*

The repository folder “/var/ftp/pub/repo” was automatically created after installing FTP

The file containing the YUM settings has been edited
[root@lombe ~]# nano /etc/yum.repos.d/rhel-source.repo
[rhel-source]
name=Red Hat Enterprise Linux $releasever - $basearch - Source
10
baseurl=file:///kwanza/pub/repo
enabled=1
gpgcheck=0
#gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
FTP

The vsftpd service was enabled to start when starting the machine
[root@lombe ~]# chkconf vsftpd on
Inicializou-se o serviço
[root@lombe ~]# service vsftpd start
HOSTS

The “/etc/hosts” file was configured
[root@lombe ~]# nano /etc/hosts
192.168.10.1 kwanza.bestsoft.com kwanza www.bestsoft.com
192.168.10.2 lombe.bestsoft.com lombe
192.168.10.10 gazela.bestsoft.com gazela

DHCP

DHCP installation and configurations were carried out
[root@lombe ~]# yum install dhcp -y
[root@lombe ~]# nano /etc/dhcpd.conf
option domain-name “bestsoft.com”
option domain-name-servers “kwanza.bestsoft.com”

Set itself as the network's official DHCP server
subnet 192.168.10.0 netmask 255.255.255.0 {
 range 192.168.10.10 192.168.10.254;
 option domain-name-servers kwanza.bestsoft.com;
 option domain-name " bestsoft.com";
 option routers 192.168.10.1;
 option broadcast-address 192.168.10.255;
 default-lease-time 600;
 max-lease-time 7200;
}
host gazela {
hardware Ethernet 00:0c:29:73:69:34;
fixed-address 192.168.10.10;
}

DHCP was configured to start when booting the machine
[root@lombe ~]# chkconfig dhcp on
[root@lombe ~]# service dhcp start

NIS Secondary

Installed the packages for the NIS server and client
[root@lombe ~]# yum install yp* -y


The NIS domain to which the NIS server and clients will belong has been defined
[root@lombe ~]# nano /etc/sysconfig/network
NETWORKING=yes
HOSTNAME=lombe.bestsoft.com
NISDOMAIN=nisbestsoft.com

All NIS servers for this client have been identified
[root@lombe ~]# nano /etc/yp.conf
domain nisbestsoft.com server kwanza.bestsoft.com
domain nisbestsoft.com server lombe.bestsoft.com
ypserver kwanza

For synchronizing user and account information across all machines in the domain
the file was edited:
[root@lombe ~]# nano /etc/nsswitch.conf

and changed the “passwd, shadow and group” fields to:
passwd: files nis
shadow: files nis
group: files nis

Configured the domainname and ypdomainname
[root@lombe ~]# domainname nisbestsoft.com
[root@lombe ~]# ypdomainname nisbestsoft.com

NIS services were configured to start when booting the machine
[root@lombe ~]# chkconfig rpcbind on
[root@lombe ~]# chkconfig ypbind on

Check ypbind
[root@lombe ~]# rpcinfo -u localhost ypbind
program 100004 version 1 ready and waiting
program 100004 version 2 ready and waiting

The NIS server database file was updated and the NIS map was initialized, where “ –s ”
indicates that this is the NIS Slave Server
[root@lombe ~]# ypcat passwd
[root@lombe ~]# /usr/lib64/yp/ypinit –s lombe

For NIS clients to mount the “/nishome” folders permanently, the file was edited
fstab on clients (Lombe and Gazelle) adding the settings at the end of the file:
[root@lombe ~]# nano /etc/fstab
kwanza:/nishome/ /nishome/ nfs defaults 0 0

Note: start the nfs service on the kwanza server and make it start whenever the
machine start.

Note2: to update fstab without restarting a machine: #mount -a

Partition management (Partition Creation, VG, PV, LVM and SWAP)
A 12 Gb IDE disk was added and the machine started

CREATION OF PARTITIONS

The command that allows us to manage disk partitions was executed, to edit the
disk added
[root@lombe ~]# fdisk /dev/sdc

1-Select the option to add partitions
Command (m for help): n

2-Defined as primary partition by selecting option “p”

3-The partition number was chosen as “1”
Partition number (1 - 4): 1

4-In the next step we leave it blank and press ENTER, because by default it searches
the first free cylinder on the disk and the file system of the partition being
created, something like the following:
First cylinder (1-1566, default 1):

5-The size of the partition was defined, which in this case is 5Gb, something like the following:
Last cylinder, +cylinders or +size{K,M,G} (1-1566, default 1566): +5GB

The other partitions were added with their respective sizes (GB for Gigabytes and MB for
Megabytes) repeating the same steps from step 1.

After finishing so that the partitions were saved, the corresponding one was chosen
option:
Command (m for help): w

The machine was restarted just to ensure that the changes were applied
[root@lombe ~]# reboot

The partition file system was created for the physical volumes (ext4 formatting)
[root@lombe ~]# mkfs.ext4 /dev/sdc1
[root@lombe ~]# mkfs.ext4 /dev/sdc2

VG, PV, LVM CREATION

PVs (Physical Volumes) were created
[root@lombe ~]# pvcreate /dev/sdc1
[root@lombe ~]# pvcreate /dev/sdc2

VG (Volume Group) was created called “vgvendafinancas”
[root@lombe ~]# vgcreate vgvendafinancas /dev/sdc1 /dev/sdc2

The 4 LVs (Logical Volumes) of each 2Gb were created in the VG vgvendafinancas, following a
naming like “lvm1, lvm2, . . . , lvmn”
[root@lombe ~]# lvcreate –n lv1 –L 2GB vgvendafinancas
[root@lombe ~]# lvcreate –n lv2 –L 2GB vgvendafinancas
[root@lombe ~]# lvcreate –n lv3 –L 2GB vgvendafinancas
[root@lombe ~]# lvcreate –n lv4 –L 2GB vgvendafinancas

The file system of the created LVMs was created (Formatting)
[root@lombe ~]# mkfs.ext4 /dev/vgvendafinancas/lv1
[root@lombe ~]# mkfs.ext4 /dev/vgvendafinancas/lv2
[root@lombe ~]# mkfs.ext4 /dev/vgvendafinancas/lv3
[root@lombe ~]# mkfs.ext4 /dev/vgvendafinancas/lv4

Directories were created where the LVMs will be mounted for better organization
[root@lombe ~]# mkdir /vgvendafinancas /vgvendafinancas/lv1 /vgvendafinancas/lv2
[root@lombe ~]# mkdir /vgvendafinancas/lv3 /vgvendafinancas/lv4

So that the LVs are mounted permanently, that is, when starting the machine, the
file “/etc/fstab” adding the settings at the end of the file
[root@lombe ~]# nano /etc/fstab
/dev/vgvendafinancas/lv1 /vgvendafinancas/lv1/ nfs defaults 0 0
/dev/vgvendafinancas/lv2 /vgvendafinancas/lv2/ nfs defaults 0 0
/dev/vgvendafinancas/lv3 /vgvendafinancas/lv3/ nfs defaults 0 0
/dev/vgvendafinancas/lv4 /vgvendafinancas/lv4/ nfs defaults 0 0

SWAP CREATION

We created two primary partitions of 512MB sdc3 and sdc4

1-Select the option to edit the type of partitions
Command (m for help): t

2-The partition type “82” was chosen (Hexadecimal value for the partition type “Linux
swap/Solaris”)
Hex code (type L to list codes):82

After finishing so that the partitions were saved, the corresponding one was chosen
option:
Command (m for help): w

Created the Partition File System for SWAP (SWAP Formatting)
[root@lombe ~]# mkswap /dev/sdc3
[root@lombe ~]# mkswap /dev/sdc4

SWAP memories have been activated
[root@lombe ~]# swapon /dev/sdc3
[root@lombe ~]# swapon /dev/sdc4

So that the SWAP memories are mounted permanently i.e. when starting the machine
the “/etc/fstab” file was edited adding the settings at the end of the file
[root@lombe ~]# nano /etc/fstab
/dev/sdc3 swap swap defaults 0 0
/dev/sdc4 swap swap defaults 0 0

NFSv4

Directories were created to share
[root@lombe ~]# mkdir /local /local/geral /local/financas /local/recursos_humanos
/local/anuncios_refeitorio
[root@lombe ~]# mkdir /partilha /partilha/nfs

Created the same directories on client machines in ways that the mount locations
are the same.

The directories and machines with which the directories will be shared have been defined
[root@lombe ~]# nano /etc/exports
/local/geral/ kwanza(rw,sync,no_root_squash) gazela(rw,sync,no_root_squash)
/local/financas/ kwanza(rw,sync,no_root_squash)
gazela(rw,sync,no_root_squash)
/local/recursos_humanos/ kwanza(rw,sync,no_root_squash)
gazela(rw,sync,no_root_squash)
/local/anuncios_refeitorio/ kwanza(rw,sync,no_root_squash)
gazela(rw,sync,no_root_squash)
/partilha/nfs/ kwanza(rw,sync,no_root_squash) gazela(rw,sync,no_root_squash)

Started the daemon that allows NFS clients to find out which port the server is on
using it, the NFS daemon was started and the services were enabled to start when booting the
machine.
[root@lombe ~]# service rpcbind start
[root@lombe ~]# service nfs start
[root@lombe ~]# chkconfig rpcbind on
[root@lombe ~]# chkconfig nfs on

For NIS clients to mount shared folders permanently, edit the
fstab file on clients (kwanza and gazela):
[root@lombe ~]# nano /etc/fstab
lombe:/local/geral/ /local/geral/ nfs defaults 0 0
lombe:/local/financas / /local/financas/ nfs defaults 0 0
lombe:/local/recursos_humanos/ /local/recursos_humanos/ nfs defaults 0 0
lombe:/local/anuncios_refeitorio/ /local/anuncios_refeitorio/ nfs defaults 0 0
lombe:/partilha/nfs/ /partilha/nfs/ nfs defaults 0 0

Security

To allow only members of the “finance” group to make changes to the files
/local/financas/ directory, the financas group was added
[root@lombe ~]# groupadd financas

The financas group was defined as the owner of the directory /local/financas/
[root@lombe ~]# chgrp financas /local/financas

Changed the permissions of the /local/financas/ directory
[root@lombe ~]# chmod u=rx,g=rwx,o=rx /local/financas

Samba

The necessary packages for samba have been installed
[root@lombe ~]# yum install -y samba samba-common samba-client

The samba sharing directory was created
[root@lombe ~]# mkdir -p /samba/share

The permissions of the samba share directory were changed so that every user can read it,
make change and execute within the directory
[root@lombe ~]# chmod 777 /samba/share

Access to samba users' home directories was enabled in the configuration file:
[root@lombe ~]# nano /etc/samba/smb.conf
workgroup = WORKGROUP
 hosts allow = 127. 192.168.9.
15
security = share
 [Share]
 path = /samba/share
 writeble = yes
 guest ok = yes
 guest only = yes
 create mode = 0777
 directory mode = 0777
 share modes = yes

Set samba passwords for added users
[root@lombe ~]# smbpasswd -a lombe

To allow all users to write to a mounting point
[root@lombe ~]# setsebool -P samba_export_all on

So that the SI does not block users' access to samba, the
following settings. The “-P” means permanent
[root@lombe ~]# setsebool -P use_samba_home_dirs on
[root@lombe ~]# setsebool -P samba_enable_home_dirs on

The service was started and the service was enabled to start when the machine was started
[root@lombe ~]# service smb start
[root@lombe ~]# chkconfig smb on

“gazelle” client configuration

YUM/FTP

FOR CentOS 6 REPOSITORY

The DVD Drive was mounted where the CentOS 6 image was located
[root@lombe ~]# mount /dev/cdrom /media

We went to the directory where the DVD Drive with the RHEL6 image was mounted and to the
Packages directory and installed the following packages
[root@lombe ~]# cd /media/CentOS_6.7_Final/Packages/
[root@lombe ~]# rpm -ivh vsftpd*

The repository folder “/var/ftp/pub/repo” was automatically created after installing FTP
The file containing the YUM settings was edited and the ftp line was changed to
“baseurl=ftp:/// kwanza/pub/repo”

FTP

Configured in the same way as the lombe FTP

NIS Client

It was configured in the same way as the lombe NIS Slave, with the exception of the command
“/usr/lib64/yp/ypinit –s”
