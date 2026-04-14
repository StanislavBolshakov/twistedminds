---
title: Не применяются vbs logon скрипты в Windows Vista / 7
author: ["Stanislav"]
date: 2010-04-19T10:38:13+00:00
url: /2010/04/vbs-logon-startup-issues/
categories:
  - Tech
tags:
  - vbs
  - windows
  - решаем проблемы

---
Извечный вопрос &#8220;что делать?&#8221;. Сегодня logon vbs скрипты, к примеру, не монтируют сетевые шары.  
&#8220;Кто виноват?&#8221; &#8211; это и предстоит выяснить.  
Для начала перейдите в ```\\DOMAIN\SYSVOL\DOMAIN\Policies\POLICY%GUID\User\Scripts\Logon``` и попробуйте запустить скрипт.  
Шары не подключились? Надо поставить правильные права на скрипт.

Шары подключились? Тогда создайте рег-файл следующего содержания и запустите его:

```Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System]  
&#8220;EnableLinkedConnections&#8221;=dword:00000001```