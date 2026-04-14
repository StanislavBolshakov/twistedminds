---
title: Citrix Receiver
author: ["Stanislav"]
date: 2012-12-24T10:10:30+00:00
url: /2012/12/citrix-receiver/
categories:
  - Tech
tags:
  - citrix
  - receiver
---

![Citrix Receiver](/wp-content/uploads/2012/12/citrix_receiver.png "Citrix Receiver")В процессе чтения форумов заметил, что у людей возникает масса вопросов на тему того, какой Citrix Receiver использовать и в каких ситуациях. Если вас, так же как и меня поглотило предновогоднее веселье, то, возможно, вы немного выпали из обоймы. Подведем итоги – у нас есть две версии Citrix Receiver (обычная и enterprise) и есть олд скульный Citrix Online plug-in (которых, в общем, тоже две версии). Давайте разберемся кто все эти люди.

#### Виды Citrix Receiver’ов

- **Citrix Receiver (Receiver for Windows)** – версия, которую большинство людей и скачивает. Отличается различными ограничениями, однако не требует прав администратора для установки.
- **Citrix Receiver Enterprise (Receiver for Windwos (Legacy PNA)** – как не трудно догадаться из название эта версия “расширена” дополнительной компонентой, отвечающий за SSO, pass-through authentication и smart card authentication.
- **Citrix Online plug-in** – это давным давно было то, во что переродился разжиревший Citrix Receiver с поддержкой облачный фишечек, StoreFront, CloudGateway, ShareFile и т.д. Оригинальное предназначение Online plug-in’а состояло в том, что ему скармливался URL PNA (*Program Neighborhood Agent*) сервиса с xml, в которой были обозначены все параметры, которые следует передать клиенту. В дайльнейшем пользователь запускал приложения так, как будто они установлены на его компьютере без необходимости скачивать и обрабатывать ICA файл. В свою очередь существовало две версии Online plug-in’а:
  - **Online plug-in Full** – то, что сейчас представляется собой Citrix Receiver Enterprise.
  - **Online plug-in Web** – то, что теперь обычный Citrix Receiver, но дополненный Self-Service Plug-in’ом и Receiver UI, который, фактически, отображает содержимое web interface в человечьем виде.
####  Какой же Citrix Receiver стоит использовать?

|  |  |  |  |
| --- | --- | --- | --- |
| Receiver | Метод доступа |  | Выполняемые функции |
| Citrix Receiver | Доступ через web к опубликованным приложениям и виртуальным рабочим станциям. | Не требует привелегии администратора для установки. | - Доступ к опубликованным приложениям и рабочим столам. - Проброс USB. - HDX Media Stream для Flash - Интеграция с другими plug-in’ами |
| Citrix Reciever Enterprise | Тоже, что и Citrix Receiver + прозрачная интеграция приложений и виртуальных рабочих станций в рабочее окружение пользователя. | Требует привелегии администратора для установки. | - Доступ к опубликованным приложениям и рабочим столам. - Проброс USB. - HDX Media Stream для Fash. - Интеграция и прозрачная аутентификациясдругимиplug-in’ами. - Поддержка PNAgent. - Приложения доступны в меню пуск и на рабочем столе. |

Вот вам крохотная табличка в помощь :)

Стоит ли вообще использовать Citrix Receiver, если есть легкий Online Plug-in? На сколько я знаю последний или уже end of support или таковым будет объявлен в ближайшее время. Плюс Citrix Receiver Enterprise поддерживает такую замечательную штуку, как Application Pre-launch. [О том что это и как ее включить можно прочитать тут](/2012/06/citrix-xenapp-6-5-speeding-up-with-app-prelaunch/ "Citrix XenApp application pre-launch").

Свободная интерпритация статьи – [http://support.citrix.com/proddocs/topic/receiver-30-windows/ica-clients-deciding-v2.html](http://support.citrix.com/proddocs/topic/receiver-30-windows/ica-clients-deciding-v2.html "Какой Citrix Receiver выбрать?")
