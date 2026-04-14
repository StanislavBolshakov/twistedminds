---
title: Настройка NetApp FAS – Часть 2
author: ["Stanislav"]
date: 2012-06-21T11:24:56+00:00
url: /2012/06/setting-up-netapp-fas-filer-part-2/
categories:
  - Tech
tags:
  - catalyst
  - cisco
  - fas2040
  - flowcontrol
  - ip aliasing
  - load-balancing
  - mtu
  - multimode vif
  - netapp
  - partner interface
---
![](/wp-content/uploads/2012/06/netapp-logo1-e1340202341425.png "netapp logo")

##### Часть вторая настройки NetApp FAS (Multimode VIF и его виды, балансировка нагрузки, интерфейсы-партнер в кластере, псевдонимы aka IP Aliasing, flowcontrol и MTU, настройка Static и Dynamic VIF с  коммутатором Cisco Catalyst).

[Часть первая >>>](/2012/06/setting-up-fas2040-part-1/ "Настройка NetApp FAS – Часть 1")
[Часть третья >>>](/2012/06/setting-up-netapp-filer-part-3/ "Настройка NetApp FAS – Часть 3")
[Часть четвертая >>>](/2012/06/setting-up-netapp-filer-part-4/ "Настройка NetApp FAS – Часть 4")
[Часть пятая >>>](/2012/06/setting-up-netapp-fas-filer-part-5/ "Настройка NetApp FAS – Часть 5")

Оттолкнемся от того, что все 4 порта каждой из голов файлера подключены к управляемому коммутатору Cisco и перед нами стоит задача объединить все это дело в виртуальный линк таким образом, чтобы IP Source-Destionation load-balancing работал эффективно. Стоит так же отметить, что для построения и понимания работы высокопроизводительных ethernet фабрик опыт работы с IP сетями оказался совершенно не лишним.

Для начала определимся с терминами. Multimode VIF – так у NetApp называется то, что у нормальных людей называется Etherchannel или Portchannel – способ объединения (агрегирования) физических интерфейсов в виртуальный, который занимается балансировкой нагрузки и обеспечивает отказоустойчивость.

###### Multimode VIF бывает двух видов – Static Multimode VIF и Dynamic Multimode VIF.

Static Multimode VIF представляет собой обычную группировку физических интерфейсов в один логический без какого-либо согласования, обнаружения и обмена сообщениями о недоступности пути. Аналог команды на коммутаторах Cisco **channel-group [номер] mode on**.

Dynamic Multimode VIF фактически является динамической агрегацией с использованием протокола 802.3ad LACP (LACP Etherchanhel в терминологии Cisco –  **channel-group [номер] mode active**). Единственное отличие Static от Dynamic состоит в том, что LACP обменивается PDU (Protocol Data Units), которые в состоянии рассказать партнеру, что порт на другой стороне был убран из группы. Так же LACP может быть полезен если между файлером и свичем существует какое-либо “тупое” устройства типа медиаконвертера. При упавшем в таком случае оптическом линке LACP больше не будет отправлять через этот интерфейс трафик. Static Multimode VIF, понятное дело, такой мониторинг не проводит.

![преимущество lacp](/wp-content/uploads/2012/06/lacp-advantage1.png "lacp advantage")

Так же в этих ваших Интернетах бытует мнение, что LACP, мол, лучше балансирует трафик между интерфейсами. Это не так. В плане балансировки они абсолютно идентичны.

###### Балансировка нагрузки.

NetApp VIF поддерживает один из трех доступный вариантов балансировки нагрузки – Round-Robin, MAC и IP (на последнем мы остановимся подробнее).

**Round-Robin** – отправляет ethernet фреймы поочередно через каждый из линков, что может создать ситуацию, когда фрейм #2 пришел раньше фрейма #1. Это, в свою очередь, приведет к ситуации, когда приложение или протокол будет вынуждено запросить повторную передачу фреймов. Еще одно распространенное заблуждение состоит в том, что при этом варианте балансировки можно получить скорость равную сумме пропускных способностей всех линков за одну передачу. Это не так :)

**MAC** – балансирует трафик на основе Source-Destination MAC. Работает, понятное дело, только если хост и файлер находятся в пределах одной подсети или VLAN.

**IP** – балансирует трафик на основе Source-Destination IP и является параметром по умолчанию в СХД NetApp. При балансировке, к слову, учитывается не весь IP адрес, а только последний откет, так что трафик от хостов 192.168.1.101, 10.0.0.101 к файлеру с одним IP будет отправлен по одному физическому каналу.

###### Псевдонимы (aka IP Aliasing).

Как и любая другая операционная система Data ONTAP позволяет задавать несколько IP адресов на одном, что является своеобразным псевдонимом. Однако пользы от того, что вы просто добавили дополнительные интерфейсы будет не много. Хосту должно быть известно что искомая сущность доступна по этим путям.

###### The last, but not the least – интерфейсы-партнеры.

Если вы помните, то в первой части, во время начальной конфигурации нас спрашивали должен ли интерфейс перехватывать на себя трафик партнера в случае нештатной ситуации. Так же штука с виртуальных Multimode VIF, при чем при сбое партнер перехватит VIF полностью, со всеми псевдонимами.

###### Flowcontrol и MTU.

На файлере и хостах, по рекомендации NetApp, следует установить flowcontrol в send, MTU 9000. На коммутаторах flowcontrol в recieve и максимально-возможное MTU (9198 для Cisco Catalyst 4948).

###### От теории к практике – putting all together.

