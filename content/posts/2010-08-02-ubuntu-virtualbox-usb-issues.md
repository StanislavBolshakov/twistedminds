---
title: Ubuntu, VirtualBox и проблемы с USB у гостевой ОС
author: ["Stanislav"]
date: 2010-08-02T10:20:20+00:00
url: /2010/08/ubuntu-virtualbox-usb-issues/
categories:
  - Tech
tags:
  - linux
  - ubuntu
  - virtualbox
  - решаем проблемы

---
После очередного обновления VirtualBox исчезла возможность подключать USB девайсы. VirtualBox видел подключенные устройства, но возможности активировать их в гостевой системе не было (пункты меню были не активны).

Лечится добавлением вашего пользователя в группу vboxusers либо через GUI (система > администрирование > пользователи и группы), либо через терминал:

```# useradd -G vboxusers username```

и проверяем

```# id username```

logoff, login, счастье