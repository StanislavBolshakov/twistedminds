---
title: postgresql tricks
author: ["Stanislav"]
date: 2010-06-10T06:21:06+00:00
url: /2010/06/postgresql-tricks/
categories:
  - Tech
tags:
  - postgresql
---

Считаем размер, занимаемой на диске базой данных в байтах:

[sql]SELECT pg_database_size(‘dbname’);[/sql]

Считаем размер, занимаемой на диске базой данных в мегабайтах:

[sql]SELECT pg_size_pretty(pg_database_size(‘dbname’));[/sql]

Считаем количество строк в всех таблицах базы данных:

[sql]SELECT relname, n_tup_ins – n_tup_del as rowcount FROM pg_stat_all_tables;[/sql]
