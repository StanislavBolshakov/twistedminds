---
title: 'Cisco, часть 2: настройка интерфейсов, NAT с overloading (PAT), проброс порта/адреса'
author: ["Stanislav"]
date: 2010-11-08T08:42:45+00:00
url: /2010/11/cisco-interface-config-nat-pat-port-forwarding/
categories:
  - Tech
tags:
  - cisco
  - nat
  - pat
---

Имеет место быть следующая топология:![network scheme](/wp-content/uploads/2010/11/simple_nat.png "network scheme")
в кошку вставлен HWIC-4ESW, что по сути представляет из себя 4х портовый l2 свич, который одним портом смотрит в локальную сеть. Интерфейс fa0/0 смотрит в интернет.
В режиме глобальной конфигурации включим ускоренную коммутацию пакетов (cef, Cisco Express Forwarding) назначим маски, ip и напишем описание к интерфейсам, а так же пропишем маршрут по умолчанию (next-hop router):
```CoreRouter#conf t
Enter configuration commands, one per line. End with CNTL/Z.
CoreRouter(config)#ip cef
CoreRouter(config)#int fa0/0
CoreRouter(config-if)#ip address 1.1.1.1 255.255.255.0
CoreRouter(config-if)#description TO BIG AND SCARY INTERNET
CoreRouter(config-if)#int vlan 1
CoreRouter(config-if)#ip address 192.168.0.254 255.255.255.0
CoreRouter(config-if)#description TO NICE AND COOL LAN
CoreRouter(config-if)#exit
CoreRouter(config)#ip route 0.0.0.0 0.0.0.0 1.1.1.2```
Теперь настроим простой NAT, обозначив интерфейсы либо как смотрящие в локальную сеть (ip nat inside), либо как смотрящие в WAN (ip nat outside).
_для извращенных случаев, когда необходимо, чтобы один и тот же интерфейс был и inside и outside см. [ip nat enable и NVI](http://www.cisco.com/en/US/docs/ios/12_3t/12_3t14/feature/guide/gtnatvi.html "http://www.cisco.com/en/US/docs/ios/12_3t/12_3t14/feature/guide/gtnatvi.html")_
Затем создадим пул NAT адресов и ACL (более подробно в части 3) для сетей, которым разрешены NAT трансляции.
```CoreRouter#conf t
Enter configuration commands, one per line. End with CNTL/Z.
CoreRouter(config)#int fa0/0
CoreRouter(config-if)#ip nat outside
CoreRouter(config-if)#int vlan 1
CoreRouter(config-if)#ip nat inside
CoreRouter(config-if)#exit
! создадим стандартный ACL, разрешающий подсеть для трансляции нат
CoreRouter(config)#access-list 10 permit 192.168.0.0 0.0.0.255
! создадим пул WAN адресов
CoreRouter(config)#ip nat pool WANPOOL 1.1.1.1 1.1.1.1 netmask 255.255.255.0
! и заставим кошку транслировать адреса из разрешенных в ACL сетей в наш пул
CoreRouter(config)# ip nat inside source list 10 pool WANPOOL overload```
Проброс порта делается следующим образом:
```CoreRouter(config)#ip nat inside source static tcp 192.168.0.2 110 1.1.1.1 110 extendable```
а всего адреса (в случае если у вас несколько внешних ip):
```CoreRouter(config)#ip nat inside source static 192.168.0.2 1.1.1.1 extendable```
вот, собственно и все.
Статистику по NAT можно посмотреть, подав в enable exec команду **sh ip nat sta**, а активные трансляции – **sh ip nat tra**.
