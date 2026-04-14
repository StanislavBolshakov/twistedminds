---
title: Cisco Site-to-Site IPsec VPNs
author: ["Stanislav"]
date: 2012-07-23T10:21:38+00:00
url: /2012/07/cisco-site-to-site-ipsec-vpns/
categories:
  - Tech
tags:
  - cisco
  - ipsec
  - lan-to-lan
  - nat
  - site-to-site
  - vpn
---
Наконец-то долгожданная практика ,)

[![](/wp-content/uploads/2012/07/vpn_logo_130.jpg "vpn")][1]

В этой статье мы рассмотрим два простых случая построения site-to-site VPN на оборудовании Cisco. В первом случае оба устройства имеют белые адреса (static crypto map), а во втором один из маршрутизаторов оказывается за NAT (dynamic crypto map). Ну и в качестве бонуса рассмотрим ситуацию совместного сожительства и static и dynamic crypto map на одном маршрутизаторе.

Те, кому лень читать 1000+ слов могут перейти сразу к конфигурациям, располагающимся в самом конце статьи.

**Шаг 0**, который вы должны предпринять в боевых условиях – это убедится в том, что IPsec трафик сможет дойти до вашего маршрутизатора. Если перед маршрутизатором стоит МСЭ или же сам маршрутизатор защищен ACL, CBAC, ZBF, you name it убедитесь в том, что разрешены протоколы AH и ESP (протоколы второй фазы ISAKMP/IKE), а так же устройство в состоянии получать udp сегменты на порты 500 (первая фаза ISAKMP/IKE) и/или 4500 (NAT-T).

И так, поехали. Включим ISAKMP:
```

Rx (config)# crypto isakmp enable
```

Создадим политики первой фазы, которые отвечают за то как мы будем устанавливать и защищать управляющее соединение, а так же производить аутентификацию соседей. Обратите внимание, у вас может быть несколько политик с разными приоритетами. Чем номер политики ниже, тем приоритет будет выше.
```

Rx (config)# crypto isakmp policy #_политики
Rx (config-isakmp)# encryption [ des | 3des | aes ]
Rx (config-isakmp)# hash [ sha | md5 ]
Rx (config-isakmp)# group [ 1 | 2 | 5 | 14-16 | 19 | 20 | 24 ]
Rx (config-isakmp)# lifetime seconds
Rx (config-isakmp)# authentication [ rsa-sig | rsa-encr | pre-share ]
```

Если какой-то из параметров будет не указан, то iOS установит его в значение по умолчанию, которое выделено красным.

Чтобы посмотреть все политики достаточно в exec mode подать команду **show crypto isakmp policy.**

Устройства установят управляющее соединение только после того, как совпадет хотя бы одна ISAKMP/IKE политика, а точнее такие параметры в политике, как алгоритм шифрования, хэш-функция, DH группа и метод аутентификации. Lifetime значение будет принято наименьшим из двух политик.

В процессе установки соединения, как вы помните, соседи обмениваются всем списком политик и сравнение начинается с самой маленькой по порядковому номеру и соответственно самой безопасной с точки зрения устройства.

###### IKE dead peer detection (DPD) – а не погиб ли наш сосед?

Что еще стоит сделать в этой части, так это включить механизм, который определяет доступен ли сосед. Делается это с помощью команды **crypto isakmp keepalive *seconds [retries] [periodic].***

Решение проблем первой фазы осуществляется с помощью **debug crypto isakmp**.

Аутентификация устройств первой фазы с помощью pre-shared key выполняется так:
```

Rx(config)# crypto isakmp key секретный_ключ remote_ip [subnet_mask] [no-xauth]
```

В случае если IP адрес соседа не известен (динамический NAT) или перед вами не стоит задачи аутентифицировать каждого соседа по отдельности, то можно использоваться wildcards:
```

Rx(config)#crypto isakmp key secret_key address 0.0.0.0 0.0.0.0 no-xauth
```

На этом можно считать, что с фазой 1 мы закончили.

В процессе настройки второй фазы нам предстоит указать какой трафик защищать (crypto acl), как его защищать (transform set) и до кого его защищать (crypto map).

Crypto acl по сути являются обычными acl. Могут быть именованными и нумерованными. Имейте ввиду, что crypto acl предполагают только одно значение permit.

Transform set – представляет собой список параметров с помощью которых вы собираетесь защищать канал передачи данных. Для успешного завершения второй фазы у каждого из соседов должен быть хотя бы один совпадающий transofrm set. В transofrm set вы можете использовать только один параметр каждого типа:

|  |  |
| --- | --- |
| **Тип** | **Параметр** |
| AH | **ah-md5-hmac** |
| **ah-sha-hmac** |
| ESP аутентификация | **esp-md5-hmac** |
| **esp-sha-hmac** |
| ESP шифрование | **esp-null** |
| **esp-des** |
| **esp-3des** |
| **esp-seal** |
| **esp-aes 128** |
| **esp-aes 192** |
| **esp-aes-256** |
| Сжатие | **comp-lzs** |

