---
title: Cisco MDS для самых маленьких
author: ["Stanislav"]
date: 2013-06-02T13:39:06+00:00
url: /2013/06/cisco-mds-basics/
categories:
  - Tech
tags:
  - cisco
  - FC
  - FCNS
  - MDS
  - NX-OS
  - SAN
  - SAN-OS
  - VSAN
  - WWN
  - Zonning
---

[![mds9124](/wp-content/uploads/2013/06/mds9124.jpg "mds9124")](/wp-content/uploads/2013/06/mds9124.jpg)Под конец моей работы в Службе до нас таки доехали новенькие MDS9124, которые пришли на замену протухшим MDS9120 и QLogic SANBOX2-8C. В этой статье я собираюсь освежить в памяти термины и оставить некоторые заметки, оставшиеся после апгрейда SAN. Чем черт не шутит, а вдруг пригодится.

Хочется так же добавить, что вендор-зависимых фич тут мало, и, принимая внимания различия в синтаксисе, те же шаги можно предпринять для настройки не только упомянутых Cisco MDS, но и оптических коммутаторов других производителей.

#### Cisco SAN-OS vs. NX-OS – в чем различия?

Признаться, с SAN коммутаторами Cisco MDS давненько общения не имел и был удивлен, увидев в выхлопе show version Nexus OS вместо SAN-OS. Все оказалось весьма банально – начиная с релиза 4.1 Cisco просто переименовала SAN-OS в NX-OS. С функциональной точки зрения не изменилось ровным счетом ничего.

Что такое VSAN? FCID? WWN? WWPN? FWWN? Zonning?- **Virtual SAN (VSAN)** – дополнительный логический уровень абстракции, который позволяет создавать виртуальные SAN поверх реальных. Мне нравится думать о VSAN как о классических VLAN. VSAN обладает индивидуальным адресным пространством и позволяет использовать те же FCID  представляет собой одновременно в разных VSAN.
- **FCID (Fibre Channel ID)** – 24 битовый идентификатор, который используется для маршрутизации фреймов по FC сети. Фреймы имеют Source ID (S_ID) и Destination (D_ID) и используются для того, чтобы направить данные к нужной цели. Помимо FCID у устройств есть еще один уникальный идентификатор – WWN.
- **WWN (Word Wide Name)** – аналог MAC адрес устройства в FC сети. WWN используются для зонирования, разделения устройств по группам безопасности, в рамках которых им разрешено общаться.
- **WWPN (World Wide Port Name)** – так если HBA имеет несколько портов, то оно имеет не только адрес себя (WWN), но и адреса все портов WWPN.
- **FWWN (Fabric WWN)** – каждый порт FC коммутатора так же имеет свой идентификатор.
- **Zonning** (зонирование) – чтобы быть до конца честным зонирование может осуществляться не только по WWN. Вполне можно объединять устройства в зоны по:
  - PWWN, FWWN
  - FCID
  - Интерфейсу
  - FC Alias (имя, ассоциированное с WWN)![zonning](/wp-content/uploads/2013/06/zonning.png "zonning")
- **VSAN vs Zonning** – VSAN используется для разделения функций фабрики, таких как FSPF (Fabric Shortest Path First – протокол маршрутизации FC сетей), Name Server, Device Discovery Service на домены. Внутри каждого VSAN будет функционировать свой Name Server, Device Discovery Service, etc и сбой одного из них не утащит за собой все устройства в фабрике.
Вы все еще тут? Продолжим :)
#### Fabric login
Или FLOGI. Процесс присоединения FC HBA к фабрике. После этого процесса происходит регистрация WWN устройств и WWPN портов:
```

FabricA# show flogi database | i fc1/1|INTERFACE
INTERFACE        VSAN    FCID           PORT NAME               NODE NAME
fc1/1            1     0xa80000  50:01:43:80:06:34:11:4c 50:01:43:80:06:34:11:4d
```

Если зарегистрированые WWN и PWWN не видны, то пора бить тревогу и занятся диагностикой физики. Так, например, выглядит плохой кабель:
```

FabricA# sh int fc1/2 tra | b Diag
    SFP Diagnostics Information:
        Temperature           :  25.18 C
        Voltage               :   3.33 V
        Current               :   7.69 mA
        Optical Tx Power      :  -3.96 dBm
        Optical Rx Power      : -35.23 dBm      --
        Tx Fault count  : 0
    Note: ++  high-alarm; +  high-warning; --  low-alarm; -low-warning
```

####  Fibre Channel Name Server

Name Server заботливо хранит все записи о хостах в FCNS базе. Если коммутаторов несколько, то Name Servers синзронизируют свою информацию с централизированной базой данных в рамках одного VSAN.
```

FabricA# show fcns database

VSAN 1:
--------------------------------------------------------------------------
FCID        TYPE  PWWN                    (VENDOR)        FC4-TYPE:FEATURE
--------------------------------------------------------------------------
0xa80000    N     50:01:43:80:06:34:11:4c (HP)            scsi-fcp:init
0xa80100    N     50:0a:09:82:88:8a:41:cc (NetApp)        scsi-fcp
0xa80200    N     50:0a:09:81:88:8a:41:cc (NetApp)        scsi-fcp

Total number of entries = 3
```

