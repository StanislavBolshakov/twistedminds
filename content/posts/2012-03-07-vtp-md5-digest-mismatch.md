---
title: VTP и MD5 Digest Mismatch
author: ["Stanislav"]
date: 2012-03-07T09:34:43+00:00
url: /2012/03/vtp-md5-digest-mismatch/
categories:
  - Tech
tags:
  - catalyst
  - cisco
  - troubleshooting
  - vlan
  - vtp
---

При добавлении нового коммутатора последний упрямо не хотел подсасывать VLANы с сервера. Пароли и имя домена были одинаковы. В выхлопе команды sh vtp status на коммутаторе-vtp-сервере имела место быть запись

```*** MD5 digest checksum mismatch on trunk: Gi1/20 ***```

ревизия базы клиента была явно меньше – 0.

Того же рода сообщения появлялись в дебаге – debug sw-vlan vtp events (не забудьте включить terminal monitor, если подключены через telnet/ssh).

Советы с форумов, сводящиеся к переводу коммутатора в режим transparent, а затем обратно в client не помогали, однако спасло добавление на vtp сервере левого vlan, подача команды no shut на нем и дальнейшее его удаление.
