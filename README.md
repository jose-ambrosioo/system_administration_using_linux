# system_administration_using_linux
(LINUX (RED HAT ENTERPRISE, CENTOS, UBUNTU) | WINDOWS)

Implemented a network infrastructure that was easily scalable based on business volume.  Configured the following services: IP, FTP, YUM, HTTP, DNS, NIS, DHCP, NFSv4, Samba, and openLAD. 

Introdução
Este relatório tem como objectivo explicar e especificar a implementação da
infraestrutura de rede Linux 100% Open Source para a empresa Best Soft Lda.

Obs: os textos em verde são comandos executados no terminal como root e os textos
em azul são alterações em ficheiros.

Obs: Desactivou-se o firewall “iptables” nos servidores e depois activou-se:
#service iptables off
#iptables –F

Servidor “kwanza”
Tabela de partições do servidor kwanza

Hostname/IP estático
Nome da máquina “kwanza.bestsoft.com”
IP da máquina “192.168.10.1”

YUM/FTP
PARA REPOSITÓRIO DO RHEL 6
Montou-se a Drive do DVD onde estava a imagem do RHEL 6
[root@kwanza ~]# mount /dev/cdrom /media

Foi-se para o directório onde estava montada a Drive do DVD com a imagem do RHEL6 e para o
directório Packages e instalou-se os seguintes pacotes

[root@kwanza ~]# cd /media/RHEL-6.6\ Server.x86_64/Packages/
[root@kwanza ~]# rpm -ivh vsftpd*
[root@kwanza ~]# rpm -ivh deltarpm*
[root@kwanza ~]# rpm -ivh python-deltarpm*
[root@kwanza ~]# rpm -ivh createrepo*

A pasta do repositório “/var/ftp/pub/repo” foi criada automaticamente após instalação do FTP

Criou-se o repositório tendo como base de dados o diretório /var/ftp/pub/repo/Packages
[root@kwanza ~]# createrepo --database /var/ftp/pub/repo/Packages/

Editou-se o ficheiro que contém as configurações do YUM

[root@kwanza ~]# nano /etc/yum.repos.d/rhel-source.repo
[rhel-source]
name=Red Hat Enterprise Linux $releasever - $basearch - Source
baseurl=file:///var/ftp/pub/repo
enabled=1
gpgcheck=0
#gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

FTP
Activou-se o serviço vsftpd para arancar ao iniciar a máquina

[root@kwanza ~]# chkconf vsftpd on

Iniciar o serviço
[root@kwanza ~]# service vsftpd start

Ver o acesso ao FTP
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

Permitir o accesso a conteúdo do FTP
[root@kwanza ~]# setsebool -P allow_ftpd_full_access on

HTTP
Instalou-se os pacotes para o HTTP
[root@kwanza ~]# yum install httpd*

Criou-se uma página index.html em /var/www/html/

Acivou-se o HTTP nos níveis certos e iniciou-se o serviço
[root@kwanza ~]# chkconfig --list httpd
httpd 0:off 1:off 2:off 3:off 4:off 5:off 6:off
[root@kwanza ~]# chkconfig httpd on
[root@kwanza ~]# service httpd start

3 – Configurou-se o httpd, adicionando a seguinte configuração
[root@kwanza ~]# nano /etc/httpd/conf/httpd.conf
<VirtualHost *:80>
 ServerAdmin root@kwanza.bestsoft.com
 DocumentRoot /var/www/html
 ServerName kwanza.bestsoft.com
 ErrorLog logs/kwanza.bestsoft.com-error_log
 CustomLog logs/kwanza.bestsoft.com-access_log common
</VirtualHost>

DNS
Instalou-se os pacotes para o DNS
[root@kwanza ~]# yum install bind*

Editou-se o ficheiro de configuração e colocou-se as principais zonas para tradução
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

Configurou-se o ficheiro “/etc/named.rfc1912.zones”
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

Criou-se e editou-se os ficheiros zone, traduzindo as máquinas que precisam de tradução nas
zonas identificadas
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

Os passos são iguais aos que foram feitos no “foward.zone” excepto as últimas 5 linhas, o “1”,
“2” e o “3” representam os últimos octetos representados nos endereços IPs
[root@kwanza]# nano /etc/reverse.zone
1 IN PTR kwanza. bestsoft.com.
2 IN PTR lombe. bestsoft.com.
3 IN PTR gazela.bestsoft.com.
1 IN PTR bestsoft.com.
1 IN PTR www. bestsoft.com.

