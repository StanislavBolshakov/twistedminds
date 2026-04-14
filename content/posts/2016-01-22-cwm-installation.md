---
title: Настройка D-Link Central WiFi Manager
author: ["Stanislav"]
date: 2016-01-22T12:09:37+00:00
url: /2016/01/cwm-installation/
categories:
  - Tech
tags:
  - cwm
  - d-link
  - howto
  - wifi
  - wireless
---

Познакомившись с ~~маркетингом~~ [теорией](/2016/01/dlink-cwm/) попробуем собрать простой лабораторный стенд следующего вида:[![cwm-lab-scheme](/wp-content/uploads/2015/12/cwm-lab-scheme.png)](/wp-content/uploads/2015/12/cwm-lab-scheme.png)

По легенде в нашей организации существуют две беспроводные сети: Production & Guest. Для того, чтобы случилась магия надо предпринять следующие шаги:

1. Выполнить базовую настройка коммутатора
   - VLAN
   - VLAN интерфейсы
   - DHCP
2. Добавить точки доступа в CWM
3. Выполнить базовую настройку точек доступа с использованием CWM
   - Добавить беспроводные сети
   - Назначить соответствующие VLANы и PVID на физическом интерфейсе точки доступа
   - Поместит беспроводные сети в соответствующие им VLANы
4. Применить настройки

##### Настройка коммутатора

**VLAN**
```

create vlan management tag 21
config vlan management add untagged 13-14,21

create vlan wireless_prod tag 22
config vlan wireless_prod add tagged 13-14

create vlan wireless_guest tag 23
config vlan wireless_guest add tagged 13-14
```

**VLAN Интерфейсы**
```

create ipif mgmt 172.16.21.254/24 management state enable
create ipif wlan_prod 172.16.22.254/24 wireless_prod state enable
create ipif wlan_guest 172.16.23.254/24 wireless_guest state enable
```

**DHCP**
```

create dhcp pool mgmt
config dhcp pool network_addr mgmt 172.16.21.0/24
config dhcp pool dns_server mgmt 8.8.4.4 8.8.8.8
config dhcp pool default_router mgmt 172.16.21.254

create dhcp pool wlan_prod
config dhcp pool network_addr wlan_prod 172.16.22.0/24
config dhcp pool dns_server wlan_prod 8.8.4.4 8.8.8.8
config dhcp pool default_router wlan_prod 172.16.22.254

create dhcp pool wlan_guest
config dhcp pool network_addr wlan_guest 172.16.23.0/24
config dhcp pool dns_server wlan_guest 8.8.4.4 8.8.8.8
config dhcp pool default_router wlan_guest 172.16.23.254

enable dhcp_server
```

##### Модули

Для того, чтобы контроллер мог управлять точками доступа в него нужно загрузить и установить модули, которые им соответствуют. Модули доступны по [ссылке](http://www.dlink.ru/ru/products/2/2086_d.html).

##### Обнаружение точек доступа с помощью CWM

Тут нужно сделать маленькое отступление.

В версиях 1.x CWM процесс обнаружения сопряжен с необходимостью выгрузки первоначальной конфигурации. Это делается с тем, чтобы перевести точку доступа в управляемый режим и сообщить ей адрес контроллера. Для этой задачи используется AP installation utility for CWM, входящая в состав контроллера и доступная к загрузке из раздела Site вкладки Configuration.

Данный процесс необходимо выполнить только один раз, после чего все изменения конфигурации можно совершать с использованием web GUI.

В версиях 2.x механизм обнаружения и обновление точки доступа в управляемую контроллером будет производиться из web интерфейса. По крайней мере так обещают разработчики.

Для минимальной рабочей конфигурации создадим *Site* и *Network* на вкладке *Configuration*. В данном пример St Petersburg и d-link. После этого можно экспортировать конфигурацию и загружать ее в точку доступа с помощью утилиты через меню *Set GroupInfo*.

![01 - AP installation utility for CWM](/wp-content/uploads/2016/01/01-AP-installation-utility-for-CWM-1024x419.png)

![02 - Select config](/wp-content/uploads/2016/01/02-Select-config.png)

![03 - AP to slave mode](/wp-content/uploads/2016/01/03-AP-to-slave-mode-1024x419.png)

Если все сделано правильно в интерфейсе CWM появится новая точка доступа. А это уже половина успеха.

![04 - AP online](/wp-content/uploads/2016/01/04-AP-online-1024x469.png)

##### Базовая настройка точки доступа с использованием CWM

С помощью пункта меню *SSID* в нашей сети d-link создадим беспроводные сети: гостевую и рабочую. В 2.4 и 5GHz диапазонах соответственно.

![05 - ssid list](/wp-content/uploads/2016/01/05-ssid-list-1024x469.png)

Теперь нужно поместить беспроводные сети в соответствующие VLAN согласно схеме.[![cwm-ap-switch-vlan](/wp-content/uploads/2016/01/cwm-ap-switch-vlan-1.png)](/wp-content/uploads/2016/01/cwm-ap-switch-vlan-1.png)Для этого:

- Создадим 22 и 23 VLANы в вкладке *Add/Edit VLAN*пункта меню *VLAN*.
- Добавим гостевую и рабочую сети в соответствующие VLAN.
- Начнем тегировать трафик этих VLANов на аплинк портах.
- Проверим значения PVID на соответствующей вкладке. Если увиденное не соответствует нашему чувству прекрасного и рабочей конфигурации, то стоит отменить радио-точку enable PVID auto assign status и взять все в свои сильные руки инженера.
В результате получим следующие настройки:
[![port](/wp-content/uploads/2016/01/port.png)](/wp-content/uploads/2016/01/port.png)

port list

 [![vlans](/wp-content/uploads/2016/01/vlans-1024x136.png)](/wp-content/uploads/2016/01/vlans.png)

vlan list

Что и будет являться рабочей конфигурацией.
##### Применение настроек
Применим конфигурацию с помощью пункта меню _Configuration_ > _Site_ (St Petersburg) > _Network_ (d-link) > _Uploading Configuration_, нажав кнопку _Complete_.
[![save config](/wp-content/uploads/2016/01/save-config-1024x447.png)](/wp-content/uploads/2016/01/save-config.png)
В результате оранжевая точка рядом с Site и Network должна пропасть.

При подключении к рабочей и гостевой беспроводным сетям вы должны получать адреса из диапазонов 172.16.22.0/24 и 172.16.23.0/24 соответственно.
