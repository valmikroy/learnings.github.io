---
layout: post
title: Raspberry Pi quick configuration 
description: "Notes around home Raspberry Pi configurations"
categories: [linux]
tags: [Raspberry, linux]
Comments: true
---



# Raspberry Pi quick configuration 

Notes around home Raspberry Pi configurations 



## Table of Contents

* Kramdown table of contents
{:toc .toc}






### Bring up USB WiFi interface 

- Know the USB device

  ```shell
  $ lsusb
  Bus 001 Device 005: ID 04f9:0061 Brother Industries, Ltd
  Bus 001 Device 004: ID 0bda:ffef Realtek Semiconductor Corp.
  Bus 001 Device 003: ID 0424:ec00 Standard Microsystems Corp. SMSC9512/9514 Fast Ethernet Adapter
  Bus 001 Device 002: ID 0424:9512 Standard Microsystems Corp. SMC9512/9514 USB Hub
  Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
  ```

  ```
  $ dmesg |grep usb
  <snip/>
  
  [    3.169119] smsc95xx 1-1.1:1.0 eth0: register 'smsc95xx' at usb-20980000.usb-1.1, smsc95xx USB 2.0 Ethernet, b8:27:eb:25:9d:b2
  [    3.283487] usb 1-1.2: new high-speed USB device number 4 using dwc_otg
  [    3.424383] usb 1-1.2: New USB device found, idVendor=0bda, idProduct=ffef, bcdDevice= 0.00
  [    3.439464] usb 1-1.2: New USB device strings: Mfr=1, Product=2, SerialNumber=3
  [    3.450304] usb 1-1.2: Product: 802.11n NIC
  [    3.457961] usb 1-1.2: Manufacturer: Realtek
  [    3.465623] usb 1-1.2: SerialNumber: 00E04C0001
  [    3.573436] usb 1-1.3: new high-speed USB device number 5 using dwc_otg
  
  <snip/>
  
  
  ```

- WPA configuration 

  ```shell
  country=US
  ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
  update_config=1
  network={
  ssid="My home ssid"
  psk="blah blah"
  scan_ssid=1
  }
  ```

- Setup interface configuration 

  ```shell
  $ cat /etc/network/interfaces.d/interfaces
  auto lo
  iface lo inet loopback
  
  auto eth0
  allow-hotplug eth0
  iface eth0 inet dhcp
  
  auto wlan0
  allow-hotplug wlan0
  iface wlan0 inet dhcp
  pre-up wpa_supplicant  -i wlan0 -Dnl80211,wext -c /etc/wpa_supplicant/wpa_supplicant.conf -B
  ```

- Restart