Adicionou-se os nomes das principais maquinas com ip fixo, para tradução de endereços e
mesmo que o servidor DNS falhe ainda haverá comunicação com essas máquinas.
Obs: se esquecer isto o servidor não conseguirá traduzir os ips e não encontrará as máquinas
pelo nome.
[root@kwanza]# nano /etc/hosts
192.168.10.1 kwanza.bestsoft.com www.bestsoft.com
192.168.10.2 lombe.bestsoft.com
192.168.10.10 gazela.bestsoft.com

Configurou-se o DNS do servidor
[root@kwanza]# nano /etc/sysconf/network-scripts/ifcfg-eth0
DNS1=192.168.10.1
DOMAIN=bestsoft.com

Activou-se o DNS nos níveis certos, iniciou-se e verificou-se o serviço
[root@kwanza]# chkconfig dnsmasq on
[root@kwanza]# service dnsmasq start
[root@kwanza]# nslookup kwanza

NIS Primário

Instalou-se os pacotes para o NIS servidor e cliente
[root@kwanza]# yum install yp* -y

Configurou-se o servidor NIS adicionando as seguintes linhas
[root@kwanza]# nano /etc/yp.conf
domain nisbestsoft.com server kwanza.bestsoft.com
ypserver kwanza
7

Definiu-se o domínio NIS no qual o servidor e os cliente NIS irão pertencer, todos os servidores
e clientes NIS devem pertencer ao mesmo domínio
[root@kwanza]# nano /etc/sysconfig/network
NETWORKING=yes
HOSTNAME=kwanza.nisbestsoft.com
NISDOMAIN= nisbestsoft.com

Configurou-se o domainname e o ypdomainname
[root@kwanza /]# domainname nisbestsoft.com
[root@kwanza /]# ypdomainname nisbestsoft.com

Configurou-se o securenets
[root@kwanza /]# nano /var/yp/securenets
host 127.0.0.1
255.255.255.0 192.168.10.0

Iniciou-se os serviços portmap e o NIS servidor
[root@kwanza /]# service rpcbind start
[root@kwanza /]# service ypserv start

Verficar o ypserv
[root@kwanza /]# rpcinfo -u localhost ypserv
program 100004 version 1 ready and waiting
program 100004 version 2 ready and waiting

Criou-se a base de dados do servidor NIS e inicializou-se o NIS map, onde “ –m ” indica que este
é o NIS Master Server
[root@kwanza]# /usr/lib64/yp/ypinit –m

Inicializou-se o serviço ypbind
[root@kwanza /]# service ypbind start

Iniciou-se o serviço que permite os usuários NIS mudarem as suas senhas
[root@kwanza /]# service yppasswdd start

Iniciou-se o daemon de transferência de base de dados NIS
[root@kwanza /]# service ypxfrd start

Configurou-se os serviços NIS para arrancarem ao inicializar a máquina
[root@kwanza /]# chkconfig rpcbind on
[root@kwanza /]# chkconfig ypserv on
[root@kwanza /]# chkconfig ypbind on
[root@kwanza /]# chkconfig yppasswdd on
[root@kwanza /]# chkconfig ypxfrd on
Criou-se o diretório “/nishome” onde ficará a home de todos os usuários NIS

[root@kwanza /]# mkdir /nishome
[root@kwanza /]# chcon --reference /home /nishome -R

Adicionou-se as contas dos utilizadores NIS
[root@kwanza /]# useradd –p joao –d /nishome/joao -c 'Joao' joao
[root@kwanza /]# useradd –p ana –d /nishome/ana -c 'Ana' ana
[root@kwanza /]# useradd –p vasco –d /nishome/vasco -c 'Vasco' vasco
[root@kwanza /]# useradd –p maria –d /nishome/maria maria -c 'Maria' maria
[root@kwanza /]# useradd –p teresa –d /nishome/teresa -c 'Teresa' teresa
[root@kwanza /]# useradd –p admin –d /nishome/admin admin -c 'Admin' admin
[root@kwanza /]# useradd –p admin_estagiario –d /nishome/admin_estagiario admin_
estagiario -c 'Admin_estagiario' admin_estagiario

