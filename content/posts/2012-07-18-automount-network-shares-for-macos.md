---
title: Автоматическое подключение к сетевым папкам в MacOS X
author: ["Stanislav"]
date: 2012-07-18T16:27:56+00:00
url: /2012/07/automount-network-shares-for-macos/
categories:
  - Tech
tags:
  - macos
  - smb
---
Функционал autofs описывается тут – [http://images.apple.com/business/docs/Autofs.pdf](http://images.apple.com/business/docs/Autofs.pdf "autofs")

Допустим перед нами стоит задача не просто подключать сетевые шары в Mac OS (что легко реализуется через объекты входа), но и автоматически подключаться к ним даже после просыпания. Открываем терминал и делаем следующее:

> sudo su
> echo ‘/Volumes/Winshare -fstype=smbfs ://username:password@hostname/sharename’ > /etc/auto_mnt
> echo ‘/- auto\_mnt’ >> /etc/auto\_master
После перезагрузки sharename появится в списке смонтированных томов finder.
