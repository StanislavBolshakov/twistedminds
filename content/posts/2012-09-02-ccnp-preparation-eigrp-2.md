---
title: 'Заметки к CCNP Route: EIGRP часть 2'
author: ["Stanislav"]
date: 2012-09-02T07:01:46+00:00
url: /2012/09/ccnp-preparation-eigrp-2/
categories:
  - Tech
tags:
  - ccnp route
  - cisco
  - eigrp
---

##### EIGRP в NBMA сетях, split horizon, аутентификация

![dynamic routing is sexy](/wp-content/uploads/2012/08/eigrp.png "dynamic routing is sexy")
В предыдущей статье мы рассмотрели материал, который должен знать каждый уважающий себя CCNA.
Откровенно говоря я не очень понимаю зачем Cisco везде тащит устаревший Frame Relay, но раз он есть в экзамене, то и разобраться с ним не помешает.

Для реализации EIGRP в NBMA сетях есть два варианта настройки:
* псевдо броадкаст
* указание соседа в ручном режиме
![](/wp-content/uploads/2012/09/eigrp-topology.png "eigrp nbma topology")

Для начала соберем стэнд, настроив frame-relay switch, hq, b1 и b2 следующим образом:
```

FRSw(config)#frame-relay switching
FRSw(config)#int s1/0
FRSw(config-if)#description to HQ
FRSw(config-if)#encapsulation frame-relay
FRSw(config-if)#frame-relay intf-type dce
FRSw(config-if)#clock rate 64000
FRSw(config-if)#frame-relay route 201 interface s1/1 102
FRSw(config-if)#frame-relay route 301 interface s1/2 103
FRSw(config-if)#no ip address
FRSw(config-if)#int s1/1
FRSw(config-if)#description to B1
FRSw(config-if)#encapsulation fra
FRSw(config-if)#encapsulation frame-relay
FRSw(config-if)#frame-relay route 102 interface s1/0 201
FRSw(config-if)#clock rate 64000
FRSw(config-if)#frame-relay intf-type dce
FRSw(config-if)#no ip address
FRSw(config-if)#int s1/2
FRSw(config-if)#description to B2
FRSw(config-if)#no ip address
FRSw(config-if)#encapsulation frame-relay
FRSw(config-if)#clock rate 64000
FRSw(config-if)#frame-relay intf-type dce
FRSw(config-if)#frame-relay route 103 interface serial 1/0 301

HQ(config)#int s1/0
HQ(config-if)#description to FR Cloud
HQ(config-if)#encapsulation frame-relay
HQ(config-if)#exit
HQ(config)#int s1/0.1 multipoint
HQ(config-subif)#ip address 192.168.1.1 255.255.255.248
HQ(config-subif)#frame-relay map ip 192.168.1.2 201 broadcast
HQ(config-subif)#frame-relay map ip 192.168.1.3 301 broadcast

B1(config)#in s1/1
B1(config-if)#encapsulation frame-relay
B1(config-if)#ip address 192.168.1.2 255.255.255.248
B1(config-if)#frame-relay map ip 192.168.1.1 102 broadcast
B1(config-if)#frame-relay map ip 192.168.1.3 102 broadcast

B2(config)#int s1/2
B2(config-if)#ip address 192.168.1.3 255.255.255.248
B2(config-if)#encapsulation frame-relay
B2(config-if)#frame-relay map ip 192.168.1.1 103
B2(config-if)#frame-relay map ip 192.168.1.2 103
```

Проверим наше облако:
```

FRSw#sh frame-relay route
Input Intf      Input Dlci      Output Intf     Output Dlci     Status
Serial1/0       201             Serial1/1       102             active
Serial1/0       301             Serial1/2       103             active
Serial1/1       102             Serial1/0       201             active
Serial1/2       103             Serial1/0       301             active

HQ#sh frame-relay map
Serial1/0.1 (up): ip 192.168.1.2 dlci 201(0xC9,0x3090), static,
              broadcast,
              CISCO, status defined, active
Serial1/0.1 (up): ip 192.168.1.3 dlci 301(0x12D,0x48D0), static,
              broadcast,
              CISCO, status defined, active
```

Настроим EIGRP с автономной системой 1, подав на маршрутизаторах незамысловатую команду и проверив соседство с B1 и B2:
```

Rx(config)#router eigrp 1
Rx(config-router)#network 192.168.1.0 0.0.0.7

HQ#sh ip eigrp neighbors
IP-EIGRP neighbors for process 1
H   Address                 Interface       Hold Uptime   SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
0   192.168.1.2             Se1/0.1          154 00:02:23   47   282  0  3
```

Ой-ой, где же B2? А у B2 в frame-relay map мы “забыли” включить псевдо броадкаст. Можно было бы согласовать с ним с одним соседство в ручном режиме, если бы не одно оно. **Одна команда согласовывающая связность в ручном режиме отключает мультикаст на всем интерфейсе, а в случае с multipoint интерфейсом это приведет к потере соседства.** По-этому не будет выпендриваться и добавим ключ broadcast к frame-relay map на B2.

