---
title: GRE over IPSeс
author: ["Stanislav"]
date: 2012-08-16T16:04:37+00:00
url: /2012/08/setting-up-gre-over-ipsec/
categories:
  - Tech
tags:
  - cisco
  - crypto map
  - gre
  - ipsec
  - ipsec profile
  - vpn
---

Настоятельно рекомендую ознакомится с [предыдущей статьей](/2012/08/tunnels/ "туннели теория"), посвященной теории.

![xzibit about gre over ipsec](/wp-content/uploads/2012/08/xzibit-about-gre-over-ipsec.jpg "xzibit about gre over ipsec")

##### Настройка GRE туннеля

Нет ничего проще, чем настроить GRE туннель:
```

Rx(config)# interface tunnel #_number
Rx(config-if)# tunnel source [ source_router_IP_addr | source_router_interface ]
Rx(config-if)# tunnel destination [ dst_router_IP_addr | dst_router_fqdn ]
Rx(config-if)# keepalive [seconds [ retries ]]
Rx(config-if)# tunnel mode mode
```

**tunnel interface** – создает туннельный интерфейс, в процессе создания туннель автоматически переходит в состояния UP, no shutdown подавать на нем не нужно;
 **tunnel source** – указывает на адрес, который должен будет использован маршрутизатором в IP заголовке GRE пакета;
 **tunnel destination** – указывает, где GRE туннель должен быть терминирован;
 Обратите внимание, что source и destionation адреса – это адреса loopback или физических интерфейсов непосредственно маршрутизатора, а не адреса самого туннельного интерфейса.
 **tunnel mode** – по умолчанию GRE point-to-point. О GRE multipoint мы поговори буквально в следующий раз, когда речь пойдет про DMVPN.
 **keepalive** – по умолчанию выключен, если при включение команде не будут переданы параметры, то keepalive будет отправляться каждые 10 секунд, а после 3-х безуспешных сообщений будет принято решение о недоступности пути.

Сделан **GRE keepalive** до ужаса хитро, а именно: от R1 до R2 GRE пакет с заголовком source R2 destionation R1 инкапсулируется в другой GRE пакет с заголовком source R1 destination R2 и отправляется от R1 до R2 по туннелю. Получив этот пакет R2 его деинкапсулирует и… увидев внутри другой GRE пакет с source R2 destination R1 отправляет его обратно. Элегантное решение, не так ли? :)
 Так же, в контексте следующего раздела, посвященного защите GRE с помощью IPSec хотелось бы упомянуть о нюансах использования keepalive. [Не вдаваясь в подробности](http://www.cisco.com/en/US/tech/tk827/tk369/technologies_tech_note09186a008048cffc.shtml#t7 "Problems with Keepalives when you Combine IPSec and GRE") скажу, что keepalive можно использовать лишь при условии, что оба конца туннеля защищены с помощью crypto map. Если вы используете IPSec профили, то тут лучше использовать протоколы динамической маршрутизации и/или IP SLA.

##### Защита GRE с помощью IPSec

Как стало понятно из последнего абзаца GRE туннель можно защитить или с помощью crypto map или IPSec профилем, примененным непосредственно на туннельном интерфейсе.
 К преимуществам **crypto map** можно отнести способность одного из соседей работать за NAT, а так же возможность использовать GRE keepalive, чтобы определить сбои на пути следования трафика.
 С **IPSec профилями** вам не надо указывать crypto acl и адрес соседа, что упрощает конфигурацию при большом количестве устройств. Оба этих параметра берутся непосредственно из конфигурации туннеля. Однако вместо keepalive, вам придется использовать динамическую маршрутизацию и позаботится о том, чтобы оба соседа имели статические, маршрутизируемые в Интернете адреса.

Стэнд выглядит следующим образом (remote2 за провайдерским NAT):

![](/wp-content/uploads/2012/08/gre-design.png "gre design")

Сначала вдарим по классике и построит отказоустойчивую конфигурацию GRE over IPSec между R1, R2 и remote1, защитив все это IPSec профилями. Настроим туннели:

**R1:**
```

R1(config)# interface Tunnel0
R1(config-if)# ip address 172.16.1.1 255.255.255.252
R1(config-if)# tunnel source FastEthernet0/1
R1(config-if)# tunnel destination 10.2.2.1
R1(config)#router eigrp 50
R1(config-router)# network 172.16.1.0 0.0.0.3
R1(config-router)# network 192.168.1.0
R1(config-router)# redistribute static
```

**R2:**
```

R2(config)# interface Tunnel0
R2(config-if)# ip address 172.16.2.1 255.255.255.252
R2(config-if)# tunnel source FastEthernet0/1
R2(config-if)# tunnel destination 10.2.2.1
R2(config)#router eigrp 50
R2(config-router)# network 172.16.2.0 0.0.0.3
R2(config-router)# network 192.168.1.0
R2(config-router)# redistribute static
```

**remote1:**
```

remote1(config)# interface Tunnel0
remote1(config-if)# ip address 172.16.1.2 255.255.255.252
remote1(config-if)# tunnel source FastEthernet0/1
remote1(config-if)# tunnel destination 10.1.1.1
remote1(config)# interface Tunnel1
remote1(config-if)# ip address 172.16.2.2 255.255.255.252
remote1(config-if)# tunnel source FastEthernet0/1
remote1(config-if)# tunnel destination 10.1.1.2
remote1(config)#router eigrp 50
remote1(config-router)# network 172.16.1.0 0.0.0.3
remote1(config-router)# network 172.16.2.0 0.0.0.3
remote1(config-router)# network 192.168.1.0
```

**Конфигурация IPSec** для всех маршрутизаторов общая и представляет собой:
```

Rx(config)#crypto isakmp policy 10
Rx(config-isakmp)#encryption aes
Rx(config-isakmp)#hash md5
Rx(config-isakmp)#group 2
Rx(config-isakmp)#authentication pre-share
Rx(config)#crypto isakmp key DERKEY address 0.0.0.0 0.0.0.0 no-xauth
Rx(config)#crypto ipsec transform-set trans-tset esp-aes esp-sha-hmac
Rx(cfg-crypto-trans)#mode transport
Rx(config)#crypto ipsec profile PROFILE-1
Rx(ipsec-profile)#set transform-set trans-tset
Rx(ipsec-profile)#exit
Rx(config)#int tun 0
Rx(config-if)#tunnel protection ipsec profile PROFILE-1
```

Главное не забыть на remote1 применить IPSec профиль и на втором туннельном интерфейсе tun1.

Красота решения заключается в том, что вся настройка, в конечном итоге сводиться к созданию туннельных интерфейсов и применении профилей. Никаких тебе crypto ACL, [никаких HSRP с RRI или, что еще хуже, SSO](/2012/08/cisco-ipsec-vpns-high-availability/ "SSO is a pain in the ass"). Все максимально прозрачно. При отказе одного из маршрутизаторов в центральном офисе протокол динамической маршрутизации выполнит всю работу :)

