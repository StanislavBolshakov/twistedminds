---
title: Shrink лог файлов в MSSQL 2008 и MSSQL 2008R2
author: ["Stanislav"]
date: 2010-09-10T05:49:06+00:00
url: /2010/09/shrinking-mssql-2008-log/
categories:
  - Tech
tags:
  - mssql
---

[sql]USE [database name]
ALTER DATABASE [database name] SET RECOVERY SIMPLE
DBCC SHRINKFILE([log file name], )
ALTER DATABASE [database name] SET RECOVERY FULL[/sql]
