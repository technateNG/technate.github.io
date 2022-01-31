---
layout: post
title:  "Read temperature of cores on Intel processor"
date:   2022-01-31 01:00:00 +0000
categories: perf
---
First we need to have `rdmsr` utility.  
We can download it from package manager of our Linux distribution.  
In Arch Linux case it's:
```
sudo pacman -S msr-tools
```
Next we need to get value of _IA32_THERM_STATUS_ msr.  
We can find this register under 0x19c address.  
We also need to have root permissions.
```
sudo rdmsr -a -c 0x19c
0x884a0000
...truncated...
```
Now to get actual temperature of cores in Celsius we need to shift value to right by 2 bytes (16 bits) and mask first byte.
Then you need to subtract value from 100. 
This is equation: 
```
100 - ((value_from_rdmsr >> 16) & 0xff)
```
and for example in python:
```
>>> 100 - ((0x884a0000 >> 16) & 0xff)
26
```
