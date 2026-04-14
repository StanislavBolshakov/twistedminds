---
title: Настройка NetApp FAS – часть 5
author: ["Stanislav"]
date: 2012-06-28T08:02:30+00:00
url: /2012/06/setting-up-netapp-fas-filer-part-5/
categories:
  - Tech
tags:
  - lun backup
  - lun clone
  - netapp
  - ontap
---

#####

##### Настройка файлера NetApp (на примере NetApp FAS2040) – часть пятая заключительная (снимки, восстановление lun как файла, клонирование lun, мониторинг файлера, оценка производительности,  полезные ссылки).

[Часть первая >>>](/2012/06/setting-up-fas2040-part-1/ "Настройка NetApp FAS – Часть 1")
[Часть вторая >>>](/2012/06/setting-up-netapp-fas-filer-part-2/ "Настройка NetApp FAS – Часть 2")
[Часть третья >>>](/2012/06/setting-up-netapp-filer-part-3/ "Настройка NetApp FAS – Часть 3")
[Часть четвертая >>>](/2012/06/setting-up-netapp-filer-part-4/ "Настройка NetApp FAS – Часть 4")

Вот, наконец, мы и получили рабочую систему хранения данных (в рамках **базовой** настройки). Вы видите как ваши LUN наполняются данными, а чакры раскрываются. Последнее чем стоило бы заняться – это обеспечить сохранность данных, использовав встроенные в операционную систему средства или отдельное программное обеспечение, но давайте посмотрим как реализовать все это со стороны файлера. Ну и последним штрихом включим failover и autosupport.

