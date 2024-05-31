>**This moved aside for now; proceed with caution!**


## Alternative (a) - Connecting the device to Mac

```
           .----------------------.
ESP-32 <-- | Mac                  |
  ^        |   '-- Multipass VM   |
  |        |          '-- ESP-IDF |
  |        `--------------- | ----'
  '--- communicate with the device
```

**Required**

- Mac
   - Apple Developer Command Line Tools installed (provides `git`):
      
      `$ xcode-select --install`

- [Multipass](https://multipass.run) installed
- willingness to install software (driver; Rust toolchain) on the Mac
- roughly 10 GB of disk space

In this alternative, you will:

- set up an ESP-IDF development toolchain, within Multipass VM
- install a device driver on your Mac, to be able to communicate with the ESP32 development board
- install Rust/Cargo and a `usbipd` server software *on the host* (Mac)

You will be able to do *embedded* ESP32 development, solely within the Mac ecosystem. :)

## Step 1- set up USB-IP

By bridging the USB device from your host (Mac) computer to the client (Ubuntu, within Multipass), we are able to do all the development actions within the Linux side of things.

This means no need to install ESP32-specific tools on your host account.


### Install CP210x bridge driver

Most ESP32 development boards use a Silicon Labs USB-to-UART chip, to allow communicating with the development board. A device driver for this chip needs to be installed on the *host* side - otherwise you won't be able to communicate with the board.

- Visit Silicon Labs > Developers > [CP210x USB to UART Bridge VCP Drivers](https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers) 
   - Download the latest `CP210x VCP Mac OSX Driver` (6.0.2 at the time of writing)
   - unzip, launch the `.dmg` and follow the instructions

In particular, when asked, open the System Settings and press `Allow` to allow the particular extension to run:

![](.images/system-extension.png)


After installation, you should be able to see device information in the device tree (`System Preferences` > `General` > `Information` <!-- tbd. or about; what in English?--> > `System report...` > `USB`):

![](.images/driver-tree.png)


### Install Rust and run `usbipd` server

The author was able to find only a single open source `usbipd` server that would work for macOS: [`jiegec/usbip`](https://github.com/jiegec/usbip) (GitHub).

To run it, you will need to have Rust and Cargo on your system.

>Note: You can compile the binary elsewhere and just pass it on to the Macs needing it.

1. Install Rust and Cargo

   Visit [https://rustup.rs](https://rustup.rs) and follow the instructions:
   
   ```
   $ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   ```
   
   >Note: running shell scripts like this *blind* 🙈 from the Internet is exactly why the author prefers the Alternative B. 🤕

   >Test Rust with:
   >
   >```
   >$ . ~/.cargo/env
   >$ cargo --version
   >cargo 1.74.1 (ecb9851af 2023-10-18)
   >```

2. Clone `usbipd` and run it

   In some suitable folder: 

   ```
   $ git clone git@github.com:jiegec/usbip.git
   [...]
   
   $ cd usbip
   
   $ env RUST_LOG=info cargo run --example host
   Updating crates.io index
   Downloaded proc-macro2 v1.0.72
   Downloaded quote v1.0.34
   [...]
   Finished dev [unoptimized + debuginfo] target(s) in 0.49s
     Running `target/debug/examples/host`
   ...     
   ```

   Leave the process running. It's now bridging USB devices to IP networking.   

   >Note: Different from `usbip-win`, there is no need for `usbip bind`. When a device is requested by a client, the process will grant access to it.

#### Host IP

You'll need the host's IP, soon. Check it from the System Preferences, Option-clicking the WLAN icon on the toolbar, or in a (new) terminal:

```
$ ipconfig getifaddr en0
192.168.1.62
```

This is the IP we'll reach from Multipass, to bind the USB development board.


### Set up a Multipass instance

You might need more than the default 5GB of disk space, so let's set up a separate Multipass (Ubuntu) instance for [ESP-IDF](https://github.com/espressif/esp-idf) (GitHub).

```
$ export NAME=esp-idf

$ multipass launch lts --name $NAME --memory 4G --disk 8G --cpus 2
Starting esp-idf |
[...]
Launched: esp-idf

