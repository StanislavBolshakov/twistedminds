---
title: Настройка NetApp FAS – Часть 1
author: ["Stanislav"]
date: 2012-06-20T14:29:43+00:00
url: /2012/06/setting-up-fas2040-part-1/
categories:
  - Tech
tags:
  - fas
  - howto
  - netapp
  - ontap
---

![](/wp-content/uploads/2012/06/netapp-logo1-e1340202341425.png "netapp logo")Тут и далее будет серия статей по **базовой** настройке файлера NetApp FAS2040.

[Часть вторая >>>](/2012/06/setting-up-netapp-fas-filer-part-2/ "Настройка NetApp FAS – Часть 2")
[Часть третья >>>](/2012/06/setting-up-netapp-filer-part-3/ "Настройка NetApp FAS – Часть 3")
[Часть четвертая >>>](/2012/06/setting-up-netapp-filer-part-4/ "Настройка NetApp FAS – Часть 4")
[Часть пятая >>>](/2012/06/setting-up-netapp-fas-filer-part-5/ "Настройка NetApp FAS – Часть 5")

##### Часть первая (Описание контроллера FAS2040, возврат к заводским настройкам, базовая настройка файлера, настройка BMC, обновление Data ONTAP, NTP сервера).

**Описание интерфейсов контроллера FAS2040.**![](/wp-content/uploads/2012/06/netapp-controller.png "netapp fas2040 controller")

Слева-направо:

- 2 порта Fibre Channel 1/2/4Gi – Target/Initiator
- SAS разъем для подключения дисковой полки DS4243
- консольный порт
- ACP – out of band управляющий интерфейс для дисковой полки DS4243
- BMC – out of band управляющий интерфейс контроллера
- 4 интерфейса 1GiE e0a e0b e0c e0d

**Возврат к заводским настройкам осуществляется с помощью пары команд:**

```
>priv set advanced
*>halt -c factory```

**Базовая настройка, обновление, BMC, NTP:**

Подключившись консольным кабелем (замечательно подходят кабели от оборудования Cisco) после заводского сброса можно лицезреть приглашение загрузчика начинающееся с LOADER. Заставим его автоматически загружать систему и перейдем к загрузке:
```
> setenv AUTOBOOT true
> boot_ontap```

Cистема, не найдя конфигурационный файл /etc/rc, предложит приступить к начальной конфигурации, принимая во внимание факт того, что настройку сети мы выполним базовую (интерфейс e0a) и более подробно рассмотрим ее в части второй. Не ключевые настройки опущены:

```
Please enter the new hostname []:FAS2040-B
Do you want to configure interface groups? [n]: n
Please enter the IP address for Network Interface e0a []: 172.16.21.101
Please enter the netmask for Network Interface e0a [255.255.0.0]: 255.255.255.0
Would you like to continue setup through the web interface? [n]: n
Please enter the name or IP address of the default gateway: 172.16.21.254
Where is the filer located? []: Server room A
What language will be used for multi-protocol files (Type ? for list)?:ru
Do you want to run DNS resolver? [n]: y
Please enter DNS domain name []: domain.local
You may enter up to 3 nameservers
Please enter the IP address for first nameserver []: 172.16.20.11
Do you want another nameserver? [n]: y
Please enter the IP address for alternate nameserver []: 172.16.20.12
Do you want another nameserver? [n]: n
Do you want to run NIS client? [n]: n
```

После чего, пройдя аутентификацию, мы попадаем в приглашение командной строки и приступим к настройке BMC:
```
FAS2040-B> bmc setup
Would you like to configure the BMC? (y/n)? y
Would you like to enable DHCP on BMC LAN interface? (y/n)? n
Please enter the IP address for the BMC []: 172.16.15.35
Please enter the netmask for the BMC []: 255.255.255.240
Please enter the IP address for the BMC gateway []: 172.16.15.33
Please enter the gratuitous ARP Interval for the BMC [10 sec (max 60)]:
> bmc reboot
```

Теперь консольный кабель можно отключить и подключившись к BMC интерфейсу по указанному вами IP продолжить конфигурацию файлера через SSH.

Ну и наконец приступим к обновлению операционной системы, выложив (в случае обновления с 8.x) архив tgz на вэб сервер. Версию текущей ОС можно узнать командой version. По окончанию файлер будет перезагружен.

```> software update http://websrvr/81_e_image.tgz -R```

Установим NTP сервера и проверим время:
```> options timed.servers ntp1.domain.local,ntp2.domain.local
> options timed.enable on
> timezone Europe/Moscow
> date
Wed Jun 20 18:42:13 MSK 2012
```

[Часть вторая настройки NetApp FAS >>> (Multimode VIF и его виды, балансировка нагрузки, интерфейсы-партнер в кластере, псевдонимы aka IP Aliasing, flowcontrol и MTU, настройка Static и Dynamic VIF с коммутатором Cisco Catalyst).](/2012/06/setting-up-netapp-fas-filer-part-2/ " Часть вторая настройки NetApp FAS (Multimode VIF и его виды, балансировка нагрузки, интерфейсы-партнер в кластере, псевдонимы aka IP Aliasing, flowcontrol и MTU, настройка Static и Dynamic VIF с  коммутатором Cisco Catalyst).")
