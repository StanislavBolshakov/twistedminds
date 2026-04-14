---
title: Настраиваем trunk между коммутаторами Cisco Catalyst и 3Com
author: ["Stanislav"]
date: 2012-05-22T13:12:57+00:00
url: /2012/05/trunk-between-cisco-and-3com/
categories:
  - Tech
tags:
  - 3com
  - catalyst
  - cisco
  - trunk
  - vlan

---
В роли Cisco будет Catalyst 4948, в роли 3Com &#8211; Switch 4210.

Просто настроить trunk было бы слишком легко, по-этому мы усложним себе задачу и сменим management vlan для 3Com. Начнем с того, что надо найти консольчик к 3Com, так как без этого ничего не получится :)

Соединяемся (19200,8,N,1,N) с 3Com и, поехали:

<!--more-->

```  
#заходим в привилегированный режим  
system-view  
#удаляем текущий интерфейс vlan 1 и управляющий vlan  
[SAN-mgmt-A]undo interface Vlan-interface 1  
[SAN-mgmt-A]undo management-vlan  
#создаем новый vlan и даем ему имя  
[SAN-mgmt-A]vlan 99  
[SAN-mgmt-A-vlan99]name IP mgmt  
[SAN-mgmt-A-vlan99]quit  
#задаем management vlan  
[SAN-mgmt-A]management-vlan 99  
#задаем ip и маршрут по умолчанию  
[SAN-mgmt-A]interface Vlan-interface 99  
[SAN-mgmt-A-Vlan-interface99]ip address 172.16.99.12 255.255.255.0  
[SAN-mgmt-A-Vlan-interface99]quit  
[SAN-mgmt-A]ip route-static 0.0.0.0 0.0.0.0 172.16.99.1  
#добавим еще VLAN и создадим на 26 порту транк и сделаем порты с 1 по 12 accses для vlan 152  
[SAN-mgmt-A]vlan 152  
[SAN-mgmt-A-vlan152]name SAN mgmt  
[SAN-mgmt-A-vlan152]port Ethernet 1/0/1 to Ethernet 1/0/12  
[SAN-mgmt-A-vlan152]quit  
[SAN-mgmt-A]interface GigabitEthernet 1/0/26  
[SAN-mgmt-A-GigabitEthernet1/0/26]port link-type trunk  
[SAN-mgmt-A-GigabitEthernet1/0/26]port trunk permit vlan 99  
Please wait&#8230; Done.  
[SAN-mgmt-A-GigabitEthernet1/0/26]port trunk permit vlan 152  
Please wait&#8230; Done.  
#проверим конфигурацию  
[SAN-mgmt-A]display vlan all  
```

Cisco:  
```  
CoreSwitchA(config)#int gi1/48  
CoreSwitchA(config-if)#switchport mode trunk  
CoreSwitchA(config-if)#switchport trunk allowed vlan 99,152  
CoreSwitchA(config-if)#description trunk to SAN-mgmt-A  
```

```  
CoreSwitchA#ping 172.16.99.12

Type escape sequence to abort.  
Sending 5, 100-byte ICMP Echos to 172.16.99.12, timeout is 2 seconds:  
.!!!!  
Success rate is 80 percent (4/5), round-trip min/avg/max = 1/8/12 ms  
```

Уня-ня.