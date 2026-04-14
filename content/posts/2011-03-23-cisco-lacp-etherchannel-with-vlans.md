---
title: Cisco LACP etherchannel c несолькими виланами
author: ["Stanislav"]
date: 2011-03-23T11:49:31+00:00
url: /2011/03/cisco-lacp-etherchannel-with-vlans/
categories:
  - Tech
tags:
  - 802.3ad
  - cisco
  - lacp
  - port aggregation
  - vlan

---
После окончательного физического переноса всех серверов возникла фантазия сделать все как надо.

Первым делом было решено создать команды из сетевых интерфейсах на всех серверах и включить их в Cisco Catalyst 4948, агрегировав несколько линков в один по 802.3ad. Частный случай состоял в том, что один из серверов был гипервизором Hyper-V, а сервера, подключенные к виртуальному свичу, должны быть доступны через отдельный Vlan. 

Первым делом, если еще не создан, создадим vlan:  
```CoreSwitchA#conf t  
Enter configuration commands, one per line. End with CNTL/Z.  
CoreSwitchA(config)#vlan 10  
CoreSwitchA(config)#name Servers  
CoreSwitchA(config)#exit  
```  
сконфигурируем интерфейс для управления &#8211;  
```  
CoreSwitchA(config)#int vlan 10  
CoreSwitchA(config-if)#ip address 172.16.20.254 255.255.255.0  
CoreSwitchA(config-if)#description to Servers VLAN  
! не обязательно &#8211; адрес, чтобы форвардить запросы к DHCP серверу  
CoreSwitchA(config-if)#ip helper-address 192.168.0.1  
CoreSwitchA(config-if)#no shut  
CoreSwitchA(config-if)#exit  
```  
создадим port-channel интерфейс, заставим его работать в транке и разрешим ему соответствующие виланы &#8211;  
```  
CoreSwitchA(config)#int port-channel 1  
CoreSwitchA(config-if)#switchport  
CoreSwitchA(config-if)#switchport trunk encapsulation dot1q  
CoreSwitchA(config-if)#switchport mode trunk  
CoreSwitchA(config-if)#switchport trunk allowed vlan 1,10  
CoreSwitchA(config-if)#exit  
```  
осталось зайти на интерфейсы, включить dot1q энкапсуляцию, включить протокол LACP и добавить их в etherchannel port-channel 1 &#8211;  
```  
CoreSwitchA(config)#int gi1/3  
CoreSwitchA(config-if)#switchport  
CoreSwitchA(config-if)#switchport trunk encapsulation dot1q  
CoreSwitchA(config-if)#channel-protocol lacp  
CoreSwitchA(config-if)#channel-group 1 mode active  
```  
в случае успеха лицезреем следующее сообщение и проделываем тоже с другим портом(ами) &#8211;  
```*Mar 23 03:35:58.971: %EC-5-BUNDLE: Interface Gi1/3 joined port-channel Po1  
CoreSwitchA(config-if)#int gi1/4  
CoreSwitchA(config-if)#switchport  
CoreSwitchA(config-if)#switchport trunk encapsulation dot1q  
CoreSwitchA(config-if)#channel-protocol lacp  
CoreSwitchA(config-if)#channel-group 1 mode active ```

that&#8217;s all, folks!

```  
CoreSwitchA#sh etherchannel 1 summary | b Num  
Number of channel-groups in use: 1  
Number of aggregators: 1

Group Port-channel Protocol Ports  
&#8212;&#8212;+&#8212;&#8212;&#8212;&#8212;-+&#8212;&#8212;&#8212;&#8211;+&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8211;  
1 Po1(SU) LACP Gi1/3(P) Gi1/4(P)  
```