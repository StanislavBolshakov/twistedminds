---
title: Отправка писем от имени группы в Microsoft Exchange 2010
author: ["Stanislav"]
date: 2013-01-18T06:59:01+00:00
url: /2013/01/send-as-distribution-group-in-ms-exchange/
categories:
  - Tech
tags:
  - exchange
  - microsoft
  - troubleshooting
---

В общем вся настройка сводится к добавлению прав send-as для distribution group через EMS:
```

Add-ADPermission grpname -ExtendedRights Send-As -User usrname -AccessRights ExtendedRight
```

либо через оснастку ADUC. Для распространения настроек требуется 2 часа либо вручную перезапущеный сервис Exchange Information Store. Последнее приведет к обновлению кэша и злым пользователям.

Если после этого вы все равно продолжаете получать отбойник о том, что, мол
```

You can't send a message on behalf of this user unless you have permission to do so.
```

```

Нельзя отправить сообщение от лица этого пользователя без соответствующего разрешения.
```

то вам следует выполнить следующие действия:

1. Попробовать отправить письмо от имени группы из OWA.
2. В случае успеха вам следует удалить OAB при выключенном Outlook.
```

C:\Users\%username%\AppData\Local\Microsoft\Outlook\Offline Address Books
```