Про снимки (snapshots) я вскользь упоминал в предыдущих статьях. NetApp создает crash consistent снимок, т.е. если вы решите, к примеру, восстановить виртуальную машину из такого снимка для нее это будет выглядеть так же, как восстановление после сбоя по питанию. На тестовом стенде в среде vSphere 5 из 40 виртуальных машин (8 debian, остальные Windows 2003R2, 2008, 2008R2) после 5-и кратного использования восстановления из crash consistent снимка все сервера продолжали функционировать нормально (некоторым понадобился fsck :)), что в прочем не является best practice. Не смотря на то, что некоторые приложения (к примеру СУБД Oracle) официально поддерживают восстановление из подобных снимков, использование [application consistent резервных копий](http://en.wikipedia.org/wiki/Data_consistency#Application_Consistency "application сonsistency") мне кажется наиболее здравой идеей.

Механизм снимков представляет из себя примерно следующее:

Снимки хранятся в каталоге .snapshot тома. При восстановлении из снимка том восстанавливается на момент времени, в который был сделан снимок. Все снимки сделанные после теряются. Список всех доступных снимков показывает команда **snap list**.

Зачастую нет никакого практического смысла восстанавливать целиком том. В случае если вам нужно восстановить часть информации на томе, то можно воспользоваться двумя способами. Восстановить LUN из снимка или клонировать LUN. В первом случае содержимое LUN по понятным причинам восстановится полностью на момент создания снимка, во втором случае можно смонтировать клон LUN и уже на хосте восстановить именно нужную часть файлов.

Проделаем следующие:

> filer> vol create voltest -l ru -s none aggr0 100g
>  filer> snap create voltest snap0
>  filer> lun create -s 10g -t windows -o noreserve /vol/voltest/lun0
>  filer> igroup show
>  FSRVRS (iSCSI) (ostype: windows):
>  iqn.1991-05.com.microsoft:fs1.1 (logged in on: vif1b)
>  filer> lun map /vol/voltest/lun0 FSRVRS

После чего искомый lun, при правильной настройке iSCSI initiator, появляется на нашем сервере. Создадим на lun какую-либо тестовую структуру каталогов и, сделав еще один снимок snap1, внесем в эту структуру изменения и попытаемся вернуть все в зад.

**Способ раз: восстановление файла из снимка**

> filer> lun show
>  /vol/voltest/lun0 10.0g (10742215680) (r/w, online, mapped)
>  filer> lun unmap /vol/voltest/lun0 FSRVRS
>  filer> lun show
>  /vol/voltest/lun0 10.0g (10742215680) (r/w, online)
>  filer> snap restore -t file -s snap1 /vol/voltest/lun0
>  filer> snap list
>  27% (27%) 0% ( 0%) Jun 28 13:39 snap1 (busy,snap restore)

Переведя lun в режим offline мы получили возможность восстановить его как файл из снимка. Тут кроется первый подводный камень – [LUN у NetApp это не только файл, но и ценный мех в виде мета данных](http://blog.aboutnetapp.ru/archives/809 "LUN у NetApp это не просто файл"). Так как метаданные на томе уже существуют то все восстанавливается нормально, но если вы попытаетесь скопировать LUN с одного тома и разместить его на другом (по CIFS/NFS например), то у вас ничего не выйдет, а если вы попытаетесь восстановить его в другое место на том же томе (**snap restore -t file -s SNAP -r /vol/vol1/fs1 /vol/vol1/fshares/fs1**), то он привратиться в тыкву, а в логе сообщений вы увидите что-то в этом роде:

> Tue Jun 26 17:31:10 MSK [filer:lun.vdisk.bad.inode:error]: failed to initialize lun LUN in volume VOL (Bad format in stream directory inode). LUN has been converted into a regular file

В нагрузку вы получите весьма паршивое состояние снимка “locked busy snapshot”.

**Способ два: восстанавливаем файлы из клонированного LUN**

> lun clone create /vol/voltest/luncln -b /vol/voltest/lun0 snap1

Создается клон luncln от родителя lun0 из снимка snap1, при этом снимок получает статус busy, LUNs. Чтобы его остановить нужно разделить родителя и клона (**lun clone split**). В последствии с клоном можно производить все те же действия, что и с родителем, включая создание клонов клона.
 Полученный LUN можно подключить к igroup и восстановить только нужные файлы, а не LUN целиком.

Перемещать LUN между томами и, даже, файлерами можно с помощью ndmpcopy.

**Мониторинг файлера, оценка производительности**

**aggr show_space -h *aggr0*** – breakdown по занимаемому пространству aggregate aggr0
 **aggr status -r** – breakdown по raid group, spare, parity, double parity дискам
 **aggr status -s** – показывает доступные spare диски
 **aggr status -f** – показывает отказавшие диски
 **vol status -v** – подробная информация по всем томам и настройками, включая autosizing, autodelete, настройки
 **df -h** – breakdown по использованию дискового пространства томами
 **version** – версия DOT
 **uptime** – время с загрузки
 **snap reserve *vol*** – устанавливает резерв для снимков тома *vol*
 **snap sched *vol*** – устанавливает расписание создания снимков
 **lun stats** – IO операции с LUN
 **lun show** – список LUN
 **netstat** – показывает текущие IP соединения
 **ifgrp stat vif interval** – breakdown по packet in/out в интерфейсах, включенных в VIF
 **sysstat -us 1** – breakdown по утилизации ресурсов каждую секунду

И, наконец, убедившись что все ок включаем autosupport и failover.

**> cf enable**
 **> options autosupport.enable on**

**Полезные ссылки**

<http://datadisk.co.uk/html_docs/netapp/netapp.htm>
 <http://www.wafl.co.uk/>
 <http://www.netapp.com/us/library/technical-reports.html>
 <http://forum.ixbt.com/topic.cgi?id=66:3353>
 [http://blog.aboutnetapp.ru](http://blog.aboutnetapp.ru/)

Отдельно хочется поблагодарить автора блога [http://blog.aboutnetapp.ru](http://blog.aboutnetapp.ru/) Романа Хмелевского за интересные статьи и ответы на вопросы в почте :)
