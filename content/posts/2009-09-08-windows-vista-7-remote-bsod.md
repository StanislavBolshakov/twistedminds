---
title: Windows Vista / 7 remote BSOD
author: ["Stanislav"]
date: 2009-09-08T12:15:47+00:00
url: /2009/09/windows-vista-7-remote-bsod/
categories:
  - Tech
tags:
  - microsoft
  - windows
  - баг
---

Laurent Gaffié в своем блоге опубликовал эксплойт, использующий уязвимость в новом smb-драйвере, входящем в состав Windows Vista и Windows7. Специально сформированный заголовок smb-пакета позволяет удаленно вызвать BSOD на атакуемой машине.
 Microsoft поставлены в известность, но никаких патчей до сих пор нет. Единственным способом защиты на данный момент является закрытие smb-портов фаерволлом.

[http://g-laurent.blogspot.com/2009/09/windows-vista7-smb20-negotiate-protocol.html](http://g-laurent.blogspot.com/2009/09/windows-vista7-smb20-negotiate-protocol.html "http://g-laurent.blogspot.com/2009/09/windows-vista7-smb20-negotiate-protocol.html")
