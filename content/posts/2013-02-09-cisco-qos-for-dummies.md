---
title: Cisco QoS для самых маленьких
author: ["Stanislav"]
date: 2013-02-09T18:35:55+00:00
url: /2013/02/cisco-qos-for-dummies/
categories:
  - Tech
tags:
  - cisco
  - QoS
---

![QoS](/wp-content/uploads/2013/02/1260110610_4e110ad3f5-300x200.jpg "QoS")Я всегда считал Quality of Service (QoS) чем-то из разряда “не дадим войсу квакать, не дадим изображению по вкс развалиться”, однако более глубокое изучение вопроса, откровенно говоря, приятно удивило. Количество средств по манипуляции трафиком, а так же различные методы предоставления гарантированного качества канала нужным приложениям сразу дали понять – нахрапом эту крепость не взять.

#### С чем полезен и с чем помогает справиться QoS?

* Маленькая пропускная способность канала – не всегда у нас на руках оказывается тот канал, который позволит и youtube в HD посмотреть и корпоративным приложениям работать адекватно.
* Потеря критичных пакетов в следствии заторов.
* Задержки (**delay**) – время, которое требуется пакету чтобы попасть от источника к получателю. Так, к примеру, 400ms RTT для голоса еще туда-сюда, а вот с значениями, которые превышают этот порог вы получите эффект накладывающегося голоса.
* Джиттер (**jitter**) или изменение задержки с течением времени. Сейчас у вас 100ms, а через минуту 200ms. Поздравляю, у вас теперь 100ms jitter, который превращается в задержку просто потому, что железкам нужно держать пакет в [буфере](http://en.wikipedia.org/wiki/Jitter#Jitter_buffers "jitter buffer") какое-то время для обеспечения плавности того же разговора.

#### Методы настройки QoS

- Command Line Interface (**CLI**) – per interface конфигурация.
- Modular QoS CLI (**MQC**) – аки ZBF позволяет группировать class map в policy map, policy map привязывать к service policy, а вот уже последнее можно и на интерфейс применить.
- Auto QoS – генерирует некий усредненный конфиг, который, конечно не one size fit all, но в некоторых случаях оказывается вполне работоспособным.
- QoS Policy Manager (**QPM**) – централизованная поддержка QoS в вашей сети. Встроена в CiscoWorks.

#### Инструменты QoS

- Классификация и группировка различных типов трафика.
- Маркировка трафика за тем, чтобы другие устройства в сети могли бы распознать его.
- Policing & shaping. Policing позволяет либо отбрасывать пакеты при превышении какого-либо значения, либо изменять маркировку. Shaping – метод размещения пакетов в очередях (i.e. FIFO, WFQ, CBWFQ) при достижении лимита.

#### Избежание заторов (congestion avoidance)

- **First-in First-out** – как понятно из названия – кто первый встал, того и тапки. Пакеты помещаются в очередь, пока не будет исчерпана пропускная способность канала. При ее исчерпании заполняется буфер. Ну а если и он закончился, то пакеты попросту отбрасываются – tail drop.
- **Random Early Detection (RED)** – RED занимается занятной штукой: как только ваш буфер начинает заполняться – “БАМ!” – в нем погибает случайный пакет. И чем быстрее заполняется буфер, тем агрессивнее работает этот механизм. На сколько я знаю Cisco не поддерживает его работу.
- **Weighted Random Early Detection (WRED)** – занимается тем же, что и RED, но вместо содомии с случайными пакетами этот механизм выбирает жертв на основании маркировки пакетов и обозначенных заранее правил. “БАМ!” – и вместо пакета с voice трафиком погибает кусок картинки с redtube’a. Если пакеты заранее не были маркированы, то WRED превращается в RED.

#### Управление заторами

Управлять заторами можно помещая пакеты в очереди, позволяя наиболее приоритетному трафику покидать интерфейс в первую очередь, а наименее приоритетный трафик может и включения WRED дождаться, всякое бывает.

#### Увеличение эффективности

- Сжатие (data compression).
- Фрагментация канала и чередование (link fragmentation & interleaving) – допустим у нас имеется медленный канал (serial, ага) в который готовится просочиться сочный пакет, содержащий не особо важные данные. На момент когда передача будет начата будет в принципе не важно настроен ли QoS – сочный пакет с радостью займет все предоставленные ему 1500 байт и не пустит крошечный голосовой пакет не смотря не на что. LFI говорит примерно следующее – больше у нас не будет больших пакетов, крошечный пакет голоса, упакованный g.711 должен пролезть no matter what.

#### Modular QoS CLI

![modular QoS CLI structure](/wp-content/uploads/2013/02/MQC.png "modular QoS CLI structure")

**Class map** – отвечают за классификацию трафика, позволяя отделить трафик и большим приоритетом от трафика с меньшим. i.e. я хочу выделить http трафик в отдельный класс class-map 2.

**Policy map** – говорит что делать с трафиком, включенным в входящие в него class map. i.e. я хочу маркировать http трафик определенным тэгом в policy-map 1 и я хочу лимитировать полосу пропускания для http трафика в 64kbps в policy-map 2.

**Service policy** – применяет policy map на интерфейсе для входящего или исходящего трафика. i.e. я хочу чтобы policy-map 1 маркировала исходящий http трафик на fa0/1, а policy map 2 лимитировала входящий на fa0/2. Одна и только одна service policy может существовать для каждого из направлений на каждом интерфейсе.

##### Настройка class map

Для class map существуют 2 критерия, по которым они будут работать – match-all (логическое И) и match-any (логическое ИЛИ). По умолчанию создается class-map match-all, означающий что все условия, занесенные в такой class map должны совпасить одновременно для того, чтобы class-map сработала.
```

R2(config)#class-map ICMP
R2(config-cmap)#match protocol icmp
R2(config-cmap)#match packet length min 200 max 400
R2(config-cmap)#do sh class-map
 Class Map match-all ICMP (id 1)
   Match protocol icmp
   Match packet length min 200 max 400

 Class Map match-any class-default (id 0)
   Match any
```

Под действие этой class map попадет ICMP трафик с размером пакета от 200 до 400 байт.

Самые наблюдательные из вас успели заметить, что помимо созданной class map ICMP существует еще одна – default, под действие которой попадает весь трафик – это class map по умолчанию. Например мы можем описать голосовой трафик и написать для него определенное правило A, а весь остальной трафик, который нам не интересен, отправится автоматом в class-default и для него будет применено правило B.

Помимо протоколов можно допускать или не допускать к фильтрации сети, хосты, а так же гибко их совмещать в одной class-map с помощью ACL. i.e.:
```

R2(config)#access-list 10 remark mail servers
R2(config)#access-list 10 permit 10.1.1.1
R2(config)#access-list 10 permit 10.1.1.2
R2(config)#class-map match-all MAIL_SRVRS
R2(config-cmap)#match access-group 10
R2(config-cmap)#match protocol smtp
```

Под действие такой class map попадут ваши почтовые сервера **и** SMTP трафик. И на 25 и на 587 порт. Умный IOS сам разберется благодаря Network Based Application Recognition (NBAR).

##### Настройка policy map

Пришло время рассказать бездушной железке что делать с классифицированным таким образом трафиком. Единственная полезная вещь, которую можно сделать с policy map сразу после создания это добавить к нему class map:
```

R2(config)#policy-map LIMIT_ICMP
R2(config-pmap)#?
QoS policy-map configuration commands:
  class        policy criteria
  description  Policy-Map description
  exit         Exit from QoS policy-map configuration mode
  no           Negate or set default values of a command
  rename       Rename this policy-map
```

А вот после применения class map как раз и начинается все веселье:
```

R2(config-pmap)#class ICMP
R2(config-pmap-c)#?
QoS policy-map class configuration commands:
  bandwidth        Bandwidth
  compression      Activate Compression
  drop             Drop all packets
  estimate         estimate resources required for this class
  exit             Exit from QoS class action configuration mode
  netflow-sampler  NetFlow action
  no               Negate or set default values of a command
  police           Police
  priority         Strict Scheduling Priority for this Class
  queue-limit      Queue Max Threshold for Tail Drop
  random-detect    Enable Random Early Detection as drop policy
  service-policy   Configure Flow Next
  set              Set QoS values
  shape            Traffic Shaping
```

Обратите внимание на изменившееся приглашение.

**Бегло** пробежимся по опциям:

- **bandwidth** – гарантирует трафику минимальную полосу пропускания в случае возникновения затора (CBWFQ). Можно устанавливать значения либо жестко, либо в процентах. Последнее решение обладает большей гибкостью, так как позволяет использовать одну и ту же policy map на интерфейсах с разными значениями bandwidth.
- **compression** – включает сжатие TCP или RTP заголовков для данного трафика, снижая network overhead;
- **drop** – отбрасывает все пакеты, попавшие под действие class map;
- **estimate** – позволяет предопределять допустимые пределы задержек или потерь пакетов для данного типа трафика;
- **netflow-sampler** – в том случае если вам не интересен 100% учет сетевого трафика посредством netflow можно настроить sampler, который будет отправлять 1 flow из n, таким образом снижая количество трафика, льющегося на ваши коллекторы.
- **policy / shape** – гибкие функции по policing’у и shaping’у трафика;
- **priority** – гарантирует полосу пропускания (LLQ). Тут внимательный читатель спросит LOLWUT?, покосится на bandwidth, потом снова на priority и снова на bandwidth в попытках найти различия. На самом деле они есть и не маленькие – priority позволяет назначать не только минимальную гарантированную полосу пропускания, но и одновременно максимальную. Более подробно о LLQ можно прочитать [тут](https://supportforums.cisco.com/thread/2016073 "LLQ understanding") или, возможно, в следующих статьях.
- **queue-limit** – команда, позволяющая указать или модифицировать количество пакетов, которые маршрутизатор сможет поместить в очередь.
- **random-detect** – включает механизм WRED.
- **service-policy** – позволяет указать другую policy map для построения иерархических (вложенных) политик.
- **set** – устанавливает различные значения в QoS поля.

Хотелось бы еще раз повторить – это краткое описание опций, подробные разговор о которых является темой не для одной статьи.

##### Применение policy map

Нет ничего проще. В режиме конфигурации интерфейса с помощью команды ***service-policy*** укажите направление и название policy map. Не так уж и трудно, не так ли? ;)

#### Практика

![qos stand](/wp-content/uploads/2013/02/qosstand.png)

Для того, чтобы разбавить унылую теорию давайте соберем небольшой стенд, запретив на r2 ICMP пакеты больше 700 байт по направлению к r3 из 192.168.1.0/24 сети, для ICMP пакетов размером от 300 до 700 выставим ограничение 8000bps.
```

R2(config)#access-list 1 permit 192.168.1.0 0.0.0.255
R2(config)#class-map match-all ICMP
R2(config-cmap)# match access-group 1
R2(config-cmap)# match protocol icmp
R2(config-cmap)# match packet length min 300 max 700
R2(config-cmap)#class-map match-all HUGE_ICMP
R2(config-cmap)# match access-group 1
R2(config-cmap)# match protocol icmp
R2(config-cmap)# match packet length min 701
R2(config-cmap)#policy-map ICMP_PMAP
R2(config-pmap)# class ICMP
R2(config-pmap-c)#   police rate 8000 bps
R2(config-pmap-c-police)#     conform-action transmit
R2(config-pmap-c-police)#     exceed-action drop
R2(config-pmap-c-police)# class HUGE_ICMP
R2(config-pmap-c)#   drop
R2(config-pmap-c)#int fa0/1
R2(config-if)# service-policy output ICMP_PMAP
R2(config-if)#^Z
```

и проверим эмпирическим путем на R1:
```

R1#ping 1.1.1.2 repeat 20 size 299

Type escape sequence to abort.
Sending 20, 100-byte ICMP Echos to 1.1.1.2, timeout is 2 seconds:
!!!!!!!!!!!!!!!!!!!!
Success rate is 100 percent (20/20), round-trip min/avg/max = 12/40/52 ms
R1#ping 1.1.1.2 repeat 20 si
R1#ping 1.1.1.2 repeat 20 size 300

Type escape sequence to abort.
Sending 20, 300-byte ICMP Echos to 1.1.1.2, timeout is 2 seconds:
!!!!!.!!!!!.!!!!!.!!
Success rate is 85 percent (17/20), round-trip min/avg/max = 24/41/64 ms
R1#ping 1.1.1.2 repeat 20 size 700

Type escape sequence to abort.
Sending 20, 700-byte ICMP Echos to 1.1.1.2, timeout is 2 seconds:
!!.!!.!!.!!.!!.!!.!!
Success rate is 70 percent (14/20), round-trip min/avg/max = 24/46/80 ms
R1#ping 1.1.1.2 repeat 20 size 701

Type escape sequence to abort.
Sending 20, 701-byte ICMP Echos to 1.1.1.2, timeout is 2 seconds:
....................
Success rate is 0 percent (0/20)
```

действительно, ICMP пакеты до 299 включительно без проблем пролезают, минуя всю маркировку через class map, ICMP пакеты от 300 до 700 байт начинают испытывать трудности, в результате установленного лимита “не более 8000 бит в секунду”, а ICMP пакеты свыше 701 байта просто и без затей отбрасываются.

На R2 с работой вашей политики можно ознакомиться следующим образом:
```

R2#sh policy-map interface fa0/1
 FastEthernet0/1

  Service-policy output: ICMP_PMAP

    Class-map: ICMP (match-all)
      195 packets, 94730 bytes
      5 minute offered rate 0 bps, drop rate 0 bps
      Match: access-group 1
      Match: protocol icmp
      Match: packet length min 300 max 700
      police:
          rate 8000 bps, burst 1500 bytes
        conformed 31 packets, 15334 bytes; actions:
          transmit
        exceeded 9 packets, 5226 bytes; actions:
          drop
        conformed 0 bps, exceed 0 bps

    Class-map: HUGE_ICMP (match-all)
      53 packets, 62262 bytes
      5 minute offered rate 0 bps, drop rate 0 bps
      Match: access-group 1
      Match: protocol icmp
      Match: packet length min 701
      drop

    Class-map: class-default (match-any)
      1538 packets, 170844 bytes
      5 minute offered rate 0 bps, drop rate 0 bps
      Match: any
```

Исходную информацию о трафике для составления QoS политик можно собрать в реальном времени c помощью того же nbar:
```

R2(config)#int fa0/1
R2(config-if)#ip nbar protocol-discovery
R2(config-if)#^Z
R2#sh ip nbar protocol-discovery top-n 2

 FastEthernet0/1
                            Input                    Output
                            -----                    ------
   Protocol                 Packet Count             Packet Count
                            Byte Count               Byte Count
                            5min Bit Rate (bps)      5min Bit Rate (bps)
                            5min Max Bit Rate (bps)  5min Max Bit Rate (bps)
   ------------------------ ------------------------ ------------------------
   icmp                     8333                     8430
                            1003954                  1093120
                            0                        0
                            22000                    22000
   bgp                      0                        0
                            0                        0
                            0                        0
                            0                        0
   unknown                  0                        0
                            0                        0
                            0                        0
                            0                        0
   Total                    8333                     8430
                            1003954                  1093120
                            0                        0
                            22000                    22000
```

Но куда кошернее делать это с помощью netflow.
