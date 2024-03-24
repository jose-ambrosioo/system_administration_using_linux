# system_administration_using_linux
(LINUX (RED HAT ENTERPRISE, CENTOS, UBUNTU) | WINDOWS)

Implemented an easily scalable network infrastructure based on business volume.  Configured the following services: IP, FTP, YUM, HTTP, DNS, NIS, DHCP, NFSv4, Samba, and openLAD. 

**Introduction**

In this project, I implemented a 100% Open Source Linux network infrastructure for a company.

Note: the texts in green are commands executed in the terminal as root, and the texts in blue are changes to files.

Note: The “iptables” firewall was disabled on the servers and then activated:
<br>#service iptables off
<br>#iptables –F

“kwanza” server
<br>kwanza server partition table
![image](https://github.com/jose-ambrosioo/system_administration_using_linux/assets/59221796/13b87c51-c7b2-4700-b609-5860e1c0a1c8)

Static hostname/IP
<br>Machine name “kwanza.bestsoft.com”
<br>Machine IP “192.168.10.1”

**YUM/FTP**
**FOR RHEL 6 REPOSITORY**

The DVD Drive was mounted where the RHEL 6 image was located
<br>[root@kwanza ~]# mount /dev/cdrom /media

We went to the directory where the DVD Drive with the RHEL6 image is mounted and to the Packages directory and installed the following packages

[root@kwanza ~]# cd /media/RHEL-6.6\ Server.x86_64/Packages/
<br>[root@kwanza ~]# rpm -ivh vsftpd*
<br>[root@kwanza ~]# rpm -ivh deltarpm*
<br>[root@kwanza ~]# rpm -ivh python-deltarpm*
<br>[root@kwanza ~]# rpm -ivh createrepo*

After installing FTP, the repository folder “/var/ftp/pub/repo” was automatically created.

The repository was created using the /var/ftp/pub/repo/Packages directory as its database
<br>[root@kwanza ~]# createrepo --database /var/ftp/pub/repo/Packages/

The file containing the YUM settings has been edited.

[root@kwanza ~]# nano /etc/yum.repos.d/rhel-source.repo
<br>[rhel-source]
<br>name=Red Hat Enterprise Linux $releasever - $basearch - Source
<br>baseurl=file:///var/ftp/pub/repo
<br>enabled=1
<br>gpgcheck=0
<br>#gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

**FTP**
<br>The vsftpd service was enabled to start when starting the machine

[root@kwanza ~]# chkconf vsftpd on

Start the service
<br>[root@kwanza ~]# service vsftpd start

View FTP access
<br>[root@kwanza ~]# getsebool -a | grep ftp
<br>allow_ftpd_anon_write --> off
<br>allow_ftpd_full_access --> off
<br>allow_ftpd_use_cifs --> off
<br>allow_ftpd_use_nfs --> off
<br>ftp_home_dir --> off
<br>ftpd_connect_db --> off
<br>ftpd_use_passive_mode --> off
<br>httpd_enable_ftp_server --> off
<br>vtftp_anon_write --> off

Allow access to FTP content
<br>[root@kwanza ~]# setsebool -P allow_ftpd_full_access on

**HTTP**
<br>HTTP packages have been installed
<br>[root@kwanza ~]# yum install httpd*

An index.html page was created in /var/www/html/

HTTP was activated at the right levels, and the service started
<br>[root@kwanza ~]# chkconfig --list httpd
<br>httpd 0:off 1:off 2:off 3:off 4:off 5:off 6:off
<br>[root@kwanza ~]# chkconfig httpd on
<br>[root@kwanza ~]# service httpd start

3 – Configure httpd, adding the following configuration
<br>[root@kwanza ~]# nano /etc/httpd/conf/httpd.conf
<br><VirtualHost *:80>
<br> ServerAdmin root@kwanza.bestsoft.com
<br> DocumentRoot /var/www/html
<br> ServerName kwanza.bestsoft.com
<br> ErrorLog logs/kwanza.bestsoft.com-error_log
<br> CustomLog logs/kwanza.bestsoft.com-access_log common
<br></VirtualHost>

**DNS**
<br>DNS packages have been installed
<br>[root@kwanza ~]# yum install bind*

<br>Edited the configuration file and placed the main areas for translation
<br>[root@kwanza ~]# nano /etc/named.conf
<br>options {
<br>listen-on port 53 { 127.0.0.1; 192.168.10.1; };
<br> #listen-on-v6 port 53 { ::1; };
<br> directory "/var/named";
<br> dump-file "/var/named/data/cache_dump.db";
<br> statistics-file "/var/named/data/named_stats.txt";
<br> memstatistics-file "/var/named/data/named_mem_stats.txt";
<br> allow-query { 192.168.9.0/24; };
<br> recursion yes;
<br> dnssec-enable yes;
<br> dnssec-validation yes;
<br> dnssec-lookaside auto;
<br> /* Path to ISC DLV key */
<br> bindkeys-file "/etc/named.iscdlv.key";
<br> managed-keys-directory "/var/named/dynamic";
<br>};

The file “/etc/named.rfc1912.zones” was configured
<br>zone "bestsoft.com" IN {
<br> type master;
<br> file "forward.zone";
<br> allow-update { none; };
<br>};
<br>zone "10.168.192.in-addr.arpa" IN {
<br> type master;
<br> file "reverse.zone";
<br> allow-update { none; };
<br>};

Zone files were created and edited, translating the machines that needed translation into the identified zones
<br>[root@kwanza]# cp /var/named/named.localhost /var/named/foward.zone
<br>[root@kwanza]# cp / var/named/named.loopback /var/named/reverse.zone
<br>[root@kwanza named]# nano /var/named/forward.zone
<br>$TTL 1D
<br>@ IN SOA bestsoft.com. root.bestsoft.com. (
<br>0 ; serial
<br>1D ; refresh
<br>6
<br>1H ; retry
<br>1W ; expire
<br>3H ) ; minimum
<br>@ IN NS kwanza.bestsoft.com.
<br>@ IN NS lombe.bestsoft.com.
<br>@ IN NS gazela.bestsoft.com.
<br>@ IN A 192.168.10.1
<br>@ IN A 192.168.10.2
<br>@ IN A 192.168.10.10
<br>kwanza A 192.168.9.1
<br>lombe A 192.168.9.2
<br>gazela A 192.168.9.10

The steps are the same as those done in “forward.zone” except the last 5 lines, the “1”, “2” and “3” represent the last octets represented in IP addresses
<br>[root@kwanza]# nano /etc/reverse.zone
<br>1 IN PTR kwanza. bestsoft.com.
<br>2 IN PTR lombe. bestsoft.com.
<br>3 IN PTR gazela.bestsoft.com.
<br>1 IN PTR bestsoft.com.
<br>1 IN PTR www. bestsoft.com.

The names of the main machines with fixed IP were added, for address translation and
<br>Even if the DNS server fails there will still be communication with these machines.
<br>Note: if you forget this, the server will not be able to translate the IPs and will not find the machines by the name.
<br>[root@kwanza]# nano /etc/hosts
<br>192.168.10.1 kwanza.bestsoft.com www.bestsoft.com
<br>192.168.10.2 lombe.bestsoft.com
<br>192.168.10.10 gazela.bestsoft.com

The server DNS has been configured
<br>[root@kwanza]# nano /etc/sysconf/network-scripts/ifcfg-eth0
<br>DNS1=192.168.10.1
<br>DOMAIN=bestsoft.com

DNS has been activated at the right levels, the service has been started and verified
<br>[root@kwanza]# chkconfig dnsmasq on
<br>[root@kwanza]# service dnsmasq start
<br>[root@kwanza]# nslookup kwanza

**NIS Primary**

Installed the packages for the NIS server and client
<br>[root@kwanza]# yum install yp* -y

The NIS server was configured by adding the following lines
<br>[root@kwanza]# nano /etc/yp.conf
<br>domain nisbestsoft.com server kwanza.bestsoft.com
<br>ypserver kwanza

The NIS domain in which the NIS server and clients will belong has been defined, and all servers and NIS clients must belong to the same domain
<br>[root@kwanza]# nano /etc/sysconfig/network
<br>NETWORKING=yes
<br>HOSTNAME=kwanza.nisbestsoft.com
<br>NISDOMAIN= nisbestsoft.com

Configured the domainname and ypdomainname
<br>[root@kwanza /]# domainname nisbestsoft.com
<br>[root@kwanza /]# ypdomainname nisbestsoft.com

Securenets has been configured
<br>[root@kwanza /]# nano /var/yp/securenets
<br>host 127.0.0.1
<br>255.255.255.0 192.168.10.0

Portmap services and NIS server started
<br>[root@kwanza /]# service rpcbind start
<br>[root@kwanza /]# service ypserv start

Check ipserv
<br>[root@kwanza /]# rpcinfo -u localhost ypserv
<br>program 100004 version 1 ready and waiting
<br>program 100004 version 2 ready and waiting

The NIS server database was created and the NIS map was initialized, where “ –m ” indicates that this is the NIS Master Server
<br>[root@kwanza]# /usr/lib64/yp/ypinit –m

The ypbind service has started
<br>[root@kwanza /]# service ypbind start

The service that allows NIS users to change their passwords has started
<br>[root@kwanza /]# service yppasswdd start

NIS database transfer daemon started
<br>[root@kwanza /]# service ypxfrd start

NIS services were configured to start when booting the machine
<br>[root@kwanza /]# chkconfig rpcbind on
<br>[root@kwanza /]# chkconfig ypserv on
<br>[root@kwanza /]# chkconfig ypbind on
<br>[root@kwanza /]# chkconfig yppasswdd on
<br>[root@kwanza /]# chkconfig ypxfrd on
<br>The “/nishome” directory was created, where the home of all NIS users will be located

[root@kwanza /]# mkdir /nishome
<br>[root@kwanza /]# chcon --reference /home /nishome -R

Added NIS user accounts
<br>[root@kwanza /]# useradd –p joao –d /nishome/joao -c 'Joao' joao
<br>[root@kwanza /]# useradd –p ana –d /nishome/ana -c 'Ana' ana
<br>[root@kwanza /]# useradd –p vasco –d /nishome/vasco -c 'Vasco' vasco
<br>[root@kwanza /]# useradd –p maria –d /nishome/maria maria -c 'Maria' maria
<br>[root@kwanza /]# useradd –p teresa –d /nishome/teresa -c 'Teresa' teresa
<br>[root@kwanza /]# useradd –p admin –d /nishome/admin admin -c 'Admin' admin
<br>[root@kwanza /]# useradd –p admin_intern –d /nishome/admin_intern admin_
intern -c 'Admin_intern' admin_intern

NIS user passwords have been configured
<br>[root@kwanza]# passwd usuario

Configured 3 NIS user accounts to expire on 2018-06-20
<br>[root@kwanza]# chage -E 2018-06-20 ana
<br>[root@kwanza]# chage -l ana
<br>Account expires : Jun 20, 2018
<br>[root@kwanza]# chage -E 2018-06-20 maria
<br>[root@ kwanza]# chage -E 2018-06-20 teresa

So that NIS users' home directories can be mounted on NIS client machines we export /nishome via NFS
<br>[root@kwanza]# nano /etc/exports
/nishome *(rw)

To allow the primary NIS server database to be transferred to the NIS
<br>Slaves:
<br>[root@kwanza]# nano /var/yp/Makefile
<br>NOPUSH=false

The database has been updated
<br>[root@kwanza]# make -C /var/yp

NOTE: Whenever a user is added, we must update the server database
<br>NIS doing:
<br>[root@kwanza]# make -C /var/yp

NOTE2: When resetting the password of a user created without a password, delete the encrypted password in the
<br>/etc/shadow file on NIS server

**Security**

To allow only the admin user to invoke the “su” command on any machine or server, on all machines, the “wheel” group was defined as the owner of the su command:
<br>[root@kwanza /]# chgrp wheel /bin/su

Changed the permissions of /bin/su
<br>[root@kwanza /]# chmod u=rws,g=rwx,o= /bin/su

The user “admin” was added to the “wheel” group
<br>[root@kwanza /]# usermod –G wheel admin

So that the user “admin_intern” only has permission to restart Apache Web Server (httpd restart) on the kwanza server and for users admin and admin_intern have additional privileges, the following was added to the file:
<br>[root@kwanza /]# visudo
<br>admin_intern kwanza.bestsoft.com=/sbin/service httpd restart
<br>admin ALL=(ALL) ALL
<br>admin_intern ALL=(ALL) ALL

So that only the two administrator accounts, admin and admin_intern, and root can login to the kwanza and lombe servers, the /etc/ssh/sshd_config file was edited in the mentioned servers, and the following lines were added:
<br>[root@kwanza /]# nano /etc/ssh/sshd_config
<br>AllowUsers admin admin_intern root
<br>DenyUsers joao ana vasco maria teresa
<br>PermitRootLogin yes

“lombe” server configuration
<br>lombe server partition table
![image](https://github.com/jose-ambrosioo/system_administration_using_linux/assets/59221796/36421297-6a9c-4dde-817a-716188325630)

Static hostname/IP
<br>Nome da máquina “lombe.bestsoft.com”
<br>IP da máquina “192.168.10.2”

**YUM/FTP**
**FOR RHEL 6 REPOSITORY**

The DVD Drive was mounted where the RHEL 6 image was located
<br>[root@lombe ~]# mount /dev/cdrom /media

We went to the directory where the DVD Drive with the RHEL6 image was mounted and to the Packages directory and installed the following packages
<br>[root@lombe ~]# cd /media/RHEL-6.6\ Server.x86_64/Packages/
<br>[root@lombe ~]# rpm -ivh vsftpd*

The repository folder “/var/ftp/pub/repo” was automatically created after installing FTP

The file containing the YUM settings has been edited
<br>[root@lombe ~]# nano /etc/yum.repos.d/rhel-source.repo
<br>[rhel-source]
<br>name=Red Hat Enterprise Linux $releasever - $basearch - Source
<br>10
<br>baseurl=file:///kwanza/pub/repo
<br>enabled=1
<br>gpgcheck=0
<br>#gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

**FTP**

The vsftpd service was enabled to start when starting the machine
<br>[root@lombe ~]# chkconf vsftpd on
<br>Inicializou-se o serviço
<br>[root@lombe ~]# service vsftpd start

**HOSTS**

The “/etc/hosts” file was configured
<br>[root@lombe ~]# nano /etc/hosts
<br>192.168.10.1 kwanza.bestsoft.com kwanza www.bestsoft.com
<br>192.168.10.2 lombe.bestsoft.com lombe
<br>192.168.10.10 gazela.bestsoft.com gazela

**DHCP**

DHCP installation and configurations were carried out
<br>[root@lombe ~]# yum install dhcp -y
<br>[root@lombe ~]# nano /etc/dhcpd.conf
<br>option domain-name “bestsoft.com”
<br>option domain-name-servers “kwanza.bestsoft.com”

Set itself as the network's official DHCP server
<br>subnet 192.168.10.0 netmask 255.255.255.0 {
<br> range 192.168.10.10 192.168.10.254;
<br> option domain-name-servers kwanza.bestsoft.com;
<br> option domain-name " bestsoft.com";
<br> option routers 192.168.10.1;
<br> option broadcast-address 192.168.10.255;
<br> default-lease-time 600;
<br> max-lease-time 7200;
<br>}
<br>host gazela {
<br>hardware Ethernet 00:0c:29:73:69:34;
<br>fixed-address 192.168.10.10;
<br>}

DHCP was configured to start when booting the machine
<br>[root@lombe ~]# chkconfig dhcp on
<br>[root@lombe ~]# service dhcp start

**NIS Secondary**

Installed the packages for the NIS server and client
<br>[root@lombe ~]# yum install yp* -y

The NIS domain to which the NIS server and clients will belong has been defined
<br>[root@lombe ~]# nano /etc/sysconfig/network
<br>NETWORKING=yes
<br>HOSTNAME=lombe.bestsoft.com
<br>NISDOMAIN=nisbestsoft.com

All NIS servers for this client have been identified
<br>[root@lombe ~]# nano /etc/yp.conf
<br>domain nisbestsoft.com server kwanza.bestsoft.com
<br>domain nisbestsoft.com server lombe.bestsoft.com
<br>ypserver kwanza

For synchronizing user and account information across all machines in the domain the file was edited:
<br>[root@lombe ~]# nano /etc/nsswitch.conf

and changed the “passwd, shadow and group” fields to:
<br>passwd: files nis
<br>shadow: files nis
<br>group: files nis

Configured the domainname and ypdomainname
<br>[root@lombe ~]# domainname nisbestsoft.com
<br>[root@lombe ~]# ypdomainname nisbestsoft.com

NIS services were configured to start when booting the machine
<br>[root@lombe ~]# chkconfig rpcbind on
<br>[root@lombe ~]# chkconfig ypbind on

Check ypbind
<br>[root@lombe ~]# rpcinfo -u localhost ypbind
<br>program 100004 version 1 ready and waiting
<br>program 100004 version 2 ready and waiting

The NIS server database file was updated and the NIS map was initialized, where “ –s ” indicates that this is the NIS Slave Server
<br>[root@lombe ~]# ypcat passwd
<br>[root@lombe ~]# /usr/lib64/yp/ypinit –s lombe

For NIS clients to mount the “/nishome” folders permanently, the file was edited fstab on clients (Lombe and Gazelle), adding the settings at the end of the file:
<br>[root@lombe ~]# nano /etc/fstab
<br>kwanza:/nishome/ /nishome/ nfs defaults 0 0

Note: start the nfs service on the kwanza server and make it start whenever the machine start.

Note2: to update fstab without restarting a machine: #mount -a

Partition management (Partition Creation, VG, PV, LVM and SWAP)
<br>A 12 Gb IDE disk was added and the machine started

**CREATION OF PARTITIONS**

The command that allows us to manage disk partitions was executed, to edit the disk added
<br>[root@lombe ~]# fdisk /dev/sdc

1-Select the option to add partitions
<br>Command (m for help): n

2-Defined as primary partition by selecting option “p”

3-The partition number was chosen as “1”
<br>Partition number (1 - 4): 1

4-In the next step we leave it blank and press ENTER, because by default it searches the first free cylinder on the disk and the file system of the partition being created, something like the following: <br>First cylinder (1-1566, default 1):

5-The size of the partition was defined, which in this case is 5Gb, something like the following:
<br>Last cylinder, +cylinders or +size{K,M,G} (1-1566, default 1566): +5GB

The other partitions were added with their respective sizes (GB for Gigabytes and MB for Megabytes) repeating the same steps from step 1.

After finishing so that the partitions were saved, the corresponding one was chosen option:
<br>Command (m for help): w

The machine was restarted just to ensure that the changes were applied
<br>[root@lombe ~]# reboot

The partition file system was created for the physical volumes (ext4 formatting)
<br>[root@lombe ~]# mkfs.ext4 /dev/sdc1
<br>[root@lombe ~]# mkfs.ext4 /dev/sdc2

**VG, PV, LVM CREATION**

PVs (Physical Volumes) were created
<br>[root@lombe ~]# pvcreate /dev/sdc1
<br>[root@lombe ~]# pvcreate /dev/sdc2

VG (Volume Group) was created called “vgvendafinancas”
<br>[root@lombe ~]# vgcreate vgvendafinancas /dev/sdc1 /dev/sdc2

The 4 LVs (Logical Volumes) of each 2Gb were created in the VG vgvendafinancas, following a naming like “lvm1, lvm2, . . . , lvmn”
<br>[root@lombe ~]# lvcreate –n lv1 –L 2GB vgvendafinancas
<br>[root@lombe ~]# lvcreate –n lv2 –L 2GB vgvendafinancas
<br>[root@lombe ~]# lvcreate –n lv3 –L 2GB vgvendafinancas
<br>[root@lombe ~]# lvcreate –n lv4 –L 2GB vgvendafinancas

The file system of the created LVMs was created (Formatting)
<br>[root@lombe ~]# mkfs.ext4 /dev/vgvendafinancas/lv1
<br>[root@lombe ~]# mkfs.ext4 /dev/vgvendafinancas/lv2
<br>[root@lombe ~]# mkfs.ext4 /dev/vgvendafinancas/lv3
<br>[root@lombe ~]# mkfs.ext4 /dev/vgvendafinancas/lv4

Directories were created where the LVMs will be mounted for better organization
<br>[root@lombe ~]# mkdir /vgvendafinancas /vgvendafinancas/lv1 /vgvendafinancas/lv2
<br>[root@lombe ~]# mkdir /vgvendafinancas/lv3 /vgvendafinancas/lv4

So that the LVs are mounted permanently, that is, when starting the machine, the file “/etc/fstab” adding the settings at the end of the file
<br>[root@lombe ~]# nano /etc/fstab
<br>/dev/vgvendafinancas/lv1 /vgvendafinancas/lv1/ nfs defaults 0 0
<br>/dev/vgvendafinancas/lv2 /vgvendafinancas/lv2/ nfs defaults 0 0
<br>/dev/vgvendafinancas/lv3 /vgvendafinancas/lv3/ nfs defaults 0 0
<br>/dev/vgvendafinancas/lv4 /vgvendafinancas/lv4/ nfs defaults 0 0

**SWAP CREATION**

We created two primary partitions of 512MB sdc3 and sdc4

1-Select the option to edit the type of partitions
<br>Command (m for help): t

2-The partition type “82” was chosen (Hexadecimal value for the partition type “Linux swap/Solaris”)
<br>Hex code (type L to list codes):82

After finishing so that the partitions were saved, the corresponding one was chosen option:
<br>Command (m for help): w

Created the Partition File System for SWAP (SWAP Formatting)
<br>[root@lombe ~]# mkswap /dev/sdc3
<br>[root@lombe ~]# mkswap /dev/sdc4

SWAP memories have been activated
<br>[root@lombe ~]# swapon /dev/sdc3
<br>[root@lombe ~]# swapon /dev/sdc4

So that the SWAP memories are mounted permanently i.e. when starting the machine the “/etc/fstab” file was edited adding the settings at the end of the file
<br>[root@lombe ~]# nano /etc/fstab
/dev/sdc3 swap swap defaults 0 0
/dev/sdc4 swap swap defaults 0 0

**NFSv4**

Directories were created to share
<br>[root@lombe ~]# mkdir /local /local/geral /local/financas /local/recursos_humanos
<br>/local/anuncios_refeitorio
<br>[root@lombe ~]# mkdir /partilha /partilha/nfs

Created the same directories on client machines in ways that the mount locations are the same.

The directories and machines with which the directories will be shared have been defined
<br>[root@lombe ~]# nano /etc/exports
<br>/local/geral/ kwanza(rw,sync,no_root_squash) gazela(rw,sync,no_root_squash)
<br>/local/financas/ kwanza(rw,sync,no_root_squash)
<br>gazela(rw,sync,no_root_squash)
<br>/local/recursos_humanos/ kwanza(rw,sync,no_root_squash)
<br>gazela(rw,sync,no_root_squash)
<br>/local/anuncios_refeitorio/ kwanza(rw,sync,no_root_squash)
<br>gazela(rw,sync,no_root_squash)
<br>/partilha/nfs/ kwanza(rw,sync,no_root_squash) gazela(rw,sync,no_root_squash)

Started the daemon that allows NFS clients to find out which port the server is on using it, the NFS daemon was started and the services were enabled to start when booting the machine.
<br>[root@lombe ~]# service rpcbind start
<br>[root@lombe ~]# service nfs start
<br>[root@lombe ~]# chkconfig rpcbind on
<br>[root@lombe ~]# chkconfig nfs on

For NIS clients to mount shared folders permanently, edit the fstab file on clients (kwanza and gazela):
<br>[root@lombe ~]# nano /etc/fstab
<br>lombe:/local/geral/ /local/geral/ nfs defaults 0 0
<br>lombe:/local/financas / /local/financas/ nfs defaults 0 0
<br>lombe:/local/recursos_humanos/ /local/recursos_humanos/ nfs defaults 0 0
<br>lombe:/local/anuncios_refeitorio/ /local/anuncios_refeitorio/ nfs defaults 0 0
<br>lombe:/partilha/nfs/ /partilha/nfs/ nfs defaults 0 0

**Security**

To allow only members of the “finance” group to make changes to the files /local/financas/ directory, the financas group was added
<br>[root@lombe ~]# groupadd financas

The financas group was defined as the owner of the directory /local/financas/
<br>[root@lombe ~]# chgrp financas /local/financas

Changed the permissions of the /local/financas/ directory
<br>[root@lombe ~]# chmod u=rx,g=rwx,o=rx /local/financas

**Samba**

The necessary packages for samba have been installed
<br>[root@lombe ~]# yum install -y samba samba-common samba-client

The samba sharing directory was created
<br>[root@lombe ~]# mkdir -p /samba/share

The permissions of the samba share directory were changed so that every user can read it, make change and execute within the directory
<br>[root@lombe ~]# chmod 777 /samba/share

Access to samba users' home directories was enabled in the configuration file:
<br>[root@lombe ~]# nano /etc/samba/smb.conf
<br>workgroup = WORKGROUP
<br> hosts allow = 127. 192.168.9.
<br>15
<br>security = share
<br> [Share]
<br> path = /samba/share
<br> writeble = yes
<br> guest ok = yes
<br> guest only = yes
<br> create mode = 0777
<br> directory mode = 0777
<br> share modes = yes

Set samba passwords for added users
<br>[root@lombe ~]# smbpasswd -a lombe

To allow all users to write to a mounting point
<br>[root@lombe ~]# setsebool -P samba_export_all on

So that the SI does not block users' access to samba, the following settings. The “-P” means permanent
<br>[root@lombe ~]# setsebool -P use_samba_home_dirs on
<br>[root@lombe ~]# setsebool -P samba_enable_home_dirs on

The service was started and the service was enabled to start when the machine was started
<br>[root@lombe ~]# service smb start
<br>[root@lombe ~]# chkconfig smb on

“gazelle” client configuration

**YUM/FTP**

**FOR CentOS 6 REPOSITORY**

The DVD Drive was mounted where the CentOS 6 image was located
<br>[root@lombe ~]# mount /dev/cdrom /media

We went to the directory where the DVD Drive with the RHEL6 image was mounted and to the Packages directory and installed the following packages
<br>[root@lombe ~]# cd /media/CentOS_6.7_Final/Packages/
<br>[root@lombe ~]# rpm -ivh vsftpd*

The repository folder “/var/ftp/pub/repo” was automatically created after installing FTP
<br>The file containing the YUM settings was edited and the ftp line was changed to “baseurl=ftp:/// kwanza/pub/repo”

**FTP**

Configured in the same way as the lombe FTP

**NIS Client**

It was configured in the same way as the lombe NIS Slave, with the exception of the command “/usr/lib64/yp/ypinit –s”
