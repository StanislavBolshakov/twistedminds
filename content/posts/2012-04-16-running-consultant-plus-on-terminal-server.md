---
title: Запуск Консультант Плюс на сервере терминалов
author: ["Stanislav"]
date: 2012-04-16T10:23:31+00:00
url: /2012/04/running-consultant-plus-on-terminal-server/
categories:
  - Tech
tags:
  - citrix
  - consultant plus
  - terminal server
  - terminal services
  - xenapp

---
По умолчанию, руководствуясь непонятной мне логикой, консультант пишет пользовательские данные и конфигурационные файлы (при запуске с ключем /group) на локальную рабочую станцию в корень диска, содержащего каталог %windir%. Сами настройки пути каталога с пользовательскими данными располагаются в реестре по адресу HKCU\Software\ConsultantPlus\ConsultantPlus\3000.

Чтобы научить Консультант Плюс хорошим манерам (читай хранить конфигурационные файлы в пользовательском каталоге) я сбыдлокодил на коленке bat файлик:

```  
SET CAT_PATH={UNC путь к shared-каталогу с Консультантом}  
SET ConsUserDataPath=%AppData%\ConsUserData  
IF EXIST %ConsUserDataPath% (goto :run) ELSE GOTO :reg

:reg  
reg ADD HKCU\Software\ConsultantPlus\ConsultantPlus\3000 /v WrkDir /t REG_SZ /d &#8220;%ConsUserDataPath%&#8221; /f  
mkdir &#8220;%ConsUserDataPath%&#8221;  
goto :run

:run  
start /d %CAT_PATH% %CAT_PATH%\CONS.EXE  
```

Комментарии, пожалуй, излишни.

added:

Как подсказал bambr в своем комментарии ниже, существует штатная возможность указать Консультанту путь к конфигам, указав их в файле complect.cfg, который нужно расположить в каталоге base. Выдержка из документации:

> В файле complect.cfg теперь можно использовать переменные окружения операционной системы.  
> Формат файла &#8211; обычный текстовый файл.  
> Первая строка &#8211; заголовок окна &#8220;Консультант Плюс&#8221;.  
> Вторая &#8211; название ярлыка &#8220;Консультант Плюс&#8221; на рабочем столе.  
> В третьей строке записывается путь до рабочей директории.
> 
> Для примера, содержание может быть таким:
> 
> Консультант Плюс  
> Консультант Плюс  
> %UserProfile%\Consultant
> 
> Здесь %UserProfile% &#8211; это каталог C:\Documents and Settings\имя пользователя (для Windows XP/2000) или C:\USERS\имя пользователя (для Windows 7/Vista).