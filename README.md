# Работа с загрузчиком #
&ensp;&ensp;Сконфигурировать систему c boot-разделом на LVM. Репозиторий с пропатченым grub:<br/>
https://github.com/thedolphin/grub2-lvm-whole-disk/tree/master/grub-install.
### Исходные данные ###
&ensp;&ensp;ПК на Linux c 8 ГБ ОЗУ или виртуальная машина (ВМ) с включенной Nested Virtualization.<br/>
&ensp;&ensp;Предварительно установленное и настроенное ПО:<br/>
&ensp;&ensp;&ensp;Hashicorp Vagrant (https://www.vagrantup.com/downloads);<br/>
&ensp;&ensp;&ensp;Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads).<br/>
&ensp;&ensp;&ensp;Все действия проводились с использованием Vagrant 2.4.0, VirtualBox 7.0.14 и образа<br/> 
&ensp;&ensp;CentOS 8 версии 20201204.2
### Ход решения ###
&ensp;&ensp;Для выполнения задания с репозитария https://github.com/thedolphin/grub2-lvm-whole-disk/tree/master/grub-install<br/>
загружался пропатченный установщик GRUB grub2-install.el8, позволяющий устанавливать GRUB в специально подготовленную<br/>
область на физическом разделе (PV). Далее на отдельном диске /dev/sdb разворачивалcя LVM, затем на него осуществлялся<br/> 
перенос текущей системы,выполнялись действия по подготовке к загрузке с LVM, в частности формировался новый образ<br/>
initramfs, осуществлялось конфигурирование GRUB и с помощью grub2-install.el8 устанавливался загрузчик на LVM.<br/> 
&ensp;&ensp;После загрузки системы с LVM раздела, производилась подготовка раздела со старой системой /dev/sda1 к переформатированию<br/>
на LVM, в частности удалялся раздел на устройстве /dev/sda, на нём создавался LVM, осуществлялись действия по переносу системы обратно.
После переноса системы на /dev/sda, LVM на /dev/sdb был разобран.<br/>
1. Просмотр начальной конфигурации дисковой инфраструктуры:<br/>
```shell
[root@lvm ~]# lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda      8:0    0  10G  0 disk 
└─sda1   8:1    0  10G  0 part /
sdb      8:16   0  10G  0 disk 
```
2. Подготовка физического раздела (PV) с областью для загрузчика:<br/>
```shell
[root@lvm ~]# pvcreate --bootloaderareasize 1m /dev/sdb
  Physical volume "/dev/sdb" successfully created.

[root@lvm ~]# pvs
  PV         VG Fmt  Attr PSize  PFree 
  /dev/sdb      lvm2 ---  10.00g 10.00g
```
3. Подготовка группы разделов (VG) и логического раздела (LV):<br/>
```shell
[root@lvm ~]# vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created

[root@lvm ~]# vgs
  VG      #PV #LV #SN Attr   VSize   VFree  
  vg_root   1   0   0 wz--n- <10.00g <10.00g

[root@lvm ~]# lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.

[root@lvm ~]# lvs
  LV      VG      Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_root vg_root -wi-a----- <10.00g 
```
4. Размещение файловой системы XFS на LVM /dev/vg_root/lv_root:<br/>
```shell
[root@lvm ~]# mkfs.xfs /dev/vg_root/lv_root 
meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

[root@lvm ~]# parted /dev/vg_root/lv_root print
Model: Linux device-mapper (linear) (dm)
Disk /dev/dm-0: 10.7GB
Sector size (logical/physical): 512B/512B
Partition Table: loop
Disk Flags: 

Number  Start  End     Size    File system  Flags
 1      0.00B  10.7GB  10.7GB  xfs
```
5. Копирование утилиты grub2-install.el8 в место назначание /usr/sbin:<br/>
```shell
root@lvm ~]# cd /vagrant/
[root@lvm vagrant]# ll
total 1452
-rw-r--r--. 1 vagrant vagrant 1482480 Apr 19 18:54 grub2-install.el8
-rw-r--r--. 1 vagrant vagrant    2077 Apr 20 09:10 Vagrantfile

[root@lvm vagrant]# cp grub2-install.el8 /usr/sbin/

[root@lvm vagrant]# cd /usr/sbin

[root@lvm sbin]# ll | grep grub
-rwxr-xr-x. 1 root root 1188968 Sep  8  2020 grub2-bios-setup
-rwxr-xr-x. 1 root root    2398 Sep  8  2020 grubK2-get-kernel-settings
-rwxr-xr-x. 1 root root 1482424 Sep  8  2020 grub2-install
-rw-r--r--. 1 root root 1482480 Apr 20 09:24 grub2-install.el8
-rwxr-xr-x. 1 root root    8835 Sep  8  2020 grub2-mkconfig
-rwxr-xr-x. 1 root root  243744 Sep  8  2020 grub2-ofpathname
-rwxr-xr-x. 1 root root 1189112 Sep  8  2020 grub2-probe
-rwxr-xr-x. 1 root root    4085 Sep  8  2020 grub2-reboot
-rwxr-xr-x. 1 root root  281512 Sep  8  2020 grub2-rpm-sort
-rwsr-xr-x. 1 root root   12016 Sep  8  2020 grubK2-set-bootflag
-rwxr-xr-x. 1 root root    3535 Sep  8  2020 grub2-set-default
-rwxr-xr-x. 1 root root    3119 Sep  8  2020 grub2-set-password
lrwxrwxrwx. 1 root root      18 Sep  8  2020 grub2-setpassword -> grub2-set-password
-rwxr-xr-x. 1 root root 1193400 Sep  8  2020 grub2-sparc64-setup
-rwxr-xr-x. 1 root root    8694 Sep  8  2020 grub2-switch-to-blscfg
-rwxr-xr-x. 1 root root     260 May 16  2020 grubby

[root@lvm sbin]# chmod a+x grub2-install.el8 

[root@lvm sbin]# ll | grep grub
-rwxr-xr-x. 1 root root 1188968 Sep  8  2020 grub2-bios-setup
-rwxr-xr-x. 1 root root    2398 Sep  8  2020 grub2-get-kernel-settings
-rwxr-xr-x. 1 root root 1482424 Sep  8  2020 grub2-install
-rwxr-xr-x. 1 root root 1482480 Apr 20 09:24 grub2-install.el8
-rwxr-xr-x. 1 root root    8835 Sep  8  2020 grub2-mkconfig
-rwxr-xr-x. 1 root root  243744 Sep  8  2020 grub2-ofpathname
-rwxr-xr-x. 1 root root 1189112 Sep  8  2020 grub2-probe
-rwxr-xr-x. 1 root root    4085 Sep  8  2020 grub2-reboot
-rwxr-xr-x. 1 root root  281512 Sep  8  2020 grub2-rpm-sort
-rwsr-xr-x. 1 root root   12016 Sep  8  2020 grub2-set-bootflag
-rwxr-xr-x. 1 root root    3535 Sep  8  2020 grub2-set-default
-rwxr-xr-x. 1 root root    3119 Sep  8  2020 grub2-set-password
lrwxrwxrwx. 1 root root      18 Sep  8  2020 grub2-setpassword -> grub2-set-password
-rwxr-xr-x. 1 root root 1193400 Sep  8  2020 grub2-sparc64-setup
-rwxr-xr-x. 1 root root    8694 Sep  8  2020 grub2-switch-to-blscfg
-rwxr-xr-x. 1 root root     260 May 16  2020 grubby
```
6. Монтирование LVM и перенос на него исходной системы:<br/>
```shell
[root@lvm ~]# mount /dev/vg_root/lv_root /mnt

[root@lvm ~]# xfsdump -J - /dev/sda1 | xfsrestore -J - /mnt
...
xfsrestore: Restore Status: SUCCESS
```
7. Создание окружения для перехода в новую систему:<br/>
```shell
[root@lvm ~]# mount --bind /dev /mnt/dev
[root@lvm ~]# mount --bind /sys /mnt/sys
[root@lvm ~]# mount --bind /run /mnt/run
[root@lvm ~]# mount --bind /proc /mnt/proc
[root@lvm ~]# chroot /mnt
```
8. Подготовка образа initramfs, конфигурирование GRUB и установка загрузчика на LVM:<br/>
```shell
[root@lvm /]# cd /boot

[root@lvm boot]# ll
total 43692
-rw-r--r--. 1 root root   189500 Nov 19  2020 config-4.18.0-240.1.1.el8_3.x86_64
drwxr-xr-x. 3 root root       17 Dec  4  2020 efi
drwx------. 4 root root       83 Apr 20 09:16 grub2
-rw-------. 1 root root 30995238 Dec  4  2020 initramfs-4.18.0-240.1.1.el8_3.x86_64.img
drwxr-xr-x. 3 root root       21 Dec  4  2020 loader
-rw-------. 1 root root  4032815 Nov 19  2020 System.map-4.18.0-240.1.1.el8_3.x86_64
-rwxr-xr-x. 1 root root  9514120 Nov 19  2020 vmlinuz-4.18.0-240.1.1.el8_3.x86_64

[root@lvm boot]# mv initramfs-4.18.0-240.1.1.el8_3.x86_64.img initramfs-4.18.0-240.1.1.el8_3.x86_64.img.old

[root@lvm boot]# dracut initramfs-`uname -r`.img

[root@lvm boot]# ll
total 74196
-rw-r--r--. 1 root root   189500 Nov 19  2020 config-4.18.0-240.1.1.el8_3.x86_64
drwxr-xr-x. 3 root root       17 Dec  4  2020 efi
drwx------. 4 root root       83 Apr 20 09:16 grub2
-rw-------. 1 root root 31235234 Apr 20 09:21 initramfs-4.18.0-240.1.1.el8_3.x86_64.img
-rw-------. 1 root root 30995238 Dec  4  2020 initramfs-4.18.0-240.1.1.el8_3.x86_64.img.old
drwxr-xr-x. 3 root root       21 Dec  4  2020 loader
-rw-------. 1 root root  4032815 Nov 19  2020 System.map-4.18.0-240.1.1.el8_3.x86_64
-rwxr-xr-x. 1 root root  9514120 Nov 19  2020 vmlinuz-4.18.0-240.1.1.el8_3.x86_64

[root@lvm boot]# cd grub2/

[root@lvm grub2]# ll
total 28
-rw-r--r--. 1 root root   64 Dec  4  2020 device.map
drwxr-xr-x. 2 root root   25 Dec  4  2020 fonts
-rw-r--r--. 1 root root 6457 Dec  4  2020 grub.cfg
-rw-------. 1 root root 1024 Apr 20 09:16 grubenv
drwxr-xr-x. 2 root root 8192 Dec  4  2020 i386-pc

[root@lvm grub2]# mv grub.cfg grub.cfg.old

[root@lvm grub2]# grub2-mkconfig -o grub.cfg
Generating grub configuration file ...
device-mapper: reload ioctl on osprober-linux-sda1 (253:1) failed: Device or resource busy
Command failed.
done

[root@lvm grub2]# cat grub.cfg
...
  set kernelopts="root=/dev/mapper/vg_root-lv_root ro no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop "
...
[root@lvm grub2]# grub2-install.el8 /dev/sdb
Installing for i386-pc platform.
Installation finished. No error reported.
```
9. Выход из конфигурируемой системы и перезагрузка:<br/>
```shell
[root@lvm grub2]# exit
exit

[root@lvm ~]# reboot
```
10. Начало работы с новой системой:<br/>
```shell
dem@calculate ~/vagrant/07_2_DZ $ vagrant ssh
Last login: Sat Apr 20 09:13:54 2024 from 10.0.2.2

[vagrant@lvm ~]$ sudo -i
[root@lvm ~]# lsblk
NAME              MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda                 8:0    0  10G  0 disk 
└─sda1              8:1    0  10G  0 part 
sdb                 8:16   0  10G  0 disk 
└─vg_root-lv_root 253:0    0  10G  0 lvm  /
```
11. Удаление раздела на диске /dev/sda:<br/>
```shell
[root@lvm ~]# fdisk /dev/sda

Welcome to fdisk (util-linux 2.32.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): d
Selected partition 1
Partition 1 has been deleted.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

[root@lvm ~]# lsblk
NAME              MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda                 8:0    0  10G  0 disk 
sdb                 8:16   0  10G  0 disk 
└─vg_root-lv_root 253:0    0  10G  0 lvm  /
```
12. Создание LMV на диске /dev/sda:<br/>
```shell
[root@lvm ~]# pvcreate --bootloaderareasize 1m /dev/sda
WARNING: dos signature detected on /dev/sda at offset 510. Wipe it? [y/n]: y
  Wiping dos signature on /dev/sda.
  Physical volume "/dev/sda" successfully created.

[root@lvm ~]# vgcreate vgr /dev/sda
  Volume group "vgr" successfully created

[root@lvm ~]# lvcreate -n lvr -l +100%FREE /dev/vgr
  Logical volume "lvr" created.

[root@lvm ~]# lvs
  LV      VG      Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_root vg_root -wi-ao---- <10.00g                                                    
  lvr     vgr     -wi-a----- <10.00g                          
```
13. 
