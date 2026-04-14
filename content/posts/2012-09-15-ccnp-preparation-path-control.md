---
title: 'Заметки к CCNP Route: Path Control'
author: ["Stanislav"]
date: 2012-09-15T15:24:40+00:00
url: /2012/09/ccnp-preparation-path-control/
categories:
  - Tech
tags:
  - ccnp route
  - cisco
  - path control
---

![](/wp-content/uploads/2012/09/path-control.png "path control")Термин path control имеет множество значений. Мы производили манипуляции над путем следования пакетов с помощью фильтрации маршрутов, тэгирования и других технологий. Однако, есть еще способы, которые стоят отдельно от манипулирования протоколами динамической маршрутизации и таблицами маршрутизации – Policy-Based Routing (PBR) и IP SLA (IP Service-Level Agreement). Первая технология оказывает влияние на data plane, изменяет логику принятия решения о маршрутизации пакетов. Вторая – IP SLA – по сути является механизмом мониторинга состояния сети и позволяет принимать решения об использовании того или иного маршрута на основании неких факторов.

##### Policy-Based Routing

Policy-Based Routing механизм, позволяющей переопределить привычку маршрутизатора заниматься destination-based маршрутизацией. PBR перехватывает пакет до того, как маршрутизатор предпримет поиск в FIB и CEF таблицах. PBR принимает решения о маршрутизации пакета, основываясь на логике ACL и инструкциях, написанных в route map.

Отбор пакетов осуществляется как по уже вам известной команде **match ip address** в процессе настройке route map, так и по **match lenght *min* *max***. Последняя позволяет принимать решение, основываясь на размере пакета в байтах. Если пакет прошел отбор, то следующее, что сделает маршрутизатор это спросит у route map: “а что мне с ним делать?”. Вариантов 4:

|  |  |
| --- | --- |
| команда | результат выполнения |
| **set ip next-hop *ip_addr* [… *ip_addr*]** | Подсеть должна быть directly connected. Пакет будет отправлен на первый ip из списка, интерфейс, ассоциированный с которым в состоянии up/up. |
| **set ip default next-hop *ip_addr* [… *ip_addr*]** | Та же логика, что и команда выше, только PBR сначала попытается принять решение на основании таблицы маршрутизации. |
| **set interface *interface* [… *interface*]** | Пакет будет отправлен в первый интерфейс из списка, который находится в состоянии UP. |
| **set default interface *interface* [… *interface*]** | Та же логика, что и команда выше, только PBR сначала попытается принять решение на основании таблицы маршрутизации. |

Пример. Допустим у нас есть 2 пк, один из которых принадлежит славному парню (PC1), а другой сильно менее славному (PC2). Так же в наличии имеются ISP1, который хороший, годный, и ISP2, у которого вся сеть на дохлых хабах начала 90ых годов прошлого века. Оба парня в рабочее время любят заглянуть на redtube и перед нами стоит задача воздать каждому по заслугам.

![](/wp-content/uploads/2012/09/pbr-example.png "pbr example")
```

CoreRouter(config)#ip access-list extended GOOD-PERVERTED-FREAK
CoreRouter(config-ext-nacl)#permit ip host 192.168.1.110 host 172.16.20.253
CoreRouter(config-ext-nacl)#exit
CoreRouter(config)#ip access-list extended BAD-PERVERTED-FREAK
CoreRouter(config-ext-nacl)#permit ip any host 172.16.20.253
CoreRouter(config)#route-map PBR-EXAMPLE
CoreRouter(config-route-map)#match ip address GOOD-PERVERTED-FREAK
CoreRouter(config-route-map)#set ip next-hop 10.10.10.2
CoreRouter(config-route-map)#route-map PBR-EXAMPLE 20
CoreRouter(config-route-map)#match ip address BAD-PERVERTED-FREAK
CoreRouter(config-route-map)#set ip next-hop 20.20.20.2
CoreRouter(config-route-map)#int fa0/1
CoreRouter(config-if)#ip policy ro PBR-EXAMPLE
```

```

*Mar 1 00:45:02.443: IP: s=192.168.1.110 (FastEthernet0/1), d=172.16.20.253, len 44, FIB policy match
*Mar 1 00:45:02.447: IP: s=192.168.1.110 (FastEthernet0/1), d=172.16.20.253, g=10.10.10.2, len 44, FIB policy routed
*Mar 1 00:45:06.215: IP: s=192.168.1.201 (FastEthernet0/1), d=172.16.20.253, len 44, FIB policy match
*Mar 1 00:45:06.219: IP: s=192.168.1.201 (FastEthernet0/1), d=172.16.20.253, g=20.20.20.2, len 44, FIB policy routed
```

![](/wp-content/uploads/2012/09/Mission-Accomplished.jpg "Mission Accomplished")

##### IP SLA

IP Service Level Agreement технология, позволяющая отслеживать состоянии сети. В старой литературе она носит название RTR – Responce Time Reporter. IP SLA использует концепцию процессов. Каждый процесс описывает тип генерируемого маршрутизатором пакета, адрес отправителя, получателя и некоторые другие характеристики. Так же в момент настройки процесса мониторинга следует указать время начала, длительность и частоту проб. IP SLA может оперировать в нескольких режимах, отправляя различные виды пробных сообщений, на основании которых будут предприниматься те или иные решения:

