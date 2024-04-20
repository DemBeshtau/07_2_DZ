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
загружался пропатченный установщик GRUB grub2-install.el8, позволяющий устанавливать GRUB в специально подготовленную область на <br/>
на физическом разделе (PV). Далее на отдельном диске /dev/sdb разворачивалcя LVM, затем на него осуществлялся перенос текущей <br/>
системы,выполнялись действия по подготовке к загрузке с LVM, в частности формировался новый образ initramfs, осуществлялось <br/>
конфигурирование GRUB и с помощью grub2-install.el8 устанавливался загрузчик на LVM. 
&ensp;&ensp;После загрузки системы с LVM раздела, производилась подготовка раздела со старой системой /dev/sda1 к переформатированию <br/>
на LVM, в частности удалялся раздел на устройстве /dev/sda, на нём создавался LVM, осуществлялись действия по переносу системы обратно.
После переноса системы на /dev/sda, LVM на /dev/sdb был разобран.
1. Просмотр начальной конфигурации дисковой инфраструктуры:<br/>
```shell
[root@lvm ~]# lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda      8:0    0  10G  0 disk 
└─sda1   8:1    0  10G  0 part /
sdb      8:16   0  10G  0 disk 
```
2. Подготовка физического раздела (PV) с областью для загрузчика:
```shell
[root@lvm ~]# pvcreate --bootloaderareasize 1m /dev/sdb
  Physical volume "/dev/sdb" successfully created.

[root@lvm ~]# pvs
  PV         VG Fmt  Attr PSize  PFree 
  /dev/sdb      lvm2 ---  10.00g 10.00g
```