Создается transform set следующим образом:
```

Rx (config)# crypto ipsec transform-set tset-name value1 [ value2 [ value3 [ value4 ]]]
```

###### Crypto map

Функцией crypto map является совмещение всех параметров (crypto acl, transfort set, адрес или имя хоста соседа, адрес с которого надо отправлять данные) необходимых для управляющего канала первой фазы и защищенного канала передачи данных, который устанавливается в процессе фазы 2. Существует static и dynamic crypto map. Crypto map привязываются к интерфейсам, только одна static  crypto map может быть активной на интерфейсе. Одна и та же static crypto map может быть активной на нескольких интерфейсах.

**Static crypto map**

Могут быть построены как с помощью ISAKMP, так и без. Последний вариант в рамках это статьи рассматриваться не будет, но вы можете познакомится с ним по ссылке <http://www.cisco.com/en/US/tech/tk583/tk372/technologies_configuration_example09186a0080093c26.shtml>

С использованием ISAKMP crypto map создается следующим образом:
```

Rx(config)# crypto map name #_number ipsec-isakmp
Rx(config-crypto-m)# match address crypto_acl_name_or_#
Rx(config-crypto-m)# set peer [ hostname | ip_addr ]
Rx(config-crypto-m)# set transform-set tset-name1 [ ... tsetname6]
```

Для просмотра crypto map есть команда **show crypto map [ interface *interface* | tag *name* ]**

Чтобы активировать static crypto map достаточно сделать следующее:
```

Rx(config)# interface interface Rx(config-if)# crypto map name
```

**Dynamic crypto map**

C static crypto map есть небольшая проблема – вам необходимо знать адрес соседа. В ситуации когда сосед находится за nat или адреса ему провайдер выдает автоматически придется использовать dynamic crypto map.  Вот где wild-carded pre-shared key сияет во всей красе :)

По хорошему счету dynamic crypto map – эта та же static crypto map, только параметры, которые раньше были завязаны на адрес соседа банально опускаются, что оставляет нам:
```

Rx(config)# crypto dynamic-map name #_number
Rx(config-crypto-m)# set transform-set tset-name1 [ ... tsetname6]
```

Crypto acl можно указывать, а можно не указывать. Если вы решите не указывать, то ваш маршрутизатор примет любой acl который ему предложит сосед в процессе установления второй фазы.

**show crypto dynamic-map [tag *name*]** показывает dynamic map легко и непринужденно.

Чтобы активировать dynamic crypto map вам нужно включить ее в статическую крипто карту. К слову это и позволяет использовать static l2l, dynamic l2l и remote access на одном маршрутизаторе. Так же желательно (cisco best practice), но не обязательно давать dynamic crypto map номера выше, чем статическим.

Включается dynamic в static следующим образом:
```

Rx(config)# crypto map stat_map_name #_number ipsec-isakmp dynamic dyn_map_name
```

Далее *static_map_name* активируется на интерфейсе так же как и любая другая статичная крипто карта.

###### Putting all together

Лабораторный стэнд: [![](/wp-content/uploads/2012/07/network-map.png "network map ipsec site-to-site vpn")][2]

Сочные конфиги для самых терпиливых:

**case 1: R1 <=> R2 Static crypto map**
```

R1(config)#crypto isakmp enable
! политики первой фазы
R1(config)#crypto isakmp policy 10
R1(config-isakmp)#encryption aes 128
R1(config-isakmp)#hash md5
R1(config-isakmp)#group 2
R1(config-isakmp)#authentication pre-share
R1(config-isakmp)#lifetime 120
R1(config)#crypto isakmp keepalive 30 periodic
! ключ аутентификации
R1(config)#crypto isakmp key DERKEY address 10.2.2.1 255.255.255.255 no-xauth
! crypto acl
R1(config)#ip access-list extended cacl-r2
R1(config-ext-nacl)#permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255 log
! transform set фазы 2
R1(config)#crypto ipsec transform-set tset1 esp-aes 128 esp-md5-hmac
R1(cfg-crypto-trans)#mode tunnel
! static crypto map
R1(config)#crypto map stat-cmap 100 ipsec-isakmp
R1(config-crypto-map)#match address cacl-r2
R1(config-crypto-map)#set peer 10.2.2.1
R1(config-crypto-map)#set transform-set tset1
R1(config)#interface fa0/1
R1(config-if)#crypto map stat-cmap
R1#ping 192.168.2.1 so 192.168.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.2.1, timeout is 2 seconds:
Packet sent with a source address of 192.168.1.1
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 104/142/180 ms
```

