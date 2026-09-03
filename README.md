# Reverse Engineering the ThermoPro TempSpike TP960W

I thought a  **ThermoPro TempSpike TP960W** would be a straightforward target for reverse engineering and hacking:

<br>

```text
open it -> find debug pins -> dump firmware -> run binwalk and ghidra -> modify something
```
<br>
I ended up opening the receiver, destroying one of the temperature probes, scraping epoxy off the electronics, soldering a DAPLink probe directly to a castellated module, reaching the ARM debug infrastructure over SWD, reverse engineering a useful chunk of the BLE protocol, and discovering the Silicon Labs OTA bootloader.
<br>
<br>
<img width="945" height="499" alt="image" src="https://github.com/user-attachments/assets/9f1f2d07-a2a0-4d31-a06c-c444883e2d65" />

<br>

<br>

> **Scope:** This is an educational reverse engineering project on hardware I own. It is not affiliated with or endorsed by ThermoPro or Silicon Labs. Everything found here is publicly accessible through the app store or on my own hardware.
<br>

---

## TL;DR

| Path | Result |
|---|---|
| Open it up and identify the hardware | Success |
| Access the SWD debug ports | Success |
| Enumerate the ARM Debug Access Port | Success |
| Halt the Cortex | Failed |
| Dump flash or memory through SWD | Failed |
| Map the BLE GATT services | Success |
| Determine command / notification services | Success |
| Trigger Silicon Labs OTA mode | Success |
| Install arbitrary modified firmware | Failed |
| Understand the temperature probe | Destructive failure |
| Inspecting the APK | Found firmware files, but not a confirmed TP960W image |
| Push modified firmware | Failed |

---

## Opening it up

Before touching the firmware I wanted to understand what the hardware actually looked like.

### Inside the receiver

<img width="724" height="231" alt="image" src="https://github.com/user-attachments/assets/69579516-e75e-4fff-81ad-997c3875320a" />


Opening the receiver shows a typical assembly built around a 14500 size cell and the main PCB. Nothing about the enclosure or PCB is particularly interesting.

<img width="384" height="461" alt="image" src="https://github.com/user-attachments/assets/ac48b79c-b155-4fcb-820d-fa6f0b8cf6d9" />


The Bluetooth MCU is mounted on a small castellated edge module soldered onto the main PCB. I researched this Bluetooth module and couldn't find anything similar. There was no reason to believe it was a tack-on module.

That turned out to be very convenient for probing. The castellated connections gave me accessible points close to the MCU signals without needing to attach directly to a 5x5 mm QFN package.

#### PCB views

<img width="310" height="675" alt="image" src="https://github.com/user-attachments/assets/4b5da296-e951-47b0-a190-d36336b53444" />

<img width="354" height="670" alt="image" src="https://github.com/user-attachments/assets/9be0b1ca-5450-408e-83d4-92178644c482" />
<br>
<br>
The upper side had epoxy covering the interesting silicon. I was able to scrape and clean it off with acetone, alcohol, and a toothbrush (not together). The silkscreen labels are accurate.

The main SoC turned out to be:

```text
Silicon Labs EFR32BG22 C224F512IM40
```

It is a Cortex M33 based Silicon Labs Series 2 SoC with considerable security features implemented.

---

### The probe

<img width="556" height="101" alt="image" src="https://github.com/user-attachments/assets/70f9e5e6-b644-4102-98f3-a7d4d98c059b" />


The "TempSpike" looks simple from the outside...

### Destroying the probe for science

I also sacrificed one of the temperature spikes to find out how the sensing element was physically constructed.

<img width="357" height="536" alt="image" src="https://github.com/user-attachments/assets/678e2929-2a4c-40b7-9e9c-7c4d270cfbd2" />

This was way less informative than I expected.

The outer material is extremely hard and inside is also very tough ceramic. Even after destroying it, the sensing and transmitting mechanism is still lost on me. The whole thing seems to be two wires running through the ceramic housing.

---

## The Main SoC

```text
EFR32BG22C224F512IM40
```

Specs:

