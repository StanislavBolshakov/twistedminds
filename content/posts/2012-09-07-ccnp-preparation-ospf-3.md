---
title: 'Заметки к CCNP Route: OSPF часть 3'
author: ["Stanislav"]
date: 2012-09-07T12:49:55+00:00
url: /2012/09/ccnp-preparation-ospf-3/
categories:
  - Tech
tags:
  - ccnp route
  - cisco
  - ospf
---

##### Различные типы LSA в OSPF

Как мы с вами выяснили каждый маршрутизатор в области должен иметь одинаковую LSDB, информация в которую собирается из отдельных LSA. Всего существует 11 типов LSA и их понимание поможет вам лучше разобраться в тонкостях работы различных типов областей, но об этом чуть позже.

1. LSA первого типа [**Router**] – каждый маршрутизатор создает этот тип LSA чтобы рассказать о себе всей области. В LSDB для каждой области содержится один LSA первого типа для каждого маршрутизатора, в котором указаны RID, IP адреса всех интерфейсов, а так же тупиковые (stub) сети. Не покидают область.
2. LSA второго типа [**Network**] – создаются DR на броадкастовом или NBMA сегменте и описывают сеседей, присоединенных к сегменту. Не покидают область.
3. LSA третьего тип [**Summary**, Network Summary] – создаются ABR для того, чтобы рассказать о маршрутах до соседей, полученных из LSA первого и второго типов в одной области и передать эту информацию в другую. Описывают подсети, стоимость маршрута исключая топологию области.
4. LSA чертвертого типа [**Summary**, ASBR Summary] – похожи на LSA третьего типа, используются чтобы передать маршрут до ASBR соседям из других областей.
5. LSA пятого типа [**External**, AS External] – создаются на ASBR для внешних маршрутов внедренных в OSPF.
6. LSA шестого типа [**Multicast**, Group Membership] – расширение OSPF протокола, которое не поддерживается в маршрутизаторах Cisco.
7. LSA седьмого типа [**NSSA External**] – создаются ASBR если он находится внутри NSSA области вместо LSA пятого типа.

Внимательный читатель тут же спросит, а куда делись еще 4 типа. Тип 8 – это внешние атрибуты, которые так же не поддерживаются маршрутизаторами Cisco, а 9-11 это расширения стандартного протокола. К примеру LSA 10 типа применяются в MPLS TE.

Посмотрим на LSA 1 и 2 типов на примере части топологии:![](/wp-content/uploads/2012/09/ospf-frag-1.png "ospf-frag-1")
```

R2#sh ip ospf database router 1.1.1.1

            OSPF Router with ID (2.2.2.2) (Process ID 1)

                Router Link States (Area 0)

  LS age: 19
  Options: (No TOS-capability, DC)
  LS Type: Router Links
  Link State ID: 1.1.1.1
  Advertising Router: 1.1.1.1
  LS Seq Number: 80000005
  Checksum: 0x6251
  Length: 60
  Number of Links: 3

    Link connected to: a Stub Network
     (Link ID) Network/subnet number: 192.168.2.0
     (Link Data) Network Mask: 255.255.255.0
      Number of TOS metrics: 0
       TOS 0 Metrics: 1

    Link connected to: a Transit Network
     (Link ID) Designated Router address: 192.168.20.2
     (Link Data) Router Interface address: 192.168.20.1
      Number of TOS metrics: 0
       TOS 0 Metrics: 10

    Link connected to: a Transit Network
   (Link ID) Designated Router address: 192.168.1.1
     (Link Data) Router Interface address: 192.168.1.1
      Number of TOS metrics: 0
       TOS 0 Metrics: 10
```

Мы видим что маршрутизатор с RID 1.1.1.1 подключен к 3 сегментам, один из которых по понятным причинам является Stub. Так как сеть 192.168.1.0/24 является транзитной и мы не можем прямо сказать какие связи будут сформированы между 2 и более маршрутизаторами в одном сегменте основываясь на LSA первого типа тут на помощь приходит тип 2.
```

R2#sh ip ospf database network 192.168.1.1

            OSPF Router with ID (2.2.2.2) (Process ID 1)

                Net Link States (Area 0)

  Routing Bit Set on this LSA
  LS age: 1522
  Options: (No TOS-capability, DC)
  LS Type: Network Links
  Link State ID: 192.168.1.1 (address of Designated Router)
  Advertising Router: 1.1.1.1
  LS Seq Number: 80000001
  Checksum: 0x7347
  Length: 32
  Network Mask: /24
        Attached Router: 1.1.1.1
        Attached Router: 3.3.3.3
```

Посмотрев и проанализировав LSA первого и второго типов на всех трех маршрутизаторах можно легко составить представление о том куда они подключены.

Разделение на области было призвано облегчить расчет маршрутов и снизить потребление памяти. Вместо того, чтобы нагружать все маршрутизаторы автономной системы LSA первого и второго типов ABR проводят своеобразную фильтрацию исключая передачу этих LSA между областями. Вместо этого передаются “выжимки” из LSDB.

Расширим схему, добавив R4 в area 1 и настроим R2 как ABR.

![](/wp-content/uploads/2012/09/ospf-frag-2.png "ospf-frag-2")
```

R2#sh ip ospf database
--- опущено ---
                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum
172.16.8.0      2.2.2.2         262         0x80000001 0x00DF53
172.16.10.0     2.2.2.2         224         0x80000001 0x0040E3
172.16.11.0     2.2.2.2         214         0x80000001 0x0035ED

--- опущено ---
                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
192.168.1.0     2.2.2.2         269         0x80000001 0x00595D
192.168.2.0     2.2.2.2         269         0x80000001 0x00F3CA
192.168.20.0    2.2.2.2         269         0x80000001 0x00F8BB
192.168.21.0    2.2.2.2         269         0x80000001 0x00EDC5
```

Теперь R2 ABR и передает в LSA 3 типа известные ему сети. Информацию о LSA 3 типа можно получить с помощью команды **ip ospf database summary *network***.

Подведем итог:

|  |  |  |  |  |  |
| --- | --- | --- | --- | --- | --- |
| Тип LSA | Имя LSA | Что означает | Как посмотреть с помощью sh ip ospf database … | LSID | Кем создается |
| 1 | Router | маршрутизатор | router | RID маршрутизатора | маршрутизатором |
| 2 | Network | Подсеть с выбранным DR | network | Алрес DR в подсети | DR в этой подсети |
| 3 | Summary | Подсеть в другой области | summary | Номер подсети | ABR |
