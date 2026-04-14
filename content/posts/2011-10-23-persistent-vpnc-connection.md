---
title: Постоянное соединение vpnc
author: ["Stanislav"]
date: 2011-10-23T13:45:18+00:00
url: /2011/10/persistent-vpnc-connection/
categories:
  - Tech
tags:
  - bash
  - howto
  - ipsec
  - vpn
  - vpnc

---
Сбыдлокодил на коленке скрипт, поддерживающий соединение с vpn-шлюзом:

```  
#!/bin/bash  
if  
[ &#8220;$(/sbin/route -n | /bin/grep &#8220;172.16.10.0&#8221; | /usr/bin/awk &#8220;{print \$8}&#8221;)&#8221; = &#8220;tun0&#8221; ]  
then  
echo &#8220;$(/bin/date &#8220;+%X %x&#8221;)&#8221; route exist >> /var/log/vpnc.log  
else  
echo &#8220;$(/bin/date &#8220;+%X %x&#8221;)&#8221; no route to scse &#8211; reconnecting >> /var/log/vpnc.log  
if [ -f /var/run/vpnc/pid ]  
then  
echo &#8220;$(date &#8220;+%X %x&#8221;)&#8221; found vpnc pid &#8211; deleting >> /var/log/vpnc.log  
&#8220;$(/bin/rm -f /var/run/vpnc/pid)&#8221;  
fi  
echo &#8220;$(date &#8220;+%X %x&#8221;)&#8221; establishing scse vpn tunnel >> /var/log/vpnc.log  
&#8220;$(/usr/sbin/vpnc /etc/vpnc/scse.conf)&#8221;  
fi

```