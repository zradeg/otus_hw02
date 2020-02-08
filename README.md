# Дисковая подсистема
## Домашнее задание
### Уровень: Base

* добавить в Vagrantfile еще дисков
* собрать R0/R5/R10 на выбор
* прописать собранный рейд в конф, чтобы рейд собирался при загрузке
* сломать/починить raid
* создать GPT  раздел и 5 партиций и смонтировать их на диск.
В качестве проверки принимаются - измененный Vagrantfile, скрипт для 
создания рейда, конф для автосборки рейда при загрузке.
* Доп. задание - Vagrantfile, который сразу собирает систему с подключенным 
рейдом


## Рабочее окружение:
* Windows 10.0.18362
* vagrant 2.2.7
* Oracle VirtualBox 5.2.36 r135684
* packer 1.5.1
* github: https://github.com/zradeg

**Приглашение командной строки хостовой системы выглядит так:**

`$`

**Приглашение командной строки гостевой системы выглядит так:**

`vagrant@$`

## Подготовка
1. Клонирую Vagrantfile из репозитория https://github.com/erlong15/otus-linux
2. Добавляю в Vagrantfile описание еще одного (пятого) диска

>:sata5 => {
>        :dfile => './sata5.vdi',
>        :size => 250, # Megabytes
>        :port => 5 
>}

3. Просмотр наличия блочных устройств

`vagrant@$ sudo lsblk`

>NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
>sda      8:0    0   40G  0 disk
>`-sda1   8:1    0   40G  0 part /
>sdb      8:16   0  250M  0 disk
>sdc      8:32   0  250M  0 disk
>sdd      8:48   0  250M  0 disk
>sde      8:64   0  250M  0 disk
>sdf      8:80   0  250M  0 disk

Исходя из того, что дисков 6, самым оптимальным будет созать RAID6

4. Пробую занулить суперблоки на неразмеченных дисках:

`vagrant@$ sudo mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}`

> mdadm: Unrecognised md component device - /dev/sdb
> mdadm: Unrecognised md component device - /dev/sdc
> mdadm: Unrecognised md component device - /dev/sdd
> mdadm: Unrecognised md component device - /dev/sde
> mdadm: Unrecognised md component device - /dev/sdf

Ответ говорит о том, что ранее на этих дисках рейда не было

5. Создаю рейд

`vagrant@$ sudo mdadm --create --verbose /dev/md0 -l 6 -n 5 /dev/sd{b,c,d,e,f}`

>mdadm: layout defaults to left-symmetric
>mdadm: layout defaults to left-symmetric
>mdadm: chunk size defaults to 512K
>mdadm: size set to 253952K
>mdadm: Defaulting to version 1.2 metadata
>mdadm: array /dev/md0 started.

6. Проверяю результат

`vagrant@$ sudo /proc/mdstat`

>Personalities : [raid6] [raid5] [raid4]
>md0 : active raid6 sdf[4] sde[3] sdd[2] sdc[1] sdb[0]
>      761856 blocks super 1.2 level 6, 512k chunk, algorithm 2 [5/5] [UUUUU]
>
>unused devices: <none>

Убеждаюсь, что уровень рейда - 6 и все пять дисков включены - [5/5] [UUUUU], выясняю, что размер чанка = 512k

`vagrant@$ sudo mdadm -D /dev/md0`

>/dev/md0:
>           Version : 1.2
>     Creation Time : Sat Feb  8 17:40:31 2020
>        Raid Level : raid6
>        Array Size : 761856 (744.00 MiB 780.14 MB)
>     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
>      Raid Devices : 5
>     Total Devices : 5
>       Persistence : Superblock is persistent
>
>       Update Time : Sat Feb  8 17:41:28 2020
>             State : clean
>    Active Devices : 5
>   Working Devices : 5
>    Failed Devices : 0
>     Spare Devices : 0
>
>            Layout : left-symmetric
>        Chunk Size : 512K
>
>Consistency Policy : resync
>
>              Name : otuslinux:0  (local to host otuslinux)
>              UUID : e4041632:2c346a4d:a1555caa:1d1c4ad5
>            Events : 17
>
>    Number   Major   Minor   RaidDevice State
>       0       8       16        0      active sync   /dev/sdb
>       1       8       32        1      active sync   /dev/sdc
>       2       8       48        2      active sync   /dev/sdd
>       3       8       64        3      active sync   /dev/sde
>       4       8       80        4      active sync   /dev/sdf

7. Создаю файл mdadm.conf и проверяю его содержимое

`vagrant@$ sudo su`

`vagrant@# mkdir /etc/mdadm`

`vagrant@#  echo "DEVICE partitions" > /etc/mdadm/mdadm.conf`

`vagrant@#  mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf`

`vagrant@$ cat /etc/mdadm/mdadm.conf`

>DEVICE partitions
>ARRAY /dev/md0 level=raid6 num-devices=5 metadata=1.2 name=otuslinux:0 UUID=e4041632:2c346a4d:a1555caa:1d1c4ad5

8. Ломаю массив, искуственно переводя в статус "fail" один из дисков, проверяю статус массива:

`vagrant@# mdadm /dev/md0 --fail /dev/sdc`

>mdadm: set /dev/sdc faulty in /dev/md0

`vagrant@# cat /proc/mdstat`