Configurou-se as senhas dos utilizadores NIS
[root@kwanza]# passwd usuario

Configurou-se 3 contas de utilizadores NIS para expirarem em 2018-06-20
[root@kwanza]# chage -E 2018-06-20 ana
[root@kwanza]# chage -l ana
Account expires : Jun 20, 2018
[root@kwanza]# chage -E 2018-06-20 maria
[root@ kwanza]# chage -E 2018-06-20 teresa

Para que os directórios home dos usuários NIS possam ser montados na máquinas clientes NIS
exportamos a /nishome via NFS
[root@kwanza]# nano /etc/exports
/nishome *(rw)

Para permitir que a base de dados do servidor NIS primário possa ser transferida para os NIS
Slaves:
[root@kwanza]# nano /var/yp/Makefile
NOPUSH=false

Actualizou-se a base de dados
[root@kwanza]# make -C /var/yp

OBS: Sempre que um utilizador for adicionado devemos actualizar a base de dados do servidor
NIS fazendo:
[root@kwanza]# make -C /var/yp

OBS2: Ao redefinir a senha de um usuário criado sem senha, excluir a senha encriptada no
ficheiro /etc/shadow no servidor NIS

Segurança

Para permitir apenas o usuário admin invocar o comando “su” em qualquer máquina ou
servidor, em todas as máquinas definiu-se o grupo “wheel” como dono do comando su:
[root@kwanza /]# chgrp wheel /bin/su

Mudou-se as permissões do /bin/su
[root@kwanza /]# chmod u=rws,g=rwx,o= /bin/su

Adicionou-se o utilizador “admin” ao grupo “wheel”
[root@kwanza /]# usermod –G wheel admin

Para que o utilizador “admin_estagiario” apenas tenha permissão de reiniciar o Apache Web
Server (restart do httpd) no servidor kwanza e para que os utilizadores admin e admin_estagiario
tenham previlégios adicionais, adicionou-se o seguinte no ficheiro:
[root@kwanza /]# visudo
admin_estagiario kwanza.bestsoft.com=/sbin/service httpd restart
admin ALL=(ALL) ALL
admin_estagiario ALL=(ALL) ALL

Para que apenas as duas contas de administradores admin e admin_estagiario e root possam
fazer login nos servidores kwanza e lombe, editou-se o ficheiro /etc/ssh/sshd_config nos
referidos servidores e adicionou-se as seguinte linhas:
[root@kwanza /]# nano /etc/ssh/sshd_config
AllowUsers admin admin_estagiario root
DenyUsers joao ana vasco maria teresa
PermitRootLogin yes

Configuração do servidor “lombe”
Tabela de partições do servidor lombe

Hostname/IP estático
Nome da máquina “lombe.bestsoft.com”
IP da máquina “192.168.10.2”

YUM/FTP
PARA REPOSITÓRIO DO RHEL 6

Montou-se a Drive do DVD onde estava a imagem do RHEL 6
[root@lombe ~]# mount /dev/cdrom /media

Foi-se para o directório onde estava montada a Drive do DVD com a imagem do RHEL6 e para o
directório Packages e instalou-se os seguintes pacotes
[root@lombe ~]# cd /media/RHEL-6.6\ Server.x86_64/Packages/
[root@lombe ~]# rpm -ivh vsftpd*

A pasta do repositório “/var/ftp/pub/repo” foi criada automaticamente após instalação do FTP

Editou-se o ficheiro que contém as configurações do YUM
[root@lombe ~]# nano /etc/yum.repos.d/rhel-source.repo
[rhel-source]
name=Red Hat Enterprise Linux $releasever - $basearch - Source
10
baseurl=file:///kwanza/pub/repo
enabled=1
gpgcheck=0
#gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
FTP

Activou-se o serviço vsftpd para arancar ao iniciar a máquina
[root@lombe ~]# chkconf vsftpd on
Inicializou-se o serviço
[root@lombe ~]# service vsftpd start
HOSTS

Configurou-se o ficheiro “/etc/hosts”
[root@lombe ~]# nano /etc/hosts
192.168.10.1 kwanza.bestsoft.com kwanza www.bestsoft.com
192.168.10.2 lombe.bestsoft.com lombe
192.168.10.10 gazela.bestsoft.com gazela
DHCP

