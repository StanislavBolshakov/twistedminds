---
title: Шлюз по умолчанию в другой подсети
author: ["Stanislav"]
date: 2012-09-20T11:09:53+00:00
url: /2012/09/default-gateway-on-different-subnet/
categories:
  - Tech
tags:
  - arp
  - networking
---
В процессе очередной сессии общения с [@hellt_ru](https://twitter.com/hellt_ru "@hellt_ru") по поводу одного из его проектов встал вопрос правомерности существования шлюза по умолчанию в другой подсети. Резкое и категоричное [“НЕТ, ГРЕШНО”](https://twitter.com/pravoslavno "ГРЕШНО!") плавненько сменилось на “oh, snap”.

Изучение RFC ([в частности 1122](http://tools.ietf.org/html/rfc1122 "rfc 1122")) выявило следующие забавные факты:

* Хост обязан передавать пакеты напрямую если адрес назначения находится в той же подсети.
* Хост обязан передавать пакеты шлюзу по умолчанию, если адрес назначения находится в иной подсети.
* Но нигде не сказано, что шлюз по умолчанию должен находится в той же подсети. Достаточно, чтобы эта сеть была connected :)
Так и получается:
![](/wp-content/uploads/2012/09/wtf.png "wtf")
```

R1#ping 10.1.1.2

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.1.1.2, timeout is 2 seconds:
..!!!
Success rate is 60 percent (3/5), round-trip min/avg/max = 40/45/56 ms
```

R1 рассылает на широковещательный адрес запрос о том, есть ли в connected сети 192.168.0.140. R2 отвечает ему – “да, это я, вот мой mac”. Больше этих ребят ничего не интересует:
[![](/wp-content/uploads/2012/09/default-gw.png "wireshark")][1]
All hail ARP!
Добавлено:

В комментариях к статье возник небольшой спор по поводу работы всей этой радости на shared сегменте. Возьмем следующую схему:![](/wp-content/uploads/2012/09/even-more-wtf.png "even-more-wtf")

Допустим мы хотим с R1 достичь lo0 на R5.
R1:
```

R1#sh ip route | b Gate
Gateway of last resort is 0.0.0.0 to network 0.0.0.0

C    192.168.1.0/24 is directly connected, FastEthernet0/0
S*   0.0.0.0/0 is directly connected, FastEthernet0/0
```

Ситуация в точности такая же. R1 с Fa0/0 отправит броадкаст всем соседям на L2 сегменте с вопросом у кого есть IP 10.1.1.1, на что получит ответ от R3 – “я знаю, выберете меня, вот мой mac”. В этом случае default gateway в явном виде вообще не задан.

[![](/wp-content/uploads/2012/09/wiredump-300x44.png "wiredump")](/wp-content/uploads/2012/09/wiredump.png)
