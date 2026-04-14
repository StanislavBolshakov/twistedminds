---
title: Настройка Citrix Access Gateway на NetScaler VPX
author: ["Stanislav"]
date: 2012-12-17T12:46:20+00:00
url: /2012/12/setting-up-cag-on-netscaler-vpx/
categories:
  - Tech
tags:
  - access gateway
  - citrix
  - citrix WI
  - netscaler
---
![citrix logo](/wp-content/uploads/2012/09/citrix-logo.gif)Подразумевается, что вся первоначальная настройка NetScaler вами уже была произведена. Если нет, то смело обращайтесь к [предыдущей статье](/2012/12/getting-started-with-netscaler-vpx/ "начальная настройка NetScaler") или официальной документации. Настройку же Citrix Access Gateway (CAG) можно условно разделить на два этапа – настройка WebInterface (WI) / Storefront aka CloudGateway (в этой статье рассматриваться не будет) и настройка непосредственно виртуального сервера на NetScaler VPX с функцией CAG. Мы создадим на WI две точки входа для клиентов, использующих Citrix Receiver и для клиентов, которые получают доступ к опубликованным приложениям через обычный браузер, создадим CAG vserver и привяжем его к этим точкам входа, настроив LDAP аутентификацию. На сладкое у нас смена темы welcome на более няшную.

#### Настройка Citrix Web Interface

Откроем оснастку Citrix Web Interface Management и создадим новый веб сайт для пользователей, использующих браузер. Настройки типичны для обычного сайта, за исключением метода безопасного доступа.

Так как меня пучит от одной только мысли о возможной необходимости копипастить сотни окошечек настроек я покажу конечный результат и опишу настройки в виде текста:

[![citrix-wi-mobile](/wp-content/uploads/2012/12/citrix-wi-mobile-800x592.png)][1]Как не трудно заметить в качестве точки аутентификации Gateway Direct с методом Explicit, аутентификация выполняется по адресу вашего будущего CAG vserver:
```

https://cag.domain.local/CitrixAuthService/AuthService.asmx
```

Теперь создадим новый сервис для пользователей Citrix Receiver. Настройка, опять же, типична для сервиса:
[![citrix-wi-pna-mobile](/wp-content/uploads/2012/12/citrix-wi-pna-mobile-800x593.png)][2]В настройках Gateway нужно указать fqdn вашего будущего CAG. На этом настройку WI можно считать законченной.
#### Настройка Citrix Access Gateway vserver
1. Первым делом добавим LDAP сервера (Configuration > Access Gateway > Policies > Authentication > LDAP > Servers):
[![ns add ldap auth server](/wp-content/uploads/2012/12/ns-auth-server.png)][3]***Это минимально-рабочая, не оптимальная и не безопасная конфигурация. После проверки лучше запихните пользователей CAG в отдельную группу в AD.***
и создадим политики аутентификации (Configuration > Access Gateway > Policies > Authentication > LDAP > Policies):
[![ns auth policy](/wp-content/uploads/2012/12/ns-auth-policy.png)][4]2. Добавим профили сессий (Configuration > Access Gateway > Policies > Session > Profiles) для PNA сервиса и WI, которые будут указывать на созданные точки входа в первом акте:
[![ns wi profile](/wp-content/uploads/2012/12/ns-wi-profile.gif)][5]
[![ns pna service profile](/wp-content/uploads/2012/12/ns-pna-service-profile.gif)][6]

Теперь давайте создадим политики  (Configuration > Access Gateway > Policies > Session > Policy), в соответствии с которыми если в User-Agent клиента будет иметь место быть “Receiver”, то пользователя перенаправят (в случае успешной аутентификации) на PNA Service. В остальных случаях пользователю будет показан WI.

[![ns session policy](/wp-content/uploads/2012/12/ns-session-policy.png)](/wp-content/uploads/2012/12/ns-session-policy.png) Все что осталось, это создать vserver CAG и привязать все созданные политики к нему ( (Configuration > Access Gateway > Virtual Servers > ADD). В конечном итоге у вас должно получиться что-то в этом роде:

![ns cag vserver](/wp-content/uploads/2012/12/ns-cag-vserver.gif)

Ну и последним штрихом осталось поменять тему.

#### Смена темы Citrix Access Gateway на NetScaler VPX

Мне известно о трех темах, информацию о которых можно найти в [блоге Citrix](http://blogs.citrix.com/?s=Theme+for+Citrix+NetScaler+Access+Gateway+Enterprise&go= "Citrix Access Gateway Theme"). Установим новую тему на примере Symphony:

* Создадим файл /tmp/Symphony#.sh
* Вставим содержимое [http://cdn.ws.citrix.com/wp-content/uploads/2012/06/Symphony.txt](http://cdn.ws.citrix.com/wp-content/uploads/2012/06/Symphony.txt "symphony script") в этот файл.
* Установим exec bit (chmod +x Symphony.sh).
* И выполним скрипт с параметром # 1 или 2. Вариант 1 – скроет список доменов, а 2 – покажет.

В итоге получим вот такую няку:

[![cag welcome page](/wp-content/uploads/2012/12/cag-welcome-page.png)](/wp-content/uploads/2012/12/cag-welcome-page.png)
