---
title: Проблемы с загрузкой Citrix Access Gateway
author: ["Stanislav"]
date: 2012-09-06T10:21:37+00:00
url: /2012/09/repairing-faulty-cag-vpx/
categories:
  - Tech
tags:
  - cag vpx
  - citrix
  - troubleshooting
---

![](/wp-content/uploads/2012/09/citrix-logo.gif "citrix logo")В один прекрасный момент сразу два виртуальных аплайинса Citrix Access Gateway VPX в failover конфигурации после потери питания отказались загружаться. Времени разбираться в чем именно дело не было, я развернул новые шлюзы и открыл тикет. Хорошо, что я не стал ждать ответа саппорта, потому что все, что я получил после недели переписки стал /dev/null.

Откровенно говоря, я забил на аплаинсы, потушил давшие сбой виртуальные машины и отправил их в архив. Ну а сегодня закончилось время действия SSL сертификатов и я про них вспомнил. Учитывая то, что подготовка к CCNP порядком надоела любая отвлеченная от нее активность воспринимается как благо.

И так, у меня был Citrix Access Gateway VPX, который в процессе загрузки выдавал ошибку запуска демона agxboss и уходил в ребут:
```

Starting agxboss...
rc=255						[ FAIL ]
```

По умолчанию аплайнс дает доступ в ограниченную унылую консоль с правами пользователя, заключенного в chroot, а значит первым делом не мешало бы получить root и полноценную консоль. Этим и займемся.

Первым делом надо загрузить виртуальную машину с какого-нибудь Linux LiveCD и смонтировать диск куда-нибудь:
```

mkdir /mnt/cag && mount -t ext3 /dev/sda3 /mnt/cag
```

Сходим в shadow, очистим рутовый пароль и дадим ему возможность заходит по SSH:
```

nano /mnt/cag/etc/shadow
```

В итоге должно получиться как-то так:
```

CAG@cag / # cat /etc/shadow | grep root
 root:::0:99999:7:::
```

Перезагружаем аплайнс и ждем пока он провалится в recovery mode после трех неудачных запусков. Теперь зайдя через админский урезанный шел включим SSH ([1] System > [2] Toggle SSH Access) и зайдем на апплайнс под рутом.

Изучение файлов инициализации натолкнуло на мысль, что демон agxboss дергает postgresql для загрузки конфигурации, но последнему что-то не нравится. Попытаемся запустить его руками:
```

su postgres -c '/opt/ag/sw/agx/db/bin/postgres -D /opt/ag/rt/db/'
FATAL:  bogus data in lock file "postmaster.pid": "r invalid-ticket.
        201206012305.48 4151"
```

Оп-оп, а вот и проблема. Найдем и удалим паршивца, а затем повторим процедуру:
```

CAG@cag /opt/ag/sw/agx/db/bin # find / -name 'postmaster.pid'
 /opt/ag/rt/db/postmaster.pid
CAG@cag /opt/ag/sw/agx/db/bin # rm -f /opt/ag/rt/db/postmaster.pid
```

```

CAG@cag /opt/ag/sw/agx/db/bin # su postgres -c '/opt/ag/sw/agx/db/bin/postgres -D /opt/ag/rt/db/'
 LOG: database system was interrupted; last known up at 2012-06-06 13:21:10 PDT
 LOG: database system was not properly shut down; automatic recovery in progress
 LOG: redo starts at 0/D2950D8
 LOG: unexpected pageaddr 0/B2A6000 in log file 0, segment 13, offset 2777088
 LOG: redo done at 0/D2A5FB8
 LOG: last completed transaction was at log time 2012-06-06 13:21:31.926276-07
 LOG: database system is ready to accept connections
 LOG: autovacuum launcher started
 ^CLOG: received fast shutdown request
 LOG: aborting any active transactions
 LOG: autovacuum launcher shutting down
 LOG: shutting down
 LOG: database system is shut down
```

Вот теперь другое дело. Можно смело перезагружать аплаинс и наслаждаться восставшим из пепла Citrix Access Gateway VPX.
