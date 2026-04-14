---
title: Cisco Remote Access IPsec VPN (EzVPN)
author: ["Stanislav"]
date: 2012-07-31T08:39:53+00:00
url: /2012/07/cisco-remote-access-ipsec-vpns/
categories:
  - Tech
tags:
  - cisco
  - easy vpn
  - ezvpn
  - howto
  - ipsec
  - remote access
  - xauth
---

![](/wp-content/uploads/2012/07/vpn_logo_130.jpg "vpn")

Меня всегда интересовало, что же такого легкого в EzVPN, что позволяет Cisco называть его так :) Настройка сервера может быть весьма и весьма развесистой, однако кардинальных отличий от Site-to-Site не так уж и много и заключаются они в так называемой ISAKMP/IKE Phase 1.5, а конкретно в механизме XAUTH.

Cisco Easy VPN оперирует двумя понятиями – Cisco Easy VPN сервер и Cisco Easy VPN Remote (удаленный агент). В качестве агента могут выступать некоторые маршрутизаторы, ASA МСЭ, компьютеры с установленным Cisco VPN Client. OS X и iPhone/iPad имеют встроенные IPSec клиенты поддерживающие XAUTH (racoon).

Я не буду останавливаться на настройке Cisco Systems Easy VPN Client, однако покажу, как можно настроить маршрутизатор в качестве этого самого клиента (что само по себе не очень актуально из-за разнообразия других технологий, обеспечивающих связность удаленных площадок, но может быть полезно для понимания механизмов взаимодействия).

Настройка Site-to-Site не сильно отличается от Remote Access. Похожие моменты это – политики первой фазы ISAKMP/IKE, transform sets, dynamic & static crypto maps. На этих вещах я подробно останавливаться не буду, если есть желание, то перечитайте [статью о site-to-site VPN’ах](/2012/07/cisco-site-to-site-ipsec-vpns/ "building cisco site-to-site ipsec vpns"). И так, приступим, начав с настройки Easy VPN Server.

###### Определимся с AAA

AAA (Authentication, Authorization, Accounting) используется Easy VPN для применения групповых политик и авторизации, а так же для аутентификации пользователей (XAUTH). Easy VPN сервер может получать информацию о пользователях и группах либо из локальной базы, либо с radius-сервера. Смысл в последнем появляется, когда у вас большая база пользователей и пара-другая точек входа. Настройка AAA сводится к следующему:
```

Rx(config)# aaa new-model
Rx(config)# aaa authentication login authentication_list method1 [method2...]
Rx(config)# aaa authorization network authorization_list method1 [method2...]
Rx(config)# username username [ password | secret ] password
```

Команда **aaa new-model** включает AAA на маршрутизаторе.
 **aaa authentication login** позволяет пользователям проходить XAUTH аутентификацию (так называемая фаза 1.5) на маршрутизаторе. *authentication_list* будет использоваться далее как один из параметров crypto map или ISAKMP профиля. В методах можно использовать локальную базу пользователей (в этом случае база заполняется с помощью команды username …) и/или radius сервер. Можно указать несколько методов аутентификации, которые маршрутизатор будет использовать в порядке указанном вами.
 **aaa authorization network** указывает AAA сервер, который будет использоваться для обработки политик IPSec remote access групп. Так же можно указать local базу, radius базу или обе.

###### Создание групп

