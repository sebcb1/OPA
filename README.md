# Oracle Personal Appliance

## Materiel

- i3-2130 2 core de 3,40Ghz
- 14G de ram
- 4 DD de 3To

## Clone du projet github

```
cd projets
ssh-keygen -f id_rsa.opa
git clone git@github.com:sebcb1/OPA.git
cd OPA
git config --global core.sshCommand "ssh -i /Users/sebastien/projets/id_rsa.opa"
git config --global user.email "sebastienbrillard@icloud.com"
git config --global user.name  "Sébastien Brillard"
git config --global push.default simple
```

## Installation de l'OS

- Oracle Linux 8.10.0
	- Language: English
	- Base env: Server with GUI
	- Kdump: disable
	- Raid 1 logiciel entre sda et sdb
	- Partitions:
		- /boot, 2Gb, xfs, raid 1 entre sda et sdb
		- /boot/efi, 2Gb, efi partition sur sda
		- swap, 10Gb sur sda et sdb
		- /, 128Gb, xfs, raid 1 entre sda et sdb
		- /u01, 128Gb, xfs, raid 1 entre sda et sdb
		- /VMs, 2Tb, xfs, raid 1 entre sda et sdb
		- hostname: opa1.myhome
	- eth0 en mode dhcp (configuration par la suite) et ipv6 disabled

## Dupliquer /boot/efi

Pour permettre au system de booter si on perd l'un des deux disques système il faut cloner /boot/efi sur le 2e disques

Après l'installation:
```
[root@opa1 ~]# df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        6.8G     0  6.8G   0% /dev
tmpfs           6.8G     0  6.8G   0% /dev/shm
tmpfs           6.8G  9.7M  6.8G   1% /run
tmpfs           6.8G     0  6.8G   0% /sys/fs/cgroup
/dev/md126      120G  6.7G  113G   6% /
/dev/md125      120G  883M  119G   1% /u01
/dev/md127      1.9G  332M  1.6G  18% /boot
/dev/sdb1       1.9G  6.0M  1.9G   1% /boot/efi
/dev/md124      1.9T   14G  1.9T   1% /VMs
tmpfs           1.4G   60K  1.4G   1% /run/user/1000
```

/dev/sdb en détail:
```
[root@opa1 ~]# fdisk /dev/sdb -l
Disk /dev/sdb: 2.7 TiB, 3000592982016 bytes, 5860533168 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: DDCC2201-7D60-4D7C-9F6C-F0AC95461FF6

Device          Start        End    Sectors   Size Type
/dev/sdb1        2048  250132479  250130432 119.3G Linux RAID
/dev/sdb2   250132480  500262911  250130432 119.3G Linux RAID
/dev/sdb3   500262912  519792639   19529728   9.3G Linux swap
/dev/sdb4   519792640  523700223    3907584   1.9G Linux RAID
/dev/sdb5   523700224 4430211071 3906510848   1.8T Linux RAID
/dev/sdb6  4430211072 4434116607    3905536   1.9G EFI System
```

La partition efi fait 3905536 sectors, on va créer sur /dev/sda une partition de 3905536 sectors:
```
parted /dev/sda --script mkpart ESP fat32 4430211072s 4434116607s
parted /dev/sda --script set 6 esp on
```

On clone:
```
dd if=/dev/sdb1 of=/dev/sda6 bs=1M status=progress conv=fsync
```

On vérifie kle boot EFI:
```
[root@opa1 ~]# efibootmgr -v
BootCurrent: 0001
Timeout: 2 seconds
BootOrder: 0001
Boot0001* Oracle Linux	HD(1,GPT,b9285e39-ce94-4408-9b1e-f6e0bfe3ffbb,0x800,0x3b9800)/File(\EFI\redhat\shimx64.efi)
```

On ajoute sda:
```
efibootmgr -c -d /dev/sda -p 6 -L "Oracle Linux EFI (Backup)" -l '\EFI\redhat\shimx64.efi'
```

