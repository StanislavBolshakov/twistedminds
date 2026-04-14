---
title: 'Заметки к CCNP Route: OSPF часть 5'
author: ["Stanislav"]
date: 2012-09-09T14:09:03+00:00
url: /2012/09/ccnp-preparation-ospf-5/
categories:
  - Tech
tags:
  - ccnp route
  - cisco
  - ospf
---

![](/wp-content/uploads/2012/09/ospf.jpg "ospf")Заключительная часть битвы с замечательным протоколом OSPF. Нам осталось поговорить про нюансы работы OSPF в NMBA сетях, а так же о замечательной технологии virtual links, как о одном из способов связать не backbone область с area 0 через другую не backbone область (другим способом могут стать gre-туннели, но на экзамене вас про это спрашивать вряд ли станут). Ну и конечно в этот раз будет много, очень много практики.

##### OSPF в NBMA сетях

Не будем сразу бросаться в омут NBMA с головой и постараемся вспомнить типы сетей, в которых может работать OSPF:

- Broadcast, multiaccess – например ethernet, token ring. В этом случае выбираются DR/BDR, Hello таймер по умолчанию равен 10 секундам, используется два multicast адерса.
- Point-to-point – serial links. DR и BDR уже не выбираются, Hello таймер равен все тем же 10 секундам, используется один multicast адрес.
- NBMA (nonbroadcast multiaccess сети) – например frame relay, atm, mpls. 5 различных режимов работы.

**Режимы работы NMBA:**

- broadcast (Cisco mode)
- non-broadcast (RFC compliant mode)
- point-to-multipoint (RFC compliant mode)
- point-to-multipoint non-broadcast (Cisco mode)
- point-to-point (Cisco mode)

С двумя из них вы уже знакомы – broadcast и point-to-point. Тут встает закономерный вопрос:

![](/wp-content/uploads/2012/09/175091_m.jpg "ospf lolwut")

Но… но… но… ведь broadcast это режим в котором OSPF работает по умолчанию. При чем тут Cisco Proprietary. Проблема в том, что broadcast в LAN сетях – это действительно режим работы по умолчанию, но с NBMA дела обстоят иначе.

|  |  |  |  |  |
| --- | --- | --- | --- | --- |
| Режим работы | Конфигурация соседей | Подсеть | Выборы DR/BDR | Примечания |
| Non-broadcast | вручную | одна | да* | Эмуляция LAN, режим по умолчанию для X.25, FR, ATM |
| Point-to-multipont | автоматически** | одна | нет | Соединение с каждым соседом воспринимается как point-to-point линк. Требует специфической топологии Hub’n’Spoke. |
| Point-to-  multipoint non-broadcast | вручную | одна | нет | Работает как Point-to-multipoint. Для тех случаев, когда нет возможности использовать псевдоброадкаст. |
| Point-to-point | автоматически | несколько | нет | Используются различные сабинтерфейсы. |

* – DR и BDR должны иметь полную связность со всем маршрутизаторами. В Hub’n’Spoke топологии DR будет Hub(s), а Spoke – DROTHER. В Full Mesh можно выбирать и DR и BDR так же как и в LAN сетях.

** Требуется включение псевдоброадкаста.

Let’s get our hands dirty :)

![](/wp-content/uploads/2012/09/ospf-nbma-1.png "ospf in nbma non-broadcast")

Начнем с такой конфигурации и настроим в Area 0 non-broadcast режим работы сети. Настройку этого Frame Relay коммутаторов я показывать не буду, вы можете самостоятельно посмотреть в [этой статье](/2012/09/ccnp-preparation-eigrp-2/ "setting up frame-relay switch").
```

R1#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
R1(config)#int s1/0
R1(config-if)#description to FR cloud
R1(config-if)#no ip address
R1(config-if)#encapsulation frame-relay
R1(config-if)#exit
R1(config)#int s1/0.1 multipoint
R1(config-subif)#ip address 192.168.1.1 255.255.255.248
R1(config-subif)#frame-relay map ip 192.168.1.2 201 broadcast
R1(config-subif)#frame-relay map ip 192.168.1.3 301 broadcast
R1(config-subif)#no shut
R1(config)#router ospf 1
R1(config-router)#router-id 1.1.1.1
R1(config-router)#network 192.168.1.0 0.0.0.7 a 0
R1(config-router)#neighbor 192.168.1.2
R1(config-router)#neighbor 192.168.1.3
R1(config-router)#^Z
R1#sh ip ospf interface s1/0.1
Serial1/0.1 is down, line protocol is down
  Internet Address 192.168.1.1/29, Area 0
  Process ID 1, Router ID 1.1.1.1, Network Type NON_BROADCAST, Cost: 64
  Transmit Delay is 1 sec, State DOWN, Priority 1
  No designated router on this network
  No backup designated router on this network
  Timer intervals configured, Hello 30, Dead 120, Wait 120, Retransmit 5
    oob-resync timeout 120
```

