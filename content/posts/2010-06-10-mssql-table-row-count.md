---
title: Подсчет количества записей в каждой из таблиц базы данных MSSQL
author: ["Stanislav"]
date: 2010-06-10T06:02:54+00:00
url: /2010/06/mssql-table-row-count/
categories:
  - Tech
tags:
  - mssql
---

[sql]select substring(o.name, 1, 30) Table\_Name ,i.rows Number\_of_Rows
from sysobjects o
inner join sysindexes i
on (o.id = i.id)
where o.xtype = ‘u’
and i.indid < 2
order by o.name[/sql]