Realizou-se a instalação e as configurações do DHCP
[root@lombe ~]# yum install dhcp -y
[root@lombe ~]# nano /etc/dhcpd.conf
option domain-name “bestsoft.com”
option domain-name-servers “kwanza.bestsoft.com”

Definiu-se como o servidor DHCP oficial da rede
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

Configurou-se o DHCP para iniciar ao inicializar a máquina e iniciou-se e ser
[root@lombe ~]# chkconfig dhcp on
[root@lombe ~]# service dhcp start

NIS Secundário

Instalou-se os pacotes para o NIS servidor e cliente
[root@lombe ~]# yum install yp* -y


Definiu-se o domínio NIS no qual o servidor e os clientes NIS irão pertencer
[root@lombe ~]# nano /etc/sysconfig/network
NETWORKING=yes
HOSTNAME=lombe.bestsoft.com
NISDOMAIN=nisbestsoft.com

Identificou-se todos os servidores NIS para este cliente
[root@lombe ~]# nano /etc/yp.conf
domain nisbestsoft.com server kwanza.bestsoft.com
domain nisbestsoft.com server lombe.bestsoft.com
ypserver kwanza

Para sincronização de usuários e informação de contas em todas as máquinas do domínio
editou-se o ficheiro:
[root@lombe ~]# nano /etc/nsswitch.conf

e mudou-se os campos “passwd, shadow e group” para:
passwd: files nis
shadow: files nis
group: files nis

Configurou-se o domainname e o ypdomainname
[root@lombe ~]# domainname nisbestsoft.com
[root@lombe ~]# ypdomainname nisbestsoft.com

Configurou-se os serviços NIS para arrancarem ao inicializar a máquina
[root@lombe ~]# chkconfig rpcbind on
[root@lombe ~]# chkconfig ypbind on

Verficar o ypbind
[root@lombe ~]# rpcinfo -u localhost ypbind
program 100004 version 1 ready and waiting
program 100004 version 2 ready and waiting

Actualizou-se o ficheiro da base de dados do servidor NIS e inicializou-se o NIS map, onde “ –s ”
indica que este é o NIS Slave Server
[root@lombe ~]# ypcat passwd
[root@lombe ~]# /usr/lib64/yp/ypinit –s lombe

Para que os clientes NIS montem as pastas “/nishome” permanentemente editou-se o ficheiro
fstab nos clientes(lombe e gazela) adicionando-se as configurações no final do ficheiro:
[root@lombe ~]# nano /etc/fstab
kwanza:/nishome/ /nishome/ nfs defaults 0 0

Obs: iniciar o serviço nfs no servidor kwanza e fazer com que este arranque sempre que a
máquina iniciar.

Obs2: para actualizar o fstab sem reiniciar uma máquina: #mount -a

Gestão de partições (Criação de Partições, VG, PV, LVM e SWAP)
Adicionou-se um disco IDE de 12 Gb e iniciou-se a máquina

CRIAÇÃO DAS PARTIÇÕES

Executou-se o comando que nos permite fazer a gestão de partições do discos, para editar o
disco adicionado
[root@lombe ~]# fdisk /dev/sdc

1-Selecionou-se a opção para adicionar partições
Command (m for help): n

2-Definiu-se como partição primária selecionando a opção “p”

3-Escolheu-se o número da partição como “1”
Partition number (1 - 4): 1

4-No passo seguinte deixamos em branco e pressionamos ENTER, porque por padrão ele busca
o primeiro cilindro livre no disco e começará ali o sistema de ficheiros da partição que está a ser
criada, algo como o seguinte:
First cylinder (1-1566, default 1):

5-Definiu-se o tamanho da partição que neste caso são 5Gb, algo como o seguinte:
Last cylinder, +cylinders or +size{K,M,G} (1-1566, default 1566): +5GB
Adicionou-se as demais partições com os respectivos tamanhos (GB para Gigabytes e MB para
Megabytes)repetindo os mesmos passos, desde o passo 1.

Depois que terminar para que as partições fossem guardadas escolheu-se a correspondente
opção:
Command (m for help): w

Reiniciou-se a máquina só para garantir que as mudanças sejam aplicadas
[root@lombe ~]# reboot