Хорошим тоном является отключение автоматического суммирования маршрутов. В нашем случае, если маршрутизаторы начнут рассказывать друг другу о 172.xx.xx.xx/16 сетях, ничего плохого не произойдет. Но если представить, что вместо этого сети, маршрутизацию для которых надо настроить лежат в 172.16.xx.xx, к примеру, то получится каша, когда все рассказывают о том, что знают маршрут до 172.16.0.0/16 сети.

Дальнейшая настройка заключается в описании локальных сетей. Проверим что получилось:
```

HQ#sh ip route eigrp
     172.17.0.0/24 is subnetted, 4 subnets
D       172.17.4.0 [90/2195456] via 192.168.1.2, 00:03:25, Serial1/0.1
D       172.17.1.0 [90/2195456] via 192.168.1.2, 00:03:25, Serial1/0.1
D       172.17.3.0 [90/2195456] via 192.168.1.2, 00:03:25, Serial1/0.1
D       172.17.2.0 [90/2195456] via 192.168.1.2, 00:03:25, Serial1/0.1
     172.18.0.0/24 is subnetted, 4 subnets
D       172.18.4.0 [90/2195456] via 192.168.1.3, 00:00:13, Serial1/0.1
D       172.18.2.0 [90/2195456] via 192.168.1.3, 00:00:13, Serial1/0.1
D       172.18.3.0 [90/2195456] via 192.168.1.3, 00:00:13, Serial1/0.1
D       172.18.1.0 [90/2195456] via 192.168.1.3, 00:00:13, Serial1/0.1
```

```

B2#sh ip route eigrp
     172.16.0.0/24 is subnetted, 4 subnets
D       172.16.4.0 [90/2195456] via 192.168.1.1, 00:14:11, Serial1/2
D       172.16.1.0 [90/2195456] via 192.168.1.1, 00:14:11, Serial1/2
D       172.16.2.0 [90/2195456] via 192.168.1.1, 00:14:11, Serial1/2
D       172.16.3.0 [90/2195456] via 192.168.1.1, 00:14:11, Serial1/2
```

Мы видим, что маршрутизатор в HQ получает всю информацию от B1 и B2, а B1 и B2 знают только о сетях, присоединенных к HQ. Это поведение связано с тем, что по умолчание включен механизм предотвращения образования петель split-horizon, смысл которого состоит в следующем: маршрутизатор не будет отправлять данные о полученном на интерфейсе маршруте с того же интерфейса. А так как маршруты о сетях за B1 и B2 приходят на один и тот же мультипоинт интерфейс маршрутизатора HQ, то согласно этому правилу HQ не распространяет информацию о сетях за B1 в сторону B2 и о сетях за B2 в сторону B1.

**По умолчанию split horizon включен на подинтерфейсах и выключен на физических интерфейсах.**

Обойти эту проблему можно с помощью PVC между B1 и B2 или если отключить split horizon на s1/0.1 мультипоинт интерфейсе маршрутизатора HQ:
```

HQ(config)#int s1/0.1
HQ(config-subif)#no ip split-horizon eigrp 1
```

Вот теперь полный порядок.

##### Утилизации канала протоколом EIGRP

По умолчанию EIGRP ограничивает себя 50%, взятыми от пропускной способности интерфейса, при чем во внимание принимается величина bandwidth. В случае если это multipoint интерфейс eigpr поделит это число на количество соседей. Изменить это поведение можно в конфигурации интерфейса следующим образом:
```

HQ(config-subif)#ip bandwidth-percent eigrp 1 60
```

##### Аутентификация

EIGRP оперирует таким понятием как связка ключей, в которую можно не просто добавить эти самые ключи, но и настроить ротацию на основе промежутков времени. Время на маршрутизаторах, понятное дело, должно быть синхронизировано. Со связками ключей работает логика ACL  – до первого совпадения. И так, займемся этим:
```

HQ(config)#key chain EIGRP_AS_1
HQ(config-keychain)#key 10
HQ(config-keychain-key)#key-string passwd-1
HQ(config-keychain-key)#accept-lifetime 00:00:00 1 sep 2012 00:00:00 1 oct 2012
HQ(config-keychain-key)#send-lifetime 00:00:00 1 sep 2012 00:00:00 1 oct 2012
HQ(config-keychain-key)#key 20
HQ(config-keychain-key)#key-string passwd-2
HQ(config-keychain-key)#accept-lifetime 00:00:00 30 sep 2012 infinite
HQ(config-keychain-key)#send-lifetime 00:00:00 30 sep 2012 infinite
```

И так же, как и ACL, связка ключей должна быть применена на интерфейсе:
```

HQ(config)#int s1/0.1
HQ(config-subif)#ip authentication mode eigrp 1 md5
HQ(config-subif)#ip authentication key-chain eigrp 1 EIGRP_AS_1
```

```

HQ# debug eigrp packet
EIGRP Packets debugging is on
    (UPDATE, REQUEST, QUERY, REPLY, HELLO, IPXSAP, PROBE, ACK, STUB, SIAQUERY, SIAREPLY)
Sep  2 13:31:29.347: EIGRP: received packet with MD5 authentication, key id = 10
Sep  2 13:31:29.347: EIGRP: Received HELLO on Serial1/0.1 nbr 192.168.1.3
```

Обратите внимание на key id – ключи должны совпадать не только по кодовой фразе, но и по идентификатору.
