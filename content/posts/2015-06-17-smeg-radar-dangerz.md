---
title: Радары и опасные зоны для SMEG+
author: ["Stanislav"]
date: 2015-06-17T12:47:51+00:00
url: /2015/06/smeg-radar-dangerz/
categories:
  - Tech
tags:
  - navi
  - navigation
  - poi
  - radar
  - smeg
  - smeg plus
---

В процессе рестайлинга 508 французы поменяли голову с RT6 на SMEG+, а вместе с ней и формат файлов обновления камер и зон для головы.

В отличии от RT6 файлы с координатами упакованы в tar gz архив.
```

delbook:POI_USER delirium$ file ZAR_POI.BIN
ZAR_POI.BIN: gzip compressed data, was "TMP_POI.TAR", from NTFS filesystem (NT), last modified: Thu Nov 6 17:38:52 2014
```

Рядом с ним лежит ZAR_POI.BIN.inf файл, в котором описаны контрольные суммы и другая служебная информация следующего содержания:
```

delbook:POI_USER delirium$ cat ZAR_POI.BIN.inf
67351db4
CONTINENT_ID:01
CONTINENT_NAME:EUROPE
MEDIA_NAME:SMEG_PLUS_I_ZAR_2014-12-03
POI_PROVIDER:PSA
VERSION:20141106
CID_SIZE_32:2359296
CID_SIZE_16:1818624
CID_SIZE_8:1589248
CID_SIZE_4:1482752
CID_SIZE:1368946
```

Где:
* Первая строчка (**67351db4**) контрольная сумма, вычисляемая через задницу (об этом ниже);
* **CONTINENT_ID** – порядковый номер континента. Россия, при этом, выделена в отдельный (!) континент с ID 017;
* **CONTINENT_NAME **– название континента;
* **MEDIA_NAME **– название файла обновления;
* **POI_PROVIDER** – автор обновления;
* **VERSION** – версия базы данных;
* **CID_SIZE** – Размер файла каталога и файла ZAR_POI.BIN. **\_4,\_8,\_16,\_32** размер каталога и файла на диске при разном размере блока файловой системы FAT32 в 4, 8, 16 и 32 кб.
Обычным способом CRC посчитать не выйдет.
```

delbook:POI_USER delirium$ crc32 ZAR_POI.BIN
f41e8763
```

Сравните с содержванием INF файла.

Вычисление контрольной суммы разделяется на два этапа. Сначало вам нужно взять программу [RTXcrc](http://mira308sw.altervista.org/) и посчитать контрольную сумму архива ZAR_POI.BIN.
```

C:\SMEG\RTXcrc (v01.00)>RTXcrc.exe -v C:\SMEG\SMEG_PLUS_UPG\DATA\MAPPE\POI_USER\ZAR_POI.BIN
RTXcrc v01.00: by mira308sw. RT4/RT5 crc calculator.

CRC = 1db4
```

И записать полученное значение первой строчкой в INF файл. Затем считаем контрольную сумму уже от INF файла:
```

C:\SMEG\RTXcrc (v01.00)>RTXcrc.exe -v C:\SMEG\SMEG_PLUS_UPG\DATA\MAPPE\POI_USER\ZAR_POI.BIN.inf
RTXcrc v01.00: by mira308sw. RT4/RT5 crc calculator.

CRC = 6735
```

Сложив CRC из второго расчета с первым мы получим исходное значение 6735, 1db4 => 67351db4.
##### Формат  ZAR_POI.BIN
Как я уже писал этот файл представляет собой tar gz архив. Внутри GZ архива расположен TMP_POI.TAR, в котором, в свою очередь лежат файлы обновлений для разных стран. 001 – Италия, 002 – Франция и т.д. В свою очередь в каталогах стран лежат радары или опасные зоны.
```

DANGERZ.LZW
DANGERZDSC.LZW
DANGERZ_PE.LZW
DANGERZ_PS.LZW
POI_VER_DANGERZ.TXT
```

```

POI_VER_RADAR.TXT
RADAR.LZW
RADARDSC.LZW
RADAR_PE.LZW
RADAR_PS.LZW
```

Особую ценность представляют POI\_VER\_DANGERZ.TXT и POI\_VER\_RADAR.TXT. В них содержаться следующие переменные
* **POI_PROVIDER** – автор обновления;
* **POI\_MACRO\_CAT** – ?;
* **POI_CAT** – категория точек. 41 – камеры, 42 – опасные зоны;
* **DATE**:20150505;
* **DATA_POI**:05/05/2015;
* **NAME_ICON**:BMP;
* **NAME_SOUND**:SOUND;
* **DESCRIPTION** – Описание: RADAR CONTROL или DANGEROUS ZONE;
* **PREFIX **– Префикс: RADAR или DANGERZ;
* **NAME_COUNTRY** – название страны;
* **CID **– ID Страны (не путать с ID континента), 33 для России;
* **NUMBER_POI **– количество камер или зон в файле обновления;
* **POI_SIZE **– размер всех файлов (LZW + TXT), \_4, \_8, \_16, \_32 – размер, который занимают файлы на файловой системе FAT32 с размером блока в 4, 8, 16 и 32 килобайта.
##### **Процесс сбора файла обновлений**
1. Вам потребуются файлы с радарами или опасными зонами. Для создания этих файлов из сsv можно (и нужно) использовать [RadarViewer](http://mira308sw.altervista.org/en/index.htm) и затем дополнить TXT файл необходимой информацией.
2. Затем файлы нужно поместить в каталог, соответствующий ID страны. Для России это 033.
3. Затем каталог с POI помещается еще в один – TMP_POI.
4. TMP\_POI помещается в tarball TMP\_POI.TAR
5. TMP\_POI.TAR упаковывается в gzip архив ZAR\_POI.BIN
6. Обновляется ZAR_POI.BIN.inf файл с правильными размерами и контрольной суммой.
PS. Проблема в том, что не смотря на то, что все размеры и контрольные суммы корректны мое ГУ отказывается принимать такой файл, выдавая ошибку Файл обновления невозможно скопировать.
##### Типичные ошибки
1. Данные на медианосителе повреждены – ошибки в заполнении контрольных сумм или размеров файлов. (CID\_SIZE, POI\_SIZE). Иногда нужно отформатировать накопитель и попробовать снова.![20150617_132134](/wp-content/uploads/2015/06/20150617_132134.jpg)
2. Файл обновления невозможно скопировать.![20150617_093658](/wp-content/uploads/2015/06/20150617_093658.jpg)
3. Не удалось проверить совместимость – неверно указаны Country ID (CID) или Continent ID.
![20150617_095256](/wp-content/uploads/2015/06/20150617_095256.jpg)
