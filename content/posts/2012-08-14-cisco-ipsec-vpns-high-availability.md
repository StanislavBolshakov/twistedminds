---
title: Построение отказоустойчивых IPSec VPN
author: ["Stanislav"]
date: 2012-08-14T09:10:44+00:00
url: /2012/08/cisco-ipsec-vpns-high-availability/
categories:
  - Tech
tags:
  - cisco
  - failover
  - ft
  - ha
  - hsrp
  - ipsec
  - rri
  - stateful failover
---

![advice borat](/wp-content/uploads/2012/08/24883385.jpg "шутка с \"не очень\"")Вот я и подошел к концу освещения темы построение виртуальных частных сетей на чистом IPSec. Последняя вещь, о которой я хотел бы поговорить – это обеспечение отказоустойчивости, а именно HSRP с RRI (chassis failover) и IPSec Stateful failover (transparent failover).

Предполагается что вы знакомы с семейством протоколов резервирования первого перехода (FHRP) и слышали о механизмах их работы.

Хотелось бы так же отметить, что для построения отказоустойчивых VPN Cisco рекомендует смотреть на другие технологии, такие как DMVPN и GETVPN.

##### Reverse Route Injection

RRI был изначально разработан для обеспечения отказоустойчивости Remote Access решений, однако, впоследствии, стал применяется в паре с HSRP для обеспечения избыточности S2S VPN-сессий. Он позволяет статическим маршрутам до сетей, траффик к которым нужно защитить, быть включенным в процесс маршрутизации автоматически.

RRI позволяет управлять метриками. По умолчанию статический маршрут попадает в таблицу с метрикой 1. Если вам требуется, к примеру, чтобы информация, полученная из протоколов динамической маршрутизации, обладала большим весом, можно увеличить административную дистанцию маршрутов, полученных с помощью RRI командой **set reverse-route distance**.

RRI может быть сконфигурирован как с static, так и с dynamic crypto maps, причем со статическими картами маршрут можно добавить в таблицу маршрутизации еще до того, как устанавливается IPSec сессия. RRI накладывает ограничение на возможность применения одной и той же crypto map на нескольких интерфейсах, иными словами – однако crypto map с RRI – один интерфейс.

Маршруты, полученные от RRI, могут быть тегированы. На основе этих тэгов можно проводить выборочную редистрибьюцию в динамические протоколы маршрутизации.

Минимальная конфигурация сводится к следующему – для static crypto map:
```

Rx(config)# crypto map stat_cmap_name #_number ipsec-isakmp
Rx(config-crypto-m)# set peer remote_peer_ip
Rx(config-crypto-m)# match address CACL_number_or_name
Rx(config-crypto-m)# set transform-set tset_name
Rx(config-crypto-m)# reverse-route
```

dynamic crypto map:
```

Rx(config)# crypto dynamic dyn_cmap_name #_number
Rx(config-crypto-m)# set transform-set tset_name
Rx(config-crypto-m)# reverse-route
Rx(config-crypto-m)# exit
Rx(config)# crypto map stat_cmap_name #_nubmer ipsec-isakmp dynamic dyn_cmap_name
```

##### HSRP

HSRP, изначально являющийся один из нескольких протоколов семейства FHRP, был улучшен Cisco для обеспечения других типов избыточности, включая IPSec. HSRP обеспечивает функционирование S2S VPN, даже если одно устройство дало сбой.

Как только происходит сбой активного HSRP маршрутизатора резервный маршрутизатор заменяет его. Существует маленький шанс ситуации split brain, когда маршрутизатор, давших сбой возвращается в оперативный режим работы и оба маршрутизатора становятся активными. Такое может случиться, к примеру, когда на коммутаторе запущен STP и порты, которые смотрят на маршрутизаторы работают без опции **PortFast**. В таком случае хорошим тоном считается конфигурация времени, которое маршрутизатор должен выждать после перезагрузки перед тем, как совершить попытку собрать HSRP группу.
```

Rx(config)# interface interface
Rx(config-if)# standby name HSRP_group_name
Rx(config-if)# standby ip virtual_ip
Rx(config-if)# standby timers hello_timer dead_timer
Rx(config-if)# standby track interface_name
Rx(config-if)# standby preempt
Rx(config-if)# standby delay minimum [minimum_delay] reload [reload_delay]
Rx(config-if)# crypto map stat_cmap_name redundancy [HSRP_group_name]
```