| Feature | Value |
|---|---|
| CPU | ARM Cortex M33 |
| CPU clock | 76.8MHz |
| Flash | 512kb |
| RAM | 32kb |
| Radio | Bluetooth Low Energy |
| Package | QFN40, 5x5 mm |
| Family | Silicon Labs EFR32 Series 2 |

The BG22 family supports security features including Secure Boot, a Secure Engine, and configurable debug access that can be restricted or authenticated using SiLabs' security mechanisms.

### Relevant SWD pins

For the QFN40 package:

| Signal | GPIO | QFN40 pin |
|---|---|---:|
| `SWCLK` | `PA01` | 22 |
| `SWDIO` | `PA02` | 23 |
| `3v3` | `VDD` | VDD |
| `GND` | `GND` | GND |
| `RESETn` | — | 11 |

<img width="674" height="605" alt="image" src="https://github.com/user-attachments/assets/defc38e6-e241-4039-bfa8-c3bf2f4208ae" />

Attached the datasheet to the repo. Page 81

---

## Get SWD Working

Based on other projects, this was supposed to be pretty straightforward.

The plan was:

1. Identify `SWDIO` and `SWCLK`
2. Connect my CMSIS-DAP probe
3. Enumerate the ARM Debug Access Port
4. Halt the Cortex M33
5. Read out flash.bin
6. profit

As per my luck it seems cryptographically impossible to reach 4, any voltage glitching experts reading this?

### A professional's setup

Because the module section sits on castellated edges, I could solder the DAPLink/debug wiring directly to the exposed edges. I verified with continuity tests. 

<img width="544" height="728" alt="image" src="https://github.com/user-attachments/assets/12b8c116-df6c-47c5-8aeb-af69c6ed3d10" />

Convenience is a blessing with projects like this and beauty is very much optional

### Initial OpenOCD configuration

I started with:

```bash
openocd \
  -f interface/cmsis-dap.cfg \
  -c "transport select swd" \
  -c "adapter speed 100" \
  -c "source [find target/swj-dp.tcl]" \
  -c "swj_newdap bg22 cpu -expected-id 0x6ba02477" \
  -c "dap create bg22.dap -chain-position bg22.cpu" \
  -c "target create bg22.cpu cortex_m -dap bg22.dap -ap-num 3" \
  -c "init"
```

Intentionally slow clock at 100Hz due to the nature of the hacky SWD connection

### Something answered

OpenOCD successfully read:

```text
SWD DPIDR 0x6ba02477
```

This confirmed that the SW-DP was alive and my SWD connection was working.