##### Защита GRE c помощью IPSec (dynamic) crypto map

Теперь перейдем к менее радужной части, которая буквально взорвала мне мозг и заставила выложить пару-другую кирпичей, а именно к той ситуации когда один из соседей оказывается за NAT.

Для начала нам надо установить IPSec туннель в tunnel mode с использованием dynamic crypto map. Tunnel mode тут для того, чтобы NAT не покоцал IP заголовок GRE.

![cisco gre dynamic map](/wp-content/uploads/2012/08/cisco-gre-dynamic-map.png "cisco gre dynamic map")

Теперь нам нужен GRE туннель поверх IPSec туннеля (xzibit, yo!).
 Благодаря тому, что с помощью dynamic crypto map мы можем затолкать в R1 со стороны remote2 любой Crypto ACL, описывающий интересующий нас трафик, мы и создадим этот GRE туннель. Со стороны R1 он будет начинаться с общедоступного fa0/1 и терминироваться на loopback интерфейсе remote2. Со стороны remote2, понятное дело, конфигурация будет зеркальная.
 Остался маленький нюанс. Как GRE туннелю установить соседство c loopback0, если путь до loopback remote2, мягко говоря, затруднен?
```

ip route loopback0[remote2] netmask[remote2] fa0/1
```

и пусть у SA второй фазы IPSec болит голова на эту тему :)

**Putting all together**

**R1:**
```

R1(config)#crypto isakmp policy 10
R1(config-isakmp)#encryption aes
R1(config-isakmp)#hash md5
R1(config-isakmp)#group 2
R1(config-isakmp)#authentication pre-share
R1(config)#crypto isakmp key DERKEY address 0.0.0.0 0.0.0.0 no-xauth
R1(config)#crypto ipsec transform-set trans-tset esp-aes esp-sha-hmac
R1(config)#crypto dynamic-map dyn-cmap 10
R1(config-crypto-map)# set transform-set tset
R1(config)#crypto map stat-cmap 10 ipsec-isakmp dynamic dyn-cmap
R1(config)#interface FastEthernet0/1
R1(config-if)# ip address 10.1.1.1 255.255.255.0
R1(config-if)# crypto map stat-cmap
R1(config)#ip route 172.16.3.5 255.255.255.255 FastEthernet0/1
R1(config)#interface Tunnel1
R1(config-if)# ip address 172.16.3.1 255.255.255.252
R1(config-if)# tunnel source FastEthernet0/1
R1(config-if)# tunnel destination 172.16.3.5
R1(config-if)#router eigrp 50
R1(config-router)# network 172.16.3.0 0.0.0.3
R1(config-router)# network 192.168.1.0
```

**remote2:**
```

remote2(config)#crypto isakmp policy 10
remote2(config-isakmp)# encr aes
remote2(config-isakmp)# hash md5
remote2(config-isakmp)# authentication pre-share
remote2(config-isakmp)# group 2
remote2(config)#crypto isakmp key DERKEY address 0.0.0.0 0.0.0.0 no-xauth
remote2(config)#crypto ipsec transform-set tset esp-aes esp-sha-hmac
remote2(config)#ip access-list extended CACL
remote2(config-ext-nacl)# permit gre host 172.16.3.5 host 10.1.1.1
remote2(config)#crypto map stat-cmap 10 ipsec-isakmp
remote2(config-crypto-map)# set peer 10.1.1.1
remote2(config-crypto-map)# set transform-set tset
remote2(config-crypto-map)# match address CACL
remote2(config)#interface Loopback0
remote2(config-if)# ip address 172.16.3.5 255.255.255.252
remote2(config)#interface Tunnel0
remote2(config-if)# ip address 172.16.3.2 255.255.255.252
remote2(config-if)# tunnel source Loopback0
remote2(config-if)# tunnel destination 10.1.1.1
remote2(config)#interface FastEthernet0/0
remote2(config-if)# ip address 192.168.3.1 255.255.255.0
remote2(config)#interface FastEthernet0/1
remote2(config-if)# ip address 172.20.100.1 255.255.255.0
remote2(config-if)# crypto map stat-cmap
remote2(config)#router eigrp 50
remote2(config-router)# network 172.16.3.0 0.0.0.3
remote2(config-router)# network 192.168.3.0
```

GRE сам поднимет IPSec туннель, а eigrp протокол  делает связность мягкой и шелковистой.
