---
title: Citrix license server крашится сразу после старта
author: ["Stanislav"]
date: 2011-07-29T06:48:45+00:00
url: /2011/07/citrix-license-server-crashes-right-after-start/
categories:
  - Tech
tags:
  - citrix
  - license server
  - xenapp
  - решаем проблемы
---

Проблема, которая, как говорят люди, тянется еще с времен XenApp 5.
**Симптомы:**
Сервер лицензий Citrix запускается, но сразу после старта аварийно завершает свою работу. В логе событий Windows остается следующая запись:
```Имя журнала: Application
Источник: Application Error
Дата: 29.07.2011 10:58:49
Код события: 1000
Категория задачи:(100)
Уровень: Ошибка
Ключевые слова:Классический
Пользователь: Н/Д
Компьютер: testnode.meh.domain
Описание:
Имя сбойного приложения: lmadmin.exe, версия: 11.9.0.0, отметка времени: 0x4d6bbff6
Имя сбойного модуля: lmadmin.exe, версия: 11.9.0.0, отметка времени 0x4d6bbff6
Код исключения: 0xc0000005
Смещение ошибки: 0x0005e222
Идентификатор сбойного процесса: 0x15e0
Время запуска сбойного приложения: 0x01cc4dbcf66a55c3
Путь сбойного приложения: C:\Program Files (x86)\Citrix\Licensing\LS\lmadmin.exe
Путь сбойного модуля: C:\Program Files (x86)\Citrix\Licensing\LS\lmadmin.exe
Код отчета: 356fef33-b9b0-11e0-9c17-000e0c3ce3c3
Xml события:

1000
2
100
0x80000000000000

6763
Application
testnode.meh.domain

lmadmin.exe
11.9.0.0
4d6bbff6
lmadmin.exe
11.9.0.0
4d6bbff6
c0000005
0005e222
15e0
01cc4dbcf66a55c3
C:\Program Files (x86)\Citrix\Licensing\LS\lmadmin.exe
C:\Program Files (x86)\Citrix\Licensing\LS\lmadmin.exe
356fef33-b9b0-11e0-9c17-000e0c3ce3c3
```
**Официальный workaround:**
The Citrix License Server might crash immediately after startup. Workaround: Delete the c:\program files\citrix\licensing\ls\conf\ activation\_state.xml and concurrent\_state.xml files and then start the license server. These files are automatically regenerated when the license server starts. [[#253576]](http://support.citrix.com/proddocs/topic/licensing-119/lic-about-11-9.html "About Citrix Licensing 11.9 for Windows (Build 11007)")
