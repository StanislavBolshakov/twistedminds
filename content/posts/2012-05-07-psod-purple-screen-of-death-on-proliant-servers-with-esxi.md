---
title: PSOD (Purple Screen Of Death) в ESXi на серверах HP Proliant
author: ["Stanislav"]
date: 2012-05-07T14:38:33+00:00
url: /2012/05/psod-purple-screen-of-death-on-proliant-servers-with-esxi/
categories:
  - Tech
tags:
  - esxi
  - hp
  - proliant
  - psod
  - vmware
---

Симптомы:
 Низкая производительность виртуальных машин, PSOD в процессе установки ESXi.

Решение проблемы:
 В BIOS, в опциях по контролю питания HP Power Profile следует выставить Maximum Performance, что, собственно контроль и отключает.

*via <http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1018206>*
