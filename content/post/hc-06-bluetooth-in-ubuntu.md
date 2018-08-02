---
title: "HC 06 Bluetooth in Ubuntu"
date: 2018-08-02T12:09:56+01:00
draft: false
index: true
tags: ["arduino", "bluetooth", "hc-06"]
categories: ["embedded-system", "linux"]
comments: true
highlight: true
---

In this article we going see how to connect HC-06 Bluetooth in Ubuntu, which is commonly using in embedded systems such arduino. But this time we will see how to use it with linux in order to interact with bluetooth channels.

<!--more-->

So first thing we need to pair this device with ubuntu using the Bluetooth GUI in Ubuntu .

![alt text](/img/hc-06-bluetooth-in-ubuntu/hc-06.png "HC-06")

Select the device and make sure you set up the Pin Option to '1234'.

Then, Open Terminal and make sure that you can see the device !

```shell
$ hcitool scan
```

Now go head and edit this file :  `/etc/bluetooth/rfcomm.conf`

```shell
$ sudo vi /etc/bluetooth/rfcomm.conf
```

uncomment and change it to :

```

    rfcomm0 {
        # Automatically bind the device at startup
        bind no;

        # Bluetooth address of the device
        device 98:D3:31:80:51:48;

        # RFCOMM channel for the connection
        channel    1;

        # Description of the connection
        comment "HC-06";
    }
```

Make sure you change the device to you're HC-06 Module Address !.

Finally, bind the device with :

```shell
$ sudo rfcomm bind rfcomm0
```

Then use minicom to communicate with the modue in serial !

```shell
$ sudo minicom -D /dev/rfcomm0 -b 9600 -8
```

and don't forget to release the device when you finishing

```shell
$ sudo rfcomm release rfcomm0
```
