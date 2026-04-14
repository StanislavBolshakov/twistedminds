---
title: Настройка NetApp FAS – Часть 4
author: ["Stanislav"]
date: 2012-06-24T18:53:01+00:00
url: /2012/06/setting-up-netapp-filer-part-4/
categories:
  - Tech
tags:
  - iscsi
  - isns
  - netapp
  - ontap
---

######

###### Настройка файлера NetApp (на примере NetApp FAS2040) – часть четвертая (iSCSI, настройка iSCSI, безопасность iSCSI, iSNS).

[Часть первая >>>](/2012/06/setting-up-fas2040-part-1/ "Настройка NetApp FAS – Часть 1")
[Часть вторая >>>](/2012/06/setting-up-netapp-fas-filer-part-2/ "Настройка NetApp FAS – Часть 2")
[Часть третья >>>](/2012/06/setting-up-netapp-filer-part-3/ "Настройка NetApp FAS – Часть 3")
[Часть пятая >>>](/2012/06/setting-up-netapp-fas-filer-part-5/ "Настройка NetApp FAS – Часть 5")

**iSCSI**

iSCSI (Internet Small Computer System Interface) – IP протокол для подключения систем хранения данных. Суть его в том, что он инкапсулирует SCSI-команды в IP пакеты и переносит их по ethernet сети. iSCSI SAN по сравнению с FC SAN стоит много меньше, позволяет использовать существующую IP инфраструктуру и легче в настройке и траблшуте.

Не могу не отметить, что существуют еще два протокола, которые по своей сути очень похожи на iSCSI – iFCP и FCIP, но они используются для соединения нескольких удаленных сетей хранения данных вместе.

**Как работает iSCSI**

iSCSI – клиент-серверный протокол. Клиентом выступает обычно сервер (initiator), а сервером – система хранения данных (target), при этом данные, которые target представляет initiator выглядят для последнего как локальный жесткий диск. Клиент осуществляет блочные операции с предоставленным хранилищем. Так как это подразумевает форматирование диски, разделение на тома и создание файловой системы следовательно только один initiator может использовать предоставленный через iSCSI том в одно время. Однако вы можете подключить тома на нескольких target в режиме read-only.

Когда приложение на initiator запрашивает данные, которые находятся на target сервер переводит запросы в чисты SCSI команды и собиарет все это дело в IP пакеты (дополнительно все это дело можно зашифровать по желанию). На target протокол iSCSI, получив пакеты, разбирает их обратно и отправляет SCSI команды на контролер. Затем данные отправляются обратно таким же образом. Так как инкапсуляция и деинкапсуляция происходит при помощью процессора (software iscsi), соответственно на высоконагруженных системах это может приводить к проседанию производительности. В таких случаях рекомендуется использовать специальные платы – iSCSI HBA (hardware iscsi).

Если вы знакомы с Fibre Channel, то знаете, что FCP использует WWPN и WWNN что идентифицировать устройства. iSCSI использует iSCSI адреса. Как только вы сконфигурируете iSCSI адреса на всех target и initiator они должны каким-то образом узнать друг о друге. Можно соседство можно установить вручную, а можно с помощью DNS-like сервера – iSNS.

iSCSI адреса бывают двух видов – iSCSI Qualified Name (iqn) или IEEE EUI-64 (eui):

- iqn: iqn.*yyyy-mm*.*backward_naming_authority*:*unique_device_name*
- eui: eui.*nnnnnnnnnnnnnnnn*

**Настройка iSCSI на файлере**

1. > iscsi [start|stop] – останавливает и запускает iscsi процесс;
2. > iscsi nodename – генерирует iSCSI адрес на основе серийного номера контролера;
3. >  iscsi security default -s none – устанавливает параметры безопасности в none;
4. Создание и экспорт LUN:

Как вы уже знаете из предыдущей части диски объединяются в raid group, те, в свою очередь, логически объединяются в aggregate на которых и создаются тома. Тома могут содержать файлы, каталоги, qtree и LUN. Qtree можно воспринимать как поддиректорию на томе, которая может содержать файлы, каталоги и LUN. Если вы собираетесь использоваться файлер и как NAS и как SAN storage, то хорошим тоном будет создать отдельный qtree для LUN и отдельный qtree для CIFS/NFS. И еще раз повторю рекомендации NetApp по поводу LUN в общем и iSCSI в частности:

