# ESP32/Rust on macOS

Instructions on how to set up an ESP32 Rust development toolchain, using macOS host (for IDE) and a Multipass VM (other tooling).

The explicit (and only) target is [Embassy](https://embassy.dev) development.

>Note: This repo builds upon [ESP32 on WSL](https://github.com/lure23/ESP32-WSL) (Windows and WSL; esp-idf; C/C++)
>
>If you are interested in using WSL instead of Multipass, check that repo together with this. Also, you can attach the ESP32 dev board directly to the Mac, but this case isn't covered (yet). Let the author know if the arrangement of two repos feels uncomfortable.

<!-- tbd. eventually, a schematics picture on how it all falls in
- USB server (usbipd, driver)
- host (IDE)
- Multipass (Rust, Cargo)
- folder sharing
-->

## Aim 🍏⛓️

- Minimal software installs on the host (Mac)
- Ability to flash the device from within Multipass VM
- RISC V only 

   <!--#whisper If you want to contribute as a co-host, maintaining the Xtensa architecture, let the author know.-->

<!-- #later???
- Ability to develop with a host-side IDE
	- ..including debugging
-->

### No nightlies needed!

The repo only handles workflows on stable Rust. Needing to use `nightly` is unfortunately out of the scope.

>*Furthermore, for ESP32-C3, a nightly version of the Rust toolchain is currently required, for this training we will use nightly-2023-11-14 version.* <sub>[Embedded Rust (no_std) on Espressif](https://docs.esp-rs.org/no_std-training/02_2_software.html)

This might not hold any more, but the above text is there (2-Jun-24). Is it true? If so, C3 is ruled out.


## Requirements

- Mac with
   - [Multipass](https://multipass.run) installed

   <!-- disable?
   - `rust-emb` development VM running from [`mp`](https://github.com/akauppi/mp) (GitHub) -->

- Windows computer (Win10 Home is enough)
   - [`usbipd-win`](https://github.com/dorssel/usbipd-win) 4.2.0 (or later) installed

   >Note: A Linux computer would do just as well, and the author would like to *eventually* provide guidance on using a Raspberry-Pi for this USB hosting. You can probably set such up on your own, with basic Linux knowledge and adapting from this guide.

   - `CP210x universal Windows driver` (11.3.0) installed from [CP210x USB to UART Bridge VCP Drivers](https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers) (SiLabs)

	>*Note: This may not be needed for ESP32-C6 development, only ESP32-C3.*
	
	<!-- disable
	For instructions on setting up USB hosting on the Windows side, see [ESP32-WSL](https://github.com/lure23/ESP32-WSL?tab=readme-ov-file#steps-set-up-usb-ip-bridging) (GitHub).
	-->
	
### Embedded hardware

**Warning!!**

When powering the ESP32 boards for the first time, <font color=dark>**BEWARE OF THE STRONG LED LIGHT!!**</font> The author used a tape on top, until he reflashed the device. 😎🩹

|board|hw version|status|notes|
|---|---|---|---|
|**[ESP32-C3-DevKitC-02](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/hw-reference/esp32c3/user-guide-devkitc-02.html)**|1.1|<font color=red>no connection</font> with `probe-rs`|
|**[ESP32-C6-DevKitM-1](https://docs.espressif.com/projects/espressif-esp-dev-kits/en/latest/esp32c6/esp32-c6-devkitm-1/user_guide.html)**|1.0|<font color=orange>**WIP to have first contact tbd.**</font>|Connect USB cable to the port on the right (ports facing you); it has JTAG circuitry.|
|**[ESP32-C3-DevKit-RUST-1](https://www.espressif.com/en/dev-board/esp32-c3-devkit-rust-1-en)**|tbd.|Waiting for delivery *tbd.*|

*Hint: To check your hw version, look at the printing on the bottom side of the board.*

<!-- disabled: #later elsewhere? tbd.
**Alternatives**

If you don't want to use a separate PC/Raspberry Pi for USB hosting, but are ready to connect the ESP32 devkit directly to your mac, there are [some instructions](...). <font color=orange>*tbd.*</font>

>TL;DR: You'll need to:
- install `CP210x` driver on the Mac
- run a USB/IP server on the Mac (*not* available as open source) **or** use a virtualization product

<p/>

>Note that physically separating the devkit from your (main?) computer has the added benefit that if something goes haywire with - say - your solderings, you are not jeopardising your main computer. This is why the author prefers this two-machine solution.
-->

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


## Prepare

### Setting up USB/IP sharing

- See [ESP32-WSL](https://github.com/lure23/ESP32-WSL?tab=readme-ov-file#steps-set-up-usb-ip-bridging) (GitHub).

	Mark down the IP of the machine (Windows or Linux) providing USB/IP bridging.


### Launching Multipass VM

- See [`mp`](https://github.com/akauppi/mp) (GitHub) and have the `rust-emb` VM running.


### Connecting the device to Multipass

We presume you have the USB-to-IP server running, and know the IP of the intermediate machine (`192.168.1.29` below - replace with yours).

The following commands are to be given in the `rust-emb` VM:

1. Check that you can see the dev board (optional)

	```
   $ usbip list -r 192.168.1.29
	```
	
	<details><summary>**ESP32-C3-DevKitC-02 v.1.1**</summary>

   ```
   Exportable USB devices
   ======================
    - 192.168.1.29
           3-1: Silicon Labs : CP210x UART Bridge (10c4:ea60)
              : USB\VID_10C4&PID_EA60\BC2F214F809DED11AAFA5F84E259FB3E
              : (Defined at Interface level) (00/00/00)
              :  0 - Vendor Specific Class / unknown subclass / unknown protocol (ff/00/00)
   ```
   </details>

	<details><summary>**ESP32-C6-DevKitM-1 v.1.0**</summary>

   ```
	Exportable USB devices
	======================
	 - 192.168.1.29
	        3-1: unknown vendor : unknown product (303a:1001)
	           : USB\VID_303A&PID_1001\54:32:04:07:15:10
	           : Miscellaneous Device / ? / Interface Association (ef/02/01)
	           :  0 - Communications / Abstract (modem) / None (02/02/00)
	           :  1 - CDC Data / unknown subclass / unknown protocol (0a/02/00)
	           :  2 - Vendor Specific Class / Vendor Specific Subclass / unknown protocol (ff/ff/01)
	```
	</details>
   
2. Attach the dev board to your VM

   ```
   $ sudo usbip attach -r 192.168.1.29 -b 3-1
   ```

3. Now you should see it in the device tree:

   ```   
   $ lsusb
   ```
   
	<details><summary>**ESP32-C3-DevKitC-02 v.1.1**</summary>

	```
   Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
   Bus 001 Device 009: ID 10c4:ea60 Silicon Labs CP210x UART Bridge
   Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
   ```
	</details>
	
	<details><summary>**ESP32-C6-DevKitM-1 v.1.0**</summary>

	```
	Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
	Bus 001 Device 003: ID 303a:1001 Espressif USB JTAG/serial debug unit
	Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
	```
	</details>

4. Check with `probe-rs`

	>Haven't gotten this working with the `ESP32-C3-DevKitC-02`, so needing to abandon that target, for now. <!--tbd. Create an issue, then mention here!!!-->
	
	<details><summary>**ESP32-C6-DevKitM-1 v.1.0**</summary>

	```
	$ probe-rs list
	The following debug probes were found:
	[0]: ESP JTAG -- 303a:1001:54:32:04:07:15:10 (EspJtag)
	```
	</details>


## Run some examples from `esp-hal`

>**Background**: The [esp-hal](https://github.com/esp-rs/esp-hal) repo (Hardware Abstraction Layer) has code that allows using the ESP32 hardware from Rust. It includes also an `examples` folder that we'll look into.


### Clone

Let's clone the [`esp-hal`](https://github.com/esp-rs/esp-hal) repo to your host (Mac):

```
$ git clone --depth 1 git@github.com:esp-rs/esp-hal.git
```

>`--depth 1` excludes history; we don't need it and same some disk space

### Share the folder with VM

```
$ multipass stop rust-emb
$ multipass mount --type=native esp-hal rust-emb:/home/ubuntu/esp-hal
```

>Note: We don't need to enter the `esp-hal` folder on the host side. Just sharing it with the VM.

<!-- disabled (too much)
<p />

>Warn: The size of the folder becomes around 1.9GB (try `du -h -d1 .`). This author prefers having such large folders on the host side. Your call, though.
-->

Now you have `esp-hal` available within the VM. 

>Note: Unfortunately, using `--type=native` requires the VM to be shut down when adding/removing mounts. On the other side, it promises better performance than default Multipass mounts.

### Restart the VM

```
$ multipass shell rust-emb
```

Within the VM, re-attach the development kit. It was lost when we restarted the VM.

```
$ sudo usbip attach -r 192.168.1.29 -b 3-1
```

### Build

Within the VM:

```
$ cd esp-hal
```

>Study the [`esp-hal/examples/`](https://github.com/esp-rs/esp-hal/tree/main/examples/) folder. 
>
>This is a treasure trove of dealing with different sensors. To begin with, we just build (and run) the `hello_world` to see connection to the development board would work.

<!-- tbd...?
-- here about editing `Cargo.toml` **IF** this helps with the console output:
   
#esp-println         = { path = "../esp-println", features = ["log"] }  // WAS 2-Jun-24
esp-println         = { path = "../esp-println", default-features=false, features = ["log", "jtag-serial", "defmt-espflash"] }
-->

Within the `esp-hal` folder in the VM:

```
$ cargo xtask run-example esp-hal esp32c6 hello_world
```

Output:

```
    Updating crates.io index
  Downloaded proc-macro2 v1.0.85
  Downloaded strum_macros v0.26.3
  Downloaded 2 crates (76.4 KB) in 0.26s
   Compiling proc-macro2 v1.0.85
   Compiling unicode-ident v1.0.12
   Compiling memchr v2.7.2
   Compiling serde v1.0.203
   Compiling quote v1.0.36
   Compiling syn v2.0.66
   Compiling utf8parse v0.2.1
   Compiling anstyle-parse v0.2.4
   Compiling aho-corasick v1.1.3
    Building [===>                       ] 10/63: aho-corasick, syn      
[...]
     Running `target/debug/xtask run-example esp-hal esp32c6 hello_world`
[2024-06-02T07:36:32Z WARN  xtask] Package 'esp-hal' specified, using 'examples' instead
[2024-06-02T07:36:32Z INFO  xtask] Building example '/home/ubuntu/esp-hal/examples/src/bin/hello_world.rs' for 'esp32c6'
[2024-06-02T07:36:32Z INFO  xtask] Package: "src/bin/hello_world.rs"
[...]
   Compiling rand_core v0.6.4
   Compiling strum_macros v0.26.3
    Building [==>                       ] 37/251: darling_macro, strum_macros                                                                              
[...]
   Compiling embassy-futures v0.1.1
    Finished `release` profile [optimized + debuginfo] target(s) in 7m 59s
     Running `espflash flash --monitor target/riscv32imac-unknown-none-elf/release/hello_world`
? Use serial port '/dev/ttyACM0' - USB JTAG/serial debug unit? (y/n) ›
```

The output pauses here. Answer `**yes**` and `**yes**`:

```
✔ Use serial port '/dev/ttyACM0' - USB JTAG/serial debug unit? · yes
✔ Remember this serial port for future use? · yes
[2024-06-02T07:55:07Z INFO ] Serial port: '/dev/ttyACM0'
[2024-06-02T07:55:07Z INFO ] Connecting...
[2024-06-02T07:55:08Z INFO ] Using flash stub
Chip type:         esp32c6 (revision v0.0)
Crystal frequency: 40 MHz
Flash size:        4MB
Features:          WiFi 6, BT 5
MAC address:       54:32:04:07:15:10
App/part. size:    33,424/4,128,768 bytes, 0.81%
[00:00:00] [========================================]      13/13      0x0                                                                                    [00:00:00] [========================================]       1/1       0x8000                                                                                 [00:00:00] [========================================]      20/20      0x10000                                                                                [2024-06-02T07:55:09Z INFO ] Flashing has completed!
Commands:
    CTRL+R    Reset chip
    CTRL+C    Exit

ESP-ROM:esp32c6-20220919
Build:Sep 19 2022
rst:0x15 (USB_UART_HPSYS),boot:0xc (SPI_FAST_FLASH_BOOT)
Saved PC:0x40800540
0x40800540 - esp_hal::interrupt::riscv::vectored::get_configured_interrupts
    at ??:??
SPIWP:0xee
mode:DIO, clock div:2
load:0x4086c410,len:0xd48
load:0x4086e610,len:0x2d68
load:0x40875720,len:0x1800
entry 0x4086c410
I (23) boot: ESP-IDF v5.1-beta1-378-gea5e0ff298-dirt 2nd stage bootloader
I (23) boot: compile time Jun  7 2023 08:02:08
I (24) boot: chip revision: v0.0
I (28) boot.esp32c6: SPI Speed      : 40MHz
I (33) boot.esp32c6: SPI Mode       : DIO
I (37) boot.esp32c6: SPI Flash Size : 4MB
I (42) boot: Enabling RNG early entropy source...
I (48) boot: Partition Table:
I (51) boot: ## Label            Usage          Type ST Offset   Length
I (58) boot:  0 nvs              WiFi data        01 02 00009000 00006000
I (66) boot:  1 phy_init         RF data          01 01 0000f000 00001000
I (73) boot:  2 factory          factory app      00 00 00010000 003f0000
I (81) boot: End of partition table
I (85) esp_image: segment 0: paddr=00010020 vaddr=42000020 size=056cch ( 22220) map
I (98) esp_image: segment 1: paddr=000156f4 vaddr=40800000 size=00014h (    20) load
I (102) esp_image: segment 2: paddr=00015710 vaddr=42005710 size=01c5ch (  7260) map
I (112) esp_image: segment 3: paddr=00017374 vaddr=40800014 size=00ef8h (  3832) load
I (120) boot: Loaded app from partition at offset 0x10000
```

Did the sample run?

```
    CTRL+R    Reset chip
    CTRL+C    Exit
```

Not sure. It seems the development board works, but we don't see `Hello world!` anywhere. <sub>[source](https://github.com/esp-rs/esp-hal/blob/main/examples/src/bin/hello_world.rs#L38)</sub>

<font color=orange>*tbd. Figure out why the output doesn't reach the host terminal.*</font>

### More samples

To see which samples are suitable for e.g. `esp32-c6`: 

```
$ git grep -P "(?<=% CHIPS:).*esp32c6" examples/src/
```

That gives 82 matches! 😃

<!-- 
### ADC
...
-->

<!-- tbd. Doesn't blink the *internal* LED; fix that?
### Blinky 🚨

Leaving the lack of `println` visibility behind, let's check:

```
$ cargo xtask run-example esp-hal esp32c6 blinky
```

-->

## Embassy!

**tbd.**

The main `loop` of the `blinky` application does busy-looping. It keeps the CPU running all the time.

By moving to [Embassy](https://embassy.dev) we can solve that for good!




<!-- bring back, IF we get things to work... tbd.
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

## Q/A & Hints

### How does `probe-rs` find a development board?

>Note the `303a:1001` in e.g. the `lsusb` output. These are the "vendor id" and "product id" of your board. 

They match a line in the `udev` rules file used by `probe-rs`:

```
$ cat /etc/udev/rules.d/69-probe-rs.rules | grep 303a | grep 1001
ATTRS{idVendor}=="303a", ATTRS{idProduct}=="1001", MODE="660", GROUP="plugdev", TAG+="uaccess"
```

### How to make the VM more performant?

You can increase the number of cores Multipass is allowed to use by:

- `multipass stop rust-emb`
- `multipass set local.rust-emb.cpus={X}`

Then `multipass shell rust-emb` to get back in action.


## References

- [Embedded Rust (`no_std`) on Espressif](https://docs.esp-rs.org/no_std-training/02_2_software.html)