Абсолютным минимумом для конфигурации HSRP группы являются команды **standby name** и **standby ip**, остальные – опциональны.

**Standby track** позволяет отслеживать интерфейс, который, к примеру, смотрит в локальную сеть. Если этот интерфейс по какой-то причине упадет, то приоритет маршрутизатора в группе будет уменьшен на указанное значение.

##### Традиционный стэнд и его конфигурация

![cisco ha vpn hsrp](/wp-content/uploads/2012/08/cisco-ha-vpn-hsrp.png "cisco ha vpn hsrp")
```

CR1(config)#crypto isakmp policy 10
CR1(config-isakmp)#encryption aes
CR1(config-isakmp)#hash md5
CR1(config-isakmp)#authentication pre-share
CR1(config-isakmp)#group 2
CR1(config-isakmp)#exit
CR1(config)#crypto isakmp key hakey123 address 0.0.0.0 0.0.0.0 no-xauth
CR1(config)#crypto ipsec transform-set tset esp-aes esp-sha-hmac
CR1(cfg-crypto-trans)#exit
CR1(config)#crypto isakmp keepalive 10
CR1(config)#crypto isakmp nat keepalive 5
CR1(config)#router eigrp 50
CR1(config-router)#redistribute static
CR1(config-router)#network 192.168.1.0
CR1(config-router)#exit
CR1(config)#crypto dynamic-map dyn-cmap 10
CR1(config-crypto-map)#set transform-set tset
CR1(config-crypto-map)#reverse-route
CR1(config-crypto-map)#exit
CR1(config)#crypto map stat-cmap 10 ipsec-isakmp dynamic dyn-cmap
CR1(config)#int fa0/1
CR1(config-if)#description Internet connection
CR1(config-if)#ip address 10.1.1.1 255.255.255.0
CR1(config-if)#standby ip 10.1.1.3
CR1(config-if)#standby timers 1 4
CR1(config-if)#standby name HA
CR1(config-if)#standby track fastEthernet 0/0 10
CR1(config-if)#crypto map stat-cmap redundancy HA
CR1(config-if)#exit
```

```

CR2(config)#crypto isakmp policy 10
CR2(config-isakmp)# encr aes
CR2(config-isakmp)# hash md5
CR2(config-isakmp)# authentication pre-share
CR2(config-isakmp)# group 2
CR2(config-isakmp)# exit
CR2(config)#crypto isakmp key hakey123 address 0.0.0.0 0.0.0.0 no-xauth
CR2(config)#crypto ipsec transform-set tset esp-aes esp-sha-hmac
CR2(cfg-crypto-trans)#exit
CR2(config)#crypto isakmp keepalive 10
CR2(config)#crypto isakmp nat keepalive 5
CR2(config)#router eigrp 50
CR2(config-router)#redistribute static
CR2(config-router)#network 192.168.1.0
CR2(config-router)#exit
CR2(config)#crypto dynamic-map dyn-cmap 10
CR2(config-crypto-map)#set transform-set tset
CR2(config-crypto-map)#reverse-route
CR2(config-crypto-map)#exit
CR2(config)#crypto map stat-cmap 10 ipsec-isakmp dynamic dyn-cmap
CR2(config)#int fa0/1
CR2(config-if)#description Internet connection
CR2(config-if)#ip address 10.1.1.2 255.255.255.0
CR2(config-if)#standby ip 10.1.1.3
CR2(config-if)#standby timers 1 4
CR2(config-if)#standby name HA
CR2(config-if)#standby track fastEthernet 0/0 10
CR2(config-if)#crypto map stat-cmap redundancy HA
CR2(config-if)#exit
```

```

client1(config)#crypto isakmp policy 10
client1(config-isakmp)# encr aes
client1(config-isakmp)# hash md5
client1(config-isakmp)# authentication pre-share
client1(config-isakmp)# group 2
client1(config-isakmp)#exit
client1(config)#crypto ipsec transform-set tset esp-aes esp-sha-hmac
client1(cfg-crypto-trans)#exit
client1(config)#crypto isakmp keepalive 10
client1(config)#crypto isakmp key hakey123 address 10.1.1.3 no-xauth
client1(config)#crypto map stat-cmap 10 ipsec-isakmp
client1(config-crypto-map)#set peer 10.1.1.3
client1(config-crypto-map)#set transform-set tset
client1(config-crypto-map)#match address IPSEC-HA
client1(config-crypto-map)#exit
client1(config)#ip access-list extended IPSEC-HA
client1(config-ext-nacl)#permit ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255 log
client1(config-ext-nacl)#exit
client1(config)#int fa0/1
client1(config-if)#crypto map stat-cmap
```

