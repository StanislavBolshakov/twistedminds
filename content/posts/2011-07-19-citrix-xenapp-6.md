---
title: Установка и настройка Citrix XenApp 6
author: ["Stanislav"]
date: 2011-07-19T09:43:39+00:00
url: /2011/07/citrix-xenapp-6/
categories:
  - Tech
tags:
  - application virtualization
  - citrix
  - howto
  - xenapp
  - решаем проблемы
---

Тут будет мой скромный опыт по установке и начальной конфигурации Citrix XenApp 6. Статья разделена на 4 раздела:

1. [Установка (это оказалось не так просто)](/2011/07/citrix-xenapp-6/#more-1804 "Установка Citrix XenApp 6")
2. [Создание и настройка фермы, подключение XenApp к серверу баз данных](/2011/07/citrix-xenapp-6/2/ "Создание и настройка фермы, подключение XenApp к серверу баз данных")
3. [Настройка лицензирования RDP, ICA. Подключение лицензий XenApp](/2011/07/citrix-xenapp-6/3/ "Настройка лицензирования RDP, ICA. Подключение лицензий XenApp")
4. Начальная конфигурация
* * *

**Установка Citrix XenApp 6**

Чтобы приготовить XenApp нам нужны:

- дистрибутив (требуется регистрация) – [http://www.citrix.com/English/ss/downloads/index.asp](http://www.citrix.com/English/ss/downloads/index.asp "xenapp6 distrib")
- сервер лицензий  – [http://www.citrix.com/English/ss/downloads/results.asp?productID=1679389](http://www.citrix.com/English/ss/downloads/results.asp?productID=1679389 "citrix license server") на момент написания 11.9. Тот, что идет в комплекте c дистрибутивом (11.6) весьма бажный :/
- рекомендуемые обновления для XenApp и для Windows 2008R2:

– [http://support.citrix.com/article/CTX129229](http://support.citrix.com/article/CTX129229 "citrix public hotfixes") (статья обновляется)

– [http://support.citrix.com/product/xa/v6.0_2008r2/hotfix/general/public](http://support.citrix.com/product/xa/v6.0_2008r2/hotfix/general/public "citrix public hotfixes")

Записываем ISO на double layer DVD (или монтируем в виртуальный привод) и приступаем. Распакованный ISO-образ у меня отказался ставиться.

1. Устанавливаем и конфигурируем сервер лицензий.
2. Запускаем autorun в корне диска и выбираем Install XenApp Server.
3. Видим, что роль License Server уже установлена.
4. Выбираем редакцию сервера (о  различия в редакциях можно прочитать тут [http://www.citrix.com/site/resources/dynamic/additional/Citrix_XenApp_Comparative_Feature_Matrix.pdf](http://www.citrix.com/site/resources/dynamic/additional/Citrix_XenApp_Comparative_Feature_Matrix.pdf "citrix xenapp version matrix")).
5. Устанавливаем непосредственно XenApp и Web Interface.
6. Дополнительно устанавливаем XML Service IIS Integration и переходим непосредственно к установке, соглашаясь на все предложения о перезагрузках. После перезагрузки повторно запускаем autorun и выбираем resume install.
7. Устанавливаем обновления.

*Во время установки XenApp на реальное железо возникла забавная ситуация – установщик вылетал с ошибкой. Закончилось все переустановкой Windows 2008R2 и далее все пошло, как по маслу.*

* * *

**Приступим к начальной конфигурации выбрав пункт Configure напротив XenApp:**

![configure xenapp](/wp-content/uploads/2011/06/configure-xenapp.png "configure xenapp")

Для начала создадим новую ферму:

![xenapp create farm](/wp-content/uploads/2011/06/xenapp-create-farm.png "xenapp create farm")

Зададим имя фермы и аккаунт администратора:

![xenapp name farm](/wp-content/uploads/2011/06/xenapp-name-farm.png "xenapp name farm")

Укажем FQDN имя или IP адрес сервера лицензий:

![xenapp lic. server](/wp-content/uploads/2011/06/xenapp-lic-server.png "xenapp lic. server")

Присосемся к существующей, предварительно созданной, базе данных (в моем случае):

![xenapp sql connection](/wp-content/uploads/2011/06/xenapp-sql-connection.png "xenapp sql connection")

Укажем сервер, базу данных, имя пользователя и пароль для доступа:

![xenapp sql setup](/wp-content/uploads/2011/06/xenapp-sql-setup.png "xenapp sql setup")

Разрешим shadowing,  укажем имя для зоны и применим конфигруцию:

![](/wp-content/uploads/2011/06/xenapp-setup-finished.png "xenapp setup finished")

* * *

**Настройка лицензирования**

Установим роль “Лицензирование удаленных рабочих столов” для “Службы удаленных рабочих столов”.

![citrix lic rds installation](/wp-content/uploads/2011/07/citrix-lic-rdp-installation.png "citrix lic rds installation")

Зайдем в диспетчер лицензирования, активируем сервер и установим сертификаты, получив в итоге следующее:

![activate rds server](/wp-content/uploads/2011/07/activate-rds-server.png "activate rds server")

Зайдем в конфигурацию узла сеансов удаленных рабочих столов и в диагностики обнаружим два предупреждения:

![rds lic errors](/wp-content/uploads/2011/07/rds-lic-errors.png "rds lic errors")

Чтобы это исправить в найстройках лицензирования укажем наш сервер лицензий для RDP и ICA (в моем случае локальный):

![ICA RDP lic server](/wp-content/uploads/2011/07/ICA-RDP-lic-server.png "ICA RDP lic server")

Перейдем непосредственно к установке лицензий Citrix XenApp (если у вас нет файла лицензий, то вы можете сгенерировать пробный 90-и дневную лицензию в личном кабинете).

Зайдем на http://hostname:8082 (порт по умолчанию или тот, который вы задали при конфигурации) и пройдем аутентификацию с именем пользователя и пароли, который вы задали при начальной конфигурации сервера лицензий.

В вкладке administration перейдем в Vendor Daemon Configuration и импортируем лицензии. В итоге получим следующую картину:

![citrix lic](/wp-content/uploads/2011/07/citrix-lic.png "citrix lic")