$ multipass shell $NAME
[...]
ubuntu@esp-idf:~$ 
```

This dives into the Ubuntu instance (Linux).

>Hint: You might want to change the style of the current terminals to highlight it's not in macOS, any more (`Command prompt` > `Inspector`).

<p />

>#### Regular maintenance (optional)
>
>```
>@esp-idf:~$ sudo apt-get update
>[...]
>@esp-idf:~$ sudo apt-get upgrade
>[...]
>```

### Reach for the USB(IP)

Within Multipass shell (launched above):

- Install `usbip` client

   ```
   ~$ sudo apt-get install linux-tools-generic
   ```

- List host devices:

   ```
   ~$ usbip list -r 192.168.1.62
   Exportable USB devices
   ======================
    - 192.168.1.62
       20-52-6: Silicon Labs : CP210x UART Bridge (10c4:ea60)
              : /sys/bus/20/52/6
              : (Defined at Interface level) (00/00/00)
              :  0 - Vendor Specific Class / unknown subclass / unknown protocol (ff/00/00)
   [...]
   ```

   Seems we can see it. `20-52-6` is the "bus id"; keep that in mind.

   >Note: If you don't get a response, check that macOS is still running the `usbip` server.

- Attach

   ```
   ~$ usbip attach -r 192.168.1.62 -b 20-52-6
   libusbip: error: udev_device_new_from_subsystem_sysname failed
   usbip: error: open vhci_driver
   ```

   So close!  This fails, because there is no `vhci-hcd` driver in the default Multipass Ubuntu image. Let's fix that!

   >The [`[1]`](https://askubuntu.com/questions/1303403/how-to-install-usbip-vhci-hcd-drivers-on-an-aws-ec2-ubuntu-kernel-version) reference points at installing `linux-image-extra-virtual` (or compiling the module from sources), but we only need one dependency of the larger package.

   For this, hop to `root` mode:

   ```
   ~$ sudo bash
   # uname -r 
   5.15.0-91-generic

   # apt install linux-modules-extra-5.15.0-91-generic

   # ls /lib/modules/5.15.0-91-generic/kernel/drivers/usb/usbip/
usbip-core.ko  usbip-host.ko  usbip-vudc.ko  vhci-hcd.ko
   ```

   We now have the `vhci-hcd.ko` and `usbip-core.ko` drivers, for the active kernel. Let's retry connecting!

   ```   
   # exit
   ```
   
   ```
   ~$ sudo modprobe vhci-hcd
   ~$ sudo usbip attach -r 192.168.1.62 -b 20-52-6
   ```

   That does it!

   ```
   $ lsusb
   Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
   Bus 001 Device 002: ID 10c4:ea60 Silicon Labs CP210x UART Bridge
   Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
   ```   

Now that we can reach the development board from within the Multipass instance, let's set up the actual toolchain.

>Note: If you feel this has been elaborate, another approach is to set up the USBIP host on a Windows computer. Once `usbip` is running, it doesn't matter where the device is physically connected - as long as they are in the same network.
>
>The author prefers using the Windows approach, since there's less installs on the main developer account.


The rest of the steps are the same for both alternatives.


<!-- disabled; no longer in focus for this repo (remove)
## Set up ESP-IDF

All the following commands happen within the Multipass (Linux) environment.

Follow the instructions on [Standard Toolchain Setup for Linux and macOS](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/get-started/linux-macos-setup.html) (Espressif docs):

- Install the pre-requisites:

   ```
   $ sudo apt-get install git wget flex bison gperf python3 python3-pip python3-venv cmake ninja-build ccache libffi-dev libssl-dev dfu-util libusb-1.0-0
   ```

- Create an installation folder

   >Note: Espressif advocates installing to `~/esp/esp-idf` and we follow it. You may choose another path; just be consistent.
   >
   >Important: The `esp-idf` is a shallow copy git repo, and the way to do updates is simply to **re-install it from `git clone`.**

   ```
   $ mkdir -p ~/esp
   $ cd ~/esp
   $ git clone --depth 1 --recursive https://github.com/espressif/esp-idf.git
	```

   While that command runs, a bit about the ESP-IDF install ideology. All you need is a snapshot of their repo, and the recommended way to upgrade is just to reinstall everything. We added the `--depth 1` to the command to reduce downloads, but it's still a hefty package.
   
   <!_-- #contribute If you know a way how we can "clone" just the files, including "--recursive", please share the command. 
   --_>
   
   Once everything's set up, you source the `export.sh` file that sets up environment variables. Note: don't do that in a `.bashrc` - it takes some seconds and prints things on the terminal. But... we're getting ahead of things.

   The clone is probably complete, by now? :)

	>Optional: remove git history, to save ~700MB
	>
   >```
   >$ rm -rf esp-idf/.git
   >```
   
   ```
   $ cd esp-idf
   $ ./install.sh esp32c3
   ```

   Here, you'll add the kinds of development board support that you need / have access to.

   We're now all set. Let's get back to the home directory.
   
   ```
   $ cd
   ```

### Activating the ESP-IDF toolchain

   Before using the toolchain in a sample project, as the last output likely showed, you need to run one shell script.
   
   ```
   $ . ~/esp/esp-idf/export.sh
   [...]
   Done! You can now compile ESP-IDF projects.
   Go to the project directory and run:
   
     idf.py build
   
   ```

   It adds environment variables to your current shell, so `idf.py` (the build command) can be used.
	
   >Hint: You'll need to rerun this every time a new shell is being used, so might be useful to make an alias for it.
   >
   >The author has `alias idf-init=". ~/esp/esp-idf/export.sh"` in `~/.bashrc`.

   
## Sample project

Note: Remember that you have the ESP-IDF environment variables set up (as mentioned above):

```
$ . ~/esp/esp-idf/export.sh
```

---

Pick a folder where you'd have a sample project. Let's say `~/my`.

```
$ install -d my
$ cd my
```

Copy a sample project to this folder

```
$ cp -r $IDF_PATH/examples/get-started/hello_world hello
```

```
$ cd hello
```

>Optional: Change something in the sources, eg. "Hello world" to - say - "Hello ESP32". `nano main/hello_world_main.c`

### Configuration

```
$ idf.py set-target esp32c3
```

This already creates quite a lot of files under `build/`. Bootloaders and such - have a look.

>Likely `idf.py set-target` is a one-time command.

<p />

>Optional: You can try `idf.py menuconfig` that allows all kinds of project settings to be changed. We are good with the defaults.


### Build and flash

```
$ idf.py build
```

This builds the three application files under `build/`:

```
-rw-rw-r--  1 ubuntu ubuntu  184240 Dec 30 19:08 hello_world.bin
-rwxrwxr-x  1 ubuntu ubuntu 3776388 Dec 30 19:08 hello_world.elf
-rw-rw-r--  1 ubuntu ubuntu 3060919 Dec 30 19:08 hello_world.map
```

>Note: The files look rather big, but this is a debug build.

```
$ idf.py flash
```

---

‼️PROBLEM!!

I *think* I had this working, earlier, but when writing this, it gives an error:

```
$ idf.py flash
Executing action: flash
Serial port /dev/ttyUSB0
Connecting.......................
/dev/ttyUSB0 failed to connect: Failed to connect to Espressif device: No serial data received.
For troubleshooting steps visit: https://docs.espressif.com/projects/esptool/en/latest/troubleshooting.html
Serial port /dev/ttyS0
/dev/ttyS0 failed to connect: Could not open /dev/ttyS0, the port is busy or doesn't exist.
([Errno 13] could not open port /dev/ttyS0: [Errno 13] Permission denied: '/dev/ttyS0')