Если вы выбрали метод авторизации ***local***для **aaa authorization network,**то совсем не лишним будет создать группы на маршрутизаторе. Делается это так:
```

Rx(config)# ip local pool VPN_POOL 1st_ip_addr 2nd_ip_addr
Rx(config)# crypto isakmp client configuration address-pool local VPN_POOL
Rx(config)# crypto isamkp client configuration group [ group_name | default ]
Rx(config-isakmp-group)# key pre_shared_key
Rx(config-isakmp-group)# pool VPN_POOL
Rx(config-isakmp-group)# domain fqdn.domain.local
Rx(config-isakmp-group)# dns 1st_DNS_server [2nd_DNS_server]
Rx(config-isakmp-group)# split-dns fqdn.domain.local
Rx(config-isakmp-group)# wins 1st_WINS_server [2nd_WINS_server]
Rx(config-isakmp-group)# include-local-lan
Rx(config-isakmp-group)# netmask mask
Rx(config-isakmp-group)# acl ACL_name_or_#
Rx(config-isakmp-group)# backup-gateway [ IP_address | hostname ]
Rx(config-isakmp-group)# save-password
Rx(config-isakmp-group)# pfs
Rx(config-isakmp-group)# max-logins #_of_simultaneous_logins
Rx(config-isakmp-group)# max-users #_of_users
Rx(config-isakmp-group)# access-restrict interface_name
Rx(config-isakmp-group)# group-lock
Rx(config-isakmp-group)# exit
```

*Объяснение очевидных настроек опущено.*

**ip local pool** – создает пул адресов, которые будут назначены пользователям remote access VPN.
 **crypto isakmp client configuration address-pool local *VPN_POOL***– позволяет вам связать адресный пул, созданный с помощью команды ip local pool. Если в дальнейшем при конфигурации remote access группы вы не укажите пул адресов, то будет использоваться пул, указанный с помощью этой команды. Так же эта команда позволяет нескольким группам использовать один и тот же пул.
 **crypto isamkp client configuration group** – непосредственно создает группу, которая будет использоваться в процессе авторизации.
 **key** – pre-shared ключ, который будет использоваться в процессе авторизации.
 **pool** – указывает пул из которого будут выделяться ip для подключения Remote Access клиентов.
 **split-dns** – при включении этой опции DNS запросы к домену будут направлены на внутренние DNS сервера, все остальные запросы будут обслуживаться DNS серверами провайдера.
 **include-local-lan** – клиент получает доступ к сетям, присутствующим в его локальной таблице маршрутизации без включения опции split-tunnel.
 **netmask** – назначает клиенту маску подсети для виртуального адаптера.
 **acl** – опция, иначе называемая split-tunnel. Клиенту сообщается список сетей, до которых нужно шифровать трафик. Весь остальной трафик будет отправляться в не шифрованном виде по общим сетям передачи данных. **Обратите внимание на то, что настраивается этот split-tunnel ACL с точки зрения маршрутизатора.** К примеру, мы хотим чтобы клиенты отправляли трафик только до сети 192.168.1.0/24, которая находится за нашим маршрутизатором-EasyVPN сервером, по зашифрованному каналу:
```

Rx(config)# ip access-list extended split-tunnel
Rx(config-ext-nacl)# remark define protected traffic
Rx(config-ext-nacl)# permit ip 192.168.1.0 0.0.0.255 any
```

**backup-gateway** – сообщает клиенту резервный Easy VPN сервер. Можно указать до 10 серверов.
 **save-password** – позволяет клиенту сохранять пароль, который указывается в процессе аутентификации (XAUTH).
 **pfs** – опция, указывающая на то, что для каждого обмена второй фазы будет использована новая пара ключей, созданных по алгоритму DH
 **access-restrict** – указывает на интерфейсы, которые могут использовать пользователи указанной группа для терминирования защищенных каналов.
 **group-lock** – опция, заставляющая клиента вводить не только имя пользователя и пароль, но и имя группы с pre-shared ключом.

###### Dynamic & Static Crypto maps

Последнее что осталось – это связать все настройки в крипто карты и применить их на нужном интерфейсе.

**Dynamic crypto map**
```

Rx(config)# crypto dynamic-map dynamic_map_name #_number
Rx(config-crypto-m)# set transform-set tset_name [ tset_name1 ... tset_name6]
Rx(cofnig-crypto-m)# reverse-route
```

Команда reverse-route, напомню, отвечает за функционал RRI (reverse route injection). При ее подаче в режиме конфигурации динамической крипто карты внутренний адрес клиента (client mode) или частный адрес сети (network extension mode) появляется как статический маршрут в таблице маршрутизации Easy VPN сервера.