Проверим таблицу маршрутизацию на маршрутизаторе **test** с в момент когда оба устройства живы:
```

test#sh ip route eigrp
D*EX  0.0.0.0/0 [170/30720] via 192.168.1.2, 00:00:40, FastEthernet0/1
                [170/30720] via 192.168.1.1, 00:00:40, FastEthernet0/1
D EX  192.168.2.0/24 [170/30720] via 192.168.1.2, 00:00:19, FastEthernet0/1
```

и с упавшим CR2:
```

D*EX  0.0.0.0/0 [170/30720] via 192.168.1.1, 00:02:19, FastEthernet0/1
D EX  192.168.2.0/24 [170/30720] via 192.168.1.1, 00:00:41, FastEthernet0/1
```

##### Отказоустойчивость с сохранением состояния сессий (stateful failover)

Вдобавок к HSRP, который занят мониторингом интерфейсов тут, в дополнении, появляется еще одна технология, трудящаяся с ним бок о бок – stateful switchover (SSO, не путать с single sign on). Именно SSO позволяет активному и резервному маршрутизаторам обмениваться информацией о сессиях фаз 1 и 2.

По понятным причинам у вас должна быть абсолютно идентичная конфигурация маршрутизаторов, включая ISAKMP политики, ISAKMP ключи, IPSec transform sets, IPSec профили, крипто карты, ACL, AAA конфигурация, адресные пулы.

Устройства должны быть однотипными, иметь одинаковую версию операционной системы, одинаковый CPU и равное количество оперативной памяти, а так же обладать одними и те же VPN модули или не иметь их вовсе.