**План базовой настройки:**
1. Создать VSAN
2. Добавить интерфейсы в VSAN
3. Включить интерфейсы
4. Создать псевдонимы для WWN
5. Создать зоны и объединить устройства по тому или иному признаку (*использование PWWN для зонирования – Cisco’s Best Practice*)
6. Создать zoneset и добавить в него зоны
7. Активировать его
So, let’s go:
**1. Создание VSAN**
```

FabricA(config)# vsan database
FabricA(config-vsan-db)# vsan 10
FabricA(config-vsan-db)# vsan 10 name Production
FabricA# sh vsan
vsan 1 information
         name:VSAN0001  state:active
         interoperability mode:default
         loadbalancing:src-id/dst-id/oxid
         operational state:up

vsan 10 information
         name:Production  state:active
         interoperability mode:default
         loadbalancing:src-id/dst-id/oxid
         operational state:down
```

**2. Добавление интерфейсов в VSAN**
```

FabricA(config)# vsan data
FabricA(config-vsan-db)# vsan 10 interface fc1/1-8,fc1/20-24
```

Как только порты будут добавлены в VSAN Operational State сменится на UP.
**3. Включение интерфейсов**
```

FabricA(config)# int fc1/1-4
FabricA(config-if)# no shut
```

и проверка:
```

FabricA# sh int brief | ex down

-------------------------------------------------------------------------------
Interface  Vsan   Admin  Admin   Status          SFP    Oper  Oper   Port
                  Mode   Trunk                          Mode  Speed  Channel
                         Mode                                 (Gbps)
-------------------------------------------------------------------------------
fc1/1      10     auto   on      up               swl    F       4    --
fc1/2      10     auto   on      notConnected     swl    --           --
fc1/3      10     auto   on      up               swl    F       4    --
fc1/4      10     auto   on      licenseNotAvail  swl    --           --
fc1/23     10     auto   on      up               swl    F       4    --
fc1/24     10     auto   on      up               swl    F       4    --

-------------------------------------------------------------------------------
```

**4. Создание WWN псевдонимов**

Может быть выполнено двумя путями, используя FC Alias и Device Alias. Разница между FC Alias и Device Alias состоит в следующем:

- FC Alias используются только для зонирования.
- FC Alias могут содержать несколько PWWN
- FC Alias работают в рамках одного VSAN
- FC Alias распространяются в рамках zoneset
- FC Alias совместимы с коммутаторами других вендоров

- Device Alias подразумевают один PWWN
- Device Alias могут быть использованы для зонирования, port security, InterVSAN Routing (IVR)
- Device Alias не привязаны к VSAN
- Device Alias распространяются по протоколу CFS (Cisco Fabric Service)
FC Alias:
```

FabricA(config)# fcalias name esxi5-n1-hba1 vsan 10
FabricA(config-fcalias)# member pwwn 50:01:43:80:06:34:11:4c
FabricA# sh fcalias
fcalias name esxi5-n1-hba1 vsan 10
  pwwn 50:01:43:80:06:34:11:4c
```

Device Alias:
```

FabricA(config)# device-alias database
FabricA(config-device-alias-db)# device-alias name esxi5-n1-hba1 pwwn 50:01:43:80:06:34:11:4c
FabricA(config)# device-alias commit
FabricA# sh device-alias database
device-alias name esxi5-n1-hba1 pwwn 50:01:43:80:06:34:11:4c

Total number of entries = 1
```

Обратите внимание на device-alias commit. Без этого псевдоним не будет занесен в базу данных. Отключить синхронизацию базы можно с помощью команды **no device-alias distribute** в глобальной конфигурации.

**5. Создание зон, добавление членов**
```

FabricA(config)# zone name esxi5-nodes-to-SPA vsan 10
FabricA(config-zone)# member ?
  device-alias       Add device-alias member to zone
  domain-id          Add member based on domain-id,port-number
  fcalias            Add fcalias to zone
  fcid               Add FCID member to zone
  fwwn               Add Fabric Port WWN member to zone
  interface          Add member based on interface
  ip-address         Add IP address member to zone
  pwwn               Add Port WWN member to zone
```

Как видно из выхлопа вы можете добавить нового члена в зону как используя свежесозданный fcalias, device-alias или, опустив предыдущий шаг, сразу добавить PWWN. Доступны так же другие вариации, но, повторюсь, Cisco рекомендует использовать PWWN.

**6. Создание zone set, добавление зон**

Zone set содержит в себе одну или несколько зон. Зона может быть членом одного или несколькоих zone set. В рамках одной зоны устройства могут общаться, а в разных уже не очень. Устройства может быть членом более чем одной зоны.
```

FabricA(config)# zoneset name active vsan 10
FabricA(config-zoneset)# member esxi5-nodes-to-SPA
FabricA(config-zoneset)# member esxi5-nodes-to-SPB
```

**7. Активация zoneset**
Только один zone set может быть активным в рамках одного VSAN. Если зонирование не настроено, то все устройства в фабрике являются членами одной default zone, общение в рамках такой default zone возможно всех со всеми. Если зонирование настроено, то устройства, которые не являются членами активной зоны в zone set находятся в default zone.
```

FabricA(config)# zoneset activate name active vsan 10
```

Хорошим тоном является сохранение настроек в startup-config :)