```
[root@opa1 ~]# efibootmgr -v
BootCurrent: 0001
Timeout: 2 seconds
BootOrder: 0000,0001
Boot0000* Oracle Linux EFI (Backup)	HD(6,GPT,d36f40b8-7f5f-49ac-bf0b-8dad61da86b9,0x1080fa800,0x3b9800)/File(\EFI\redhat\shimx64.efi)
Boot0001* Oracle Linux	HD(1,GPT,b9285e39-ce94-4408-9b1e-f6e0bfe3ffbb,0x800,0x3b9800)/File(\EFI\redhat\shimx64.efi)
```

Maintenant le systèm peut démarrer sur l'un ou l'autre !

## Gestion des disques

Les disques:

| devices | etiquette| uuid disk
|---  |--- |--- |
|/dev/sda| F6 |DDCC2201-7D60-4D7C-9F6C-F0AC95461FF6|
|/dev/sdb| 62 |782D750C-766E-4A1E-997D-FE2DDB8A1F62|
|/dev/sdc| C7 |A55FE146-D319-4D72-91B9-B6741E09AFFA|
|/dev/sdd| 8D |C42F57A1-F420-46DA-BE1C-B51756C032AA| 

### Verifier le miroir

```
[root@opa1 ~]# cat /proc/mdstat
Personalities : [raid1] 
md124 : active raid1 sda6[0]
      1953123328 blocks super 1.2 [2/1] [U_]
      bitmap: 13/15 pages [52KB], 65536KB chunk

md125 : active raid1 sda3[0]
      124998656 blocks super 1.2 [2/1] [U_]
      bitmap: 1/1 pages [4KB], 65536KB chunk

md126 : active raid1 sda2[0]
      124998656 blocks super 1.2 [2/1] [U_]
      bitmap: 1/1 pages [4KB], 65536KB chunk

md127 : active raid1 sda5[0]
      1951744 blocks super 1.2 [2/1] [U_]
      bitmap: 1/1 pages [4KB], 65536KB chunk
```
```
[root@opa1 ~]# mdadm --detail /dev/md127
/dev/md127:
           Version : 1.2
     Creation Time : Thu Jun 19 08:13:12 2025
        Raid Level : raid1
        Array Size : 1951744 (1906.00 MiB 1998.59 MB)
     Used Dev Size : 1951744 (1906.00 MiB 1998.59 MB)
      Raid Devices : 2
     Total Devices : 1
       Persistence : Superblock is persistent

     Intent Bitmap : Internal

       Update Time : Thu Jun 19 09:53:43 2025
             State : clean, degraded 
    Active Devices : 1
   Working Devices : 1
    Failed Devices : 0
     Spare Devices : 0

Consistency Policy : bitmap

              Name : opa1.myhome:boot  (local to host opa1.myhome)
              UUID : 6f2f49ae:f9e52bcd:c2023d09:74892e7e
            Events : 154

    Number   Major   Minor   RaidDevice State
       0       8        5        0      active sync   /dev/sda5
       -       0        0        1      removed
```

### Resync le miroir

Status d'une grappe raid:
```
[root@opa1 ~]# cat /proc/mdstat
Personalities : [raid1] 
md124 : active raid1 sda6[0]
      1953123328 blocks super 1.2 [2/1] [U_]
      bitmap: 14/15 pages [56KB], 65536KB chunk
```

Liste des partitions raid:
```
[root@opa1 ~]# lsblk -f | grep raid
├─sda2    linux_raid_member opa1.myhome:root bf6df99b-9415-7a91-d412-5b2910b9ff7c 
├─sda3    linux_raid_member opa1.myhome:u01  a686c22b-ed9e-3c02-4ea1-f0578ef7002d 
├─sda5    linux_raid_member opa1.myhome:boot 6f2f49ae-f9e5-2bcd-c202-3d0974892e7e 
└─sda6    linux_raid_member opa1.myhome:VMs  ed75db96-d5bf-95b2-589f-a80f8cfa6c14 
├─sdb1    linux_raid_member opa1.myhome:root bf6df99b-9415-7a91-d412-5b2910b9ff7c 
├─sdb2    linux_raid_member opa1.myhome:u01  a686c22b-ed9e-3c02-4ea1-f0578ef7002d 
├─sdb4    linux_raid_member opa1.myhome:boot 6f2f49ae-f9e5-2bcd-c202-3d0974892e7e 
├─sdb5    linux_raid_member opa1.myhome:VMs  ed75db96-d5bf-95b2-589f-a80f8cfa6c14 
```