**Static crypto map и XAUTH**

Для начало нам надо связать dynamic crypto map с static crypto map:
```

Rx(config)# crypto map stat-cmap-name seq_# ipsec-isakmp dynamic dyn-cmap-name
```

Если вы планируете использоваться и dynamic и static crypto map, то назначайте динамическим картам номера выше, чем статическим. Одновременную работу S2S & remote access VPN (функционал ISAKMP профилей) [мы рассмотрим в следующей статье](/2012/08/cisco-isakmp-profiles/ "cisco site-to-site vpn and remote access ezvpn on the same router").

Затем нам надо указать группу, которая будет использоваться для авторизации в данной крипто карте:
```

Rx(config)# crypto map stat-cmap-name isakmp authorization list authorization_list
```

Укажем маршрутизатору, где искать данные, для аутентификации пользователей:
```

Rx(config)# crypto map stat-cmap-name client authentication list authentication_list
```

Дальше решается, кто должен инициировать применение политик IKE Mode Config – сервер (ключ initiate) или клиент (ключ respond):
```

Rx(config)# crypto map stat-cmap-name client configuration address [initiate | respond]
```

Программные и аппаратные Cisco Easy VPN клиенты сами инициируют IKE Mode Config. Ключ initiate видимо остался для совместимости с какими-то старыми реализациями протокола.

Альтернативно можно указать время, которое отводится пользователю для прохождения аутентификации командой  **crypto isakmp xauth timeout** ***seconds.***Значение по умолчанию равно 60 секундам.

###### Мониторинг

**show crypto session group** – показывает активные группы и количество подключенных клиентов
 **show crypto session brief –** показывает активные группы и пользователей

На этом настройка Cisco Easy VPN Server окончена и после применения static crypto map на интерфейса пользователи cisco easy vpn client должны получить возможность подключаться к вашей сети. Настройку клиента я опущу из-за ее очевидности, а вот на конфигурации аппаратных клиентов мы остановимся подробнее.

###### Настройка Cisco Easy VPN Remote

В качестве Cisco Easy VPN Client могут выступать не только программные клиенты, но и вполне себе взрослые маршрутизаторы. Настройка их весьма проста по сравнению с сервером и единственное что стоит держать в голове это три режима работы Easy VPN клиента – client mode, network extension mode, network extension plus mode.

Главное ограничение client mode состоит в том, что устройства или пользователи, которые находятся за Cisco Easy VPN Server не могут инициировать подключения к устройствам, которые находятся за Cisco Easy VPN Client. В client mode NAT/PAT настраивается автоматически.

Network extension mode симулирует S2S подключение, позволяя устройствам, находящимся за Cisco Easy VPN Server получать доступ к ресурсам за Cisco Easy VPN Client. В этом случае никакие адреса из внутреннего диапазона не назначаются клиенту. Каждая сеть за клиентом, работающим в network extension mode, должна быть уникальной по понятным причинам.

Network extension plus mode – есть комбинация из client mode и network extension mode. В процессе IKE Mode Config loopback адресу маршрутизатора назначается адрес из пула.

Непосредственно настройка сводится к следующему – указывается DHCP Pool (по желанию):
```

Rx(config)# ip dhcp pool DHCP_POOL
Rx(dhcp-config)# network IP_network [subnet_mask | /prefix_length ]
Rx(dhcp-config)# default-router Rx_ip_addr
Rx(dhcp-config)# domain-name fqdn.domain.local
Rx(dhcp-config)# dns-server 1st_DNS_server 2nd_DNS_server
Rx(dhcp-config)# netbios-name-server 1st_WINS_server 2nd_WINS_server
Rx(dhcp-config)# lease [ days [hours [minutes]] | infinite ]
```

