---
title: Проблема с установкой SP1 на Windows 2008R2
author: ["Stanislav"]
date: 2011-12-01T08:49:19+00:00
url: /2011/12/sp1-win2k8r2-installation-issue/
categories:
  - Tech
tags:
  - microsoft
  - SP1
  - troubleshooting
  - Windows Server 2008R2
---

После того, как я устал смотреть на нашего одмина, запускающего по 12му разу sfc /scannow после неудачной ошибки обновления до SP1 сервера пришлось взять дело в свои руки.
В event viewer были следующие ошибки:
```Установка пакета обновления завершилась с ошибкой; код ошибки: 0x800706be.```
```
Имя сбойного приложения: TrustedInstaller.exe, версия: 6.1.7600.16385, отметка времени: 0x4a5bc4b0
Имя сбойного модуля: ntdll.dll, версия: 6.1.7600.16695, отметка времени 0x4cc7b325
Код исключения: 0xc00000fd
Смещение ошибки: 0x0000000000054a07
Идентификатор сбойного процесса: 0xf9c
Время запуска сбойного приложения: 0x01ccaffe65793418
Путь сбойного приложения: C:\Windows\servicing\TrustedInstaller.exe
Путь сбойного модуля: C:\Windows\SYSTEM32\ntdll.dll
Код отчета: 50dd7494-1bf4-11e1-848e-000c29097f71```
```Служба Установщик модулей Windows была неожиданно завершена.
Это произошло 1 раз(а). Следующее корректирующее действие будет предпринято через 120000 мсек: Перезапуск службы.```
Вспоминая проблемы с установкой SP1 для Vista в свое время, я отправился искать System Update Readiness Tool для Windows 2008 R2 и, в общем-то, [не обломался](http://www.microsoft.com/downloads/ru-ru/details.aspx?familyid=c4b0f52c-d0e4-4c18-aa4b-93a477456336&displaylang=ru "System Update Readiness Tool for Windows 2008R2").
Readiness tool успешно отработал, но в отличии от Vista Service Pack ставиться все равно не хотел. CheckSUR.log можно найти найти в каталоге
```C:\Windows\Logs\CBS```
и выяснить кто всему виной. В моем случае это было
```
Unavailable repair files:
servicing\packages\Package\_for\_KB2586448_RTM~31bf3856ad364e35~amd64~~6.1.1.2.mum
servicing\packages\Package\_for\_KB2586448_RTM~31bf3856ad364e35~amd64~~6.1.1.2.cat
```
No problem, идем в гугл, ищем [KB2586448](http://www.microsoft.com/downloads/ru-ru/details.aspx?familyid=646a9a56-c343-45cb-a255-303602aa5a64&displaylang=ru "update") для нашей ОС и языка, потрошим его дважды (msu, потом cab) командой expand F:* и заменяем косячные файлы (предварительно став владельцем каталога packages и установив соответствующие параметры безопасности). И… sp1 ставится как по маслу.
