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

| devices | uuid |
|---  |--- |
|/dev/sda| DDCC2201-7D60-4D7C-9F6C-F0AC95461FF6|
|/dev/sdb| 782D750C-766E-4A1E-997D-FE2DDB8A1F62| 
|/dev/sdc| 5609A76A-4FA0-4916-8710-0954CEAB32C7| 
|/dev/sdd| AF592D4F-D020-44B5-B55E-94E4F2209D8D| 

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
nmcli con add type ethernet con-name eth0-slave ifname eth0 master br0
```

```
nmcli con up br0
nmcli con up eth0-slave
```

```
ip a show br0
```

## Préparation du system

### Mise à jour

### Installation de KVM

### Partitionnement des disques ASM


Netoyer le disque:
```
parted -s /dev/sdb mklabel gpt
```

Créer les x partitions:
```
for i in $(seq 0 6); do
  start=$((i * 128))
  end=$(((i + 1) * 128))
  parted -s /dev/sdb mkpart primary ${start}GiB ${end}GiB
done
```

