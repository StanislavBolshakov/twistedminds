---
title: Cisco и неподдерживаемые SFP модули
author: ["Stanislav"]
date: 2011-02-18T07:27:09+00:00
url: /2011/02/cisco-unsupported-sfp-module/
no_lj:
  - 1
categories:
  - Tech
tags:
  - cisco
  - sfp
  - решаем проблемы

---
Захотелось подружить Cisco 4948 c 122-54.SG и древний Picolight SFP модуль, по злой рок в виде

```C4K_CHASSIS-3-SFPINTEGRITYCHECKFAILED:SFP```

не давал этим двум славным железкам слиться в экстазе. Подружить негодников удалось 3мя волшебными командами:

```  
service unsupported-transceiver  
no errdisable detect cause gbic-invalid  
errdisable recovery cause gbic-invalid  
```

и, аллелуя!

<!--more-->

```

CoreSwitchA#show interfaces gi1/45 controller  
GigabitEthernet1/45 is up, line protocol is up (connected)  
Hardware is Gigabit Ethernet Port, address is 0024.9760.372c (bia 0024.9760.372c)  
MTU 1500 bytes, BW 1000000 Kbit, DLY 10 usec,  
reliability 255/255, txload 1/255, rxload 1/255  
Encapsulation ARPA, loopback not set  
Keepalive set (10 sec)  
Full-duplex, 1000Mb/s, link type is auto, media type is 1000-DWDM-38.19  
Media-type configured as SFP connector  
```  
```

CoreSwitchA#show idprom interface gi1/45

GBIC Serial EEPROM Contents:  
Common Block:  
Identifier = Not available [0x3]  
Extended Id = GBIC is compliant with MOD_DEF 4 [0x4]  
Connector = LC Connector [0x7]  
Transceiver  
Type = Gbic DWDM-3819  
Speed(FC,byte 10)= 100 MBytes/Sec 200 MBytes/Sec [0x5]  
Media = Multi-mode, 50u (M5) Multi-mode, 62.5u (M6) [0xC]  
Technology = Shortwave laser w/o OFC (SN) [0x40]  
Link Length = Intermediate Distance [0x20]  
GE Comp Codes = Not available [0x0]  
SONET Comp Codes = Not available [0x0]  
10GE Comp Codes = Extended [0x0]  
Encoding = 8B10B [0x1]  
BR, Nominal = 2100 Mbps (million bits per second)  
Length(9u) in km = GBIC does not support single mode fibre, or the length  
must be determined from the transceiver technology.  
Length(9u) = GBIC does not support single mode fibre, or the length  
must be determined from the transceiver technology.  
Length(50u) = 300 m  
Length(62.5u) = 150 m  
Length(Copper) = GBIC does not support copper cables, or the length must  
be determined from the transceiver technology.  
Vendor name = PICOLIGHT  
Vendor OUI = 1157  
Vendor Part No. = PL-XPL-00-S23-24  
Vendor Part Rev. = Unspecified  
Wavelength = 850 nanometers  
CC_BASE = 0x65

Extended ID Fields  
Options = Loss of Signal implemented TX_FAULT signal implemented TX_DISABLE is implemented and disables the serial output [0x1A]  
BR, max = 4%  
BR, min = 52%  
Vendor Serial No. = 231CL0QB  
Date code = 020809  
Diag monitoring = Not implemented  
Rx pwr measuremnt = Optical Modulation Amplitude (OMA)  
Address change = Not required  
CC_EXT = 0xAD  
```