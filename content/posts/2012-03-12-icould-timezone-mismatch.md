---
title: Ошибка с часовым поясом iCal календаре в iCloud
author: ["Stanislav"]
date: 2012-03-12T09:02:14+00:00
url: /2012/03/icould-timezone-mismatch/
categories:
  - Tech
tags:
  - apple
  - ical
  - icloud
  - timezone mismatch
  - временной пояс
  - решаем проблемы
---
![ical logo](/wp-content/uploads/2012/03/hero_ical.jpg "ical logo")Симптомы проблемы следующие: вы создаете событие через вэб интерфейс iCloud, с вашими iOS устройствами события синхронизируются нормально, однако в календаре iCloud время начала события установлено на час раньше или на час позже. При этом в свойствах события в том же вэб календаре время верное.
Исправить это можно следующим образом:
1. Убедимся что у нас выставлен правильный часовой пояс в настройках учетной записи iCloud.

[![icloud timezone settings](/wp-content/uploads/2012/03/IMG_01021-300x217.jpg "icloud timezone settings")](/wp-content/uploads/2012/03/IMG_01021.jpg)

2. Убедимся что у нас включена настройка “Включить поддержку часового пояса” в настройках iCal.
[![ical timezone settings](/wp-content/uploads/2012/03/IMG_01041-300x177.jpg "ical timezone settings")][1]
3. Перейдем к событию, имеющему ошибку в выставленном времени.
[![ical wrong timezone](/wp-content/uploads/2012/03/IMG_01051-300x176.jpg "ical wrong timezone")][2]
4. Исправим часовой пояс с “Стандартное московское” на “Плавающий”.

[![ical floating timezone](/wp-content/uploads/2012/03/IMG_01031-300x176.jpg "ical floating timezone")](/wp-content/uploads/2012/03/IMG_01031.jpg)

5. ???
6. Profit.
