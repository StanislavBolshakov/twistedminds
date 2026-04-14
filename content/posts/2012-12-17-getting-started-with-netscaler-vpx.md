---
title: Netscaler VPX – общие сведения, первоначальная настройка
author: ["Stanislav"]
date: 2012-12-17T10:30:59+00:00
url: /2012/12/getting-started-with-netscaler-vpx/
categories:
  - Tech
tags:
  - citrix
  - netscaler
---
#### citrix logoNetscaler VPX

Сегодня у нас в планах приседания с замечательным комбайном, который включает в себя целый ряд вкусных и полезных функций. Citrix позиционирует Netscaler как Application Delivery Controller (ADC), способный выполнять такие функции как балансировка нагрузки, SSL offloading, CloudBridge (l2 DC bridging), DNS Server, HTTP compression, Load Balancing, Web Application Firewall и многие другие.  В этой статье мы разберемся с типами IP адресов, понятием виртуального сервера, режимами пересылки пакетов и займемся первоначальной настройкой.

Вся представленная информация является произвольной интерпретаций документации с knowledge center Сitrix, по этому, возможно, имеет смысл обратиться к первоисточнику если возникнут какие-либо вопросы – [http://support.citrix.com/proddocs/topic/access-gateway/ag-edocs-landing.html](http://support.citrix.com/proddocs/topic/access-gateway/ag-edocs-landing.html "Access Gateway 10 documentation")

#### NetScaler: типы IP адресов

- NetScaler IP address (**NSIP**) – управляющий IP адрес для доступа к системе, heartbeat интерфейс для HA, syslog source, etc;
- Virtual IP address (**VIP**) – IP ассоциированный с виртуальным сервером. Зачастую является публичным IP адресом с которым соединяются клиенты;
- Mapped IP address (**MIP**) – MIP адрес используется для соединений с серверами. Зачастую является адресом смотрящим в сторону ваших серверов для которых нужно осуществлять балансировку нагрузки и/или обеспечивать доступ через Access Gateway и/или защитить приложение с помощью Web Application Firewall (WAF). Когда MIP принимает пакет обычно происходит замена IP адреса источника на MIP до того как пакет будет отправлен по направлению к серверу.
- Subnet IP address (**SNIP**) – когда Netscaler взаимодействует с множеством подсетей SNIP могут быть настроены как MIP для предоставления доступа к этим подсетям. SNIP можно привязывать к VLAN и интерфейсам.
#### Виртуальные сервера

Виртуальный сервер (**vserver**) именованная сущность, целью которой является предоставить внешним пользователям доступ к приложениям. Vserver имеет имя (имеет значение только локально, служит для легкой идентификации при настройке), VIP, порт и протокол. Когда клиент пытается получить доступ к серверу он запрашивает именно VIP, а не адрес сервера. В этом случае Netscaler выступает как TCP proxy.

#### Режимы пересылки пакетов

Netscaler может либо маршрутизировать пакеты, предназначенные для IP адресов отличных от адресов Netscaler’a (NSIP, VIP, MIP, SNIP), либо коммутировать их. По умолчанию включен режим маршрутизации (L3 routing mode), а коммутация (L2 bridging mode) выключена.

![netscaler packet flowchart](/wp-content/uploads/2012/12/netscaler.png)
**Layer 2 mode**

Этот режим контролирует функцию коммутации пакетов. В этом режиме Netscaler работает как мост для пакетов, которые предназначены не для него. Как уже говорилось по умолчанию этот режим выключен и NetScaler просто отбрасывает пакеты, которые не предназначены не для одного из его MAC адресов. Осторожно включайте L2 режим. Вы не должны допускать ситуации, когда два и более интерфейсов NetScaler смотрят в один и тот же сегмент сети или параллельно установлен какое-нибудь L2 устройство. Так как NetScaler не поддерживает протоколы семейства STP вы получите петлю. Если существует необходимость чтобы несколько интерфейсов смотрели в один и тот же broadcast domain, то соберите интерфейсы в группу и включите LACP.

**Layer 3 mode**

Layer 3 mode контролирует функции маршрутизации пакетов, в этом режиме NetScaler производит пересылку пакетов согласно таблице маршрутизации для всех адресов, которые не принадлежат NetScaler.

**MAC-based forwarding mode**

MAC-based forwarding позволяет обрабатывать трафик более эффективно, позволяя избежать многочисленных ARP запросов и поисков в таблице маршрутизации. В этом режиме NetScaler кэширует MAC адрес отправителя для каждого соединения и возвращает данные на тот же самый MAC адрес.

Когда это может быть полезно? Рассмотрим следующую ситуацию:

![netscaler mac based forwarding](/wp-content/uploads/2012/12/netscaler-mac-based.png)

1. Поступает входящий запрос на VIP сервиса vSRVR-LB.
2. NetScaler смотрит в кэш и выбирает первый сервер.
3. Сервер отвечает NetScaler.
4. NetScaler смотрит в кэш и выбирает маршрутизатор – инициатор соединения.

Особую полезность данная технология приобретает в том случае если Router1 и Router2 терминирует VPN сессии.

Однако, не все йогурты одинаково полезны. Имеют место быть дизайны, которые требуют чтобы входящие и исходящие пути проходили через разные маршрутизаторы. В таких ситуация MAC-based forwarding ломает всю схему.

Вот в принципе по теории и все. На самом деле тема куда более обширная и я рекомендую обратиться к официальной документации в случае если нужен более глубокий тюнинг вашего ADC.

### Первоначальная настройка

Настроим hostname, IP (с их настройкой проблем возникнуть не должно, справку по типам адресов можно получить в начале статьи), DNS, NTP, timezone, syslog, установим SSL сертификат.

Если не указано обратное, то настройка выполняется с помощью шела. Либо встроенного (приглашение >), либо bash (приглашение #). Провалиться в bash из встроенного shell можно с помощью команды ***shell***. Логин и пароль по умолчанию – nsroot/nsroot.

**hostname**
Приведем файл /nsconfig/rc.conf к следующему виду:
```

root@ns# cat /nsconfig/rc.conf
hostname=ns.domain.ltd
```

**DNS**
Добавим адреса DNS серверов:
```

>add dns nameServer 172.16.20.11

>add dns nameServer 172.16.20.12
```

и проверим:
```

> show dns nameServer
1)       172.16.20.11  -  State: UP     Protocol: UDP
2)       172.16.20.12  -  State: UP     Protocol: UDP
 Done
```

**NTP**
Приведем файл /nsconfig/ntp.conf к следующему виду:
```

root@ns# cat /etc/ntp.conf
# keys /nsconfig/ntp/ntp.keys #fixed location TODO: Create /nsconfig/ntp/
server 172.16.20.11 autokey minpoll 6 maxpoll 10 prefer
server 172.16.20.12 minpoll 6 maxpoll 10
```

**timezone**
Временная зона настраивается следующим образом:
В консоле открывается утилита configns и выбирается пункт 4. По завершению нужно сохранить настройки, нажав клавишу 6.
**Syslog**
Добавим сервер:
```

add audit syslogAction <name> <serverIP> [-serverPort <port>] -logLevel <logLevel> ... [-dateFormat ( MMDDYYYY | DDMMYYYY )] [-logFacility <logFacility>] [- tcp ( NONE | ALL )] [-acl ( ENABLED | DISABLED )] [- timeZone ( GMT_TIME | LOCAL_TIME )]
```

```

add audit syslogAction statsrvr 172.16.20.55 -serverPort 20103 -logLevel ALL -dateFormat DDMMYYYY -logFacility LOCAL0 - tcp NONE -acl DISABLED -timeZone LOCAL_TIME
```

привяжем сервер к политике:
```

add audit syslogPolicy <name> <rule> <action>
```

```

add audit syslogPolicy statsrvr_pol ns_true statsrvr
```

привяжем эту политику либо глобально:
```

bind system global <policyName>
```

либо непосредственно к сервису.
**SSL**
Легче всего настроить с использованием вебинтерфейса, зайдя на NSIP Netscaler.
[http://support.citrix.com/article/CTX109260](http://support.citrix.com/article/CTX109260 "installing SSL certificate") – long story short:
* Генерируется RSA ключ (Configuration > SSL > Create RSA key);
* Создается CSR (Configuration > SSL > Create CSR);
* Устанавливаются сертификаты (Configuration > SSL > Certificates > Install);
* Серверный сертификат линкуется с CA и Intermediate Authority, образуя цепь (Configuration > SSL > Certificates > Link).
Получиться должно что-то в этом роде:
```

> show ssl certlink
1)      Cert Name: Comodo        CA Cert Name: Intermediate Comodo
2)      Cert Name: Intermediate Comodo   CA Cert Name: CAroot
```

```

> show ssl certKey
1)      Name: ns-server-certificate
        Cert Path: /nsconfig/ssl/ns-server.cert
        Key Path: /nsconfig/ssl/ns-server.key
        Format: PEM
        Status: Valid,   Days to expiration:5780
        Certificate Expiry Monitor: DISABLED
2)      Name: Comodo
        Cert Path: /nsconfig/ssl/server_cert.crt
        Key Path: /nsconfig/ssl/rsa2048
        Format: PEM
        Status: Valid,   Days to expiration:264
        Certificate Expiry Monitor: DISABLED
3)      Name: CAroot
        Cert Path: /nsconfig/ssl/AddTrustExternalCARoot.crt
        Format: PEM
        Status: Valid,   Days to expiration:2721
        Certificate Expiry Monitor: DISABLED
4)      Name: Intermediate Comodo
        Cert Path: /nsconfig/ssl/PositiveSSLCA2.crt
        Format: PEM
        Status: Valid,   Days to expiration:2721
        Certificate Expiry Monitor: DISABLED
 Done
```
