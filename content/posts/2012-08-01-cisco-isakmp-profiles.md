---
title: 'Cisco ISAKMP профили: S2S и Remote Access VPN на одном маршрутизаторе'
author: ["Stanislav"]
date: 2012-08-01T10:42:31+00:00
url: /2012/08/cisco-isakmp-profiles/
categories:
  - Tech
tags:
  - cisco
  - easy vpn
  - ezvpn
  - ike
  - ipsec
  - isakmp
  - isakmp profile
---

![](/wp-content/uploads/2012/07/vpn_logo_130.jpg "vpn")Настало время поговорить о материях, наиболее близких к реальной жизни, а именно о моменте когда нам надо обеспечить работу одновременно S2S и Remote Access VPN’ов на одной маршрутизаторе. В этой статье речь пойдет о ISAKMP профилях.

Представим: у вас есть удаленная площадка за NAT и мобильные пользователи, которым надо предоставить доступ в ЛВС. Из-за того, что маршрутизатор в центральном офисе понятия не имеет какой адрес получит сосед, находящийся за NAT, нам придется использовать wild carded pre-shared key (0.0.0.0 0.0.0.0 no-xauth). Но как только мы подадим эту команду, так сразу наши remote access клиенты и превратятся в тыкву. Одним из вариантов решения проблемы могло бы стать использование цифровых сертификатов, однако согласитесь, разворачивать PKI ради двух-трех площадок есть существенный overkill.

Другим вариантом могут стать ISAKMP профили, о которых мы сейчас и будем говорить.

В дополнении к конфигурации, с которыми вы познакомились в заметках про S2S и Remote Access VPN вам предстоит выполнить пару лишних телодвижений, а именно:

* Указать список pre-shared ключей (брелок, keyring), которые будут использоваться соседями.
* Создать профили для S2S (Site-to-Site, Lan-to-Lan, L2L) соседей.
* Создать профили для Remote Access клиентов.
* Создать dynamic crypto map, которая включает в себя записи о ISAKMP/IKE профилях первой фазы для S2S соседей и Remote Access клиентов.

###### Keyrings

Списки pre-shared ключей создаются следующим образом:
```

Rx(config)# crypto keyring name
Rx(conf-keyring)# description description
Rx(conf-keyring)# pre-shared-key address address [subnet_mask] key key
```

###### S2S ISAKMP/IKE профиль

Для клиентов, которые получают адрес динамически, создадим ISAKMP профиль:
```

Rx(config)# crypto isakmp profile ISAKMP_profile
Rx(conf-isa-prof)# description description
Rx(conf-isa-prof)# match identity address IP_addr
Rx(conf-isa-prof)# self-identity [ address | fqdn ]
Rx(conf-isa-prof)# keyring keyring_name
Rx(conf-isa-prof)# keepalive seconds
```

**crypto isakmp profile** – создает профиль для первой фазы ISAKMP/IKE. Каждый профиль должен обладать уникальным именем.
 **match identitiy** – указывает каким образом нужно сопоставлять соседей с этим профилем. В качестве IP адреса можно указывать wild card маски в том случае если адрес заранее не известен. Альтернативно можно указать адрес хоста (fqdn.domain.ltd) или даже адрес домена (domain.ltd).
 **self-identity –** указывает каким образом наш маршрутизатор будет “представляться” соседу – по IP адресу или по доменному имени.

ISAKMP профили можно в дальнейшем использовать для построения DMVPN, подав в режиме конфигурации профиля команду set isakmp-profile, указывающую на нужный ISAKMP/IKE профиль первой фазы. Подробнее в другой статье :)

###### Remote Access ISAKMP/IKE профиль

Вместо keyring нам нужно будет указать на группу, а так же сообщить маршрутизатору как проходить и где брать сведения для XAUTH аутентификации и авторизации. Не забудем так же рассказать бездушной железке о том, стоит ли ей самой посылать IKE Config Mode информацию или отвечать на запрос с помощью опции **client configuration address**.
```

Rx(conf)# crypto isakmp profile ISAKMP_profile
Rx(conf-isa-prof)# description description
Rx(conf-isa-prof)# match identity group group_name
Rx(conf-isa-prof)# self-identity [ address | fqdn ]
Rx(conf-isa-prof)# client authentication list authentication_list
Rx(conf-isa-prof)# isakmp authorization list group_name
Rx(conf-isa-prof)# client configuration address [ respond | initiate ]
Rx(conf-isa-prof)# keepalive seconds
```