Порты e0a-d головы А включены в GiE1/1-4, на них поднимем LACP Etherchannel, порты e0a-d головы B включены в GiE1/5-8 и сконфигурированы в Static Etherchannel.

**Static Multimode VIF:**

NetApp содержимое /etc/rc файла:

```hostname FAS2040-B
ifgrp create multi vif1b -b ip e0a e0b e0c e0d
ifconfig vif1b 172.16.21.11 netmask 255.255.255.0 mtusize 9000 partner vif1a
ifconfig vif1b alias 172.16.21.12 netmask 255.255.255.0
ifconfig vif1b alias 172.16.21.13 netmask 255.255.255.0
ifconfig vif1b alias 172.16.21.14 netmask 255.255.255.0
ifconfig e0a flowcontrol send
ifconfig e0b flowcontrol send
ifconfig e0c flowcontrol send
ifconfig e0d flowcontrol send
route add default 172.16.21.254 1
routed on
options dns.domainname domain.local
options dns.enable on
options nis.enable off
savecore```
Настройка коммутатора Cisco Catalyst:
```SAN-sw-A(config)#int range gi1/5-8
SAN-sw-A(config-if-range)#switchport
SAN-sw-A(config-if-range)#switchport mode access
SAN-sw-A(config-if-range)#switchport access vlan 21
SAN-sw-A(config-if-range)#flowcontrol receive on
SAN-sw-A(config-if-range)#channel-group 1 mode on
SAN-sw-A(config-if-range)#no shut
SAN-sw-A(config-if-range)#exit
SAN-sw-A(config)#int port-channel 1
SAN-sw-A(config-if)#mtu 9198```
**Dynamic Multimode VIF:**
NetApp содержимое /etc/rc файла:
```hostname FAS2040-A
ifgrp create lacp vif1a -b ip e0a e0b e0c e0d
ifconfig vif1a 172.16.21.1 netmask 255.255.255.0 mtusize 9000 partner vif1b
ifconfig vif1a alias 172.16.21.2 netmask 255.255.255.0
ifconfig vif1a alias 172.16.21.3 netmask 255.255.255.0
ifconfig vif1a alias 172.16.21.4 netmask 255.255.255.0
ifconfig e0a flowcontrol send
ifconfig e0b flowcontrol send
ifconfig e0c flowcontrol send
ifconfig e0d flowcontrol send
route add default 172.16.21.254 1
routed on
options dns.domainname domain.local
options dns.enable on
options nis.enable off
savecore
```
Настройка коммутатора Cisco Catalyst:```
SAN-sw-A(config)#int range gi1/1-4
SAN-sw-A(config-if-range)#switchport
SAN-sw-A(config-if-range)#switchport mode access
SAN-sw-A(config-if-range)#switchport access vlan 21
SAN-sw-A(config-if-range)#flowcontrol receive on
SAN-sw-A(config-if-range)#channel-group 2 mode active
SAN-sw-A(config)#int po2
SAN-sw-A(config-if)#mtu 9198```
**Проверяем:**
Cisco:
```
SAN-sw-A#sh etherchannel summary | b Group
Group Port-channel Protocol Ports
——+————-+———–+———————————————–
1 Po1(SU) – Gi1/5(P) Gi1/6(P) Gi1/7(P)
Gi1/8(P)
2 Po2(SU) LACP Gi1/1(P) Gi1/2(P) Gi1/3(P)
Gi1/4(P)
```
NetApp:
```
FAS2040-A> ifconfig -a
e0a: flags=0xaf08867 mtu 9000
ether 02:a0:98:2b:16:bb (auto-1000t-fd-up) flowcontrol send
trunked vif1a
e0b: flags=0xaf08867 mtu 9000
ether 02:a0:98:2b:16:bb (auto-1000t-fd-up) flowcontrol send
trunked vif1a
e0c: flags=0xaf08867 mtu 9000
ether 02:a0:98:2b:16:bb (auto-1000t-fd-up) flowcontrol send
trunked vif1a
e0d: flags=0xaf08867 mtu 9000
ether 02:a0:98:2b:16:bb (auto-1000t-fd-up) flowcontrol send
trunked vif1a
e0P: flags=0x2348867 mtu 1500 PRIVATE
inet 192.168.1.12 netmask 0xfffffc00 broadcast 192.168.3.255 noddns
ether 00:a0:98:2b:16:b6 (auto-unknown-down) flowcontrol full
lo: flags=0x1b48049 mtu 8160
inet 127.0.0.1 netmask 0xff000000 broadcast 127.0.0.1
ether 00:00:00:00:00:00 (VIA Provider)
losk: flags=0x40a400c9 mtu 9188
inet 127.0.20.1 netmask 0xff000000 broadcast 127.0.20.1
vif1a: flags=0x22f48863 mtu 9000
inet 172.16.21.1 netmask 0xffffff00 broadcast 172.16.21.255
inet 172.16.21.4 netmask 0xffffff00 broadcast 172.16.21.255
inet 172.16.21.3 netmask 0xffffff00 broadcast 172.16.21.255
inet 172.16.21.2 netmask 0xffffff00 broadcast 172.16.21.255
partner vif1b (not in use)
ether 02:a0:98:2b:16:bb (Enabled interface groups)
```
[Часть третья настройки NetApp FAS >>> (disk ownership, raid group, aggregate, volume, qtree, lun, share).](/2012/06/setting-up-netapp-filer-part-3/ "Часть третья настройки NetApp FAS (disk ownership, raid group, aggregate, volume, qtree, lun, share).")