Criou-se o Sistema de ficheiros das partições para os volumes físicos (Formatação ext4)
[root@lombe ~]# mkfs.ext4 /dev/sdc1
[root@lombe ~]# mkfs.ext4 /dev/sdc2

CRIAÇÃO DE VG, PV, LVM

Criou-se os PVs (Physical Volumes)
[root@lombe ~]# pvcreate /dev/sdc1
[root@lombe ~]# pvcreate /dev/sdc2

Criou-se o VG (Volume Group) chamado “vgvendafinancas”
[root@lombe ~]# vgcreate vgvendafinancas /dev/sdc1 /dev/sdc2

Criou-se os 4 LVs (Logical Volumes) de cada 2Gb no VG vgvendafinancas, seguindo uma
nomeclatura do tipo “lvm1, lvm2, . . . , lvmn ”
[root@lombe ~]# lvcreate –n lv1 –L 2GB vgvendafinancas
[root@lombe ~]# lvcreate –n lv2 –L 2GB vgvendafinancas
[root@lombe ~]# lvcreate –n lv3 –L 2GB vgvendafinancas
[root@lombe ~]# lvcreate –n lv4 –L 2GB vgvendafinancas
Criou-se o Sistema de ficheiros dos LVMs criados (Formatação)
[root@lombe ~]# mkfs.ext4 /dev/vgvendafinancas/lv1
[root@lombe ~]# mkfs.ext4 /dev/vgvendafinancas/lv2
[root@lombe ~]# mkfs.ext4 /dev/vgvendafinancas/lv3
[root@lombe ~]# mkfs.ext4 /dev/vgvendafinancas/lv4

Criou-se os diretórios onde as LVMs serão montadas para melhor organização
[root@lombe ~]# mkdir /vgvendafinancas /vgvendafinancas/lv1 /vgvendafinancas/lv2
[root@lombe ~]# mkdir /vgvendafinancas/lv3 /vgvendafinancas/lv4

Para que as LVs sejam montados permanentemente ou seja, ao iniciar a máquina editou-se o
ficheiro “/etc/fstab” adicionando-se as configurações no final do ficheiro
[root@lombe ~]# nano /etc/fstab
/dev/vgvendafinancas/lv1 /vgvendafinancas/lv1/ nfs defaults 0 0
/dev/vgvendafinancas/lv2 /vgvendafinancas/lv2/ nfs defaults 0 0
/dev/vgvendafinancas/lv3 /vgvendafinancas/lv3/ nfs defaults 0 0
/dev/vgvendafinancas/lv4 /vgvendafinancas/lv4/ nfs defaults 0 0

CRIAÇÃO DE SWAP

Criamos duas partições primárias de 512MB sdc3 e sdc4

1-Selecionou-se a opção para editar o tipo das partições
Command (m for help): t

2-Escolheu-se o tipo de partição “82” (Valor Hexadecimal para o tipo de partição “Linux
swap/Solaris”)
Hex code (type L to list codes):82

Depois que terminar para que as partições fossem guardadas escolheu-se a correspondente
opção:
Command (m for help): w

Criou-se o Sistema de ficheiros das partições para SWAP (Formatação SWAP)
[root@lombe ~]# mkswap /dev/sdc3
[root@lombe ~]# mkswap /dev/sdc4

Activou-se as memórias SWAP
[root@lombe ~]# swapon /dev/sdc3
[root@lombe ~]# swapon /dev/sdc4

Para que as memórias SWAP sejam montadas permanentemente ou seja, ao iniciar a máquina
editou-se o ficheiro “/etc/fstab” adicionando-se as configurações no final do ficheiro
[root@lombe ~]# nano /etc/fstab
/dev/sdc3 swap swap defaults 0 0
/dev/sdc4 swap swap defaults 0 0

NFSv4

Criou-se os diretórios para partilhar
[root@lombe ~]# mkdir /local /local/geral /local/financas /local/recursos_humanos
/local/anuncios_refeitorio
[root@lombe ~]# mkdir /partilha /partilha/nfs

Criou-se os mesmos diretórios nas máquinas clientes de maneiras que os locais de montagem
sejam os mesmos.