###### Связываем ISAKMP профиль и dynamic crypto map

Порядок, в котором вы разместите S2S и Remote Access профили в dynamic crypto map имеет существенно значение. Remote Access записи должны располагаться с приоритетом выше (а значит номером последовательности ниже) чем S2S, иначе remote access клиенты попадут под wild-carded записи pre-shared ключей в keyring и XAUTH аутентификация провалится.

Чтобы создать dynamic crypto map нужны всего две записи – transform set и указание на ISAKMP профиль, ради которых весь сыр-бор и был затеян:
```

Rx(config)# crypto dynamic-map dyn-cmap #_number
Rx(config-crypto-map)# set transform-set transform_set
Rx(config-crypto-map)# set isakmp-profile ISAKMP_profile
```

А как привязывать dynamic crypto map к static и применять последнюю на интерфейсы вы уже знаете :)

###### Практика

По легенде маршрутизатор main в центральном офисе будет терминировать два Site-to-Site соединения с помощью dynamic (dyns2s) и static (stats2s) crypto map, а так же Remote Access подключение от удаленного офиса (RA1) в режиме network extension plus mode. При чем RA1 и dyns2s располагаются за NAT/PAT одного провайдера.

![cisco isakmp profile topology](/wp-content/uploads/2012/07/cisco-isakmp-profile.png "cisco isakmp profile topology")

топология сети

**main**:
```

main(config)# username ra1 password ra1pwd
main(config)# aaa authentication login rausrs local
main(config)# aaa authorization network grpauth local
main(config)# crypto isakmp policy 10
main(config-isakmp)# encryption aes 256
main(config-isakmp)# hash sha
main(config-isakmp)# authentication pre-share
main(config-isakmp)# group 2
main(config-isakmp)# exit
main(config)# ip local pool RAPOOL 192.168.10.10 192.168.10.100
main(config)# crypto isa client configuration group ra1grp
main(config-isakmp-group)# key ra1key
main(config-isakmp-group)# save-password
main(config-isakmp-group)# domain fqdn.domain.ltd
main(config-isakmp-group)# pool RAPOOL
main(config-isakmp-group)# exit
main(config)# crypto isakmp profile RA-profile
main(conf-isa-prof)# description profile for remote access VPN
main(conf-isa-prof)# match identity group ra1grp
main(conf-isa-prof)# client authentication list rausrs
main(conf-isa-prof)# isakmp authorization list grpauth
main(conf-isa-prof)# client configuration address respond
main(conf-isa-prof)# keepalive 20 retry 3
main(conf-isa-prof)# exit
main(config)# crypto keyring dyn-L2L-keyring
main(conf-keyring)# description pre-shared keys for users with unknown ip addr
main(conf-keyring)# pre-shared-key addr 0.0.0.0 0.0.0.0 key dynkey
main(conf-keyring)# exit
main(config)# crypto isakmp profile dyn-L2L-profile
main(conf-isa-prof)# description profile for dyn L2L peers
main(conf-isa-prof)# keyring dyn-L2L-keyring
main(conf-isa-prof)# match identity address 0.0.0.0
main(conf-isa-prof)# keepalive 20 retry 3
main(conf-isa-prof)# exit
main(config)# crypto ipsec transform-set tset esp-aes esp-sha-hmac
main(cfg-crypto-trans)# exit
main(config)# ip access-list extended IPSEC-stats2s
main(config-ext-nacl)# permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255 log
main(config-ext-nacl)# exit
main(config)# ip access-list extended IPSEC-dyns2s
main(config-ext-nacl)# permit ip 192.168.1.0 0.0.0.255 192.168.3.0 0.0.0.255 log
main(config-ext-nacl)# exit
main(config)# crypto keyring stat-L2L-keyring
main(conf-keyring)# description keyring for real ip peers
main(conf-keyring)# pre-shared-key address 10.2.2.1 key statkey
main(conf-keyring)# exit
main(config)# crypto isakmp profile stat-L2L-profile
main(conf-isa-prof)# match identity address 10.2.2.1
main(conf-isa-prof)# keyring stat-L2L-keyring
main(conf-isa-prof)# exit
main(config)#crypto map stat-cmap 10 ipsec-isakmp
main(config-crypto-map)# set peer 10.2.2.1
main(config-crypto-map)# set transform-set tset
main(config-crypto-map)# set isakmp-profile stat-L2L-profile1
main(config-crypto-map)# match address IPSEC-stats2s
main(config)# crypto dynamic-map dyn-cmap 100
main(config-crypto-map)# set transform-set tset
main(config-crypto-map)# set isakmp-profile RA-profile
main(config-crypto-map)# exit
main(config)# crypto dynamic-map dyn-cmap 200
main(config-crypto-map)# set transform-set tset
main(config-crypto-map)# set isakmp-profile dyn-L2L-profile
main(config-crypto-map)# exit
main(config)# crypto map stat-cmap 100 ipsec-isa dynamic dyn-cmap
main(config)# int fa0/1
main(config-if)# crypto map stat-cmap
```

