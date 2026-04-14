---
title: 'Заметки к CCNP Route: EIGRP часть 3'
author: ["Stanislav"]
date: 2012-09-03T06:30:34+00:00
url: /2012/09/ccnp-preparation-eigrp-3/
categories:
  - Tech
tags:
  - ccnp route
  - cisco
  - eigrp
---

![dynamic routing is sexy](/wp-content/uploads/2012/08/eigrp.png "dynamic routing is sexy")Больше, больше хардкора. В заключительной части мы посмотрим как можно оптимизировать конвергенцию протокола EIGRP с помощью фильтрации маршрутов на основе таких средств, как ACL, prefix-list, route-map. Выясним кто такие stub маршрутизаторы, а так же какое влияние оказывают суммарные маршруты на query сообщений. Ну и на сладенькое выясним что за состояние “Stuck in Active” и как испортить load sharing с помощью offset lists.

##### Оптимизация конвергенции EIGRP

Рассмотрим следующую топологию:

![eigrp stuck in active](/wp-content/uploads/2012/09/aef2aea86b7c75c2e4fe9b3b021cf019.gif "eigrp stuck in active")

Сеть, находящаяся за R1 внезапно умирает. В этом случае R1 генерирует query на всех интерфейсах, за исключением отвалившегося, спрашивая низлежашие маршрутизаторы не знают ли они чего-нибудь про 10.1.0.0/16. Те, понятное дело не знают, но они в свою очередь генерируют query и спрашивают других соседей. В большой сети это может затянуться на весьма продолжительное время, но проблема в том, что если R1 не получит ответ на свой query в течении 3х минут… он сбросит соседство (сосед stuck in active) :) Допустим, B6 нашел маршрут до 10.1.0.0/16 через богом забытый dial-up. R1 не установит маршрут в таблицу маршрутизации до тех пока не получит от всех ответ на свои query сообщения.

А теперь, чтобы стало совсем грустно, представим fullmesh топологию с десяткой другой удаленных площадок. Тут к Stuck in Active может добавится проблема инсталляции в таблицу маршрутизации не оптимальных маршрутов.

Тут на помощь приходят несколько технологий, которые призваны решить эти проблемы и оптимизировать работу протокола. Это:

- route filtering
- stub маршрутизаторы
- суммарзиция

Стэнд будет выглядеть следующим образом:

![](/wp-content/uploads/2012/09/lab.png "eigrp lab")

##### Фильтрация маршрутов (route filtering)

Позволяет вам контролировать какие маршруты будут отправляться соседям с update сообщениями. Например мы хотим чтобы маршрутизаторы в филиалах ничего не знали о локальных сетях друг друга. Такая фильтрация уменьшает размер таблиц маршрутизации, уменьшает потребление памяти, делает маршрутизацию безопаснее, а способность ее настроить отличает вас от CCENT и положительно сказывается на размере эго.

EIGRP осуществляет фильтрацию маршрутов с помощью distribute-list подкоманды в процессе настройки инстанса EIGRP. Distribute-list осуществляет фильтрацию с помощью ACL, prefix-list или route-map. Все эти славные парни определяют может ли какой-либо маршрут быть принятым/отправленным в update сообщение или он должен быть запрещен (отфильтрован). Так же в distribute-list можно указать направление – входящие update, исходящие update и/или добавить специфический интерфейс.

##### Фильтрация на основе ACL
```

R1(config)#ip access-list standard 10
R1(config-std-nacl)#deny 172.16.18.1 255.255.255.0
R1(config-std-nacl)#exit
R1(config)#router eigrp 25
! можем фильтровать на всех интерфейсах
R1(config-router)#distribute-list 10 out
! или только на конкретном
R1(config-router)#distribute-list 10 out fa0/0
```

##### Фильтрация на основе prefix-lists

Могут осуществлять фильтрацию как на основе префикса маршрута (номер подсети), так и на основе длины префикса (маска подсети). Отфильтруем горемычную 172.16.18.0/24 сетку, а заодно и все сети из диапазона 172.16.0.0/16, но разрешим те, у которых маска /25. Дополнительно запретим рассказывать о point-to-point линках с маской /30 – информация в эти сети все равно не отправляется, вот и нечего таблицу маршрутизации раздувать.
```

R1(config)#ip prefix-list TIMETOFILTER seq 10 permit 172.16.0.0/16 ge 25 le 25
R1(config)#ip prefix-list TIMETOFILTER seq 20 deny 172.16.0.0/16 le 32
R1(config)#ip prefix-list TIMETOFILTER seq 30 deny 0.0.0.0/0 ge 30 le 30
R1(config)#ip prefix-list TIMETOFILTER seq 40 permit 0.0.0.0/0 le 32
R1(config)#router eigrp 25
R1(config-router)#distribute-list prefix TIMETOFILTER out
```

Как и в примере с ACL мы можем указать конкретный интерфейс для действия политики на основе prefix-lists.

##### Фильтрация на основе route map

Route map используют логику if.. else, с помощью запрещающих и разрешающих номеров последовательностей и match *criteria*. Отбор по какому-то признаку может вестись как по ACL, так и по prefix-list. С этим возникает небольшая путаница. **Фильтрация осуществляется на основании утверждений permit или deny в route map, к фильтрации допускаются маршруты, разрешенные в ACL и/или prefix-list**.