```

R2(config)#crypto isakmp enable
R2(config)#crypto isakmp policy 10
R2(config-isakmp)#encryption aes 128
R2(config-isakmp)#hash md5
R2(config-isakmp)#group 2
R2(config-isakmp)#authentication pre-share
R2(config-isakmp)#lifetime 120
R2(config)#crypto isakmp keepalive 30 periodic
R2(config)#crypto isakmp key DERKEY address 10.1.1.1 255.255.255.255 no-xauth
R2(config)#ip access-list extended cacl-r1
R2(config-ext-nacl)#permit ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255 log
R2(config)#crypto ipsec transform-set tset1 esp-aes 128 esp-md5-hmac
R2(cfg-crypto-trans)#mode tunnel
R2(config)#crypto map stat-cmap 100 ipsec-isakmp
R2(config-crypto-map)#match address cacl-r1
R2(config-crypto-map)#set peer 10.1.1.1
R2(config-crypto-map)#set transform-set tset1
R2(config)#interface fa0/1
R2(config-if)#crypto map stat-cmap
```

**case 2: R1 <=> R3 Dynamic crypto map**
```

R1(config)#crypto isakmp enable
R1(config)#crypto isakmp policy 10
R1(config-isakmp)#encryption aes 128
R1(config-isakmp)#hash md5
R1(config-isakmp)#group 2
R1(config-isakmp)#authentication pre-share
R1(config-isakmp)#lifetime 120
R1(config)#crypto isakmp nat keepalive 10
R1(config)#crypto isakmp key DYNKEY address 0.0.0.0 0.0.0.0 no-xauth
R1(config)#ip access-list extended cacl-r3
R1(config-ext-nacl)#permit ip 192.168.1.0 0.0.0.255 192.168.3.0 0.0.0.255
R1(config)#crypto ipsec transform-set tset1 esp-aes 128 esp-md5-hmac
R1(cfg-crypto-trans)#mode tunnel
R1(config)#crypto dynamic-map dyn-cmap 1000
R1(config-crypto-map)#match address cacl-r3
R1(config-crypto-map)#set transform-set tset1
R1(config)#crypto map stat-cmap 200 ipsec-isakmp dynamic dyn-cmap
R1(config)#int fa0/1
R1(config-if)#crypto map stat-cmap
```

```

R3(config)#crypto isakmp enable
R3(config)#crypto isakmp policy 10
R3(config-isakmp)#encryption aes 128
R3(config-isakmp)#hash md5
R3(config-isakmp)#group 2
R3(config-isakmp)#authentication pre-share
R3(config-isakmp)#lifetime 120
R3(config)#crypto isakmp keepalive 30 periodic
R3(config)#crypto isakmp key DERKEY address 10.1.1.1 255.255.255.255 no-xauth
R3(config)#ip access-list extended cacl-r1
R3(config-ext-nacl)#permit ip 192.168.3.0 0.0.0.255 192.168.1.0 0.0.0.255 log
R3(config)#crypto ipsec transform-set tset1 esp-aes 128 esp-md5-hmac
R3(cfg-crypto-trans)#mode tunnel
R3(config)#crypto map stat-cmap 100 ipsec-isakmp
R3(config-crypto-map)#match address cacl-r1
R3(config-crypto-map)#set peer 10.1.1.1
R3(config-crypto-map)#set transform-set tset1
R3(config)#interface fa0/1
R3(config-if)#crypto map stat-cmap
```

** case 3: R1 <=> R2 / R3 Static & Dynamic crypto map**
```

R1(config)#crypto isakmp policy 10
R1(config-isakmp)#encryption aes 128
R1(config-isakmp)#hash md5
R1(config-isakmp)#authentication pre-share
R1(config-isakmp)#group 2
R1(config-isakmp)#lifetime 120
R1(config)#crypto isakmp key DERKEY address 10.2.2.1 no-xauth
R1(config)#crypto isakmp key DYNKEY address 0.0.0.0 0.0.0.0 no-xauth
R1(config)#crypto isakmp keepalive 30 periodic
R1(config)#crypto isakmp nat keepalive 10
R1(config-if)#ip access-list extended cacl-r2
R1(config-ext-nacl)#permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255 log
R1(config-if)#ip access-list extended cacl-r3
R1(config-ext-nacl)#permit ip 192.168.1.0 0.0.0.255 192.168.3.0 0.0.0.255
R1(config)#crypto ipsec transform-set tset1 esp-aes esp-md5-hmac
R1(cfg-crypto-trans)#crypto dynamic-map dyn-cmap 1000
R1(config-crypto-map)#set transform-set tset1
R1(config-crypto-map)#match address cacl-r3
R1(config-crypto-map)#crypto map stat-cmap 100 ipsec-isakmp
R1(config-crypto-map)#set peer 10.2.2.1
R1(config-crypto-map)#set transform-set tset1
R1(config-crypto-map)#match address cacl-r2
R1(config-crypto-map)#crypto map stat-cmap 200 ipsec-isakmp dynamic dyn-cmap
R1(config)#interface FastEthernet0/1
R1(config-if)#crypto map stat-cmap
```

Настройку R2 & R3 можно посмотреть выше.
И…
[![](/wp-content/uploads/2012/07/23844421.jpg "it works")][3]
