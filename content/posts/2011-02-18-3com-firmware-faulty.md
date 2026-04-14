---
title: Проблема с прошивкой 3com 4210
author: ["Stanislav"]
date: 2011-02-18T08:23:59+00:00
url: /2011/02/3com-firmware-faulty/
no_lj:
  - 1
categories:
  - Tech
tags:
  - 3com
  - 4210
  - sfp
  - решаем проблемы
---

Нашел на складе старые трансиверы Picolight и возникла фантазия в связи с скорым переносом серверной связать циску с 3com 2мя гигабитными оптическими шнурками. После того, как [циска таки съела Picolight](/2011/02/cisco-unsupported-sfp-module/ "/2011/02/cisco-unsupported-sfp-module/") я начал дружить эти модули с 3com 4210. Свичи обладали какой-то старой прошивкой и радостно рапортовали о Module Faulty, а в статистике интерфейса лавинообразно росли input/output errors.

Первое, что пришло в голову – залезть на сайт 3com и скачать свежачек, но не тут-то было. HP,  купив 3com, угробил раздел downloads и редиректил на свой сайт, на котором черт ногу сломит. Фирмваря ищется по Product # 3CR17333A-91 и ведет на страницу [http://h17007.www1.hp.com/us/en/support/converter/index.aspx?productNum=JF427A](http://h17007.www1.hp.com/us/en/support/converter/index.aspx?productNum=JF427A "http://h17007.www1.hp.com/us/en/support/converter/index.aspx?productNum=JF427A") для 26-и портового 4210. Но не спешите его ставить :)

Скачав прошивку и распоковав архив я обнаружил файлы с расширениями btm (bootrom), web (web interface) и … bin вместо app :) В вебинтерфейсе в требованиях к фирмвари были файлы bin/app и на этом я успокоился. Сделав бэкапп прошивки, бутрома и вэбморды я приступив к обновлению (процесс описан у Николая Ульянитского (надеюсь написал фамилию правильно) [http://blog.lystor.org.ua/2010/04/3com-switch-4210-software-upgrading.html](http://blog.lystor.org.ua/2010/04/3com-switch-4210-software-upgrading.html "http://blog.lystor.org.ua/2010/04/3com-switch-4210-software-upgrading.html")).

У 4210 не хватает ROM, по-этому старую прошивку приходится затирать, а новая… у меня не встала :) Bootrom прошился удачно, а firmware вылетала с ошибкой **Set bootfile unsuccessfully on unit 1!** Parameter(s) invalid, вне зависимости от того шился ли я из вэбморды или из  CLI.

Решив все вернуть в зад и залив старые бутром и прошивку обратно меня ждал небольшой fail – bootrom не шился, вылетая с ошибкой **Upgrade Bootrom failed !**, а firmware с уже надоевшей **Set bootfile unsuccessfully on unit 1!** Видимо произошел косяк во время трансфера на tftp или еще по какой-то неведомой мне причине.

Спас меня Николай, выложивший s4o03_01_12s56.app и s4p04_08.btm, с которыми все вышло в лучшем виде.

```
<4210>display interface GigabitEthernet 1/0/27
GigabitEthernet1/0/27 current state : UP
IP Sending Frames’ Format is PKTFMT\_ETHNT\_2, Hardware address is 0024-738f-eb47
Media type is not sure, loopback not set
Port hardware type is SFP\_UNKNOWN\_CONNECTOR
1000Mbps-speed mode, full-duplex mode
Link speed type is autonegotiation, link duplex type is autonegotiation
Flow-control is not enabled
The Maximum Frame Length is 1536
Broadcast MAX-ratio: 100%
PVID: 1
Mdi type: auto
Port link-type: access
Tagged VLAN ID : none
Untagged VLAN ID : 1
Last 300 seconds input: 36 packets/sec 2950 bytes/sec
Last 300 seconds output: 2 packets/sec 650 bytes/sec
Input(total): 165197 packets, 13301628 bytes
51939 broadcasts, 12120 multicasts, 0 pauses
Input(normal): 165197 packets, 13301628 bytes
51939 broadcasts, 12120 multicasts, 0 pauses
Input: 0 input errors, 0 runts, 0 giants, – throttles, 0 CRC
0 frame, – overruns, 0 aborts, – ignored, – parity errors
Output(total): 9179 packets, 2483625 bytes
14 broadcasts, 79 multicasts, 0 pauses
Output(normal): 9179 packets, – bytes
14 broadcasts, 79 multicasts, – pauses
Output: 0 output errors, – underruns, – buffer failures
0 aborts, 0 deferred, 0 collisions, 0 late collisions
– lost carrier, – no carrier
```
Их и оставлю тут, если кому-нибудь пригодится :)
[/3com.tar.bz2](/3com.tar.bz2 "/3com.tar.bz2")
