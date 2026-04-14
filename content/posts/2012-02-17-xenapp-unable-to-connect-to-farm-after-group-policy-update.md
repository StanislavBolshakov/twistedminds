---
title: Citrix XenApp 6.5 – невозможность подключения к ферме после применения групповых политик
author: ["Stanislav"]
date: 2012-02-17T06:41:49+00:00
url: /2012/02/xenapp-unable-to-connect-to-farm-after-group-policy-update/
categories:
  - Tech
tags:
  - citrix
  - Windows Server 2008R2
  - xenapp
  - решаем проблемы
---
[![XenApp Logo](/wp-content/uploads/2012/02/XenAppLogo.png "XenApp Logo")][1]Если в результате применения групповой политики (легко повторить баг выполнив gpupdate /force) ваши сервера Citrix XenApp становятся недоступны для пользователей и к примеру возникают ошибки вида
```Возникла ошибка при создании запрошенного подключения.```
на портале опубликованных приложений, а qfarm /load возвращает отчет о том, что сервер запрещает новые подключения
```
C:\Users\administrator>qfarm /load
Server Name Server Load Load Throttling Load Logon Mode
——————– ———– ——————– ——————-
CTRX0 200 0 ProhibitLogons
```
значит, скорее всего, вы натолкнулись на этот досадный баг.
Не смотря на то, что Citrix об этом известно бог знает сколько патча, к несчастью, нет, но есть официальный workaround – установить значение ключа реестра fDenyTSConnections, расположенного по пути
```HKEY\_LOCAL\_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server```
в “0”.
На сколько я знаю баг затрагивает XenApp 6.5 (некоторые жалуются и на XenApp 6), установленном на Windows Server 2008R2 с Service Pack 1.
