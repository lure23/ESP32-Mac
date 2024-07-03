# Setting up `usbipd` on Raspberry Pi

Instructions on how to set up USB/IP server, on a Raspberry Pi.

Based on:

- ["How to Setup a Raspberry Pi as a USB-over-IP Server - A Comprehensive Guide"](https://fleetstack.io/blog/how-to-setup-a-raspberry-pi-as-a-usb-over-ip-server-a-comprehensive-guide) (Jul '23)

---

## On the RPi

```
$ sudo apt install usbip
```

>Note: On Linux, the USB/IP daemon (`usbipd`) is installed together with the client.

```
$ lsusb
Bus 001 Device 005: ID 303a:1001 Espressif USB JTAG/serial debug unit
...
```

```
$ usbip list --local
 - busid 1-1.1 (0424:ec00)
   Microchip Technology, Inc. (formerly SMSC) : SMSC9512/9514 Fast Ethernet Adapter (0424:ec00)

 - busid 1-1.4 (303a:1001)
   unknown vendor : unknown product (303a:1001)
```   

The `1-1.4` is the "bus id" that we need to identify the device.

```
$ sudo usbip bind -b 1-1.4
usbip: info: bind device on busid 1-1.4: complete
```

```
$ sudo usbipd -D
```

## On the client (VM)

```
$ usbip list -r 192.168.1.199
Exportable USB devices
======================
 - 192.168.1.199
      1-1.4: unknown vendor : unknown product (303a:1001)
           : /sys/devices/platform/soc/3f980000.usb/usb1/1-1/1-1.4
           : Miscellaneous Device / ? / Interface Association (ef/02/01)
```

>Optionally, if you have updated the kernel, you might need to:
>
>```
>$ sudo apt install -y linux-tools-generic linux-modules-extra-$(uname -r)
>```
>
>```
>$ sudo modprobe vhci-hcd
>```

```
$ sudo usbip attach -r 192.168.1.199 -b 1-1.4
```

```
$ lsusb
[...]
Bus 001 Device 002: ID 303a:1001 Espressif USB JTAG/serial debug unit
```

## Test

Prove that everything works:

```
$ espflash board-info
✔ Use serial port '/dev/ttyACM0' - USB JTAG/serial debug unit? · yes
✔ Remember this serial port for future use? · no
[2024-07-03T06:37:51Z INFO ] Serial port: '/dev/ttyACM0'
[2024-07-03T06:37:51Z INFO ] Connecting...
[2024-07-03T06:37:52Z INFO ] Using flash stub
Chip type:         esp32c6 (revision v0.0)
Crystal frequency: 40 MHz
Flash size:        4MB
Features:          WiFi 6, BT 5
MAC address:       54:32:04:07:15:10
```

## More...

If you wish the device to be automatically attached after, say, a reboot of the RPi, some more steps may be needed. <!-- tbd. document, but also consider moving these to `docs/{win|rpi|mac}_usbipd.md`. -->