>Personalities : [raid6] [raid5] [raid4]
>md0 : active raid6 sde[5] sdf[4] sdd[2] sdc[1](F) sdb[0]
>      761856 blocks super 1.2 level 6, 512k chunk, algorithm 2 [5/4] [U_UUU]
>
>unused devices: <none>

9. Восстанавливаю рейд, сначала удалив диск из массива, а затем "вставив" его же назад
 
 `vagrant@# mdadm --remove /dev/md0 /dev/sdc`
 
 > mdadm: hot removed /dev/sdc from /dev/md0
 
 `vagrant@# mdadm --add /dev/md0 /dev/sdc`
 
 > mdadm: added /dev/sdc
 
 Несмотря на несмотря на отсутствие данных, процесс rebuild проходил около минуты.
 
10. После того как перестроение массива завершилось, размонтирую /dev/md0 и создаю на нем gpt раздел, в разделе четыре партиции, создаю на них файловую систему, смотрю вывод lsblk и монтирую их по каталогам

`vagrant@# umount /dev/md0`

`vagrant@# parted -s /dev/md0 mklabel gpt`

`vagrant@# parted /dev/md0 mkpart primary ext4 0% 25%`

`vagrant@# parted /dev/md0 mkpart primary ext4 25% 50%`

`vagrant@# parted /dev/md0 mkpart primary ext4 50% 75%`

`vagrant@# parted /dev/md0 mkpart primary ext4 75% 100%`

`vagrant@# lsblk`

>NAME    MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
>md0       9:0    0   744M  0 raid6
>|-md0p1 259:0    0 184.5M  0 md
>|-md0p2 259:1    0   186M  0 md
>|-md0p3 259:2    0   186M  0 md
>`-md0p4 259:3    0 184.5M  0 md
>sda       8:0    0    40G  0 disk
>`-sda1    8:1    0    40G  0 part  /

11. Добавляю следующие строки в Vagrantfile в секцию box.vm.provision, делаю vagrant reload --provision, проверяю

>mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}
>mdadm --create --verbose /dev/md0 -l 6 -n 5 /dev/sd{b,c,d,e,f}
>mkdir /etc/mdadm
>echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
>mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf

>parted -s /dev/md0 mklabel gpt
>parted /dev/md0 mkpart primary ext4 0% 25%
>parted /dev/md0 mkpart primary ext4 25% 50%
>parted /dev/md0 mkpart primary ext4 50% 75%
>parted /dev/md0 mkpart primary ext4 75% 100%
>for i in $(seq 1 4); do sudo mkfs.ext4 /dev/md0p$i; done
>mkdir -p /raid/part{1,2,3,4}
>for i in $(seq 1 4); do mount /dev/md0p$i /raid/part$i; done

`vagrant@# cat /proc/mdstat`

>Personalities : [raid6] [raid5] [raid4]
>md0 : active raid6 sdf[4] sde[3] sdd[2] sdc[1] sdb[0]
>      761856 blocks super 1.2 level 6, 512k chunk, algorithm 2 [5/5] [UUUUU]
>
>unused devices: <none>

`vagrant@# lsblk`

>NAME      MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
>sda         8:0    0    40G  0 disk
>`-sda1      8:1    0    40G  0 part  /
>sdb         8:16   0   250M  0 disk
>`-md0       9:0    0   744M  0 raid6
>  |-md0p1 259:0    0 184.5M  0 md    /raid/part1
>  |-md0p2 259:1    0   186M  0 md    /raid/part2
>  |-md0p3 259:2    0   186M  0 md    /raid/part3
>  `-md0p4 259:3    0 184.5M  0 md    /raid/part4
>sdc         8:32   0   250M  0 disk
>`-md0       9:0    0   744M  0 raid6
>  |-md0p1 259:0    0 184.5M  0 md    /raid/part1
>  |-md0p2 259:1    0   186M  0 md    /raid/part2
>  |-md0p3 259:2    0   186M  0 md    /raid/part3
>  `-md0p4 259:3    0 184.5M  0 md    /raid/part4
>sdd         8:48   0   250M  0 disk
>`-md0       9:0    0   744M  0 raid6
>  |-md0p1 259:0    0 184.5M  0 md    /raid/part1
>  |-md0p2 259:1    0   186M  0 md    /raid/part2
>  |-md0p3 259:2    0   186M  0 md    /raid/part3
>  `-md0p4 259:3    0 184.5M  0 md    /raid/part4
>sde         8:64   0   250M  0 disk
>`-md0       9:0    0   744M  0 raid6
>  |-md0p1 259:0    0 184.5M  0 md    /raid/part1
>  |-md0p2 259:1    0   186M  0 md    /raid/part2
>  |-md0p3 259:2    0   186M  0 md    /raid/part3
>  `-md0p4 259:3    0 184.5M  0 md    /raid/part4
>sdf         8:80   0   250M  0 disk
>`-md0       9:0    0   744M  0 raid6
>  |-md0p1 259:0    0 184.5M  0 md    /raid/part1
>  |-md0p2 259:1    0   186M  0 md    /raid/part2
>  |-md0p3 259:2    0   186M  0 md    /raid/part3
>  `-md0p4 259:3    0 184.5M  0 md    /raid/part4

12. Итого отработано:
* создание массива
* поломка и последующее восстановление массива
* массив разбит на разделы
* обе процедуры автоматизированы через Vagrantfile