Definiu-se os diretórios e as máquinas com as quais os diretórios serão partilhados
[root@lombe ~]# nano /etc/exports
/local/geral/ kwanza(rw,sync,no_root_squash) gazela(rw,sync,no_root_squash)
/local/financas/ kwanza(rw,sync,no_root_squash)
gazela(rw,sync,no_root_squash)
/local/recursos_humanos/ kwanza(rw,sync,no_root_squash)
gazela(rw,sync,no_root_squash)
/local/anuncios_refeitorio/ kwanza(rw,sync,no_root_squash)
gazela(rw,sync,no_root_squash)
/partilha/nfs/ kwanza(rw,sync,no_root_squash) gazela(rw,sync,no_root_squash)

Iniciou-se o daemon que permite que os clientes NFS descubram qual porta o servidor está
utilizando, iniciou-se o daemon NFS e activou-se os serviços para iniciarem ao arrancar a
máquina.
[root@lombe ~]# service rpcbind start
[root@lombe ~]# service nfs start
[root@lombe ~]# chkconfig rpcbind on
[root@lombe ~]# chkconfig nfs on

Para que os clientes NIS montem as pastas compartilhadas permanentemente editou-se o
ficheiro fstab nos clientes (kwanza e gazela):
[root@lombe ~]# nano /etc/fstab
lombe:/local/geral/ /local/geral/ nfs defaults 0 0
lombe:/local/financas / /local/financas/ nfs defaults 0 0
lombe:/local/recursos_humanos/ /local/recursos_humanos/ nfs defaults 0 0
lombe:/local/anuncios_refeitorio/ /local/anuncios_refeitorio/ nfs defaults 0 0
lombe:/partilha/nfs/ /partilha/nfs/ nfs defaults 0 0

Segurança

Para permitir apenas os membros do grupo “financas” possam fazer alterações nos ficheiros do
diretório /local/financas/, adicionou-se o grupo financas
[root@lombe ~]# groupadd financas

Definiu-se o grupo financas como proprietário do diretório /local/financas/
[root@lombe ~]# chgrp financas /local/financas
Mudou-se as permissões do diretório /local/financas/
[root@lombe ~]# chmod u=rx,g=rwx,o=rx /local/financas

Samba

Instalou-se os pacotes necessários para o samba
[root@lombe ~]# yum install -y samba samba-common samba-client

Criou-se o directório de partilha samba
[root@lombe ~]# mkdir -p /samba/share

Alterou-se as permissões do directório de partilha samba para que todo utilizador possa ler,
fazer alteração e executar dentro do directório
[root@lombe ~]# chmod 777 /samba/share

Habilitou-se o acesso aos diretórios home dos utilizadores samba no ficheiro de configuração:
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

Definir as passwords samba dos users adicionados
[root@lombe ~]# smbpasswd -a lombe

Para permitir que todos os utilizadores possam escrever num mounting point
[root@lombe ~]# setsebool -P samba_export_all on

De maneiras que o SI não bloqueie o acesso dos utilizadores ao samba, habilitou-se as
seguintes configurações. O “ -P ” significa permanente
[root@lombe ~]# setsebool -P use_samba_home_dirs on
[root@lombe ~]# setsebool -P samba_enable_home_dirs on

Iniciou-se o serviço e activou-se o serviço para iniciar ao arrancar a máquina
[root@lombe ~]# service smb start
[root@lombe ~]# chkconfig smb on

Configuração do cliente “gazela”

YUM/FTP

PARA REPOSITÓRIO DO CentOS 6
Montou-se a Drive do DVD onde estava a imagem do CentOS 6
[root@lombe ~]# mount /dev/cdrom /media

Foi-se para o directório onde estava montada a Drive do DVD com a imagem do RHEL6 e para o
directório Packages e instalou-se os seguintes pacotes
[root@lombe ~]# cd /media/CentOS_6.7_Final/Packages/
[root@lombe ~]# rpm -ivh vsftpd*

A pasta do repositório “/var/ftp/pub/repo” foi criada automaticamente após instalação do FTP
Editou-se o ficheiro que contém as configurações do YUM e alterou-se a linha do ftp para
“baseurl=ftp:/// kwanza/pub/repo”

FTP

Configurou-se da mesma forma que o FTP do lombe

NIS Cliente

Configurou-se da mesma forma que o NIS Slave do lombe, com a excecção do comando
“/usr/lib64/yp/ypinit –s”