Повторим задачу из прошлого примера с использованием route map, но немного изменим ее. Разрешим из диапазона 172.16.0.0/16 только сеть 172.16.30.0/25:
```

R1(config)#access-list 10 permit 172.16.30.0 0.0.0.127
R1(config)#access-list 20 permit 172.16.0.0 0.0.255.255
R1(config)#access-list 30 permit 0.0.0.0 0.0.0.3
R1(config)#route-map EIGRP-RM permit 1
R1(config-route-map)#match ip address 10
R1(config-route-map)#route-map EIGRP-RM deny 2
R1(config-route-map)#match ip address 20
R1(config-route-map)#route-map EIGRP-RM deny 3
R1(config-route-map)#match ip address 30
R1(config-route-map)#route-map EIGRP-RM permit 4
R1(config)#router eigrp 25
R1(config-router)#distribute-list route-map EIGRP-RM out
```

Обратите на последнее утверждение с порядковым номером 4. Оно, фактически разрешает все, что не запрещено. По умолчанию в конце route-map находится утверждение deny any.

##### Stub routers

Маршрутизатор, превращающийся в stub, больше не будет пересылать трафик между сетями, маршруты до которых были заучены из EIGRP. Такие маршрутизаторы больше не рассказывают соседям о сетях, информацию о которых они получили от других соседей и, что наиболее важно, non-stub маршрутизаторы не направляют query сообщения по направлению к stub.

Настраивается stub маршрутизатор весьма просто:
```

R4(config-router)#eigrp stub ?
  connected      Do advertise connected routes
  receive-only   Set IP-EIGRP as receive only neighbor
  redistributed  Do advertise redistributed routes
  static         Do advertise static routes
  summary        Do advertise summary routes
```

Если вы просто подадите команду eigrp stub, то по умолчанию она будет сконфигурирована с использованием ключей connected и summary. Для соседей это будет выглядеть как-то так:
```

R3#sh ip eigrp neighbors detail
IP-EIGRP neighbors for process 25
H   Address                 Interface       Hold Uptime   SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
1   192.168.2.2             Fa0/0             12 00:00:11   48   288  0  46
   Version 12.4/1.2, Retrans: 0, Retries: 0, Prefixes: 1
   Stub Peer Advertising ( CONNECTED ) Routes
   Suppressing queries
0   172.16.19.1             Fa0/1             13 01:53:16   35   210  0  33
   Version 12.4/1.2, Retrans: 2, Retries: 0, Prefixes: 3
```

##### Влияние суммаризации на query запросы

Если мы отдаем соседям суммарный маршрут до сетей, в которую входит упавшая, то на наш вопрос-query о том, знают ли соседи о других путях достижения этой сети нам ответят коротко и лаконично:

![](/wp-content/uploads/2012/09/no.jpg "no")

И дальше, соответственно, query-сообщения лавинно рассылаться не будут. Суммаризацию мы научились настраивать [еще в прошлый раз](/2012/08/ccnp-preparation-eigrp-1/ "eigrp summarization").

##### Stuck in Active

Как я уже говорил маршрутизатор будет ожидать 3 минуты до того, как пометить соседа stuck in active и сбросить соседство, но есть нюанс (с). Начиная с 12.2 IOS после 90 секунд (половина SIA таймера) соседям будет отправлен SIA-query запрос, но которые они могут ответить: все ок, мы еще ищем маршруты, не сбрасывай нас со счетов.

##### Offset lists

EIGRP offset lists представляют собой механизм манипуляции метрикой, которые выполняют следующие функции:

- могут фильтровать изменения метрик только для определенных префиксов используя IP ACL
- могут применяться на update сообщения в каком-либо направлении (отправка – out, прием – in)
- могут применяться на интерфейсе, который принимает и/или отправляет update сообщения
- число, добавляемое к метрике, учитывается в вычислении как FD, так и RD

Возьмем R4, который знает о двух способах достичь 172.16.18.0/24:
```

R4#sh ip route
--- опущено ---
D       172.16.18.0 [90/309760] via 192.168.2.1, 00:37:33, FastEthernet0/0
                    [90/309760] via 172.16.20.1, 00:37:33, FastEthernet0/1
```

и изменим положение вещей:
```

R4(config)#ip access-list standart 10
R4(config-ext-nacl)#permit ip host 172.16.18.0 host 255.255.255.0
R4(config)#router eigrp 25
R4(config-router)#offset-list 10 in 10 fastEthernet 0/1
*Mar  1 01:36:48.507: %DUAL-5-NBRCHANGE: IP-EIGRP(0) 25: Neighbor 172.16.20.1 (FastEthernet0/1) is resync: route configuration changed
R4(config-router)#do sh ip eig topo
--- опущено ---
P 172.16.18.0/24, 1 successors, FD is 309760
        via 192.168.2.1 (309760/284160), FastEthernet0/0
        via 172.16.20.1 (309770/284170), FastEthernet0/1
--- опущено ---
R4(config-router)#do sh ip proto
Routing Protocol is "eigrp 25"
--- опущено ---
 Incoming routes in FastEthernet0/1 will have 10 added to metric if on list 10
```

Вот так легко и непринужденно мы испортили equal path load sharing.

На этом можно считать, что с EIGRP мы закончили. Говорят, что в следующей серии сериала  “маленький системный инженер готовится к CCNP” на сцену ворвется OSPF.
