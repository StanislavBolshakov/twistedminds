---
title: Проблемы с установкой Starcraft 2 в MacOS X
author: ["Stanislav"]
date: 2010-07-28T11:05:00+00:00
url: /2010/07/starcraft2-macos-x-installation-issues/
categories:
  - Tech
tags:
  - macos
  - starcraft2
  - решаем проблемы

---
```  
28.07.10 15:09:56 com.apple.launchd.peruser.501[104] ([0x0-0x25025].com.blizzard.installer[319]) posix_spawn(&#8220;/Volumes/data/delirium/Downloads/SC2-WingsOfLiberty-ruRU-Installer/Installer.app/Contents/MacOS/InstallerLauncher&#8221;, &#8230;): Permission denied  
28.07.10 15:09:56 com.apple.launchd.peruser.501[104] ([0x0-0x25025].com.blizzard.installer[319]) Exited with exit code: 1```

Заходим в папку с игрой и делаем

```chmod -R 0777 Installer*```