I could also enumerate several [Access Ports](https://piolabs.com/blog/engineering/diving-into-arm-debug-access-port.html):

```text
AP0: IDR 0x84770001  -> MEM-AP AHB3
AP1: IDR 0x54770002  -> MEM-AP APB2/APB3
AP3: IDR 0x84770001  -> MEM-AP AHB3
```

AP0 reported:

```text
AP # 0x0
AP ID register 0x84770001
Type is MEM-AP AHB3
MEM-AP BASE 0xe00fe003
Valid ROM table present
Component base address 0xe00fe000
Can't read component, the corresponding core might be turned off
```

AP1 exposes a separate debug path and identified as **Energy Micro**, which is consistent with the Silicon Labs lineage. This company was [acquired](https://news.silabs.com/2013-06-07-Silicon-Labs-to-Acquire-Energy-Micro-a-Leader-in-Low-Power-ARM-Cortex-Based-Microcontrollers-and-Radios) by SiLabs. AP1 is mainly a security/debug-control path, so it can report or manage debug state but does not provide the direct flash/memory access needed to dump firmware.

I still did not have a debuggable CPU.

---

### Checking other APs

AP0 was more interesting. Its base address appeared to fall in [Cortex M CoreSight](https://support.arm.com/documentation/100231/0002/Programmers-model/Memory-model) region, suggesting that AP0 may have been the main processor debug access port.

Outputs:

```text
> mdw 0x00000000 8         // Read 8 words at 0x0
Target not examined yet

> mdw 0x20000000 8        // Read 8 words at 0x2
Target not examined yet

> halt                    // Stop the application and debug
Target not examined yet
```

'Target not examined yet' means the debugger could not attach to the cortex, since everything was working and enumerable, then it could not attach because it had been disabled. There are 3 types of locks:

  - **Standard:** Debug access is blocked but a full device erase can unlock it. The firmware and memory is wiped in the process
  
  - **Secure:** Debug access is blocked unless you provide a private key (ECC P-256/Certs)
  
  - **Permanent:**	Debug access is closed permanently

(I have not confirmed which is the case because I do not have a J-Link adapter to use SiLabs Commander software and I did not want to reset the firmware)

### Reset / halt did not help

A reset-halt attempt gave:

```text
SWD DPIDR 0x6ba02477
Failed to read memory at 0xe000ed00
[bg22.cpu] AP write error, reset will not halt
[bg22.cpu] VECTRESET is not supported on this Cortex-M core, using SYSRESETREQ instead.
[bg22.cpu] DP initialisation failed
```

`0xE000ED00` is the Cortex-M System Control Block's `CPUID` register, and the debugger was unable to read it.


---

## Attempt Two: Bluetooth LE

While the hardware and firmware were likely locked out, the BLE portion didn't have anything to hide. There are actually other repos that have also reverse engineered the BLE service to attach it to ESP32s and Home Assistant. See [/alexpacini/tpy357](https://github.com/alexpacini/tpy357) and [Bluetooth-Devices/thermopro-ble](https://github.com/Bluetooth-Devices/thermopro-ble)

The receiver advertises itself as:

```text
TP960R
```

It can be placed into pairing mode by applying power while holding the button on the PCB. (Power can be USB or the 3v3 VDD). 

**NOTE:** When killing the rest of the electronics and powering just the USB, it inexplicably dies after about one minute. Requires a power cycle to re-connect.

Using `bluetoothctl`:

```text
[bluetoothctl]> power on
[bluetoothctl]> agent on
[bluetoothctl]> default-agent
[bluetoothctl]> scan on

[CHG] Device 38:39:8F:C2:EC:9A RSSI: -39
[CHG] Device 38:39:8F:C2:EC:9A Name: TP960R
```

### The useful GATT services

GATT stands for Generic Attribute Profile. It is the part of Bluetooth Low Energy that defines how a device exposes data and commands. A GATT device is set up like so:

  - **Services**:  Groups of related functionality

  - **Characteristics**: Individual values or command/data endpoints inside each service (supports read, write, or notify)

  - **Descriptors**: Extra information or configuration for a characteristic

The normal Bluetooth services were present, plus two vendor-specific services that mattered:

| Service | UUID | Role |
|---|---|---|
| ThermoPro application | `93bb7cab-eb14-c5a9-2354-a24f4df9330f` | Commands and TempSpike data |
| Silicon Labs OTA | `1d14d6ee-fd63-4fa1-bfa4-8f47b42119f0` | AppLoader / firmware upload mode |

The ThermoPro service exposed:

| Characteristic | Observed role |
|---|---|
| `0000ff01-0000-1000-8000-00805f9b34fb` | Notifications / responses |
| `0000ff02-0000-1000-8000-00805f9b34fb` | Commands |
| `0000ff03-0000-1000-8000-00805f9b34fb` | Firmware/version-related data |

`FF01` exposes a CCCD descriptor (0x2902) which the client writes to enable or disable notifications.

---

## Temperature Data: Command In, Notification Out

The TP960 does not return temperature data from a normal GATT read. Instead the host sends a command to **FF02** service and the device sends the result back as a notification on **FF01**.

```text
Host
  |
  | write command
  v
FF02
  |
  v
TP960 processes request
  |
  | notification
  v
FF01
  |
  v
temperature / battery / status
```

### The APK reveals all

Decompiling the official app with Jadx confirmed the BLE processes

<img width="185" height="184" alt="image" src="https://github.com/user-attachments/assets/d2e2fb7f-deef-457c-a29f-325a151b7734" />


Relevant files:

* `/Source/cn.com.adsmart.wifibbq/ble/BleController.kt` sends the temperature request after connecting
* `/Source/cn.com.adsmart.wifibbq/ble/MyBleManager.kt` enables notifications on `FF01` and handles the returned packets
* `/Source/cn.com.adsmart.wifibbq/ble/device_data/TP960Data.kt` defines the TP960 commands, including `01 00` for requesting the current temperature
* `/Source/cn.com.adsmart.wifibbq/ble/utils/TP960ChangeUtils.kt` decodes the returned temperature, battery, and status data


Pretty useful for device ownership. As mentioned previously, the other repos allow you to skip the app and BLE connection to your phone and poll the raw data with an ESP32 or similar controller

With `bluetoothctl`:

```text
menu gatt

// Enable notifications on FF01
select-attribute /org/bluez/hci0/dev_38_39_8F_C2_EC_9A/service0013/char0014
notify on

// Select FF02 for commands
select-attribute /org/bluez/hci0/dev_38_39_8F_C2_EC_9A/service0013/char0017
write "01 00"
```

<img width="540" height="157" alt="image" src="https://github.com/user-attachments/assets/a53678f8-2927-4860-abd6-0cf1a26fc0de" />

---

## Accidentally Finding OTA Mode

While exploring some of the vendor-specific characteristics with `bluetoothctl`, I was spamming bytes and watching the outcome

```text
select-attribute /org/bluez/hci0/dev_38_39_8F_C2_EC_9A/service001b/char001c
write "00"
```

The receiver immediately:

1. Disconnected
2. Rebooted
3. Reconnected back advertising as **`OTA`** instead of **`TP960R`**.

I thought this might introduce another attack vector, but upon more research, this is Silicon Labs' typical OTA AppLoader mode. The firmware packages can be encrypted and signed securely. The device stores the public signing key used to verify firmware signatures, while encrypted GBL updates use a separate AES decryption key stored on the device. I'll explain the binaries later.

Silicon Labs AppLoader receives firmware in a GBL (Gecko Bootloader) container and the application chunks it up and sends it in OTA mode. The process can be found in `/Source/cn.com.adsmart.wifibbq/ble/config/OtaConfig.kt`

<img width="928" height="230" alt="image" src="https://github.com/user-attachments/assets/1e832557-4985-4b79-bb31-9a057b6fc3cc" />


Output:

```
[TP960R:/service001b/char001c]> write "00"
Attempting to write /org/bluez/hci0/dev_38_39_8F_C2_EC_9A/service001b/char001c

hci0 38:39:8F:C2:EC:9A type LE Public disconnected with reason 3
[CHG] Device 38:39:8F:C2:EC:9A ServicesResolved: no

[SIGNAL] LE.Disconnected - org.bluez.Reason.Remote, Connection terminated by remote user
[SIGNAL] Disconnected - org.bluez.Reason.Remote, Connection terminated by remote user

[CHG] Device 38:39:8F:C2:EC:9A Connected: no
[CHG] Device 38:39:8F:C2:EC:9A RSSI: 0xffffffcd (-51)
[CHG] Device 38:39:8F:C2:EC:9A TxPower: 0x0000 (0)

[CHG] Device 38:39:8F:C2:EC:9A Name: OTA
[CHG] Device 38:39:8F:C2:EC:9A Alias: OTA
```


## Firmware Discoveries: Re-wrote this 3 times

Two firmware binaries were found inside the APK in the `raw` resources directory.

This was a roller coaster: first I thought I had found the TP960 or generic firmware and that it was encrypted, then I realized this binary was not encrypted (SCORE), and finally I realized it was for a completely different ThermoPro model.

### `ast6s53_v25_ota.bin`

A firmware image for an **[TP922](https://www.homedepot.com/p/ThermoPro-TP922W-Wi-Fi-Wireless-Grill-Thermometer-with-Dual-Probes-TP922W/332104431)** based ThermoPro device (D'OH!)

Initially I assumed `ast6s53_v25_ota.bin` was the firmware for the TP960 receiver because it was bundled with the ThermoPro APK. Based on the Silicon Labs OTA path I had just found, I also believed the firmware was AES-128 encrypted and ECDSA signed.

After loading the binary into Ghidra and examining the image, it became clear that this particular file was not encrypted. The binary identifies itself as **ASR BLE 560X** and follows the ASR560X OTA image format, displayed below. This is completely different MCU family but I was not familiar with it by name

<img width="759" height="712" alt="image" src="https://github.com/user-attachments/assets/14e703f3-ff1c-45a0-a6a3-cb71b19f194b" />

Further inspection revealed multiple `TP922` strings, indicating this image belongs to the ThermoPro **TP922** rather than the TP960 or a generic image.

<img width="230" height="96" alt="image" src="https://github.com/user-attachments/assets/2daec4e3-8cb1-47e9-a0a4-538164da1c13" />

The APK only stores this single firmware update for the TP922, I suspect it still may be possible to retrieve the TP960 through some sniffery. That may be part 2 and hopefully a succesful PWN

#### Loading the bin image into Ghidra anyways

The first 0x80 bytes of `ast6s53_v25_ota.bin` are the ASR560X OTA header, so the actual application starts at offset:

```text
0x80
```

The first two words of the application expose the Cortex M stack pointer and reset vector:

```text
0x20008C00
0x1004A9BD ---> LSnibble 1101
```

`0x20008C00` matches the documented ASR560X `_estack` value. The reset vector has bit 0 set because Cortex M uses Thumb mode. That low bit is used as a flag to indicate thumb mode, not as part of the actual address, so clearing it gives the real reset handler address:

```text
0x1004A9BC  ---> LSnibble 1100
```

The application firmware area begins at:

```text
0x10048000
```

Then import `firmware.bin` into Ghidra and choose:

```text
Processor: ARM
Endian: Little
Mode: Cortex-M / Thumb
Base address: 0x10048000
```

<img width="630" height="125" alt="image" src="https://github.com/user-attachments/assets/59e8cb88-67c5-4e7b-8d0c-23d024fe7a5b" />


### `ota_hpyke25_2101_v117.gbl`

The second file is a Silicon Labs Gecko Bootloader (GBL) update package.

Unlike the ASR560X binary above, this one matches the Silicon Labs OTA system used by the EFR32BG22 receiver. A GBL is basically the firmware update package that gets sent to the bootloader, and it can contain the actual application firmware.

This GBL is encrypted with **AES-128** and signed with **ECDSA P-256**. The bootloader decrypts it, verifies the signature, and then installs the firmware inside it.

So this file is much more interesting for the TP960W than the ASR560X image. However, I'm unable to confirm if it's for the TP960W let alone decrypt it

---

### Conclusion

At this point I am still proverbially cooked. I would like to work on sniffing the BT and HTTP/S traffic from the app but I am unfamiliar with this as of now. I might be able to encourage the firmware update by reporting an outdated firmware version to the app. The decompiled code didn't show an upgrade mechanism for the 960. Shoutout to ChatGPT for the significant crutch in this project; as with all my other projects, my goal is to learn, document, and share

Happy Hacking!


---

## References

### Silicon Labs

- EFR32BG22 product page: https://www.silabs.com/wireless/bluetooth/efr32bg22-series-2-socs/device.efr32bg22c224f512im40
- EFR32BG22 datasheet: https://www.silabs.com/documents/public/data-sheets/efr32bg22-datasheet.pdf
- Series 2 Secure Debug: https://docs.silabs.com/iot-security/latest/series2-secure-debug/
- Programming Series 2 devices using SWD: https://docs.silabs.com/connect-stack/latest/efr32-dci-swd-programming/
- Gecko Bootloader: https://docs.silabs.com/bluetooth/latest/using-gecko-bootloader-with-bluetooth-apps/
- ASR560X Docs: https://asriot.readthedocs.io/en/latest/ASR560X/Quick-Start/Developer_Guide.html
- Simplicity Commander security: https://docs.silabs.com/simplicity-commander/latest/simplicity-commander-commands/security-commands