Создается Easy VPN Client профиль и активируется на интерфейсах:
```

Rx(config)# crypto ipsec client ezvpn profile_name
Rx(config-crypto-ezvpn)# group group_name key group_password_key
Rx(config-crypto-ezvpn)# peer EzVPN_server_remote_ip_addr
Rx(config-crypto-ezvpn)# mode [ client | network-extension ]
Rx(config-crypto-ezvpn)# connect [auto | manual]
Rx(config-crypto-ezvpn)# username username password password
Rx(config)# interface interface
Rx(config-if)# crypto ipsec client ezvpn profile_name [ outside | inside ]
```

Если групповой профиль на Easy VPN сервере сконфигурирован без опции save-password, позволяющей клиентам сохранять пароль, то толку от настройки клиента
```

Rx(config-crypto-ezvpn)# username username password password
```

не будет. Вам будет предложено руками в режиме user или exec mode указать имя пользователя и пароль с помощью команды **crypto ipsec client ezvpn xauth**.

###### Мониторинг и выявление неполадок Cisco Easy VPN Client (Remote)

**show crypto ipsec client ezvpn** – отображает информацию о текущем состоянии активных туннелей включая политики, полученные с сервера в процессе IKE Config Mode.
 **show ip nat statistics** – отображает настройку NAT/PAT клиента работающего в client mode.
 **debug crypto ipsec client ezvpn** – помогает выявлять неполадки в конфигурации Cisco Easy VPN.
 **clear crypto ipsec client ezvpn** – удаляет все активные туннели.

Ну и традиционно сочные конфиги для самых терпеливых:

![cisco remote access vpn network map](/wp-content/uploads/2012/07/cisco-remote-access-vpn.png "cisco remote access vpn network map")

network map

По легенде RA1 работает в client mode и на группу, в которую он входит применен ACL разрешающий ему отправлять трафик, который не предназначен для 192.168.1.0/24 в обход защищенного канала передачи информации. RA2 работает в network extension mode и весь трафик будет отправлен в зашифрованный канал.

**main (Cisco Easy VPN Server) configuration:**
```

main(config)# aaa new-model
main(config)# aaa authentication login ra-usrs local
main(config)# aaa authorization network ra-grps local
main(config)# username ra1 secret ra1pwd
main(config)# username ra2 secret ra2pwd
main(config)# crypto isakmp policy 100
main(config-isakmp)# encryption aes 128
main(config-isakmp)# hash sha
main(config-isakmp)# authentication pre-share
main(config-isakmp)# group 2
main(config-isakmp)# exit
main(config)# crypto isakmp keepalive 20 3
main(config)# crypto isakmp nat keepalive adasdasd
main(config)# ip local pool ra1pool 192.168.20.10 192.168.20.20
main(config)# ip access-list extended split-acl
main(config-ext-nacl)# permit ip 192.168.1.0 0.0.0.255 any
main(config-ext-nacl)# exit
main(config)# crypto isakmp client configuration group ra1_grp
main(config-isakmp-group)# key ra1key
main(config-isakmp-group)# save-password
main(config-isakmp-group)# pool ra1pool
main(config-isakmp-group)# domain fqdn.domain.ltd
main(config-isakmp-group)# acl split-acl
main(config-isakmp-group)# exit
main(config)# crypto isakmp client configuration group ra2_grp
main(config-isakmp-group)# key ra2key
main(config-isakmp-group)# save-password
main(config-isakmp-group)# domain fqdn.domain.ltd
main(config-isakmp-group)# exit
main(config)# crypto ipsec transform-set tset1 esp-aes esp-sha-hmac
main(cfg-crypto-tran)# exit
main(config)# crypto dynamic-map dyn-cmap 10
main(config-crypto-m)# set transform-set tset1
main(config-crypto-m)# reverse-route
main(config-crypto-m)# exit
main(config)# crypto map stat-cmap client authentication list ra-usrs
main(config)# crypto map stat-cmap isakmp authorization list ra-grps
main(config)# crypto map stat-cmap client configuration address respond
main(config)# crypto map stat-cmap 10 ipsec-isakmp dynamic dyn-cmap
main(config)# interface fa0/0
main(config-if)# description Local LAN
main(config-if)# ip address 192.168.1.1 255.255.255.0
main(config-if)# exit
main(config)# interface fa0/1
main(config-if)# description Internet Connection
main(config-if)# ip address 10.1.1.1 255.255.255.0
main(config-if)# crypto map stat-cmap
```