* ICMP (echo, jitter)
* RTP (VoIP)
* TCP (3-way handshake), UDP (echo, jitter)
* DNS
* DHCP
* HTTP
* FTP

и так далее. С полным списком процессов и возможных проб [можно ознакомится тут](http://www.cisco.com/en/US/technologies/tk648/tk362/tk920/technologies_qas0900aecd8017bd5a.html "IP SLA operations and probes").

В качестве целей для проб могут выступать как другие маршрутизаторы, так и обычные сервера. К примеру маршрутизатор-инициатор проб может даже отправлять HTTP GET запросы и ожидать на них ответа.

Настройка состоит из 4х простых шагов:

|  |  |
| --- | --- |
| шаг | действие |
| 1 | Создать процесс IP SLA: **ip sla monitor *#_id*** |
| 2 | Указать тип процесса и его параметры. Например для ICMP echo запросов вам следует указать адрес или hostname отправителя, адрес или hostname получателя. |
| 3 | Опционально можно изменить частоту отправки проб с значения по умолчанию, используя команду **frequency *seconds***. |
| 4 | Ну и в конце осталось создать расписание: **ip sla monitor schedule *#_id* [ life (*forever* | *seconds* ) ] [ start-time (*hh:mm:ss*) | *pending* | *now* | *after* (*hh:mm:ss*) ] [ ageout *seconds* ] [ recurring ]** в режиме глобальной конфигурации. |

Посмотреть текущую конфигурация процессов ip sla можно с помощью команды **show ip sla monitor configuration**, а статистику – **show ip sla monitor statistics.**

##### **Отслеживание IP SLA процессов**

Сами по себе пробы нам не очень интересны, а интересно отслеживать успешность тех или иных действий и на основании этого принимать какие-либо решения. Например использование статического маршрута или PBR. Механизм отслеживание IP SLA смотрит на последний статус пробы (return code) и на основе этого принимает решение о состоянии пробы этого процесса (up/down).

**Настройка отслеживания IP SLA для статических маршрутов**

|  |  |
| --- | --- |
| шаг | действие |
| 1 | Настроить процесс слежения за IP SLA, используя команду **track *track_id* rtr *sla_id* [ *state* | *reachability* ]** * |
| 2 | Опционально можно настроить задержку возврата к предыдущему состоянию, указав значение в секундах командой **delay ( down *seconds* | up *seconds* ) **** |
| 3 | Настроить статический маршрут, указав id процесса слежения: **ip route *network* *mask* [ *interface* | *next-hop* ] track *track_id*** |

* судя по [этому документу](http://www.cisco.com/en/US/docs/ios/ipsla/command/reference/sla_05.html "difference between state and reachability") разница между state и reachability заключается в том, что reachability возвращает статус up для процесса отслеживания IP SLA даже если значение для, скажем, jitter, превышает пороговое.
 ** есть механизм предотвращения возвращения маршрута в таблицу маршутизации при route flapping, когда путь становится то доступен, то недоступен.

**Настройка отслеживания IP SLA для PBR**

Происходит с использованием дополнительного ключа verify-availablitiy в процессе установки следующего хопа: **set ip next-hop verifiy-availability *ip_addr* track *track_id***.

Если процесс отслеживания получает статус “down”, то PBR начинает работать так, как будто не существует утверждения **set ip next-hop…** Маршрутизатор будет пробовать отправить пакет так, как если бы PBR не существовало.

Вернемся к нашей схеме, использованной в теме про PBR и настроим IP SLA для мониторинга доступности интернетов через двух наших провайдеров. Объектом для мониторинга будет выступать webserver. В реальной жизни я использую адреса корневых DNS серверов.
```

CoreRouter#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
CoreRouter(config)#no ip sla monitor 10
CoreRouter(config)#ip sla monitor 10
CoreRouter(config-sla-monitor)#type echo protocol ipIcmpEcho 172.16.20.253 source-ipaddr 10.10.10.1
CoreRouter(config-sla-monitor-echo)# frequency 5
CoreRouter(config-sla-monitor-echo)# ip sla monitor schedule 10 life forever start-time now
CoreRouter(config)#ip sla monitor 20
CoreRouter(config-sla-monitor)#$172.16.20.253 source-ipaddr 20.20.20.1
CoreRouter(config-sla-monitor-echo)# frequency 5
CoreRouter(config-sla-monitor-echo)# ip sla monitor schedule 20 life forever start-time now
CoreRouter(config)#track 1 rtr 10 state
CoreRouter(config-track)# delay down 20 up 20
CoreRouter(config-track)#track 2 rtr 20 reachability
CoreRouter(config-track)# delay down 20 up 20
CoreRouter(config-track)#exit
CoreRouter(config)#ip route 0.0.0.0 0.0.0.0 10.10.10.2 track 1
CoreRouter(config)#ip route 0.0.0.0 0.0.0.0 20.20.20.2 track 2
```

Теперь если путь до webserver с IP 172.16.20.253 через, скажем, ISP2 окажется недоступен, вы получите в логе сообщение
```

*Mar  1 03:39:53.643: %TRACKING-5-STATE: 2 rtr 20 reachability Up->Down
```

а маршрут по умолчанию до ISP2 будет убран из таблицы маршрутизации. Как только webserver станет снова доступен через ISP2 все вернется взад с соответствующим оповещением:
```

*Mar  1 03:45:08.647: %TRACKING-5-STATE: 2 rtr 20 reachability Down->Up
```
