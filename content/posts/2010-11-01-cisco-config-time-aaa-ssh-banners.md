---
title: 'Cisco, часть 1: time, AAA, ssh, console tricks, banners'
author: ["Stanislav"]
date: 2010-11-01T15:51:52+00:00
url: /2010/11/cisco-config-time-aaa-ssh-banners/
categories:
  - Tech
tags:
  - cisco
---

Подключаемся COM кабелем, затираем startup-config и отправляем железку в ребут:
```CoreRouter#erase startup-config
Erasing the nvram filesystem will remove all configuration files! Continue? [confirm]
[OK]
Erase of nvram: complete
*Nov 1 13:55:40.978: %SYS-7-NV\_BLOCK\_INIT: Initialized the geometry of nvram
CoreRouter#reload
Proceed with reload? [confirm]
*Nov 1 13:55:50.754: %SYS-5-RELOAD: Reload requested by root on console. Reload Reason: Reload Command.```
ждем, пока она загрузится, отказываемся от диалога начальной конфигурации и получаем приглашение user mode:
 ```
— System Configuration Dialog —
Would you like to enter the initial configuration dialog? [yes/no]: no
Press RETURN to get started!
Router>
```
переходим в привилегированный режим (команда enable, приглашение меняется на Hostname#), а затем в режим глобальной конфигурации (команда configure terminal, приглашение меняется на Hostname(config)#) и назначим железке host name и domain name, а так же сразу запрещаем интерпретатору командной строки обрабатывать команду, которую не удается распознать, как символьное имя ip адреса:
```Router(config)#hostname CoreRouter
CoreRouter(config)#ip domain-name my.domain.ltd
CoreRouter(config)#no ip domain-lookup
```
###### Настройки Времени
установим часовой пояс, время и интервал перехода на летнее время, а так же ntp серверы:
```
CoreRouter#clock set 00:00:00 1 JAN 2011
CoreRouter#conf t
CoreRouter(config)#clock timezone Moscow 3
CoreRouter(config)#clock summer-time Moscow recurring last Sun Mar 1:00 last Sun Oct 1:00
CoreRouter(config)#ntp server 192.168.0.1 prefer
CoreRouter(config)#ntp server 192.168.0.2```
###### ААА, доступ по SSH
добавим защиту на основе пароля к консольному порту и линиям VTY:
```
! включаем службу шифрования паролей
CoreRouter(config)#service password-encryption
! создаем новую модель AAA (Authentication, Authorization, Accounting)
CoreRouter(config)#aaa new-model
! и локальную базу пользователей
CoreRouter(config)#aaa authentication login default local
! и пользователя с именем root, паролем P@$sW0rD и наивысшими правами в системе
CoreRouter(config)#username root privilege 15 secret P@$sW0rD
! установим пароль на вход в привилегированный режим
CoreRouter(config)#enable secret 3n@b1e
! сгенерируем ключ длинной 1024 бита для SSH
CoreRouter(config)#crypto key generate rsa modulus 1024
The name for the keys will be: CoreRouter.my.domain.ltd
% The key modulus size is 1024 bits
% Generating 1024 bit RSA keys, keys will be non-exportable…[OK]
*Nov 1 14:31:20.771: %SSH-5-ENABLED: SSH 1.99 has been enabledip ssh time-out 60
! ограничим количество попыток аутентификации двумя
CoreRouter(config)#ip ssh authentication-retries 2
! только вторая версия ssh
CoreRouter(config)#ip ssh version 2
! режим конфигурации линий vty
CoreRouter(config)#line vty 0 4
! установим уровень привелегий, который получает пользователь после логина – 1 – user exec mode, 15 – enable exec mode
CoreRouter(config-line)# privilege level 1
! ограничим возможность подключения к линиям только с помощью SSH
CoreRouter(config-line)# transport input ssh```
###### Тюнинг консоли
Можно, но не обязательно заняться тюнингом консольки (в режиме конфигурации con или vty линий):
* logging synchronous – синхронизируем вывод незапрошенных сообщений и команд отладки с запрошенным выводом
* history size 100 – устанавливает размер буфера истории в 100 команд
* exec-timeout 60 – задает время бездействия перед разрывом сеанса в минутах (у меня 60 для консольной линии и 2 для vty)
###### Баннеры:
Не вдаваясь в [подробности](http://www.cisco.com/en/US/tech/tk583/tk617/technologies_tech_note09186a00800949e2.shtml "http://www.cisco.com/en/US/tech/tk583/tk617/technologies_tech_note09186a00800949e2.shtml") есть 3 вида баннеров: login, motd и exec.
Напугаем хацкера login banner’ом (лучше набив его предварительно где-нибудь моноширинным шрифтом, а затем скопипастить) и сохраним конфигурацию в NVRAM командой wr (или copy running-config startup-config
):```
CoreRouter(config)#banner login @
Enter TEXT message. End with the character ‘@’
+——————————————————–+
| |
| CoreRouter |
| |
| Access to this system is for authorized users only. |
| Any unauthorized entry or attempt to enter |
| is strictly forbidden and will result in prosecution |
| to the maximum extent allowable by applicable law. |
| |
+——————————————————–+@
CoreRouter(config)#exit
*Nov 1 16:07:10.887: %SYS-5-CONFIG_I: Configured from console by root on console
CoreRouter#wr
Building configuration…
[OK]
CoreRouter#
```
Часть 2 – [Конфигурация интерфейсов, настройка простого NAT с overloading (PAT)](/2010/11/cisco-interface-config-nat-pat/ "/2010/11/cisco-interface-config-nat-pat/")
