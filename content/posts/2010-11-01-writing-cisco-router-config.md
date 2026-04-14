---
title: Конфигурация роутеров Cisco для самых маленьких (intro)
author: ["Stanislav"]
date: 2010-11-01T14:43:47+00:00
url: /2010/11/writing-cisco-router-config/
categories:
  - Tech
tags:
  - cisco
---

Сижу и настраиваю свой первый роутер, судорожно вспоминая материал курсов ICND1/2, просматривая маны и хэндбуки. Пусть мои потуги останутся тут, как шпаргалка самому себе и интернетам.

В роли роутера выступает 2811 с следующими платами расширения:

```CoreRouter#sh diag | include (FRU)
Product (FRU) Number : CISCO2811
Product (FRU) Number : HWIC-4ESW
Product (FRU) Number : HWIC-2FE
```
* Часть 1 – [Начальная конфигурация, установка времени, AAA, доступ по SSH, тюнинг консоли и баннеры](/2010/11/cisco-config-time-aaa-ssh-banners/)
* Часть 2 – [Конфигурация интерфейсов, настройка простого NAT с overloading (PAT)](/2010/11/cisco-interface-config-nat-pat/ "/2010/11/cisco-interface-config-nat-pat/")