Hint: Try to add user into dialout group: sudo usermod -a -G dialout $USER

No serial ports found. Connect a device, or use '-p PORT' option to set a specific port.
```

That's unfortunate. 

But it also highlights the dependency on a single, Rust-based `usbipd` server implementation - whose focus is not in providing a server, but in being a USB simulation tool.

IF YOU KNOW HOW TO FIX THE ALTERNATIVE A, LET ME KNOW.  Best, as a PR.

Thus, let's look at the Alternative B - attaching the ESP32 development board to a Windows PC and getting the `usbip` connection from there. 😀

---
-->


### Uninstalling the device driver

The Release Notes of the CP210x driver have this to say:

```
[...]
Uninstalling the Driver
-----------------------

	To uninstall the driver, run the `uninstaller.sh` shell script from
	a Terminal command prompt. 
```

If you didn't keep it around, here's a copy: ;)


<details><summary>`uninstaller.sh` for CP210x driver</summary>
```
#!/bin/bash

if [ -d /System/Library/Extensions/SiLabsUSBDriver.kext ]; then
sudo kextunload /System/Library/Extensions/SiLabsUSBDriver.kext
sudo rm -rf /System/Library/Extensions/SiLabsUSBDriver.kext
fi

if [ -d /System/Library/Extensions/SiLabsUSBDriver64.kext ]; then
sudo kextunload /System/Library/Extensions/SiLabsUSBDriver64.kext
sudo rm -rf /System/Library/Extensions/SiLabsUSBDriver64.kext
fi

if [ -d /Library/Extensions/SiLabsUSBDriverYos.kext ]; then
sudo kextunload /Library/Extensions/SiLabsUSBDriverYos.kext
sudo rm -rf /Library/Extensions/SiLabsUSBDriverYos.kext
fi

if [ -d /Library/Extensions/SiLabsUSBDriver.kext ]; then
sudo kextunload /Library/Extensions/SiLabsUSBDriver.kext
sudo rm -rf /Library/Extensions/SiLabsUSBDriver.kext
fi

if [ -d /Applications/CP210xVCPDriver.app/Contents/Library/SystemExtensions/com.silabs.cp210x.dext ]; then
/Applications/CP210xVCPDriver.app/Contents/MacOS/CP210xVCPDriver uninstall
sudo rm -rf /Applications/CP210xVCPDriver.app
fi
```
</details>
