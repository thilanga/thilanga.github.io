---
layout: post
title: How to Setup smstools in debian
date: 2011-12-30
comments: true
tags: 3G debian linux smstools
---

3G dongle is the only bandwidth service I have when i'm home. (If my neighbor didn't turn on his WIFI connection ).
Due to heavy usage of mobile broadband, my 3g Dongle frequently get caught to service disruptions by exceeding the reserved data bundle,
Most of the time I had to call 3G service provider to get the connection temporary to do an online payment.

Normally service provider send a SMS when reaching the reserved quota (Probably when reaching 75%).
But as I can't use Mobile Partner software in linux, I always missed those SMSs, if I didn't connect my dongle in to a Windows powered machine
(As I hardly use Windows OS, most of the time I was in trouble).

As a solution I installed and configured smstools to avoid this mess and getting alone without the Internet.

###Install smstools
>Install the package

```bash
    sudo apt-get install smstools
```
>Find the device. ex: /dev/ttyUSB0

```bash
    dmesg | grep usb
```

###Configuration (/etc/smsd.conf)

>Change the smstools config file according to your service provider

```bash
    sudo vim /etc/smsd.conf
```

```
    devices = GSM1
    outgoing = /var/spool/sms/outgoing
    checked = /var/spool/sms/checked
    incoming = /var/spool/sms/incoming
    logfile = /var/log/smstools/smsd.log
    infofile = /var/run/smstools/smsd.working
    pidfile = /var/run/smstools/smsd.pid
    outgoing = /var/spool/sms/outgoing
    checked = /var/spool/sms/checked
    failed = /var/spool/sms/failed
    incoming = /var/spool/sms/incoming
    sent = /var/spool/sms/sent
    receive_before_send = no
    autosplit = 3

    [GSM1]
    device = /dev/ttyUSB0
    incoming = yes
    baudrate = 19200
    memory_start = 1
```

>Start smstools after changing configurations

```bash
    sudo /etc/init.d/smstools start ()
```
>To check incoming sms

```bash
    cd  /var/spool/sms/incoming
```

If you configured the smstools properly you will get sms to  `/var/spool/sms/incoming`

###Test your settings

Send a test sms to your 3G Dongle from a mobile phone, then
```bash
    ls /var/spool/sms/incoming
```
>If the dongle received the sms, you will see a file, name is slimier to `GSM1.AuvV6s`.
To read the sms

```bash
    vim  /var/spool/sms/incoming/$filename
```
###Debug your setings

```bash
    sudo tail -f /var/log/smstools/smsd.log
```
>Note: I didn't test sms sending through the 3G dongle as my service provider has blocked the facility.