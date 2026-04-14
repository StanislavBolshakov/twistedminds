---
title: Canon UFRII 10322 error на MacOS X
author: ["Stanislav"]
date: 2010-12-21T14:35:52+00:00
url: /2010/12/canon-ufrii-10322-error/
categories:
  - Tech
tags:
  - canon
  - macos
  - ufr
  - решаем проблемы
---

На 10.6.5 была предпринята попытка поставить UFRII драйвер Canon, с тем, чтобы завести irc2380i. После установки драйверов и подключения принтера печать обламывалась с ошибкой 10322
```”Cannot communicate with the printer, or the printer is not supported.”```
Решилась проблема следующим образом –
* удален принтер
* удалены следующие файлы и каталоги:
```System HDD/Library/Printers/Canon/UFR2
System HDD/Library/Launch Agents/jp.co.canon.UFR2.BG.plist
System HDD/Library/Printers/PPDs/Contents/Resources/CNTD**\*Z\*2*.ppd.gz```
* очищена корзина
* системы была запущены в safe-mode (shift во время загрузки до появления полосы)
* с помощью Disk Utility (DU) [были исправлены права](http://support.apple.com/kb/HT1452?viewlocale=en_US "http://support.apple.com/kb/HT1452?viewlocale=en_US") [(2)](https://kb.iu.edu/data/aoxn.html "https://kb.iu.edu/data/aoxn.html")
* установлен UFRII 2.10 (!!!) драйвер
* система загружена в обычном режиме
* установлен UFRII 2.20 драйвер поверх
Только после этого я смог печатать без каких-либо ошибок.
