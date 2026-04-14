---
title: Исчез пункт меню “Все программы”
author: ["Stanislav"]
date: 2010-08-19T11:53:19+00:00
url: /2010/08/missing-all-programs/
categories:
  - Tech
tags:
  - parallels
  - windows
  - решаем проблемы
---

Parallels как-то через попу установил гостя Windows Server 2003 в связи с чем в меню пуск отсутствовал пункт “Все программы”.

Лечится правкой следующих ключей в реестре:

в ветке ```HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders```

исправить или создать cтроковой параметр Start Menu на %USERPROFILE%\Главное меню, а в ветке

```HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders```

исправить или создать строковой параметр Common Start Menu на %ALLUSERSPROFILE%\Главное меню
