---
title: 'Citrix Access Gateway на NetScaler VPX – tips & tricks'
author: ["Stanislav"]
date: 2012-12-18T07:47:21+00:00
url: /2012/12/citrix-access-gateway-на-netscaler-vpx-tips-tricks/
categories:
  - Tech
tags:
  - access gateway
  - citrix
  - netscaler
  - troubleshooting
---

[![citrix logo](/wp-content/uploads/2012/09/citrix-logo.gif)](/wp-content/uploads/2012/09/citrix-logo.gif)В заключительной части хотелось бы поделиться некоторыми проблемами, с которыми я столкнулся в результате работы, а так же разъяснить пару моментов, которые могут быть не совсем очевидными. А именно: мы рассмотрим процесс подключения пользователя к CAG, посмотрим как выявить проблемы с LDAP аутентификацией, как отфильтровать пользователей по заданным атрибутам в LDAP, выявлять неполадки в STA, запуск HTTP vserver вместо HTTPS и некоторые другие моменты.

**Процесс подключения**

Для начала рассмотрим процесс подключение, взяв простейшую схему. Откровенно говоря это следовало бы сделать в первых двух частях, but meh.

![cag access scheme](/wp-content/uploads/2012/12/cag-access-cheme.png "cag access scheme")

- Пользователь попадает на https://cag.company.ltd
- Access Gateway терминирует SSL и производит аутентификацию пользователя (а если включен Smart Access еще и валидацию оборудования).
- В соответствии с заданными политиками происходит коммуникация с Web Interface (WI) и пользователю отображается сайт. CAG выступает в роли reverse proxy.
- Пользователь запускает приложение, кликнув на иконку.
- WI запрашивает тикет от XML сервиса (STA – Secure Ticket Authority).
- WI отправляет пользователю тикет в ICA файле.
- Citrix Receiver запускает ICA клиент, заворачивает ICA сессию в SSL и отправляет все это дело до Access Gateway.
- Если Access Gateway получает информацию о том, что тикет валидный, то ICA сессия устанавливается, пользователю показывается приложение.
**Выявление проблем с LDAP аутентификацией**
В tmp лежит named pipe, в который сливается лог аутентификации. Посмотреть его можно так:
```

root@ns# cat /tmp/aaad.debug
[...]
Tue Dec 18 06:51:18 2012
 /usr/home/build/rs_100_71_5/usr.src/usr.bin/nsaaad/../../netscaler/aaad/ldap_drv.c[1243]: receive_ldap_user_bind_event Other invalid credentials
Tue Dec 18 06:51:18 2012
 /usr/home/build/rs_100_71_5/usr.src/usr.bin/nsaaad/../../netscaler/aaad/naaad.c[1683]: send_reject sending reject to kernel for : user
```

пример провалившейся аутентификации в следствии неправильно введенных данных.
```

[...]
Tue Dec 18 06:51:22 2012
/usr/home/build/rs_100_71_5/usr.src/usr.bin/nsaaad/../../netscaler/aaad/naaad.c[1587]: send_accept sending accept to kernel for : user
```

а это пример того, как все должно работать. Больше информации о LDAP auth named pipe можно получить из этой статьи – [http://support.citrix.com/article/CTX114999](http://support.citrix.com/article/CTX114999 "LDAP auth debug named pipe").
**Фильтр поиска LDAP**
Подобная конфигурация

![ldap search filter](/wp-content/uploads/2012/12/ldap-search-filter.png "ldap search filter")

фактически разрешает любому пользователю в Active Directory получать доступ к ресурсам, опубликованным на вашей ферме терминальных серверов.

Выдать разрешения пользователям можно объединив их в группу, например cag-enabled-users. Затем можно выполнять поиск конкретно по этой группе с помощью фильтра поиска, например так:
```

memberOf=CN=cag-enabled-users,CN=Users,DC=domain,DC=local
```

**Выявление неполадок с STA**

Первым делом следует проверить ошибки на стороне сервера с XML Services, установив log level 3 в CtxSta.config. Логи находятся в **С:\Program Files\Citrix\logs.**

Проверить ошибки STA на netscaler можно следующим образом:
```

> sh vpn stats | grep STA
STA connection success                            49
STA connection failure                             0
STA request sent                                  64
STA response received                             34
```

На сервере с WebInterface попытайтесь открыть в браузере url, заменив cag.domain.company.local на fqdn вашего cag **https://cag.domain.compnay.local/CitrixAuthService/AuthService.asmx**. В результате вы должны получить ошибку, мол, this page could not be displayed. Никаких ошибок SSL в результате этого теста появляться не должно.

**Ошибки, связанные с agsso.aspx и setclient?wica**

Если браузер пользователя “застрял” на этих страницах, то весьма вероятно, что между CAG vserver, WI и фермой существуют проблемы связности по 80, 443, 1494, 2598 портам. Вооружитесь telnet и проверьте все в обе стороны. Так же не исключены проблемы с DNS. Для troubleshoot’a воспользуйтесь либо IP адресами, либо занесите IP и hostname для CAG, WI, фермы в hosts файлы.

**Логирование и мониторинг**

Первым делом когда что-то идет не так и вы не можете понять почему – проверьте /var/log/license.log

Так, в процессе ввода в эксплуатацию я получил ошибку о невозможности генерации CSR на основании RSA ключи длиннее 512 бит. Лог сервера лицензирования грязно ругался на неправильную лицензию, а GUI NetScaler уверял, что все ок. В итоге пришлось удалять лицензию, перезагружать NetScaler и устанавливать ее снова.

Так же особой ценностью обладает лог /var/log/ns.log, являющийся syslog NetScaler.

Для логирования проблем с различными vserver можно создать отдельную политику (о том, как это сделать можно прочитать в первой статье) и привязать ее к сервису. Для CAG это делается так:
```

bind vpn vserver cag.domain.local -policy cag_mon_pol -priority 100
```

**Создание и настройка HTTP CAG vserver**

Согласитесь, не очень удобно выявлять неполадки расшифровывая wireshark трейсы с помощью приватного ключа SSL сертификата CAG. Вместо этого можно легко настроить HTTP CAG vserver и смотреть все в clear text. Делается это так:
```

root@ns# nsapimgr -ys add_http_vpn_vserver=1
Changing add_http_vpn_vserver from 0 to 1 ... Done.
> Add vpn vserver debug HTTP 10.10.10.10 80
```