**RA1 (Cisco Easy VPN Client) configuration – client mode:**
```

RA1(config)# crypto ipsec client ezvpn ra1
RA1(config-crypto-ezvpn)# connect auto
RA1(config-crypto-ezvpn)# mode client
RA1(config-crypto-ezvpn)# peer 10.1.1.1
RA1(config-crypto-ezvpn)# username ra1 password ra1pwd
RA1(config-crypto-ezvpn)# group ra1_grp key ra1key
RA1(config-crypto-ezvpn)# xauth userid mode local
RA1(config-crypto-ezvpn)# exit
RA1(config)# interface FastEthernet0/0
RA1(config-if)# ip address 192.168.4.1 255.255.255.0
RA1(config-if)# crypto ipsec client ezvpn ra1 inside
RA1(config-if)# exit
RA1(config)# interface FastEthernet0/1
RA1(config-if)# ip address 172.16.20.1 255.255.255.0
RA1(config-if)# crypto ipsec client ezvpn ra1
```

```

*Mar 1 04:36:04.650: %CRYPTO-6-ISAKMP_ON_OFF: ISAKMP is ON
*Mar 1 04:36:05.734: %CRYPTO-6-EZVPN_CONNECTION_UP: (Client) User=ra1 Group=ra1_grp Client_public_addr=172.16.20.1 Server_public_addr=10.1.1.1 Assigned_client_addr=192.168.20.10
*Mar 1 04:36:07.390: %LINK-3-UPDOWN: Interface Loopback0, changed state to up
*Mar 1 04:36:08.390: %LINEPROTO-5-UPDOWN: Line protocol on Interface Loopback0, changed state to up
```

**RA2 (Cisco Easy VPN Client) configuration – network extension mode:**
```

RA2(config)# crypto ipsec client ezvpn ra2
RA2(config-crypto-ezvpn)# connect auto
RA2(config-crypto-ezvpn)# group ra2_grp key ra2key
RA2(config-crypto-ezvpn)# mode network-extension
RA2(config-crypto-ezvpn)# peer 10.1.1.1
RA2(config-crypto-ezvpn)# username ra2 password ra2pwd
RA2(config-crypto-ezvpn)# xauth userid mode local
RA2(config-crypto-ezvpn)# exit
RA2(config)# int fa0/1
RA1(config-if)# ip address 172.16.30.1 255.255.255.0
RA2(config-if)# crypto ipsec client ezvpn ra2 outside
RA2(config)#int fa0/0
RA1(config-if)# ip address 192.168.5.1 255.255.255.0
RA2(config-if)# crypto ipsec client ezvpn ra2 inside
```

```

*Mar 1 00:31:54.643: %CRYPTO-6-ISAKMP_ON_OFF: ISAKMP is ON
*Mar 1 00:31:55.431: %CRYPTO-4-IKMP_NO_SA: IKE message from 10.1.1.1 has no SA and is not an initialization offer
*Mar 1 00:31:56.351: %CRYPTO-6-EZVPN_CONNECTION_UP: (Client) User=ra2 Group=ra2_grp Client_public_addr=172.16.30.1 Server_public_addr=10.1.1.1 NEM_Remote_Subnets=192.168.5.0/255.255.255.0
*Mar 1 00:31:56.999: %LINEPROTO-5-UPDOWN: Line protocol on Interface NVI0, changed state to up
```

Вот такой вот Easy VPN от Cisco Systems.![cisco y u no](/wp-content/uploads/2012/07/24206552.jpg "cisco y u no")
