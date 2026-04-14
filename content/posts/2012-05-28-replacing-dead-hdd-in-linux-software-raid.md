---
title: Замена диска в Linux Software Raid
author: ["Stanislav"]
date: 2012-05-28T18:15:09+00:00
url: /2012/05/replacing-dead-hdd-in-linux-software-raid/
geo_latitude:
  - 59.910179
geo_longitude:
  - 30.310190
geo_public:
  - 1
categories:
  - Tech
tags:
  - linux
  - mdadm
  - raid

---
Жил-был диск, который исправно трудился в Linux software raid, но, как и все хорошее в этой жизни, он внезапно кончился. Пусть его звали /dev/sda.

Для начала удалим мертвеца из md0 и md1:

```mdadm /dev/md0 -r /dev/sda  
mdadm /dev/md1 -r /dev/sda```

Выключаем сервер, меняем диск на новый, загружаемся с второго (у вас же все впрорядке с grub и mbr? kekeke).  
<!--more-->

  
Убеждаемся в том, что новый диск девственно чист (ату, его fdisk, ату).

Копируем схему разделов с старого диска:  
```sfdisk -d /dev/sdb > part  
sfdisk /dev/sda < part```

И собираем рейд в зад:  
```mdadm /dev/md0 -a /dev/sda1  
mdadm /dev/md1 -a /dev/sda2```

Медитируем, следя за процессом:  
```localhost:~# cat /proc/mdstat  
Personalities : [raid1]  
md2 : active raid1 sdc[0] sdd[1]  
1465138496 blocks [2/2] [UU]

md1 : active raid1 sda2[2] sdb2[1]  
488255424 blocks [2/1] [_U]  
[=================>&#8230;] recovery = 86.9% (424484736/488255424) finish=22.9min speed=46354K/sec

md0 : active raid1 sda1[0] sdb1[1]  
128384 blocks [2/2] [UU]

```  
Не лишним будет записать mbr на sda:  
```dd if=/dev/sdb of=/dev/sda bs=512 count=1```  
и записать grub на каждый диск:  
```localhost:~# grub  
grub> root (hd0,0)  
grub> setup (hd0)  
grub> root (hd1,0)  
grub> setup (hd1)```