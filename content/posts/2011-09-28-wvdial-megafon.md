---
title: 'wvdial & megafon'
author: ["Stanislav"]
date: 2011-09-28T06:49:15+00:00
url: /2011/09/wvdial-megafon/
categories:
  - Tech
tags:
  - 3g
  - howto
  - huawei
  - linux
  - megafon
  - modem
  - wvdial

---
```  
root@localhost:~# cat /etc/wvdialer.conf

Init1 = ATZ  
Init2 = AT+CGDCONT=1,&#8221;IP&#8221;,&#8221;internet.nw&#8221;  
Baud = 115200  
New PPPD = yes  
Modem = /dev/ttyUSB0  
Phone = *99#  
Password = internet  
Username = internet  
Abort on Busy = on  
Stupid Mpde = yes  
```