Все, больше ничего настраивать не надо. Как мы помним для FR non-broadcast – это режим работы по умолчанию. В случае с R2 и R3 хотелось бы убедиться, что они никогда не станут DR или BDR, а значит нам надо либо задать либо RID, либо приоритет ниже таковых у R1. Так же обратите внимание на статически сконфигурированных соседей, ведь если не дунуть, то чуда не произойдет.
```

R2(config)#int s1/2
R3(config-if)#ip address 192.168.1.3 255.255.255.248
R2(config-if)#encapsulation frame-relay
R2(config-if)#no shut
R2(config-if)#frame-relay map ip 192.168.1.1 102 broadcast
R2(config-if)#frame-relay map ip 192.168.1.3 102 broadcast
R2(config-if)#ip ospf priority 0
R2(config-if)#router ospf 1
R2(config-router)#network 192.168.1.0 0.0.0.7 a 0
R2(config-router)#router-id 2.2.2.2
R2(config-router)#neighbor 192.168.1.1

R3(config)#int s1/1
R3(config-if)#ip address 192.168.1.3 255.255.255.248
R3(config-if)#no shut
R3(config-if)#encapsulation frame-relay
R3(config-if)#frame-relay map ip 192.168.1.1 103 broadcast
R3(config-if)#frame-relay map ip 192.168.1.2 103 broadcast
R3(config-if)#ip ospf priority 0
R3(config-if)#router ospf 1
R3(config-router)#network 192.168.1.0 0.0.0.7 a 0
R3(config-router)#router-id 3.3.3.3
R3(config-router)#neighbor 192.168.1.1
```

R1 сформировал соседство с R2 и R3 и ведет себя как типичный DR:
```

R1#sh ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           0   FULL/DROTHER    00:01:44    192.168.1.2     Serial1/0.1
3.3.3.3           0   FULL/DROTHER    00:01:32    192.168.1.3     Serial1/0.1
```

Продолжим развлекаться и добавим Area 51 с R4 таким образом, что R3 становится ABR. Настроим эту сеть как point-to-point.

![](/wp-content/uploads/2012/09/ospf-nbma-21.png "ospf nbma point-to-point")

R3:
```

R3(config)#int s1/3
R3(config-if)#no ip address
R3(config-if)#encapsulation frame-relay
R3(config-if)#router ospf 1
R3(config-router)#network 192.168.10.0 0.0.0.3 a 51
R3(config-router)#exit
R3(config)#int s1/3.1 point-to-point
R3(config-subif)#ip address 192.168.10.1 255.255.255.252
R3(config-subif)#frame-relay interface-dlci 401
R3(config-subif)#do sh ip ospf int s1/3.1
Serial1/3.1 is down, line protocol is down
  Internet Address 192.168.10.1/30, Area 51
  Process ID 1, Router ID 3.3.3.3, Network Type POINT_TO_POINT, Cost: 64
  Transmit Delay is 1 sec, State DOWN
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    oob-resync timeout 40
```

Альтернативно можно задать point-to-point mode на интерфейсе командой i**p ospf network point-to-point**.

R4:
```

R4#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
R4(config)#int s1/0
R4(config-if)#encapsulation frame-relay
R4(config-if)#no ip address
R4(config)#int s1/0.1 point-to-point
R4(config-subif)#ip address 192.168.10.2 255.255.255.252
R4(config-subif)#frame-relay interface-dlci 104
R4(config-subif)#exit
R4(config)#router ospf 1
R4(config-router)#router-id 4.4.4.4
R4(config-router)#network 192.168.10.0 0.0.0.3 a 51
R4(config-router)#^Z
R4#sh ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
3.3.3.3           0   FULL/  -        00:00:32    192.168.10.1    Serial1/0.1
```

Идем дальше. Добавим еще одну область area 740 с R21 и R22 таким образом, чтобы R2 стал ABR.

![](/wp-content/uploads/2012/09/ospf-nbma-3.png "ospf nbma point to multipoint ")

R2:
```

R2#conf t
R2(config)#interface Serial1/0
R2(config-if)#description to FR cloud
R2(config-if)#no ip address
R2(config-if)#encapsulation frame-relay
R2(config-if)#serial restart-delay 0
R2(config-if)#no shut
R2(config-if)#interface Serial1/0.1 multipoint
R2(config-subif)#ip address 192.168.1.3 255.255.255.248
R2(config-subif)#frame-relay map ip 192.168.100.1 501 broadcast
R2(config-subif)#frame-relay map ip 192.168.100.2 502 broadcast
R2(config-subif)#no shut
R2(config-subif)#ip ospf network point-to-multipoint
R2(config-subif)#router ospf 1
R2(config-router)#network 192.168.100.0 0.0.0.7 area 740

R2#sh ip ospf int s1/0.1
Serial1/0.1 is down, line protocol is down
  Internet Address 192.168.1.3/29, Area 0
  Process ID 1, Router ID 2.2.2.2, Network Type POINT_TO_MULTIPOINT, Cost: 64
  Transmit Delay is 1 sec, State DOWN
  Timer intervals configured, Hello 30, Dead 120, Wait 120, Retransmit 5
    oob-resync timeout 120
```

