---
title: Оценка производительности raid-массива
author: ["Stanislav"]
date: 2010-08-26T07:03:16+00:00
url: /2010/08/raid-massive-performance-benchmark/
categories:
  - Tech
---

В силу внедрения 1с-предприятия (буээээ) было принято решение обзавестись новым сервером баз данных в замен отдающего концы старого.

На raid-контроллере SRCSASBB8I (raid10) при использовании политик чтения normal и записи write-through были получены следующие результаты для чтения/записи при объеме теста 50mb и 1000mb:

```
Sequential Read : 255.045 MB/s
Sequential Write : 235.498 MB/s
Random Read 512KB : 73.339 MB/s
Random Write 512KB : 139.554 MB/s
Random Read 4KB (QD=1) : 1.127 MB/s [ 275.1 IOPS]
Random Write 4KB (QD=1) : 5.884 MB/s [ 1436.4 IOPS]
Random Read 4KB (QD=32) : 9.335 MB/s [ 2279.2 IOPS]
Random Write 4KB (QD=32) : 6.019 MB/s [ 1469.4 IOPS]
Test : 50 MB [D: 0.0% (0.2/1660.8 GB)] (x9)
```
```
Sequential Read : 268.659 MB/s
Sequential Write : 284.862 MB/s
Random Read 512KB : 77.980 MB/s
Random Write 512KB : 117.371 MB/s
Random Read 4KB (QD=1) : 0.946 MB/s [ 231.0 IOPS]
Random Write 4KB (QD=1) : 4.962 MB/s [ 1211.3 IOPS]
Random Read 4KB (QD=32) : 7.329 MB/s [ 1789.2 IOPS]
Random Write 4KB (QD=32) : 4.816 MB/s [ 1175.8 IOPS]
Test : 1000 MB [D: 0.0% (0.2/1660.8 GB)] (x5)
```
Сильно улучшить ситуацию в случае 50мб позволяет включение write-back и read-ahead. Для 1000мб теста результаты бенчмарка немного ухудшаются, но это жертва на которую я готов пойти.
**ня!:**
```
Sequential Read : 900.660 MB/s
Sequential Write : 638.849 MB/s
Random Read 512KB : 908.009 MB/s
Random Write 512KB : 611.975 MB/s
Random Read 4KB (QD=1) : 92.827 MB/s [ 22662.9 IOPS]
Random Write 4KB (QD=1) : 84.774 MB/s [ 20696.8 IOPS]
Random Read 4KB (QD=32) : 206.488 MB/s [ 50412.0 IOPS]
Random Write 4KB (QD=32) : 189.847 MB/s [ 46349.4 IOPS]
Test : 50 MB [D: 0.0% (0.2/1660.8 GB)] (x9)
```
```
Sequential Read : 262.669 MB/s
Sequential Write : 283.437 MB/s
Random Read 512KB : 56.689 MB/s
Random Write 512KB : 113.080 MB/s
Random Read 4KB (QD=1) : 0.811 MB/s [ 198.1 IOPS]
Random Write 4KB (QD=1) : 5.068 MB/s [ 1237.2 IOPS]
Random Read 4KB (QD=32) : 7.207 MB/s [ 1759.5 IOPS]
Random Write 4KB (QD=32) : 4.692 MB/s [ 1145.5 IOPS]
Test : 1000 MB [D: 0.0% (0.2/1660.8 GB)] (x5)
```
