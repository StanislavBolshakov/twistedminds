---
title: Исправляем кодировку в PDF/CSV отчетах в GLPI 0.80.2
author: ["Stanislav"]
date: 2011-08-29T12:43:04+00:00
url: /2011/08/codepage-pdf-csv-error-in-glpi/
categories:
  - Tech
tags:
  - codepage
  - fusioninventory
  - glpi
  - решаем проблемы
---

Если для вас такой отчет:
![GLPI PDF CSV codepage issue](/wp-content/uploads/2011/08/Снимок-экрана-2011-08-29-в-16.18.15.png "GLPI PDF CSV codepage issue")
является несколько менее читаемым нежели такой:
![GLPI PDF CSV codepage issue fix](/wp-content/uploads/2011/08/Снимок-экрана-2011-08-29-в-16.19.10.png "GLPI PDF CSV codepage issue fix")
есть смысл заморочиться с исправлением кодировки и сейчас мы постараемся причесать GLPI, чтобы PDF/CSV отчеты стали мягкими и шелковистыми.
Для начала берем шрифты [отсюда](http://sisyphus.ru/ru/srpm/Sisyphus/glpi/sources "http://sisyphus.ru/ru/srpm/Sisyphus/glpi/sources") или [отсюда](/fonts.tar.gz "GLPI patched fonts").
Заменяем содержимое каталога
```/var/www/glpi/lib/ezpdf/fonts``` содержимом архива и патчим search.class.php
``` sed -i -e ‘s/windows-1252/windows-1251/g’ search.class.php``` который лежит в ```/var/www/glpi/inc/```
Если вам нужны CSV отчеты, то надо сделать резервную копию search.class.php, и пропатчить ее. После этого FusionInventory откажется переносить данные в GLPI. Вот [патч](/patch.patch "/patch.patch"), вот [diff](/diff.txt "/diff.txt").
