---
title: Site-to-Site IPSec VPN между Cisco и Mikrotik
author: ["Stanislav"]
date: 2012-08-02T18:21:42+00:00
url: /2012/08/s2s-ipsec-vpn-mikrotik-cisco/
categories:
  - Tech
tags:
  - cisco
  - ipsec
  - mikrotik
  - s2s
  - site-to-site
  - vpn
---

Не смотря на то, что статей в этих ваших интернетах море я все равно внесу немного энтропии и напишу еще одну.

Я всегда с осторожностью отношусь к хвалебным отзывам и хайпу на форумах, особенно когда дело касается железки стоимостью 60$. Как оказалось, делаю я это совсем не зря. Mikrotik не подвел и порадовал таким набором фееричных багов, что пару раз белому мерзавцу грозила неминуемая смерть.

Стабильной работы удалось добиться с следующими параметрами:

Phase1&2: шифрование: aes 128, аутентификация, md5, DH group 2 на прошивке 5.19

Если вы наблюдаете печальную картину того, как разваливается управляющее соединения первой фазы, SA второй фазы остаюся активными, а трафик перестает ходить, то пора нервничать поиграйте с параметрами соединения, временем жизни SA, а так же версией прошивки. Если и это не помогает, то по поисковому запросу **mikrotik flush SA** на официальных форумах есть несколько рецептов того, как с помощью костылей в виде скрипта, крона и какой-то матери проблему эту если и не решить, то уж точно нивелировать.

**Собственно настройки:**

За маршрутизатором Cisco существуют две подсети 192.168.0.0/23 и 172.16.20.0/24, к которым не плохо было бы иметь доступ. Mikrotik расположен за динамическим NAT’ом и локальная подсеть предствалена 172.16.100.0/24
```

! связка ключей для микротика
crypto keyring mikrotik
pre-shared-key address 0.0.0.0 0.0.0.0 key derkey
! политики первой фазы
crypto isakmp policy 10
 encr aes
 hash md5
 authentication pre-share
 group 2
! профиль
crypto isakmp profile mikrotik-profile
keyring mikrotik
match identity address 0.0.0.0
no keepalive
! политики второй фазы
crypto ipsec transform-set ESP-AES-MD5 esp-aes esp-md5-hmac
! криптокарта
crypto dynamic-map dyn-cmap 2000
 set transform-set ESP-AES-MD5
 set isakmp-profile mikrotik-profile
 reverse-route
! связываем динамическую карту с статической
crypto map stat-cmap 100 ipsec-isakmp dynamic dyn-cmap
! применяем ее на интерфейсе
interface gi0/0
 crypto map stat-cmap
```

По желанию на криптокарту можно привязать ACL.

На стороне микротика добавляем соседа (фаза 1), не забыв включить NAT-T:
```

/ip ipsec peer
add address=xx.xx.xx.xx/32 auth-method=pre-shared-key comment=remote-peer dh-group=modp1024 disabled=no dpd-interval=5s dpd-maximum-failures=10 enc-algorithm=aes-128 \
    exchange-mode=aggressive generate-policy=no hash-algorithm=md5 lifebytes=0 lifetime=1h my-id-user-fqdn="" nat-traversal=yes port=500 proposal-check=obey secret=\
    derkey send-initial-contact=yes
```

Добавляем transorm set (proposal):
```

/ip ipsec proposal
add auth-algorithms=md5 disabled=no enc-algorithms=aes-128 lifetime=1h name=tset1 pfs-group=none
```

И в политиках описываем интересующий нас траффик:
```

/ip ipsec policy
add action=encrypt comment="servers vlan" disabled=no dst-address=172.16.20.0/24 dst-port=any ipsec-protocols=esp level=unique priority=1 proposal=tset protocol=all \
    sa-dst-address=xx.xx.xx.xx sa-src-address=0.0.0.0 src-address=172.16.100.0/24 src-port=any tunnel=yes
add action=encrypt comment="generic subnet" disabled=no dst-address=192.168.0.0/23 dst-port=any ipsec-protocols=esp level=unique priority=0 proposal=tset protocol=all \
    sa-dst-address=xx.xx.xx.xx sa-src-address=0.0.0.0 src-address=172.16.100.0/24 src-port=any tunnel=yes
```

Обратите внимение на опцию level=unique. Она заставит маршрутизатор установить еще одну пару SA если вы направите траффик в сеть, для которой их еще не существует.

Ну и добавим исключения для NAT:
```

/ip firewall nat
add action=accept chain=srcnat disabled=no dst-address=192.168.0.0/23 src-address=172.16.100.0/24 place-before 0
add action=accept chain=srcnat disabled=no dst-address=172.16.20.0/24 src-address=172.16.100.0/24 place-before 0
```

Easy as that.
