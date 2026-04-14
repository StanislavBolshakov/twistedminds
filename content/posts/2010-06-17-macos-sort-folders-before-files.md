---
title: 'MacOS X: сортировка директорий перед файлами'
author: ["Stanislav"]
date: 2010-06-17T10:53:36+00:00
url: /2010/06/macos-sort-folders-before-files/
categories:
  - Tech
tags:
  - finder
  - macos
  - решаем проблемы
---

Подчиняясь неведомой логике Finder сортирует файлы и директории по алфавиту, когда в файловых менеджерах линукса и венды директории имеют более высокий приоритет. Исправляется следующим образом:

```cd /System/Library/CoreServices/Finder.app/Contents/Resources/ru.lproj/```

(ru, соответственно надо заменить на язык системы)

```sudo cp InfoPlist.strings InfoPlist.strings.bk```

делаем бэкап файла с конфигом

```sudo vi InfoPlist.strings```

Открываем файл и в разделе /* General kind strings */ меняем ```”Folder” = “Папка”;``` на```”Folder” = ” Папка”;```

Сохраняем, закрываем, перезагружаемся, сортируем по типу.
