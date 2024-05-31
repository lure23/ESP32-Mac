# ESP32 on macOS - w/ Rust

Instructions on how to set up an ESP32 Rust development toolchain, using macOS host (for IDE) and Multipass VM's (other tooling).

This repo builds upon [ESP32 on WSL](https://github.com/lure23/ESP32-WSL) (working on Windows and WSL). However, that one sets up `esp-idf`, a C/C++ toolchain, whereas we are concerned about Rust, and [Embassy](https://embassy.dev) in particular.

>*<small>Note: Sandboxed development is definitely possible (and should be easy) by using virtualization software such as VirtualBox, Parallels and so on. We rule those out, because.. just because. The author is keen to know, whether Multipass can be used for embedded development.</small>*

## Aim 🍏⛓️

- Minimal software installs on the host (Mac)
- Ability to flash the device from within Multipass VM
- RISC V only <!--#whisper Unless you want to contribute as a co-host of the repo, maintaining the Xtensa architecture.-->

**Further aim**

- Ability to develop with a host-side IDE
	- ..including single-stepping etc. debugging

<!-- tbd. eventually, a schematics picture on how it all falls in
- USB server (usbipd, driver)
- host (IDE)
- Multipass (Rust, Cargo)
- folder sharing
-->

## Requirements

- Mac with
   - [Multipass](https://multipass.run) installed
   - `rust-emb` development VM running from [`mp`](https://github.com/akauppi/mp) (GitHub)

- Windows computer (Win10 Home is enough)
   - [`usbipd-win`](https://github.com/dorssel/usbipd-win) 4.2.0 (or later) installed

   >Note: A Linux computer would do just as well, and the author would like to *eventually* provide guidance on using a Raspberry-Pi for this USB hosting. You can probably set such up on your own, with basic Linux knowledge and adapting from this guide.

   - `CP210x universal Windows driver` (11.3.0) installed from [CP210x USB to UART Bridge VCP Drivers](https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers) (SiLabs)

>For instructions on setting up USB hosting on the Windows side, see [ESP32-WSL](https://github.com/lure23/ESP32-WSL?tab=readme-ov-file#steps-set-up-usb-ip-bridging) (GitHub).

**Embedded hardware**

- [ESP32-C3-DevKitC-02](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/hw-reference/esp32c3/user-guide-devkitc-02.html)

   - Cables (USB-A to USB-micro) to connect the board to your PC.

   You'll likely want to have some breadboard, LEDs, resistors, sensors.. etc. but let's set up the Rust dev environment, first! 😊


---

**Alternatives**

If you don't want to use a separate PC/Raspberry Pi for USB hosting, but are ready to connect the ESP32 devkit directly to your mac, there are [some instructions](...). <font color=orange>*tbd.*</font>

>TL;DR: You'll need to:
- install `CP210x` driver on the Mac
- run a USB/IP server on the Mac (*not* available as open source) **or** use a virtualization product

<p/>

>Note that physically separating the devkit from your (main?) computer has the added benefit that if something goes haywire with - say - your solderings, you are not jeopardising your main computer. This is why the author prefers this two-machine solution.


### Warning!!

When plugging the ESP32-C3-DevKitC-02 development board for the first time, <font color=darkaqua>**BEWARE OF THE STRONG LED LIGHT!!**</font> The author used a tape on top, until he reflashed the device. 😎🩹

<!-- 
Developed on:

Mac:
- macOS 14.5
- (CP210x device driver v. 6.0.2 optional; if running on single computer)
- Multipass 1.13.1

Windows:
- Windows 10 Home
   - CP210x universal Windows driver (11.3.0)
- usbipd-win v. 4.2.0
- WSL version 2.1.5.0 (> wsl --version); Ubuntu 22.04.4 LTS ($ lsb_release -a)
-->


## Connecting the device to Multipass

We presume you have the USB-to-IP server running, and know the IP of the Windows machine.

If so, within the `rust-emb` VM (See [`mp`](https://github.com/akauppi/mp) > `rust+emb`):

1. Check that you can see the devkit (optional)

   ```
   $ usbip list -r 192.168.1.29
   Exportable USB devices
   ======================
    - 192.168.1.29
           3-1: Silicon Labs : CP210x UART Bridge (10c4:ea60)
              : USB\VID_10C4&PID_EA60\BC2F214F809DED11AAFA5F84E259FB3E
              : (Defined at Interface level) (00/00/00)
              :  0 - Vendor Specific Class / unknown subclass / unknown protocol (ff/00/00)
   ```
   
2. Attach the devkit to your VM

   ```
   $ sudo usbip attach -r 192.168.1.29 -b 3-1
   ```

3. Now you should see it in the device tree:

   ```   
   $ lsusb
   Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
   Bus 001 Device 009: ID 10c4:ea60 Silicon Labs CP210x UART Bridge
                           ^^^^ ^^^^
	...                           
   ```

	Note the `10c4:ea60` (highlighted). They are the "vendor id" and "device id" designators.

## Using `probe-rs`

...


<!-- revise... tbd.
## Outcome

We have set up the development tools fully within Multipass. On the Mac side, there's nothing that needs to be installed.

Unfortunately, we weren't able to get the `usbipd` server running properly on macOS, but that may change over the years. Also - you may look into commercial or closed source solutions, if you don't wish to pitch in the extra PC.

The author is rather pleased about the arrangement, though. This is enough to let him proceed with the rest of the toolchain setup - see [EmbeddedRover](https://github.com/lure23/EmbeddedRover) for more.

And - there's still 10 minutes of 2023 remaining. 

Happy 🎉
New Year
Everyone!

Make great things!

-->