**stats2s**:
```

stats2s(config)# crypto isakmp policy 10
stats2s(config-isakmp)# encryption aes 256
stats2s(config-isakmp)# hash sha
stats2s(config-isakmp)# authentication pre-share
stats2s(config-isakmp)# group 2
stats2s(config-isakmp)# exit
stats2s(config)# crypto isakmp key statkey address 10.1.1.1 no-xauth
stats2s(config)# ip access-list extended IPSEC-main
stats2s(config-ext-nacl)# permit ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255 log
stats2s(config-ext-nacl)# exit
stats2s(config)# crypto ipsec transform-set tset esp-aes esp-sha-hmac
stats2s(cfg-crypto-trans)# exit
stats2s(config)# crypto map stat-cmap 100 ipsec-isakmp
stats2s(config-crypto-map)# set peer 10.1.1.1
stats2s(config-crypto-map)# set transform-set tset
stats2s(config-crypto-map)# match address IPSEC-main
stats2s(config-crypto-map)# exit
stats2s(config)# int fa0/1
stats2s(config-if)# crypto map stat-cmap
```

**dyns2s**:
```

dyns2s(config)# crypto isakmp policy 10
dyns2s(config-isakmp)# encryption aes 256
dyns2s(config-isakmp)# hash sha
dyns2s(config-isakmp)# authentication pre-share
dyns2s(config-isakmp)# group 2
dyns2s(config-isakmp)# exit
dyns2s(config)# crypto isakmp key dynkey address 10.1.1.1 no-xauth
dyns2s(config)# ip access-list extended IPSEC-main
dyns2s(config-ext-nacl)# permit ip 192.168.3.0 0.0.0.255 192.168.1.0 0.0.0.255 log
dyns2s(config-ext-nacl)# exit
dyns2s(config)# crypto ipsec transform-set tset esp-aes esp-sha-hmac
dyns2s(cfg-crypto-trans)# exit
dyns2s(config)# crypto map stat-cmap 100 ipsec-isakmp
dyns2s(config-crypto-map)# set peer 10.1.1.1
dyns2s(config-crypto-map)# set transform-set tset
dyns2s(config-crypto-map)# match address IPSEC-main
dyns2s(config-crypto-map)# exit
dyns2s(config)# int fa0/1
dyns2s(config-if)# crypto map stat-cmap
```

**ra1**:
```

RA1(config)# crypto ipsec client ezvpn ra1
RA1(config-crypto-ezvpn)# connect auto
RA1(config-crypto-ezvpn)# mode network-plus
RA1(config-crypto-ezvpn)# username ra1 password ra1pwd
RA1(config-crypto-ezvpn)# group ra1grp key ra1key
RA1(config-crypto-ezvpn)# peer 10.1.1.1
RA1(config-crypto-ezvpn)# xauth userid mode local
RA1(config-crypto-ezvpn)# exit
RA1(config)# int fa0/1
RA1(config-if)# crypto ipsec client ezvpn ra1 outside
RA1(config-if)# exit
RA1(config)# int fa0/0
RA1(config-if)# crypto ipsec client ezvpn ra1 inside
RA1(config-if)# exit
```

И все ради заветных трех строчек в выхлопе **sh crypto session breif**:
```

main#sh cry se br
Status: A- Active, U - Up, D - Down, I - Idle, S - Standby, N - Negotiating
        K - No IKE
ivrf = (none)
           Peer     I/F        Username          Group/Phase1_id   Uptime Status
       10.3.3.1   Fa0/1             ra1                   ra1grp 00:17:31    UA
       10.2.2.1   Fa0/1                                 10.2.2.1 00:15:43    UA
       10.3.3.1   Fa0/1                              172.16.10.1 00:14:48    UA
```