- Не создавайте LUN или NFS/CIFS шары на root томе.
- Используйте qtree чтобы отделять iSCSI/FC LUN от NFS/CIFS расшаренных каталогов.
- Используйте qtree чтобы хранить все LUN одного хоста в отдельности.
- Выключите резерв места под снимки (snapshot reserve в 0%).
- Отключите автоматическое создание снимков (nosnap on).
- Убедитесь что у тома, на котором расположен LUN, свич create_ucode стоит в on.

Процесс создание тома, qtree и LUN (для Windows, с резервированием пространства, 9Gb размером, для группы инициаторов iSCSI_TEST, инициатора с iSCSI адресом iqn.2001-04.com.test:test.device.0 в частности) происходит следующим образом:

> > vol create test0 -s volume aggr0 10g
> > vol options test0 nosnapdir on
> > vol options test0 nosnap on
> > vol options test0 create_ucode on
> > qtree create /vol/test0/iscsi
> > lun setup
> Do you want to create a LUN? [y]: y
> Multiprotocol type of LUN: windows
> Enter LUN path: /vol/test0/iscsi/win1-lun0
> Do you want the LUN to be space reserved? [y]: y
> Enter LUN size: 9g
> Enter comment string: test lun
> Name of initiator group []: iSCSI_TEST
> Type of initiator group iSCSI_TEST (FCP/iSCSI) [FCP]: iSCSI
> Enter comma separated nodenames: iqn.2001-04.com.test:test.device.0
> OS type of initiator group “iSCSI_TEST” [windows]: ⏎

Вот и все, настройку target можно считать законченной. Можно приступать к настройке target, указав iqn файлера (получить его можно по команде **iscsi nodename**).

**Пара слов про iSCSI Security.**

Существуют два механизма обеспечения безопасности: аутентификация (CHAP) и шифрование (IPSEC).

На файлерах NetApp безопасность сервиса iSCSI может быть сконфигурирован как CHAP (входящая/исходящая CHAP аутентификация), deny (если initiator нет в листе, то доступ к target будет запрещен) и none (все iSCSI сессии разрешены по умолчанию).

Сгенерируем пароль, укажем CHAP как метод аутентификации по умолчанию и проверим как это работает, после чего вернем все обратно:

> > iscsi security generate
> Generated Random Secret: 0xd5a59081ffb9b0e0d898e2c1f6698433
> > iscsi security default -s CHAP -n TEST-INCOME -p 0xd5a59081ffb9b0e0d898e2c1f6698433
> > iscsi security show
> Default sec is CHAP Local Inbound password: **** Inbound username: TEST-INCOME Outbound password: **** Outbound username:
> > iscsi security default -s none
> > iscsi security show
> Default sec is None

Еще одним путем обеспечения безопасности может стать включение и отключение iSCSI на некоторых интерфейсах (**iscsi interface**).

**Пара слов про iSNS**

Если вы еще раз обратите внимание, то при настройке LUN мы указывали iqn адрес тестового хоста. На хосте, в процессе настройки initiator нам тоже нужно было бы указать iqn (но уже файлера).

Чтобы разрешать FQDN в IP существует DNS и для iSCSI существует подобный сервис – iSNS. В нашем случае iSNS клиентами выступают хосты и файлер. Для одного-двух initiator смысла в нем пожалуй и нет, но когда сеть хранения данных становится более комплексной (десятки initiator, пара-другая target), да еще и смешанной появляется смысл в iSNS, который позволяет не только гибко управлять IP SAN, но и через WWN-proxy рулить уже FC SAN.

##### [Настройка файлера NetApp (на примере NetApp FAS2040) – часть пятая заключительная >>> (снимки, восстановление lun как файла, клонирование lun, мониторинг файлера, оценка производительности,  полезные ссылки)](/2012/06/setting-up-netapp-fas-filer-part-5/ "/2012/06/setting-up-netapp-fas-filer-part-5/")
