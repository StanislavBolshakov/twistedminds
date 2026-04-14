---
title: Шрифты в линуксе
author: ["Stanislav"]
date: 2009-09-21T17:36:03+00:00
url: /2009/09/windows-fonts-in-linux/
categories:
  - Tech
tags:
  - fonts
  - linux
  - решаем проблемы
---

В отличии от России у линукса бед гораздо больше и проблема рендеринга шрифтов занимает не последнее место. Шрифты такие, что, порой, хочется вырвать свои глаза.

Но рецепт кошерных шрифтов есть и он, как обычно, лежит на первой странице гугла. Но так как все равно никто не пользуется поиском, а от размазанных шрифтов тянет  поставить Windows и забыть об этом кошмаре я, приняв волевое решение, задумал написать статейку, а точнее перевод хау-то с [http://www.sharpfonts.com/](http://www.sharpfonts.com/ "http://www.sharpfonts.com/")

Первое качаем эти файлы:

> [andale32.exe](http://www.sharpfonts.com/fonts/arial32.exe)
>  [arial32.exe](http://www.sharpfonts.com/fonts/arial32.exe)
>  [arialb32.exe](http://www.sharpfonts.com/fonts/arialb32.exe)
>  [comic32.exe](http://www.sharpfonts.com/fonts/comic32.exe)
>  [courie32.exe](http://www.sharpfonts.com/fonts/courie32.exe)
>  [georgi32.exe](http://www.sharpfonts.com/fonts/georgi32.exe)
>  [impact32.exe](http://www.sharpfonts.com/fonts/impact32.exe)
>  [tahoma32.exe](http://www.sharpfonts.com/fonts/tahoma32.exe)
>  [times32.exe](http://www.sharpfonts.com/fonts/times32.exe)
>  [trebuc32.exe](http://www.sharpfonts.com/fonts/trebuc32.exe)
>  [verdan32.exe](http://www.sharpfonts.com/fonts/verdan32.exe)
>  [webdin32.exe](http://www.sharpfonts.com/fonts/webdin32.exe) и
>  [xml files](http://www.sharpfonts.com/fontconfig.tbz)

Второе:

Убеждаемся в том, что у нас стоит fontconfig, cabextract, переходим в каталог с загруженными шрифтами и делаем:

> **mkdir -p /usr/share/fonts/truetype/**
>  **cabextract -d /usr/share/fonts/truetype *.exe**
>  **tar xvjpf *.tbz -C /etc/fonts/**

Осталось перезапустить иксы.

Просто – как орбит “просто” (с)
