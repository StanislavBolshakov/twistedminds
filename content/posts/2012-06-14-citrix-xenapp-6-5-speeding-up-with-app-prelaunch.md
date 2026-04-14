---
title: 'Citrix XenApp 6.5: ускоряем загрузку приложений с prelaunch'
author: ["Stanislav"]
date: 2012-06-14T14:32:16+00:00
url: /2012/06/citrix-xenapp-6-5-speeding-up-with-app-prelaunch/
categories:
  - Tech
tags:
  - citrix
  - howto
  - xenapp
---
[![XenApp Logo](/wp-content/uploads/2012/02/XenAppLogo.png "Citrix XenApp Logo")][1]Session lingering, базару нет, есть хорошо, но запуск отдельных buisness critical приложений можно дополнительно ускорить с помощью Application Prelaunch.

Для начала убедимся что у нас:
1. установлен On-line Plug-in версии 13.x и выше.
2. ветка реестра HCLM\SOFTWARE\Citrix\ICA Client\Prelaunch выглядит следуюищм образом –
[![prelaunch значения реестра](/wp-content/uploads/2012/06/pre-launch-300x84.png "prelaunch registry entry")][2]
На сервере заходим в свойства фермы и создаем application prelaunch:
[![создать prelaunch приложение](/wp-content/uploads/2012/06/create-prelaunch-app-300x247.png "create prelaunch application")][3]

На клиенте подключаемся снова:

[![переподключение к серверу](/wp-content/uploads/2012/06/re-log-300x124.png "xenapp server relogging")](/wp-content/uploads/2012/06/re-log.png)

И на сервере можем наблюдать выбранное приложение в состоянии prelaunch.

[![предзапущенное приложение](/wp-content/uploads/2012/06/prelaunched-application-800x15.png "prelaunched application")](/wp-content/uploads/2012/06/prelaunched-application.png)

Если у вас так же настроен session lingering, то prelaunch приложение провалится в lingering сессию и позволит запускать уже любые приложения в мгновение ока. Ессесно и application prelaunch и session lingering жрут лицензии.