Ajout:
```
mdadm --add /dev/md124 /dev/sdb5
```

Surveillance de la sync:
```
watch cat /proc/mdstat
```

## Configuration du réseaux

Pour pouvoir accrocher des VMs sur le résaux local il faut reconfigurer l'interface réseaux eth0 de la machine physique en mode bridge.
Il n'y pas d'usage de VLAN içi.

```
nmcli con add type bridge con-name br0 ifname br0
nmcli con modify br0 ipv4.addresses 192.168.1.50/24
nmcli con modify br0 ipv4.gateway 192.168.1.254
nmcli con modify br0 ipv4.dns "192.168.1.254"
nmcli con modify br0 ipv4.method manual
nmcli con modify br0 ipv6.method ignore
nmcli con add type ethernet con-name enp2s0-slave ifname enp2s0 master br0
```

```
nmcli con delete enp2s0
```

```
nmcli con up br0
nmcli con up enp2s0-slave
```

```
ip a show br0
```

## Préparation du system

### Upgrade du système

Upgrade OS:
```
yum update -y
```

Resync de /boot/efi (exemple si /boot/efi est actif sur /dev/sda1 et /dev/sdb6 est le secours)
```
dd if=/dev/sda1 of=/dev/sdb6 bs=1M status=progress conv=fsync
```

### Installation de KVM

### Partitionnement des disques ASM

Les disques ASM sont /dev/sdc et /dev/sdd dans cette exemple.

Netoyer le disque:
```
parted -s /dev/sdc mklabel gpt
parted -s /dev/sdd mklabel gpt
```

Créer les 21 partitions sur sdc:
```
for i in $(seq 0 20); do
  start=$((i * 128 + 35))
  end=$(((i + 1) * 128 + 34))
  echo "parted -s /dev/sdc mkpart primary ${start}GiB ${end}GiB"
  parted -s /dev/sdc mkpart primary ${start}GiB ${end}GiB
done
```

Créer les 21 partitions sur sdd:
```
for i in $(seq 0 20); do
  start=$((i * 128 + 34))
  end=$(((i + 1) * 128 + 34))
  parted -s /dev/sdd mkpart primary ${start}GiB ${end}GiB
done
```

## Fichiers nécessaire

A déposer dans /tmp:
- oracleasmlib-3.1.0-6.el8.x86_64.rpm
- LINUX.X64_193000_grid_home.zip  

## Prerequis Oracle

```
yum install -y oracle-database-preinstall-19c 
yum install -y java-1.8.0-openjdk
yum install -y ksh libaio-devel.x86_64
dnf install -y oraclelinux-release-el8
dnf config-manager --enable ol8_addons
yum install oracleasm-support
yum install kmod-oracleasm
```
```
rpm -i /tmp/oracleasmlib-3.1.0-6.el8.x86_64.rpm
```

## Préparation du system

```
groupadd asmadmin
groupadd oinstall
groupadd asmdba
useradd -g oinstall -G asmadmin,asmdba grid
```

```
mkdir -p /u01/app/grid
mkdir -p /u01/app/19/grid
chown -R grid:oinstall /u01
chmod -R 775 /u01
```

```
vi /etc/sysctl.d/97-oracle-database-sysctl.conf
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 2097152
kernel.shmmax = 4294967295
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
```
```
/sbin/sysctl --system
```
```
[root@opa1 ~]# cat /etc/selinux/config|grep SELINUX=
# SELINUX= can take one of these three values:
SELINUX=disabled
```
```
setenforce 0
```

