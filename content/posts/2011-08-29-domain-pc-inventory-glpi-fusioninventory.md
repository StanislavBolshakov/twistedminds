---
title: Инветоризация компьютеров в домене с помощью GLPI (0.80.2) + FusionInventory
author: ["Stanislav"]
date: 2011-08-29T11:10:35+00:00
url: /2011/08/domain-pc-inventory-glpi-fusioninventory/
categories:
  - Tech
tags:
  - active directory
  - fusioninventory
  - glpi
  - howto
  - manual
  - инвенторизация
---
Ну вы знаете как это бывает – прибегают из ~~ада~~ бухгалтерии с распечатками за 2006 и спрашивают нет ли у вас более актуальной статистике по парку пк, а то, видите ли, за 5 лет она утратила актуальность. А вы, такие, делаете жест “не беспокойтесь, юзеры, я тут, чтобы решить ваши проблемы”, заходите на http://localhost/glpi и скармливаете им красивенький отчетик в pdf или csv.
Делается это (в моем случае) так – берется сервер на debian с apache2, php5 и mysql5.1, качаются [GLPI](http://www.glpi-project.org/spip.php?article41 "http://www.glpi-project.org/spip.php?article41") (0.80.2):
```wget -c https://forge.indepnet.net/attachments/download/943/glpi-0.80.2.tar.gz```
и [FusionInventory](http://fusioninventory.org/wordpress/download-fusioninventory/ "http://fusioninventory.org/wordpress/download-fusioninventory/") (2.4.0 RC2 для GLPI 0.80.X):
```wget -c http://forge.fusioninventory.org/attachments/download/417/fusioninventory-for-glpi-metapackage_2.4.0-RC2.tar.gz```
устанавливаем glpi и FusionInventory:
```tar -zxvf glpi-0.80.2.tar.gz && mv glpi /var/www/glpi/```
```tar -zxvf fusioninventory-for-glpi-metapackage_2.4.0-RC2.tar.gz && rm -f fusion\*.gz && mv fusi\* /var/www/glpi/plugins/```
Исправляем права:
```chown -R www-data:www-data /var/www/glpi/```
Затем подключаемся к mysql серверу, создаем базу, и даем пользователю glpi полные права:
```root@monsrvr:/var/www# mysql -u root -p
Enter password:
Welcome to the MySQL monitor.
mysql> create database glpi;
Query OK, 1 row affected (0.02 sec)
mysql> grant all privileges on glpi.* to ‘glpi’@’localhost’ identified by ‘glpi’;
Query OK, 0 rows affected (0.14 sec)
mysql> exit
Bye```
Открываем браузер, заходим по адресу http://tehserver/glpi, указываем все, что требуется и попадаем на страницу аутентификации (username: glpi, password: glpi).
Плагин FusionInventory можно активировать перейдя в пункт меню “Настройки” > “Дополнения”. Вот в принципе и все для серверной части.
Функционал этой связки значительно шире чем просто инвенторизация компьютеров, однако я напишу только о том, как подключить обычный Windows клиент и распространить его через Active Directory.
Качается [агент](http://forge.fusioninventory.org/projects/fusioninventory-agent/wiki/Platforms_tested "http://forge.fusioninventory.org/projects/fusioninventory-agent/wiki/Platforms_tested") и выкладывается на шару (у меня samba-шара с read-only доступом для всех на этом же сервере):
```wget -c http://prebuilt.fusioninventory.org/stable/windows-i386/fusioninventory-agent\_windows-i386\_2.1.9-3.exe```
а в Active Directory для вашего OU с компьютерами подлежащими инвенторизации создается следующий [logon script](http://forge.fusioninventory.org/projects/fusioninventory-agent/wiki/Vbs_install_upgrade "http://forge.fusioninventory.org/projects/fusioninventory-agent/wiki/Vbs_install_upgrade"):
```Option Explicit
Dim versionverification, fusionarguments, uninstallocsagent, fusionsetupURL
””’ USER SETTINGS ””’
versionverification = “2.1.9-3”
fusionarguments = “/S /server=http://server1/glpi/plugins/fusioninventory/ /rpc-trust-localhost /runnow”
‘ Depending on your needs, you can use either HTTP or Windows share
fusionsetupURL = “\\server1\data\fusioninventory-agent\_windows-i386\_” & versionverification & “.exe”
‘fusionsetupURL = “http://prebuilt.fusioninventory.org/stable/windows-i386/fusioninventory-agent\_windows-i386\_” & versionverification & “.exe”
uninstallocsagent = “yes”
””’ DO NOT EDIT BELOW ””’
Function baseName (strng)
Dim regEx, ret
Set regEx = New RegExp
regEx.Global = true
regEx.IgnoreCase = True
regEx.Pattern = “.*[/\\]([^/\]+)$”
baseName = regEx.Replace(strng,”$1″)
End Function
Function isHttp (strng)
Dim regEx, matches
Set regEx = New RegExp
regEx.Global = true
regEx.IgnoreCase = True
regEx.Pattern = “^(http(s?)).*”
If regEx.Execute(strng).count > 0 Then
isHttp = True
Else
isHttp = False
End If
Exit Function
End Function
‘ http://www.ericphelps.com/scripting/samples/wget/index.html
Function SaveWebBinary(strUrl) ‘As Boolean
Const adTypeBinary = 1
Const adSaveCreateOverWrite = 2
Const ForWriting = 2
Dim web, varByteArray, strData, strBuffer, lngCounter, ado
‘ On Error Resume Next
‘Download the file with any available object
Err.Clear
Set web = Nothing
Set web = CreateObject(“WinHttp.WinHttpRequest.5.1”)
If web Is Nothing Then Set web = CreateObject(“WinHttp.WinHttpRequest”)
If web Is Nothing Then Set web = CreateObject(“MSXML2.ServerXMLHTTP”)
If web Is Nothing Then Set web = CreateObject(“Microsoft.XMLHTTP”)
web.Open “GET”, strURL, False
web.Send
If Err.Number <> 0 Then
SaveWebBinary = False
Set web = Nothing
Exit Function
End If
If web.Status <> “200” Then
SaveWebBinary = False
Set web = Nothing
Exit Function
End If
varByteArray = web.ResponseBody
Set web = Nothing
‘Now save the file with any available method
On Error Resume Next
Set ado = Nothing
Set ado = CreateObject(“ADODB.Stream”)
If ado Is Nothing Then
Set fs = CreateObject(“Scripting.FileSystemObject”)
Set ts = fs.OpenTextFile(baseName(strUrl), ForWriting, True)
strData = “”
strBuffer = “”
For lngCounter = 0 to UBound(varByteArray)
ts.Write Chr(255 And Ascb(Midb(varByteArray,lngCounter + 1, 1)))
Next
ts.Close
Else
ado.Type = adTypeBinary
ado.Open
ado.Write varByteArray
ado.SaveToFile CreateObject(“WScript.Shell”).ExpandEnvironmentStrings(“%Temp%”) & “\fusioninventory.exe”, adSaveCreateOverWrite
ado.Close
End If
SaveWebBinary = True
End Function
Function removeOCS()
On error resume next
Dim OCS
‘ Uninstall agent ocs if is installed
‘ Verification on OS 32 Bits
On error resume next
OCS = WshShell.RegRead(“HKEY\_LOCAL\_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\OCS Inventory Agent\UninstallString”)
If err.number = 0 then
WshShell.Run “CMD.EXE /C net stop “”OCS INVENTORY SERVICE”””,0,True
WshShell.Run “CMD.EXE /C “”” & OCS & “”” /S /NOSPLASH”,0,True
WshShell.Run “CMD.EXE /C rmdir “”%ProgramFiles%\OCS Inventory Agent”” /S /Q”,0,True
WshShell.Run “CMD.EXE /C rmdir “”%SystemDrive%\ocs-ng”” /S /Q”,0,True
End If
‘ Verification on OS 64 Bits
On error resume next
OCS = WshShell.RegRead(“HKEY\_LOCAL\_MACHINE\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\OCS Inventory Agent\UninstallString”)
If err.number = 0 then
WshShell.Run “CMD.EXE /C net stop “”OCS INVENTORY SERVICE”””,0,True
WshShell.Run “CMD.EXE /C “”” & OCS & “”” /S /NOSPLASH”,0,True
WshShell.Run “CMD.EXE /C rmdir “”%ProgramFiles(x86)%\OCS Inventory Agent”” /S /Q”,0,True
WshShell.Run “CMD.EXE /C rmdir “”%SystemDrive%\ocs-ng”” /S /Q”,0,True
End If
End Function
Function needFusionInstall ()
Dim Fusion
‘ install fusion if version is different or if not installed
needFusionInstall = False
On error resume next
Fusion = WshShell.RegRead(“HKEY\_LOCAL\_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\FusionInventory Agent\DisplayVersion”)
If err.number = 0 Then
‘ Verification on OS 32 Bits
If Fusion <> versionverification Then
needFusionInstall = True
Else
needFusionInstall = False
Return
End If
Else
‘ Verification on OS 64 Bits
On error resume next
Fusion = WshShell.RegRead(“HKEY\_LOCAL\_MACHINE\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\FusionInventory Agent\DisplayVersion”)
If err.number = 0 Then
If Fusion <> versionverification Then
needFusionInstall = True
End if
Else
needFusionInstall = True
End If
End If
End Function
”’ MAIN
Dim WshShell
Set WshShell = Wscript.CreateObject(“Wscript.shell”)
If uninstallocsagent = “yes” Then
removeOCS()
End If
If needFusionInstall() Then
If (isHttp(fusionsetupURL)) Then
SaveWebBinary(fusionsetupURL)
WshShell.Run “CMD.EXE /C %TEMP%\fusioninventory.exe ” & fusionarguments,0,True
Else
WshShell.Run “CMD.EXE /C “”” & fusionsetupURL & “”” ” & fusionarguments,0,True
End If
End If```
не забываем указать в fusionargument, fusionsetupURL актуальные для вас данные, радостные потираем потные ладошки и с чувством выполненного долга отправляемся к ближайшей стойке за холодным жидким хлебом.
[![glpi + fusion inventory](/wp-content/uploads/2011/08/Снимок-экрана-2011-08-29-в-15.18.22-300x135.png "glpi + fusion inventory")][1]
PS. [Как исправить не кошерную кодировку в отчетах для GLPI + FusionInventory](/2011/08/codepage-pdf-csv-error-in-glpi/ "/2011/08/codepage-pdf-csv-error-in-glpi/").