Ограничений, на самом деле, еще больше. Целиком с ними можно ознакомится в Feature Navigator’е на сайте Cisco ->[http://tools.cisco.com/ITDIT/CFN/jsp/index.jsp](http://tools.cisco.com/ITDIT/CFN/jsp/index.jsp "feature navigator")

Для настройки stateful failover надо совершить несколько простых шагов:

1. Настроить HSRP
 2. Настроить SSO
 3. *опционально* Настроить RRI
 4. Включить stateful failover для IPSec
 5. *опционально* Защитить SSO трафик

**Настройка HSRP** – см. предыдущую часть

**Настройка SSO**

Как я уже говорил выше – SSO есть механизм обмена информацией, необходимой для поддержания IPSec сессий. Настроивается SSO так:
```

Rx(config)# redundancy inter-device
Rx(config-red-interdevice)# scheme standby HSRP_group_name
Rx(config-red-interdevice)# exit
Rx(config)# ipc zone default
Rx(config-ipczone)# association assoc_ID
Rx(config-ipczone)# no shutdown
Rx(config-ipc-assoc)# protocol sctp
Rx(config-ipc-protocol-sctp)# local-port local_port
Rx(config-ipc-protocol-sctp-l)# local-ip local_IP
Rx(config-ipc-protocol-sctp-l)# exit
Rx(config-ipc-protocol-sctp)# remote-port remote_port
Rx(config-ipc-protocol-sctp-r)# remote-ip remote_IP
Rx(config-ipc-protocol-sctp-r)# exit
Rx(config-ipc-protocol-sctp)# retransmit-timeout min_msec max_msec
Rx(config-ipc-protocol-sctp)# path-retransmit max_path_retries
Rx(config-ipc-protocol-sctp)# assoc-retransmit max_association_retries
```

**redundancy inter-device** включает SSO;
 **scheme standby** указывает HSRP группу (имя указывается с помощью команды standby name);
 **ipc zone default** конфигурирует SSO протокол и IPC (inter process communication);
 **assotiation**указывает ID процесса, который будет использоваться между двумя устройствами, должен быть одинаковым на обоих устройствах и может принимать значения от 1 до 255;
 **protocol sctp** настраивает stream control transmission protocol, который будет использоваться для передачи SSO сообщений;
 Далее идут команды, которые отнюдь не являются опциональными, как большинство таймеров. **retransmit-timeout (RTO)** указывает на время, которое процесс SCTP будет ожидать перед передачей информации. Для подсчета можно использовать следующую формулу:
```

RTT = ((152*8)/(пропускная способность в битах))*2
```

Где 152 = 20 байт заголовка IP, 32 байта заголовка SCTP и 100 байт полезной нагрузки, 8 – конвертация в биты;
 **path-retransmit (PTO)** указывает на количество последовательных передач, после которого будет принято решение о том, что канал до соседа может быть немного мертв. По умолчанию количество попыток равно 4, может быть увеличено до 210. Например при RTO 1с и PTO 4 общее время конвергенции составит 4 секунды.
 **assoc-retransmit** указывает на количество последовательных передач, после которых будет принято решение о том, что сосед погиб.

Для траблшутинга SSO есть команда **debug redundancy**.

**Настройка RRI** – описывает в предыдущей части.

**Включение Stateful Failover для IPSec**

Может быть включена двумя путями – как непосредственно на crypto map, добавив ключевое слово stateful после имени HSRP группы:
```

Rx(config)# crypto map stat_cmap_name redundancy HSRP_group_name stateful
```

так и на ipsec профиле (VTI, IPIP, DMVPN), например:
```

Rx(config)# crypto ipsec profile profile_name
Rx(ipsec-profile)# redundancy HSRP_group_name stateful
Rx(ipsec-profile)# exit
Rx(config)# interface tunnel #_number
Rx(config-if)# tunnel source {IP_address_on_router}
Rx(config-if)# tunnel destination {IP_address_of_dst_router}
Rx(config-if)# tunnel mode ipsec ipv4
Rx(config-if)# tunnel protection ipsec profile profile_name
```

Опционально можно защитить SSO трафик
```

Rx(config)# crypto isakmp key pre_shared_key address remote_IP no-xauth
Rx(config)# crypto ipsec profile SSO_profile_name
Rx(ipsec-profile)# set transform-set tset_name
Rx(ipsec-profile)# exit
Rx(config)# redundancy inter-device
Rx(config-red-interdevice)# security ipsec SSO_profile_name
Rx(config-red-interdevice)# exit
```

Политики первой фазы, при этом, должны совпадать.

**Мониторинг**

**show redundancy –**показывает текущее состояние SSO; как только два маршрутизатора закончили процедуру установления соседства один из них становится “ACTIVE”, второй “STANDBY HOT”.
 **show crypto ha** – показывает виртуальный IP, используемый соединениями фазы 1 и 2.

Чтобы заработал Stateful failover к предыдущей конфигурации надо добавить следующее:
```

CR1(config)#redundancy inter-device
CR1(config-red-interdevice)#scheme standby HA
CR1(config-red-interdevice)#exit
CR1(config)#ipc zone default
CR1(config-ipczone)#association 1
CR1(config-ipczone-assoc)#no shutdown
CR1(config-ipczone-assoc)#protocol sctp
CR1(config-ipc-protocol-sctp)#local-port 21999
CR1(config-ipc-local-sctp)#local-ip 10.1.1.1
CR1(config-ipc-local-sctp)#exit
CR1(config-ipc-protocol-sctp)#remote-port 21999
CR1(config-ipc-remote-sctp)#remote-ip 10.1.1.2
CR1(config)#int fa0/1
CR1(config-if)#no crypto map
CR1(config-if)#crypto map stat-cmap redundancy HA stateful
```

```

CR2(config)#redundancy inter-device
CR2(config-red-interdevice)#scheme standby HA
CR2(config-red-interdevice)#exit
CR2(config)#ipc zone default
CR2(config-ipczone)#association 1
CR2(config-ipczone-assoc)#no shutdown
CR2(config-ipczone-assoc)#protocol sctp
CR2(config-ipc-protocol-sctp)#local-port 21999
CR2(config-ipc-local-sctp)#local-ip 10.1.1.2
CR2(config-ipc-local-sctp)#exit
CR2(config-ipc-protocol-sctp)#remote-port 21999
CR2(config-ipc-remote-sctp)#remote-ip 10.1.1.1
CR2(config)#int fa0/1
CR2(config-if)#no crypto map
CR2(config-if)#crypto map stat-cmap redundancy HA stateful
```

и перезагрузить устройства. That’s all folks.
