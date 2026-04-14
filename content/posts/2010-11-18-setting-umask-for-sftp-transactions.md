---
title: Устанавливаем umask для SFTP транзакций
author: ["Stanislav"]
date: 2010-11-18T07:47:30+00:00
url: /2010/11/setting-umask-for-sftp-transactions/
categories:
  - Tech
tags:
  - linux
  - sftp
---

```root@localhost# nano /etc/ssh/sshd_config
Subsystem sftp /bin/sh -c ‘umask 0002; /usr/libexec/openssh/sftp-server'```
via: [http://jeff.robbins.ws/articles/setting-the-umask-for-sftp-transactions](http://jeff.robbins.ws/articles/setting-the-umask-for-sftp-transactions "http://jeff.robbins.ws/articles/setting-the-umask-for-sftp-transactions")
