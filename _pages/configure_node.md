---
layout: single
classes: wide
title: Configure your bitcoin payment node in 5 steps 
permalink: /configure_node/
toc: true
toc_icon: "cog"
---

## 1. Get hardware and software

Buy and assemble hardware:

* Raspberry Pi 4 Computer, Model B 4GB RAM
* 16 Gb microSD
* 1 Tb SSD
* USB to SATA or NvME adapter
* Proper heat dissipation

Install [Raspberry Pi Imager](https://www.raspberrypi.com/software/), as shown in the video tutorial.

{% include video id="LlN1gqM70cQ" provider="youtube" %}

## 2. Flash the Raspberry Pi firmware to boot from USB

The Raspberry Pi OS will boot directly from the SSD drive. 
It's necessary to configure the Raspberry Pi device to boot from USB instead of the MicroSD.
This can be done by flashing the Raspberry Pi firmware with a special distribution that can be found in Raspberry Pi Imager, and must be flashed to the MicroSD. Insert the MicroSD in the Raspberry Pi and boot!

{% include video id="LxKIxr9Vt80" provider="youtube" %}

## 3. Prepare the Microsd for backups

The MicroSD can be used to contain backups that are very useful to restore the device in case of failures. Format the MicroSD.

{% include video id="iK3RrAjYMKU" provider="youtube" %}

## 4. Prepare the SSD with the operating system

Now you can install the operating system to the SSD. You can use our custom operating system image, that we have prepared for you.

* [Download the custom Raspberry Pi OS from here (14GB)](https://mega.nz/file/xtdVSSBL#kErbuXf8ZOrlSmYZpoBfMnr56msQCEkrBRgY_HbIdQI).
* Verify that SHA256 of the downloaded file is BBD593D3F5157F0BE1DA6108F2FBBA6013BE8B2B0188EC3243F6685D8014703D

Lean how to flash a custom image in the video tutorial.

{% include video id="StpabPdHMHs" provider="youtube" %}

If you prefer to not use the ready-to-go image, you can also read the (heavy) technical procedure which was used to create the image. Learn how to configure [Raspberry Pi OS full disk encryption from scratch](/configure_fde).

## 5. Run the setup!!

You are almost done.

1. Connect your Raspberry Pi to a display and start. You will be asked to unlock the device with a password. Enter **start123** as the default password, and do not forget to change it later.
2. Connect your Raspberry Pi to the internet, via Wi-Fi or Ethernet.
3. Run the setup and wait for the configuration to complete. The setup can take around 30 minutes to complete, depending on your network connection.
4. Reboot and you are ready to go.

{% include video id="pHUML0avgw8" provider="youtube" %}