```
su - grid
vi .bash_profile
# .bash_profile
# Get the aliases and functions
if [ -f ~/.bashrc ]; then
. ~/.bashrc
fi
ORACLE_SID=+ASM; export ORACLE_SID
ORACLE_BASE=/u01/app/grid; export ORACLE_BASE
ORACLE_HOME=/u01/app/19/grid; export ORACLE_HOME
ORACLE_TERM=xterm; export ORACLE_TERM
TNS_ADMIN=$ORACLE_HOME/network/admin; export TNS_ADMIN
PATH=.:${JAVA_HOME}/bin:${PATH}:$HOME/bin:$ORACLE_HOME/bin
PATH=${PATH}:/usr/bin:/bin:/usr/local/bin
export PATH
export TEMP=/tmp
export TMPDIR=/tmp
umask 022
```

## Configuration d'asmlib

```
[root@opa1 tmp]# oracleasm configure -i
Configuring the Oracle ASM system service.

This will configure the on-boot properties of the Oracle ASM system
service.  The following questions will determine whether the service
is started on boot and what permissions it will have.  The current
values will be shown in brackets ('[]').  Hitting <ENTER> without
typing an answer will keep that current value.  Ctrl-C will abort.

Default user to own the ASM disk devices []: grid
Default group to own the ASM disk devices []: oinstall
Start Oracle ASM system service on boot (y/n) [y]: y
Scan for Oracle ASM disks when starting the oracleasm service (y/n) [y]: y
Maximum number of ASM disks that can be used on system [2048]: 
Enable iofilter if kernel supports it (y/n) [y]: y
Writing Oracle ASM system service configuration: done

Configuration changes only come into effect after the Oracle ASM
system service is restarted.  Please run 'systemctl restart oracleasm'
after making changes.

WARNING: All of your Oracle and ASM instances must be stopped prior
to restarting the oracleasm service.
```
```
systemctl start oracleasm.service
```

## Creation des disques ASM

```
for i in $(seq 1 21); do
  oracleasm createdisk ASMDISK1P${i} /dev/sdc${i}
done
```
```
for i in $(seq 1 21); do
  oracleasm createdisk ASMDISK2P${i} /dev/sdd${i}
done
```
```
oracleasm listdisks
```

## Installation de OraGrid

```
su - grid
cd /tmp
unzip LINUX.X64_193000_grid_home.zip -d $ORACLE_HOME
```
```
su -
cd  /u01/app/19/grid/cv/rpm/
CVUQDISK_GRP=oinstall; export CVUQDISK_GRP
rpm -iv cvuqdisk-1.0.10-1.rpm
```
```
su - grid
cd $ORACLE_HOME
export CV_ASSUME_DISTID=OL8.10
./gridSetup.sh
```

Les options:
- Configure Oracle Grid Infrastructure for a standalone Server
- Disk group name: DATA
- Redundancy: Normal
- Choisir les disks: scd1 et sdd1
- password: changeme
- group OSASM: asmadmin
- group OSDBA: asmdba
- ORACLE_BASE /u01/app/grid
- inv directory /u01/app/oraIventory

Depuis une session root quand demandé:
```
/u01/app/oraInventory/orainstRoot.sh
/u01/app/19/grid/root.sh
```

## Suppression du grid si besoin

Sous root:
```
/u01/app/19/grid/crs/install/roothas.sh -deconfig -force -verbose
rm -rf /u01/app/grid
rm -rf /u01/app/19/grid/*
rm -rf /u01/app/19/grid/.*
rm -rf /u01/app/oraInventory
rm -f /etc/systemd/system/oracle-ohasd.service
chown grid.oinstall /u01/app/19/grid
systemctl daemon-reexec
systemctl daemon-reload
```

Netoie les disques:
```
wipefs -a /dev/sdc
sgdisk --zap-all /dev/sdc
dd if=/dev/zero of=/dev/sdc bs=1M count=10
partprobe /dev/sdc

wipefs -a /dev/sdd
sgdisk --zap-all /dev/sdd
dd if=/dev/zero of=/dev/sdd bs=1M count=10
partprobe /dev/sdc
```

```
oracleasm scandisks
oracleasm listdisks
```