Далее мы можем настроить R21 и R22 как point-to-point или задать определенный network type.

R21:
```

R21(config)#interface Serial1/1
R21(config-if)#ip address 192.168.100.1 255.255.255.248
R21(config-if)#encapsulation frame-relay
R21(config-if)#ip ospf network point-to-multipoint
R21(config-if)#frame-relay map ip 192.168.100.3 105 broadcast
R21(config-if)#frame-relay map ip 192.168.100.2 105 broadcast
R21(config)#router ospf 1
R21(config-router)#router-id 1.1.2.1
R21(config-router)#network 192.168.100.0 0.0.0.7 a 740

R21#sh ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           0   FULL/  -        00:01:48    192.168.100.3   Serial1/1
R21#sh ip ospf interface s1/1
  Process ID 1, Router ID 1.1.2.1, Network Type POINT_TO_MULTIPOINT, Cost: 64
  Transmit Delay is 1 sec, State POINT_TO_MULTIPOINT
```

R22:
```

R22(config)#interface Serial1/2
R22(config-if)#no shut
R22(config-if)#encapsulation frame-relay
R22(config-if)#int s1/2.1 point-to-point
R22(config-subif)#no shut
R22(config-subif)#ip address 192.168.100.2 255.255.255.248
R22(config-subif)#frame-relay interface-dlci 205
R22(config-subif)#ip ospf hello-interval 30
R22(config-fr-dlci)#router ospf 1
R22(config-router)#network 192.168.100.0 0.0.0.7 a 740
R22(config-router)#^Z

R22#sh ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           0   FULL/  -        00:01:40    192.168.100.3   Serial1/2.1
R22#sh ip route ospf
     192.168.10.0/30 is subnetted, 1 subnets
O IA    192.168.10.0 [110/192] via 192.168.100.3, 00:00:56, Serial1/2.1
     192.168.1.0/29 is subnetted, 1 subnets
O IA    192.168.1.0 [110/128] via 192.168.100.3, 00:00:56, Serial1/2.1
     192.168.100.0/24 is variably subnetted, 3 subnets, 2 masks
O       192.168.100.1/32 [110/128] via 192.168.100.3, 00:00:56, Serial1/2.1
O       192.168.100.3/32 [110/64] via 192.168.100.3, 00:00:56, Serial1/2.1
R22#ping 192.168.10.2

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.10.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 100/104/112 ms
```

Обратите внимание на изменение hello таймера. По умолчанию на point-to-point линках он равен 10 секундам, а на point-to-multipoint – 30. Если значение не изменить, то соседство не сформируется. В итоге мы получили автономную систему, в которой все знают про всех.

В очередной раз добавим автономную систему area 1, таким образом, чтобы R22 стал ABR. Тут внимательный читатель решит, что самое время для мишки

![](/wp-content/uploads/2012/09/175091_m.jpg "ospf lolwut")

и будет абсолютно прав. Ведь несколько раз речь шла о том, что любая non-backbone область должна быть присоединена к area 0. No shit, должна. Этим и займемся, расчехлив технологию virtual link.

##### OSPF Virtual Links

![](/wp-content/uploads/2012/09/ospf-virtual-links.png "ospf virtual links")

Чтобы не нарушать правила дизайна OSPF о том, что все области должны иметь связность с backbone областью существует технология виртуальных каналов (virtual links).

Что нам нужно сделать это настроить сквозное подключение в 740 области между R2 (RID 2.2.2.2) и R22 (RID 1.1.2.2). В случае если между маршрутизаторами R2 и R22 существовало бы еще 10-20-300 других маршрутизаторов настройка была бы точно такой же.
```

R2(config)#router ospf 1
R2(config-router)#area 740 virtual-link 1.1.2.2

R22(config)#router ospf 1
R22(config-router)#area 740 virtual-link 2.2.2.2
```

Все. Этого достаточно для поднятия туннельного интерфейса OSPF_VL0 в нашем случае и получении на R8 все маршрутов автономной системы.

##### OSPF virtual link authentication

Just for kicks можем настроить аутентификацию на виртуальном канале:

|  |  |  |
| --- | --- | --- |
| тип | номер | синтаксис |
| none |  | area ***#_num*** virtual-link ***router-id*** authentication null |
| clear text | 1 | area ***#_num*** virtual-link***router-id*** authentication authentication-key ***KEY*** |
| md5 | 2 | area ***#_num*** virtual-link***router-id*** authentication message-digest  message-diget-key ***#_num*** md5 ***KEY*** |

а можем и не настроить. Вот, в принципе, и все. До сих пор не могу поверить что c OSPF покончено и можно переходить к более интересным темам :)
