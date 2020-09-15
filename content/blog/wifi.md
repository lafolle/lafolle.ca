---
title: "Wifi"
date: 2020-09-05T15:26:36-04:00
draft: true
---

Was unable to capture traffic for my wifi using `airport sniff `.  It turns out my wifi is on channel 44 and not 1.

airport sniff 44 # works

Beacon frame - 

`wlan.fc.type_subtype != 0x08 && wlan.ssid == "Waves in Space-5G" && wlan.ta == e4:76:84:ce:61:0d`
