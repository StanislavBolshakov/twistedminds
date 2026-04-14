---
title: Dr.Web ES Database verification failed
author: ["Stanislav"]
date: 2010-04-13T08:43:02+00:00
url: /2010/04/dr-web-es-database-verification-failed/
categories:
  - Tech
tags:
  - dr.web es
  - решаем проблемы
---

Новая работа, новые проблемы. Много, много новых проблем.
Одна из проблем возникла в результате запуска Dr.Web Enterprise Server, после чего в логе оставался такой выхлоп:
```20100413.100131.65 inf [ 5388] noname Dr.Web (R) Enterprise Server REL-500 Build 5.00.0.200908050 (WinNT 5.2/PPro) has terminated
20100413.100131.65 ERR [ 5388] noname Server execution failed because of
20100413.100131.65 ERR [ 5388] noname   database verification failed
20100413.100131.65 ntc [ 5388] noname [Server] Process exit code is 0x42```
Гугление выхлопа результатов не дало, служба поддержки Dr.Web обитала только в DC, так что пришлось обратить к официальному форуму.
Первым советом стала попытка ручного сжатия базы (метод описан [тут](http://wiki.drweb.com/index.php/%D0%A0%D1%83%D1%87%D0%BD%D0%BE%D0%B5_%22%D1%81%D0%B6%D0%B0%D1%82%D0%B8%D0%B5%22_%D0%B2%D0%BD%D1%83%D1%82%D1%80%D0%B5%D0%BD%D0%BD%D0%B5%D0%B9_%D0%B1%D0%B0%D0%B7%D1%8B_Dr.Web%C2%AE_Enterprise_Suite "ручное сжатие")), но сия процедура завершилась с ошибками:
```drwidbsh> .read delold
DELETE FROM station_offline WHERE starttime <= 20100409000000000;
SQL error: no such table: station_offline
DELETE FROM update_state WHERE rectime <= 20100409000000000;
SQL error: no such table: update_state
DELETE FROM deleted_stations WHERE created <= 20100409000000000;
SQL error: no such column: created
VACUUM;
SQL error: columns sid, gid are not unique```
Пришлось воспользоваться вторым советом, а именно восстановить базу из бэкапа. Делается это так:
* Останавливаем ES сервер, если он запущен (в моем случае он вообще не запускался)
* Инициализируем новую базу данных командой ```”C:\Program Files\DrWeb Enterprise Server\bin\drwcsd.exe” -home=”C:\Program Files\DrWeb Enterprise Server” -var-root=”C:\Program Files\DrWeb Enterprise Server\var” -verbosity=all initdb c:\agent.key – – root```  при условии, что DrWeb ES установлен в Program Files, а agent.key лежит в корне диска C:
* Восстанавливаем из резервной копии ```%ES%\var\backup\%guid%\%date%``` командой ```”C:\Program Files\DrWeb Enterprise Server\bin\drwcsd.exe” -home=”C:\Program Files\DrWeb Enterprise Server” -var-root=”C:\Program Files\DrWeb Enterprise Server\var” -verbosity=all importdb “disc:\path\_to\_the\_backup\_file\database.dz”```
* Запускаем сервер и подключаемся к консоли.
