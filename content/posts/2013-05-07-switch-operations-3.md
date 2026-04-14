---
title: Switch operations – часть 3
author: ["Stanislav"]
date: 2013-05-07T14:17:19+00:00
url: /2013/05/switch-operations-3/
categories:
  - Tech
tags:
  - adjacency table
  - ccnp switch
  - cef
  - cisco
  - fib
  - mls
---

![cable-nightmare](/wp-content/uploads/2013/05/cable-nightmare.jpg)В финальной части мы быстренько пробежимся по CEF, посмотрим какое место в этой технологии занимают Forward Information Base (FIB) и ее часть – Adjacency Table.

CEF MLS содержит в себе два функциональных блока – **layer 3 engine**, формирующий информацию о маршрутах, которую может использовать **layer 3 forwarding engine** для обработки пакетов аппаратными средствами.

#### Forward Information Base

**Layer 3 engine** (фактически маршрутизатор) формирует таблицу маршрутизации, поддерживает ее в актуальном состоянии и, что самое главное, форматирует и компилирует ее в новый формат – Forward Information Base. FIB строится согласно следующей логике: таблица маршрутизации форматируется в упорядоченный список, где для каждой сети существует один или несколько маршрутов. Первым, понятное дело, идет наиболее длинный префикс (longest prefix match). Одновременно с этим FIB содержит next-hop адрес устройства для каждой записи. Помимо информации из таблицы маршрутизации в FIB попадают маршруты до directly conntect хостов.

Любая актуализация таблицы маршрутизации тут же отражается в FIB благодаря стараниям Layer 3 engine, так же как изменения next-hop и ARP записей.![mls operations](/wp-content/uploads/2013/05/mls-operations.png "mls operations")

После того, как FIB скомпилирована, загружена в TCAM/SRAM за дело берется **layer 3 forwarding engine**, занимаясь маршрутизацией данных с помощью аппаратного обеспечения, за исключением нескольких случаев, описанных в первой статье.

#### Типы CEF в модульных MLS

- **Accelerated CEF (aCEF)** – встречается в line card, которые не могут загрузить FIB целиком. Представляет собой горячий кэш недавних процессов маршрутизации. То, что попадает мимо такого кэша провоцирует запрос к layer 3 engine, чтобы получить актуальную информацию.
- **Distributed CEF (dCEF)** – централизованный layer 3 engine поддерживает одну FIB и она реплицируется на все line cards, поддерживающие dCEF.

#### Adjacency Table

Таблицы содержащие инофрмацию о подсети и адресе next-hop устройства и ARP таблицы соответствия Layer 3 адресов к Layer 2 держаться отдельно. Часть FIB, содержащая информацию о MAC адресе next-hop устройства, носит название ***adjacency table***.

Посмотреть на содержимое adjacency table можно следующим образом:
```

Switch# show adjacency [ interface #_num | vlan #_id ] [ summary | detail ]
```

```

Switch#sh adjacency vlan 20 detail
Protocol Interface                 Address
IP       Vlan20                    172.16.20.2(8)
                                   14830 packets, 2912675 bytes
                                   epoch 0
                                   sourced in sev-epoch 0
                                   Encap length 14
                                   D8D385DD609EF0F755B5A23F0800
                                   L2 destination address byte offset 0
                                   L2 destination address byte length 6
                                   Link-type after encap: ip
                                   ARP
IP       Vlan20                    172.16.20.5(8)
                                   3826122 packets, 1466477703 bytes
                                   epoch 0
                                   sourced in sev-epoch 0
                                   Encap length 14
                                   005056B56178F0F755B5A23F0800
                                   L2 destination address byte offset 0
                                   L2 destination address byte length 6
                                   Link-type after encap: ip
                                   ARP
< ... опущено ... >
```

Эта таблица содержит и Layer 3 адреса и MAC адреса в записях о всех next-hop и directly connected устройствах. MAC адрес записан в длиннющей последовательности шестнадцатеричных символов, в которую, помимо этого входим MAC SVI VLAN 20 (в моем примере) и значение EtherType (0x0800 – IP).

#### Состояние CEF glean

Adjacency table строится Layer 3 engine из ARP таблицы. В том случае если нет записи о next-hop адресе в ARP таблице CEF помечает такую FIB запись как “CEF glean”. Это означает, что Layer 3 forwarding engine стоит дернуть Layer 3 engine, которые должен в свою очередь генерировать ARP request и справедливо ожидать на него ARP reply.

