---
title: Регистрируем A и PTR записи linux сервера на Windows DNS
author: ["Stanislav"]
date: 2012-10-17T09:26:48+00:00
url: /2012/10/register-a-ptr-records-of-linux-server-on-windows-dns/
categories:
  - Tech
tags:
  - debian
  - dns
  - howto
  - linux
  - microsoft
  - windows
---

Включим опцию и сменим имя хоста на необходимое:
```

root@localhost:~# cat /etc/dhcp/dhclient.conf | grep 'send host-name'
send host-name "server.fqdn.domain.ltd";
```

Открыв оснастку DHCP и выбрав свойства области включим опцию “Динамически обновлять A- и PTR- записи для DHCP клиентов не требующих обновления (например, клиенты с Windows NT 4.0)” во вкладке Служба DNS.
