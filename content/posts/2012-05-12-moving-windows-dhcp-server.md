---
title: Перенос Windows DHCP сервера
author: ["Stanislav"]
date: 2012-05-12T10:34:55+00:00
url: /2012/05/moving-windows-dhcp-server/
categories:
  - Tech
tags:
  - dhcp
  - troubleshooting
  - Windows Server 2003
  - Windows Server 2003R2
  - Windows Server 2008R2

---
В жизни бы не подумал, что мне потребуется это знание больше одного раза, а случилось так, что за последнюю неделю потребовалось перенести аж три штуки.

И если один из трех перенесся нормально простой операцией Архивировать > Восстановить, то с двумя другими возникла следующая проблема: база восстанавливалась, но вкладка арендованные адреса (leases) была девственно пуста, более того при попытке доступа к ней в журнале событий возникала ошибка:

```Possible Memory Leak.  Application (&#8220;C:\Windows\system32\mmc.exe&#8221; &#8220;C:\Windows\system32\dhcpmgmt.msc&#8221; )```

Правильно перенести DHCP сервер с Windows 2003 и Windows 2003R2 на Windows 2008R2 помогла следующая последовательность:

1. Был остановлен DHCP сервер на Windows 2008R2 серверах.  
2. Удалено содержимое %windir%\system32\dhcp  
3. На Windows 2003/Windows 2003R2 серверах была выполнена команда

```netsh dhcp server export C:\dhcp.txt all```

а dhcp.txt был перенесен на целевые сервера.  
4. На Windows 2008R2 серверах был запущен DHCP сервер и выполнена команда  
```netsh dhcp server import C:\dhcp.txt all```

После этой процедуры dhcp сервера заработали в штатном режима, а mmc оснастка перестала радовать memory leak&#8217;ом.