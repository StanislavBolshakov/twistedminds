---
title: Event ID 12 – предупреждения от W32TIME
author: ["Stanislav"]
date: 2010-04-19T11:03:31+00:00
url: /2010/04/ntp-client-event-id-12-pdc-warning/
categories:
  - Tech
tags:
  - ntp
  - w32time
  - windows
  - решаем проблемы

---
После переноса первичного контроллера домена (PDC) стал получать сообщения вида: ```

NTP-клиент поставщика времени: этот компьютер использует доменную структуру для определения своего источника времени, но для этого домена в корне леса находится PDC-эмулятор&#8230;```

Лечится следующим образом:

```  
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\Config]  
&#8220;AnnounceFlags&#8221;=dword:00000005

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\Parameters]  
&#8220;NtpServer&#8221;=&#8221;north-america.pool.ntp.org,0×1  
&#8220;Type&#8221;=&#8221;NTP&#8221;

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\TimeProviders\NtpServer]  
&#8220;Enabled&#8221;=dword:00000001  
```  
И перезапускам сервис:  
```net stop w32time && net start w32time```