Мониторится это следующим образом:
```

Switch# show ip cef adjacency glean
Prefix               Next Hop             Interface
172.16.15.0/27       attached             Vlan151
172.16.15.32/28      attached             Vlan152
172.16.16.0/27       attached             Vlan161
172.16.20.0/24       attached             Vlan20
172.16.21.0/24       attached             Vlan21
< ... опущено ... >
```

```

Switch#show ip cef 172.16.20.2 255.255.255.255 internal
172.16.20.2/32, epoch 1, flags attached, refcount 5, per-destination sharing
  sources: Adj
  subblocks:
   Adj source: IP adj out of Vlan20, addr 172.16.20.2 2051FFE0
    Dependent covered prefix type adjfib cover 172.16.20.0/24
  ifnums:
   Vlan20(75): 172.16.20.2
  path 200C69F4, path list 200BE078, share 1/1, type adjacency prefix, for IPv4
  attached to Vlan20, adjacency IP adj out of Vlan20, addr 172.16.20.2 2051FFE0
  output chain: IP adj out of Vlan20, addr 172.16.20.2 2051FFE0
```

В adjacency table помимо соответствия layer 3 – layer 2 адресов хранятся другие типы данных:- **Null adjacency** – используется для коммутации пакетов, которые направляются в blackhole, null интерфейс. Это логический интерфейс присутствующий на коммутаторах и маршрутизаторах, функция которого состоит в тихом убиение пакетов без какой-либо обработки и генерации пакетов.
- **Drop adjacency** – используется для коммутации пакетов, которые невозможно перенаправить в нормальном режиме. Такое может случиться по разным причинам – ошибки CRC, неподдерживаемый протокол, отсутствие маршрута и т.д. Отслеживать количество отброшенных пакетов можно так:
```

CoreSwitchA#show cef drop
  % Command accepted but obsolete, see 'show (ip|ipv6) cef switching statistics [feature]'

  IPv4 CEF Drop Statistics
  Slot  Encap_fail  Unresolved Unsupported    No_route      No_adj  ChkSum_Err
  RP             0           0      385494           5           0           0
```

- **Discard adjacency** – используется тогда, когда пакеты должны быть отброшены по причине правила в Security ACL или по другим соображениям (QOS, PBR).
- **Punt adjacency** – используется тогда, когда пакет должен быть отправлен на layer 3 engine для дальнейшей обработки. См. тот же CEF glean state выше. Отслеживать количество пакетов, которые подверглись process switching’у можно так:
```

CoreSwitchA#show cef not-cef-switched
  % Command accepted but obsolete, see 'show (ip|ipv6) cef switching statistics [feature]'

  IPv4 CEF Packets passed on to next switching layer
  Slot  No_adj No_encap Unsupp'ted Redirect  Receive  Options   Access     Frag
  RP         0       0      385382   385382    84015        0        0        0
```

где:

  - **slot** – номер line card, на который пришел пакет.
  - **no_adj** – не полная или отсутствующая запись в результате невозможности использовать ARP для разрешения next-hop адреса устройства или недавно выполненная очистка ARP таблицы с помощью **clear ip arp** / **clear adjacency**.
  - **no_encap** – количество пакетов, которое подверглась processor switching’у в результате необходимости выяснить MAC адрес с помощью ARP.
  - **unsupp’ted** – количество пакетов, отправленных на layer 3 engine из-за каких-либо неподдерживаемых функций (hardware NAT).
  - **redirect** – в счетчик пакетов, на которые MLS должен был генерировать ICMP сообщение.
  - **receive** – количество пакетов, предназначенное непосредственно MLS.
  - **options** – количество пакетов, отправленных на обработку процессором, в которых было установлено 14-е поле “options”. ([wiki](https://ru.wikipedia.org/wiki/IPv4#.D0.A1.D1.82.D1.80.D1.83.D0.BA.D1.82.D1.83.D1.80.D0.B0_.D0.BF.D0.B0.D0.BA.D0.B5.D1.82.D0.B0 "options field"))
  - **access** – количество пакетов, отброшенных из-за совпадений с deny ACE.
  - **frag** – количество пакетов, отброшенных из-за проблем с фрагментацией.
