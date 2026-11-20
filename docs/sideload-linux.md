# Sideload for ECarX E02 Android 9 — Complete Linux Guide

Flash `lk.bin` and `boot.bin` once — everything after that is done on the running system over UART. Natively on Linux: no Windows, no WSL, no UsbDk. Every command is copy-paste ready.

**Platform:** Linux (native) · **IHU:** ECarX E02 · **Android:** 9 · **Method:** UART + USB drive + mtkclient CLI · **Flash:** lk + boot only

---

## Contents

- [Step 0 — Overview](#step-0--overview)
- [Step 1 — Prerequisites](#step-1--prerequisites)
- [Step 2 — Set up the Linux PC](#step-2--set-up-the-linux-pc)
- [Step 3 — Connect UART (first session)](#step-3--connect-uart-first-session)
- [Step 4 — Pull services.vdex off the IHU](#step-4--pull-servicesvdex-off-the-ihu)
- [Step 5 — Build services.jar](#step-5--build-servicesjar)
- [Step 6 — Patch lk.bin (after the 8.3 dump)](#step-6--patch-lkbin)
- [Step 7 — Patch boot.bin (Magisk) (after the 8.3 dump)](#step-7--patch-bootbin-magisk)
- [Step 8 — BROM: back up, erase, unlock, flash](#step-8--brom-back-up-erase-unlock-flash)
- [Step 9 — Connect UART again](#step-9--connect-uart-again)
- [Step 10 — Get root & remount rw](#step-10--get-root--remount-rw)
- [Step 11 — Deploy services.jar](#step-11--deploy-servicesjar)
- [Step 12 — Reboot & test](#step-12--reboot--test)
- [Step 13 — Stabilise Magisk](#step-13--stabilise-magisk)
- [Step 14 — File Manager](#step-14--file-manager)
- [Step 15 — SwipeBack (optional)](#step-15--swipeback-optional)
- [Fix Fast](#fix-fast)
- [Panic recovery — get back to stock](#panic-recovery--get-back-to-stock)
- [Notes](#notes)

---

## Step 0 — Overview

`Read First`

This is the Linux-native equivalent of all three Windows guides — the [MTKClient install](./mtkclient-windows-install.md), [Build services.jar](./build-services-jar.md) and [Sideload](./sideload.md) guides — combined into one linear workflow. Nothing here needs Windows, WSL, UsbDk, WinFsp, PuTTY or a COM port.

> [!TIP]
> **What gets flashed:** Two small files, one time — `lk.bin` (~1MB) and `boot.bin` (~32MB). The writing takes minutes. The full partition backup in [8.3](#83--session-a-back-up-everything-read-only) comes first and takes longer; it is your only way back if anything goes wrong, so it is not the place to save time. Everything after the flash happens on the running system over UART, so deploying `services.jar` takes seconds.

**What actually blocks installs:**

Stock firmware ships `/system/framework/services.jar` as a **183-byte stub** — an empty file. The real one is about 3.7MB and holds a real `classes.dex`. Because of that stub, `pm install` is blocked completely. Replace the file with the real one and the block disappears. That is the whole point of this method; every other step just gets you there safely.

**The full flow:**

**Reference** — From nothing to installing APKs on your own
```text
Phase 1  Set up Linux PC → udev/groups, mtkclient venv, serial tool, workspace
Phase 2  UART pull        → services.vdex off the IHU → build patched services.jar
Phase 3  BROM read-only   → dump every partition, verify, copy lk/boot into ORI   ← checkpoint, nothing written
Phase 4  Prepare files    → patch lk.bin, patch boot.bin with Magisk on a phone
Phase 5  BROM destructive → gated: erase + unlock → power-cycle → write lk + boot → read back
Phase 6  Live deploy      → UART: root → remount rw → swap services.jar → Magisk/File Manager
```

> [!IMPORTANT]
> **True reading order — Step 8 is not last on the bench.** The read-only dump lives in [8.1–8.3](#83--session-a-back-up-everything-read-only), but it **must** run before [Step 6](#step-6--patch-lkbin) and [Step 7](#step-7--patch-bootbin-magisk): `ORI/lk.bin` and `ORI/boot.bin` are copies of that dump, and patching before dumping is exactly how a unit ends up with no way back. The correct physical order is **1 → 2 → 3 → 4 → 5 → 8.1–8.3 → 6 → 7 → 8.4–8.6 → 9 → … → 15**. Step 6 tells you the same thing again before it touches anything.

> [!CAUTION]
> **No original image comes from the internet.** Any `lk.bin`, `boot.bin`, `system.bin` or `seccfg.bin` from another unit is a brick waiting to happen. Every image flashed in this guide must be byte-identical to your own dump, except for the four `lk` bytes and the Magisk patch you make yourself. The preflight scripts below enforce that.

> [!WARNING]
> **Where does root come from?** Not from Magisk. ECarX leaves a binary at `/system/xbin/su` in stock firmware — setuid root, group shell. Since the UART shell runs as uid `shell`, that binary hands you root directly. Magisk does something else entirely: it disables dm-verity so `/system` becomes **writable**. Two separate things, and you need both.

> [!CAUTION]
> **Risk:** This process unlocks and modifies a car head unit. A wrong flash can brick the IHU. Follow every step in order, don't skip. All original files are **renamed**, never deleted — so if it bootloops you can recover over UART. Keep the `ORI` folder and the partition dump safe.


### Does this apply to your unit?

Check both numbers before you start. The software version alone is not enough — the same model can ship more than one hardware revision, and a method verified on one is not automatically safe on the other.

| Model | IHU software versions | Hardware version |
|---|---|---|
| **Proton S70** Flagship & Flagship X | v333, v300, v209, v202 | `HW0SS110506M0101` |
| **Proton X50 RC** | v144, v115 | `HW0SX110506M0101` or `HW0SX110506H0101` |
| **Proton X90** Flagship | v735, v722, v694, v651, v647, v638 | `HW0VX110506H0101` |

All of the above run **Android 9** on the ECarX E02 platform. Newer platforms running **Android 10** (dynamic partitions, `super.bin`) use a different partition layout and are **not** supported by this guide.

### What is different from the Windows guides

| Windows + WSL guide | This Linux guide |
|---|---|
| Ubuntu WSL, `/mnt/c/Users/...` paths | Native shell, `~/IHU_DEPLOY` workspace |
| UsbDk + WinFsp to see BROM | libusb + udev rules; no driver install |
| `gui.bat`, "Run as administrator" | `mtk` CLI in a venv (or the GUI, with one patch) |
| PuTTY, COM ports, Device Manager | `picocom` (or `screen`), `/dev/ttyUSB0` |
| `CH341SER.exe` driver | In-kernel `ch341` module — already there |
| Flatpak/pyenv Python PATH traps | Standard venv; extra deps in one apt line |
| The GUI double-init bug | Avoided entirely by using the CLI; GUI patch optional in [8.6](#86--optional-use-the-gui-instead) |

Everything else — the byte patterns, offsets, four smali patches, Magisk options, cache rename order and on-device commands — is identical.

---

## Step 1 — Prerequisites

`Hardware + Software`

### 1.1 — Hardware

| Item | Notes |
|---|---|
| CH340G USB-to-TTL adapter | Must be set to **3.3V**, not 5V. 5V will permanently damage the IHU UART pins. |
| Micro JST GH 6-pin cable, single connector | GH series is 1.25 mm pitch — a 6-pin cable from any other JST series will not fit. Full pinout in [8.2](#82--enter-brom-mode). **The three wires you are not actively using must be insulated.** |
| **A second, dedicated 4-wire BROM jumper** | White↔blue and green↔red soldered/sleeved, with yellow and black **removed from the housing**. Do not reuse the six-wire UART cable for BROM. |
| Heat shrink tube + heat gun | To cover the bare ends of the blue, green and red wires whenever you are not shorting them for BROM. Electrical tape works too — but they must not be left bare. |
| USB A-to-A cable (male-to-male) | For mtkclient during flashing. Must be a data cable, not charge-only. **Only connect it after the green↔red short is verified** — with that short missing, both ends are hosts. |
| USB 2.0 port on the PC | The black ones. BROM handshake is unreliable on some USB 3.0 controllers. |
| USB drive, **FAT32** | To move files onto and off the IHU. exFAT and NTFS are not read. |
| Android phone | Android 10 or newer, to patch boot.bin with the Magisk app. |
| Linux PC | Any mainstream distro with USB and a serial port. |
| **Stable 12 V supply** | Bench supply set to 12.0–12.6 V with at least 5 A, or a healthy car battery on a charger. Never flash on a weak battery, with the engine cranking, or where start-stop can cut accessory power. |
| Multimeter | To prove white↔blue and green↔red are the only two joins before anything is powered. |


![CH340G USB-to-TTL adapter seen from above, pin header labelled 5V, 3V3, TXD, RXD, GND](../images/ch340g-adapter-front.jpg)

*CH340G adapter. The pin header is labelled `5V`, `3V3`, `TXD`, `RXD`, `GND`.*

![The same CH340G adapter at an angle, showing the yellow voltage-select jumper](../images/ch340g-adapter-angled.jpg)

*The yellow jumper selects the voltage. It must sit on **3V3**, never 5V.*

![Micro JST GH 6-pin cable with the white connector on the right and bare tinned wire ends](../images/jst-6pin-cable-connector.jpg)

*Micro JST GH 6-pin cable, single connector. The connector plugs into the IHU; the bare ends go to the adapter.*

![A plain USB-A flash drive photographed from above](../images/usb-flash-drive-top.jpg)

*Any small USB-A drive works, as long as it is formatted **FAT32**.*

> [!CAUTION]
> **Before you buy anything else, get heat shrink or electrical tape.** The Micro JST GH cable arrives with six bare tinned ends. Three (yellow, black, white) are your permanent UART wires; the other three — blue, green and red — you short only briefly in [Step 8](#step-8--brom-back-up-erase-unlock-flash) for BROM, and the rest of the time they sit bare and live next to your working wires, with red carrying **3.3V from the IHU** whenever the unit is powered. Cover them whenever you are not deliberately shorting them.

### 1.2 — Software on Linux

| Software | Used for |
|---|---|
| `python3`, `git`, `build-essential`, `zlib1g-dev`, `libfuse-dev`, `zip`, `xxd` | Build toolchain and mtkclient dependencies (installed in [Step 2](#step-2--set-up-the-linux-pc)). |
| `smali`, `baksmali` | Disassemble / reassemble DEX. Debian/Ubuntu packages; on other distros use the fat jars from [smali releases](https://github.com/baksmali/smali/releases) with a JDK. |
| `picocom` (or `screen`) | Serial terminal at 921600 baud. Replaces PuTTY. |
| mtkclient | Upstream [bkerler/mtkclient](https://github.com/bkerler/mtkclient), installed from source in a venv in [2.4](#24--install-mtkclient). Nothing prebuilt. |
| Magisk app | Official releases only — [github.com/topjohnwu/Magisk/releases](https://github.com/topjohnwu/Magisk/releases). Take `Magisk-vXX.X.apk` from the newest release — never a repackaged mirror. |
| Android phone | For the Magisk patch in [Step 7](#step-7--patch-bootbin-magisk). |

### 1.3 — Files you need

| File | Size | Where it comes from |
|---|---|---|
| `lk.bin` | ~1MB | Original firmware from your own unit, copied from your partition dump in [8.3](#83--session-a-back-up-everything-read-only). |
| `boot.bin` | ~32MB | Original firmware from your own unit, from the same dump. |
| `services.jar` (modified) | ~3.7MB | Built from your own unit's firmware in [Step 5](#step-5--build-servicesjar). The single most important file. |
| `Magisk.apk` | ~11MB | Official releases only. |
| `Files.apk` | — | File Manager+ (`com.alphainventor.filemanager`) from [APKPure](https://apkpure.com/file-manager/com.alphainventor.filemanager). Take an **arm64-v8a** build that still supports Android 9. |
| `SwipeBack.apk` | ~1MB | Optional. Swipe-back button (`ace.jun.simpleback`) from [APKPure](https://apkpure.com/swipe-back/ace.jun.simpleback). |

> [!TIP]
> You build `services.jar` yourself from your own unit's firmware. A stock jar will not work, and one built against a different firmware version may not either.

---

## Step 2 — Set up the Linux PC

`Shell`

### 2.1 — Install packages

**Debian / Ubuntu**
```bash
sudo apt-get update
sudo apt-get install -y python3 python3-venv python3-pip git build-essential \
    zlib1g-dev libfuse-dev smali baksmali zip unzip xxd picocom
```

On Ubuntu the `smali` package provides both `smali` and `baksmali` (v2.5.x, which handles Android 9 / API 28). On Fedora or Arch, install the equivalents (`python3 libusb1 fuse-devel` / `python libusb fuse2`) and use the [smali fat jars](https://github.com/baksmali/smali/releases) instead of the apt packages:

```bash
# Fedora
sudo dnf install python3 python3-devel python3-pip git gcc make zlib-devel fuse-devel libusb1-devel picocom zip unzip vim-common
# Arch
sudo pacman -S python python-pip git base-devel zlib fuse2 libusb picocom zip unzip vim
```

### 2.2 — Serial permissions and udev rules

Your user needs to open the CH340 adapter and the mtkclient USB endpoint. Add yourself to the serial groups now; the udev rule copy happens in [2.4](#24--install-mtkclient), after the clone exists:

```bash
for g in dialout plugdev uucp; do
  getent group "$g" >/dev/null && sudo usermod -aG "$g" "$USER"
done
```

**Reboot now** so the group change takes effect.

> [!WARNING]
> **ModemManager / brltty can steal UART and break BROM handshakes.** On desktop distros ModemManager probes both `/dev/ttyUSB0` (CH340) and `/dev/ttyACM0` (which is how the MediaTek `0e8d:0003` BROM endpoint enumerates). An `AT` probe sent to BROM consumes the single-shot handshake and makes `mtk` fail until you power-cycle. The udev rule in [2.4](#24--install-mtkclient) tells ModemManager to ignore MediaTek USB IDs; you should also stop both services before UART or BROM sessions:
> ```bash
> sudo systemctl stop ModemManager brltty 2>/dev/null || true
> ```
> They are only needed if you actually use a cellular modem or braille display.

### 2.3 — Find the serial device

Plug in the CH340G (power still disconnected from the IHU) and check the kernel log:

```bash
dmesg | grep -i -E 'ch34|ttyUSB'
ls -l /dev/ttyUSB*
```

You should see a `ch341` driver message and a `/dev/ttyUSB0` (or `ttyUSB1` if you have another adapter). Remember the name — it is used everywhere below as `/dev/ttyUSB0`.

> [!TIP]
> The Linux kernel's in-kernel `ch341` driver handles the CH340G at up to 3 Mbps, so 921600 baud needs no driver install and no extra kernel module setup. Nothing equivalent to the Windows `CH341SER.exe` exists here.

### 2.4 — Install mtkclient

Use a venv — modern distros block system-wide pip (PEP 668).

```bash
cd ~
git clone --recursive https://github.com/bkerler/mtkclient
cd mtkclient
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip
pip install -r requirements.txt
pip install .
```

Install the udev rules now that the clone exists, then check the CLI runs (neither touches hardware):

```bash
sudo cp ~/mtkclient/Setup/Linux/*.rules /etc/udev/rules.d/
printf '\n# Prevent ModemManager from probing MediaTek BROM/Preloader CDC ACM endpoints\nATTRS{idVendor}=="0e8d", ENV{ID_MM_DEVICE_IGNORE}="1", ENV{ID_MM_PORT_IGNORE}="1"\n' \
  | sudo tee -a /etc/udev/rules.d/52-mtk.rules >/dev/null
sudo udevadm control -R && sudo udevadm trigger
mtk --help | head -30
```

> [!NOTE]
> `pip install -r requirements.txt` pulls in a surprising number of packages (`Flask`, `SQLAlchemy`, `oslo.*`, …). That is an upstream typo — `keystone` on PyPI resolves to OpenStack Keystone, not the assembler (`keystone-engine`, also listed). Harmless: a few hundred MB of disk. Leave it.

> [!TIP]
> We use the **CLI** (`mtk`) for flashing. It performs the same read/erase/unlock/write operations as the GUI but calls `preloader.init()` only once, so it completely avoids the GUI's double-init handshake bug (see [Fix Fast](#fix-fast)). You do not need PySide6 or the GUI at all.

### 2.5 — Create the workspace

```bash
export BASE="$HOME/IHU_DEPLOY"
mkdir -p "$BASE/ORI" "$BASE/MOD" "$BASE/DUMP" "$BASE/svc"
ls -d "$BASE"/*
```

Directory layout used by every command below:

```text
IHU_DEPLOY/
├── ORI/    untouched originals: lk.bin, boot.bin  (never modify)
├── MOD/    patched files to flash: lk.bin, boot.bin, services.jar
├── DUMP/   full partition backup from mtkclient
└── svc/    services.jar build workspace
```

> [!WARNING]
> `export` lives only in the shell you set it in. Open a new terminal, or a second one, and it is gone — re-run the `export BASE=...` line before continuing, or commands will write to the wrong place. Every original file goes in `ORI` and is **never modified**; all patching is done on copies in `MOD`. `ORI` is your way back. Keep the path free of spaces, semicolons and commas — `mtk multi` splits commands on spaces and semicolons in [Step 8](#step-8--brom-back-up-erase-unlock-flash).

---

## Step 3 — Connect UART (first session)

`UART · picocom`

This first UART session is only to pull `services.vdex` off the IHU. You will repeat the physical setup after flashing in [Step 9](#step-9--connect-uart-again).

> [!CAUTION]
> **3.3V only.** Confirm the CH340G jumper is set to 3.3V before connecting anything. 5V will damage the IHU UART pins beyond repair.

### 3.1 — Insulate the unused wires

Port location and the full six-pin table are in [8.2](#82--enter-brom-mode). For UART you use only three wires: **yellow, black and white**. Green, blue and red are shorted only for BROM; keep them insulated for all UART work — leave the green pin un-shorted or the USB drive in [Step 4](#step-4--pull-servicesvdex-off-the-ihu) will not mount.

![Side panel of the ECarX E02 chassis with the Micro JST GH port low down, next to a red TAP marking](../images/jst-port-location.jpg)

*Where to look: the port sits low on the side panel of the chassis. The red **TAP** marking here is hand-written; your unit will not have it.*

![Close-up of the Micro JST GH port showing six gold pins inside a white plastic housing](../images/jst-port-closeup.jpg)

*The same port close up — six pins in a white housing.*

![Micro JST GH 6-pin cable with the wires fanned out, showing white, blue, yellow, green, black and red](../images/jst-6pin-cable-wires.jpg)

*The six wire colours in pin order (1 to 6): white (GND), blue (Recovery), green (USB mode), black (UART RX), yellow (UART TX), red (3.3V).*

> [!CAUTION]
> **Insulate blue, green and red before the UART working phase. This is the highest-risk part of the hardware setup.** Their ends are **bare tinned copper**, and while the unit is powered:
>
> - **Red is a live 3.3V rail.** Red touching white (GND), or the adapter's GND pin, is a dead short across the IHU's 3.3V supply — the worst outcome on this list.
> - **Blue is Recovery.** Blue touching GND drops the unit into BROM / recovery instead of booting normally.
> - **Green is USB mode.** Green touching red flips the USB port into device mode, and then a USB drive will not mount.
>
> All three ends hang loose right next to the wires you *are* using, and the cable moves every time you reach around the unit. Cover them.

| Method | How |
|---|---|
| **Heat shrink tube** (best) | Slide a ~15mm piece of 2–3mm tube over each bare end **individually**, then shrink with a heat gun at roughly 100–150 °C, a few seconds each, turning the wire as it tightens. Adhesive-lined tube seals best. |
| No heat gun? | The barrel of a hot soldering iron held close (not touching), or a lighter held a few cm below the tube — move it constantly, never let the flame touch the tube or the wire. A hair dryer is usually not hot enough. |
| PVC electrical tape | Wrap each end separately, two or three turns past the tip, then fold the tip back and tape it down so nothing can poke out. |
| Self-amalgamating silicone tape | Stretch and wrap each end; it fuses to itself and leaves no adhesive residue. |
| Spare dupont housing | Push an empty 1-pin dupont shell or a short offcut of empty wire sleeving over each end. Fast, reversible, no heat needed. |

> [!WARNING]
> **Insulate them separately, never bundled together.** Taping the three bare ends into one lump is how you short blue to red. One cover per wire. **Do not cut them off** — they are your BROM recovery path.

### 3.2 — Wire it up and open the console

- Make sure the IHU **POWER socket is unplugged** before wiring anything — **Socket Block A — Power**, the black connector sitting lowest in the ISO stack.

  ![The ISO connector block on the back of the IHU with all sockets plugged in](../images/power-socket-before.jpg)

  *The ISO connectors as they normally sit, everything still plugged in.*

  ![The same ISO connector block with a red box drawn around the black power socket at the bottom of the stack](../images/power-socket-highlighted.jpg)

  *Unplug the one in the red box: **Socket Block A — Power**, the black connector at the very bottom of the ISO stack.*

- Check blue, green and red are still covered — do this every session, not just the first one.
- Yellow (IHU TX) → CH340G **RXD**; black (IHU RX) → CH340G **TXD**; white (GND) → CH340G **GND**. TX crosses to RX — wiring TX to TX gives silence.
- Open the serial console **before** powering on, so you catch the boot log:

```bash
picocom -b 921600 /dev/ttyUSB0
```

- Plug the IHU POWER socket back in — the boot log starts scrolling. Wait for boot to finish, then press **Enter** once to get a prompt.


![CH340G adapter from above with three jumper wires attached to the pin header](../images/ch340g-jumper-wires-top.jpg)

*Only three wires are used — yellow, black and white — on `TXD`, `RXD` and `GND`.*

![The same three jumper wires at an angle, with the pin labels readable](../images/ch340g-jumper-wires-angled.jpg)

*Angled view, so you can check each wire against its pin label.*

![CH340G adapter joined by jumper wires to the Micro JST GH 6-pin cable, ready to plug into the IHU](../images/ch340g-jst-cable-wired.jpg)

*The finished lead: adapter, three jumpers, and the JST GH cable that plugs into the IHU.*

> [!TIP]
> The prompt should look like `console:/ $`. The `$` means you are **not** root yet — that is normal, [Step 4](#step-4--pull-servicesvdex-off-the-ihu) calls the factory `su`.
>
> `screen /dev/ttyUSB0 921600` also works on Linux (the kernel defines `B921600`). picocom is used here for consistency. To leave picocom: `Ctrl-A`, then `Ctrl-X`.

> [!WARNING]
> **No text at all?** TX and RX are probably swapped — swap those two wires and reconnect. Also confirm the speed is exactly `921600`, the device name is right, and ModemManager/brltty are not holding it ([2.2](#22--serial-permissions-and-udev-rules)).

---

## Step 4 — Pull services.vdex off the IHU

`UART · picocom`

The real framework code sits in `/system/framework/oat/arm64/services.vdex`. We copy it to a FAT32 USB drive using the factory root shell — nothing is modified on the device.

### 4.1 — Get root with the factory su

In the UART shell, the prompt starts as `$` (uid shell). Call the factory `su` by full path — do **not** type a bare `su`, that hits MagiskSU which is not set up here:

```bash
/system/xbin/su
```

The prompt changes to `#`. Reading a file only needs root read access.

### 4.2 — Copy the vdex to a USB drive

Confirm the file and its size, then copy it to a FAT32 drive:

```bash
ls -lh /system/framework/oat/arm64/services.vdex*
ls /mnt/media_rw/
cp /system/framework/oat/arm64/services.vdex /mnt/media_rw/usbotg-otg1/services.vdex
sync
ls -lh /mnt/media_rw/usbotg-otg1/services.vdex
```

> [!WARNING]
> **Drive not visible?** Make sure the green pin is **not** shorted to red. While green is shorted, the IHU USB port is in device mode and will not read a drive. On a stock unit the port is in host mode by default, so the drive mounts normally.

### 4.3 — Move the file onto the PC

Pull the drive out, plug it into your Linux PC, find where it mounted and copy the file into the build workspace:

```bash
lsblk -f                       # find the FAT32 partition
ls /media/"$USER"/ /run/media/"$USER"/ 2>/dev/null
cp /media/"$USER"/*/services.vdex "$BASE/svc/" 2>/dev/null || \
  cp /run/media/"$USER"/*/services.vdex "$BASE/svc/"
ls -lh "$BASE/svc/services.vdex"
```

> [!NOTE]
> `/mnt/media_rw/usbotg-otg1/` is the path **inside the IHU**. On the PC the stick mounts under `/media/$USER/<label>` or `/run/media/$USER/<label>` — use `lsblk` or `mount | grep -i vfat` to find the right one.

Everything from here on happens on the PC.

---

## Step 5 — Build services.jar

`Shell`

This turns the stock, odexed `services.vdex` into a working, sideload-enabled `services.jar`. The flow:

```text
services.vdex  ─(vdexExtractor)→  services.cdex   (CompactDex)
services.cdex  ─(cdex2dex.py)→    services.dex    (standard DEX)
services.dex   ─(baksmali)→       smali/          (editable)
patch 4 changes ─(smali)→         classes.dex
classes.dex    ─(zip)→            services.jar    ← done
```

> [!WARNING]
> Stock firmware ships `services.jar` as a 183-byte stub — the real code lives in `services.vdex` in Android's internal **CompactDex** (`cdex001`) format. Standard tools (baksmali) cannot read CompactDex, and Google's `compact_dex_converter` needs a full AOSP build to compile. This guide uses a pure-Python converter instead — no AOSP, no untrusted binaries.

### 5.1 — Build vdexExtractor

```bash
cd ~
git clone https://github.com/anestisb/vdexExtractor.git
cd vdexExtractor
sed -i 's/-Werror/-Wno-error/g' src/Makefile
./make.sh
ls -l ~/vdexExtractor/bin/vdexExtractor
```

> [!TIP]
> Newer GCC treats one old warning as an error, so we disable `-Werror` before building. A few `-Wvla-parameter` warnings are fine — only errors stop the build. You want the binary at `~/vdexExtractor/bin/vdexExtractor`.

### 5.2 — Create the cdex2dex.py converter

baksmali cannot read CompactDex, and Google's converter needs a full AOSP tree. This pure-Python tool does the conversion with no external dependencies. Run the one-liner in the build folder — it decodes a ~430-line script into `cdex2dex.py`:

```bash
cd "$BASE/svc"
echo aW1wb3J0IHN0cnVjdCwgc3lzLCB6bGliLCBoYXNobGliCgpkZWYgdWxlYihiLG8pOgogICAgcj0wO3M9MAogICAgd2hpbGUgVHJ1ZToKICAgICAgICB4PWJbb107bys9MTtyfD0oeCYweDdmKTw8cwogICAgICAgIGlmIHg8MHg4MDogcmV0dXJuIHIsbwogICAgICAgIHMrPTcKZGVmIHd1bGViKHYpOgogICAgb3V0PWJ5dGVhcnJheSgpCiAgICB3aGlsZSBUcnVlOgogICAgICAgIHg9diYweDdmOyB2Pj49NwogICAgICAgIGlmIHY6IG91dC5hcHBlbmQoeHwweDgwKQogICAgICAgIGVsc2U6IG91dC5hcHBlbmQoeCk7IGJyZWFrCiAgICByZXR1cm4gYnl0ZXMob3V0KQoKY2xhc3MgQ29udjoKICAgIGRlZiBfX2luaXRfXyhzZWxmLCBwYXRoKToKICAgICAgICBiPXNlbGYuYj1vcGVuKHBhdGgsJ3JiJykucmVhZCgpCiAgICAgICAgYXNzZXJ0IGJbOjhdPT1iJ2NkZXgwMDFceDAwJwogICAgICAgIGc9bGFtYmRhIG86IHN0cnVjdC51bnBhY2tfZnJvbSgnPElJJyxiLG8pCiAgICAgICAgc2VsZi5uX3N0cixzZWxmLm9fc3RyPWcoNTYpOyBzZWxmLm5fdHlwLHNlbGYub190eXA9Zyg2NCkKICAgICAgICBzZWxmLm5fcHJvLHNlbGYub19wcm89Zyg3Mik7IHNlbGYubl9mbGQsc2VsZi5vX2ZsZD1nKDgwKQogICAgICAgIHNlbGYubl9tdGgsc2VsZi5vX210aD1nKDg4KTsgc2VsZi5uX2NscyxzZWxmLm9fY2xzPWcoOTYpCiAgICAgICAgc2VsZi5kYXRhX3NpemUsc2VsZi5kYXRhX29mZj1nKDEwNCkKICAgIGRlZiBEKHNlbGYsb2ZmKTogcmV0dXJuIHNlbGYuZGF0YV9vZmYrb2ZmIGlmIG9mZiBlbHNlIDAKCiAgICAjIC0tLS0tLS0tLS0gZW5jb2RlZF92YWx1ZSAvIGFubm90YXRpb24gbGVuZ3RoIG1lYXN1cmluZyAtLS0tLS0tLS0tCiAgICBkZWYgZW5jX3ZhbHVlX2xlbihzZWxmLCBvKToKICAgICAgICBiPXNlbGYuYjsgc3RhcnQ9bwogICAgICAgIGFyZ3R5cGU9YltvXTsgbys9MQogICAgICAgIHZ0PWFyZ3R5cGUmMHgxZjsgdmE9YXJndHlwZT4+NQogICAgICAgIGlmIHZ0IGluICgweDFjLCk6ICAgICAgICAjIEFSUkFZCiAgICAgICAgICAgIHNpemUsbz11bGViKGIsbykKICAgICAgICAgICAgZm9yIF8gaW4gcmFuZ2Uoc2l6ZSk6IG89c2VsZi5lbmNfdmFsdWVfbGVuKG8pCiAgICAgICAgZWxpZiB2dD09MHgxZDogICAgICAgICAgICMgQU5OT1RBVElPTgogICAgICAgICAgICBvPXNlbGYuZW5jX2Fubm90YXRpb25fbGVuKG8pCiAgICAgICAgZWxpZiB2dD09MHgxZSBvciB2dD09MHgxZjogICMgTlVMTCAvIEJPT0xFQU4gOiBubyBwYXlsb2FkCiAgICAgICAgICAgIHBhc3MKICAgICAgICBlbHNlOgogICAgICAgICAgICBvKz12YSsxICAgICAgICAgICAgICAjIChzaXplLTEpIHN0b3JlZCBpbiB2YQogICAgICAgIHJldHVybiBvCiAgICBkZWYgZW5jX2Fubm90YXRpb25fbGVuKHNlbGYsIG8pOgogICAgICAgIGI9c2VsZi5iCiAgICAgICAgXyxvPXVsZWIoYixvKSAgICAgICAgICAgICMgdHlwZV9pZHgKICAgICAgICBzaXplLG89dWxlYihiLG8pCiAgICAgICAgZm9yIF8gaW4gcmFuZ2Uoc2l6ZSk6CiAgICAgICAgICAgIF8sbz11bGViKGIsbykgICAgICAgICMgbmFtZV9pZHgKICAgICAgICAgICAgbz1zZWxmLmVuY192YWx1ZV9sZW4obykKICAgICAgICByZXR1cm4gbwogICAgZGVmIGVuY19hcnJheV9sZW4oc2VsZiwgbyk6CiAgICAgICAgYj1zZWxmLmI7IHNpemUsbz11bGViKGIsbykKICAgICAgICBmb3IgXyBpbiByYW5nZShzaXplKTogbz1zZWxmLmVuY192YWx1ZV9sZW4obykKICAgICAgICByZXR1cm4gbwogICAgZGVmIHN0cmluZ19kYXRhX2xlbihzZWxmLCBvKToKICAgICAgICBiPXNlbGYuYjsgXyxwPXVsZWIoYixvKQogICAgICAgIHdoaWxlIGJbcF0hPTA6IHArPTEKICAgICAgICByZXR1cm4gcCsxLW8KICAgIGRlZiBhbm5vdGF0aW9uX2l0ZW1fbGVuKHNlbGYsIG8pOgogICAgICAgIHJldHVybiBzZWxmLmVuY19hbm5vdGF0aW9uX2xlbihvKzEpLW8gICAjICsxIHZpc2liaWxpdHkKCiAgICAjIC0tLS0tLS0tLS0gY29tcGFjdCBjb2RlX2l0ZW0gLT4gc3RhbmRhcmQgYnl0ZXMgLS0tLS0tLS0tLQogICAgZGVmIGNvbnZfY29kZShzZWxmLCBvZmYpOgogICAgICAgIGI9c2VsZi5iOyBwPXNlbGYuRChvZmYpCiAgICAgICAgZmllbGRzXyxpYWNmPXN0cnVjdC51bnBhY2tfZnJvbSgnPEhIJyxiLHApCiAgICAgICAgaW5zbnNfY291bnQ9aWFjZj4+NTsgZmxhZ3M9aWFjZiYweDFmCiAgICAgICAgcmVnPShmaWVsZHNfPj4xMikmMHhGOyBpbnM9KGZpZWxkc18+PjgpJjB4Rjsgb3V0cz0oZmllbGRzXz4+NCkmMHhGOyB0cmllcz1maWVsZHNfJjB4RgogICAgICAgIHByZT1wCiAgICAgICAgaWYgZmxhZ3MmMHgxMDoKICAgICAgICAgICAgcHJlLT0yOyBpbnNuc19jb3VudCs9c3RydWN0LnVucGFja19mcm9tKCc8SCcsYixwcmUpWzBdCiAgICAgICAgICAgIHByZS09MjsgaW5zbnNfY291bnQrPXN0cnVjdC51bnBhY2tfZnJvbSgnPEgnLGIscHJlKVswXTw8MTYKICAgICAgICBpZiBmbGFncyYweDAxOiBwcmUtPTI7IHJlZys9c3RydWN0LnVucGFja19mcm9tKCc8SCcsYixwcmUpWzBdCiAgICAgICAgaWYgZmxhZ3MmMHgwMjogcHJlLT0yOyBpbnMrPXN0cnVjdC51bnBhY2tfZnJvbSgnPEgnLGIscHJlKVswXQogICAgICAgIGlmIGZsYWdzJjB4MDQ6IHByZS09Mjsgb3V0cys9c3RydWN0LnVucGFja19mcm9tKCc8SCcsYixwcmUpWzBdCiAgICAgICAgaWYgZmxhZ3MmMHgwODogcHJlLT0yOyB0cmllcys9c3RydWN0LnVucGFja19mcm9tKCc8SCcsYixwcmUpWzBdCiAgICAgICAgcmVnPXJlZytpbnMgICAgIyBDb21wYWN0RGV4IHN0b3JlcyByZWdpc3RlcnNfc2l6ZSBhcyAocmVnaXN0ZXJzIC0gaW5zKTsgcmVzdG9yZSBhYnNvbHV0ZQogICAgICAgIGluc25zX3N0YXJ0PXArNAogICAgICAgIGluc25zPWJbaW5zbnNfc3RhcnQ6aW5zbnNfc3RhcnQraW5zbnNfY291bnQqMl0KICAgICAgICByZXN0X29mZj1pbnNuc19zdGFydCtpbnNuc19jb3VudCoyCiAgICAgICAgdGFpbD1iJycKICAgICAgICBpZiB0cmllcz4wOgogICAgICAgICAgICByZXN0X29mZj0ocmVzdF9vZmYrMykmfjMgICAgICAgICMgdHJ5X2l0ZW1zIDQtYWxpZ24gKEFCU09MVVRFKSBpbiBjb21wYWN0IGRleAogICAgICAgICAgICAjIHRyeV9pdGVtczogOCBieXRlcyBlYWNoCiAgICAgICAgICAgIHRzdGFydD1yZXN0X29mZgogICAgICAgICAgICB0cmllc19ieXRlcz1iW3RzdGFydDp0c3RhcnQrdHJpZXMqOF0KICAgICAgICAgICAgaHN0YXJ0PXRzdGFydCt0cmllcyo4CiAgICAgICAgICAgICMgaGFuZGxlcnM6IGVuY29kZWRfY2F0Y2hfaGFuZGxlcl9saXN0LiBtZWFzdXJlIGxlbmd0aC4KICAgICAgICAgICAgaHA9aHN0YXJ0CiAgICAgICAgICAgIGhzaXplLGhwPXVsZWIoYixocCkKICAgICAgICAgICAgZm9yIF8gaW4gcmFuZ2UoaHNpemUpOgogICAgICAgICAgICAgICAgc3osaHA9dWxlYl9zKGIsaHApCiAgICAgICAgICAgICAgICBjbnQ9YWJzKHN6KQogICAgICAgICAgICAgICAgZm9yIF8gaW4gcmFuZ2UoY250KToKICAgICAgICAgICAgICAgICAgICBfLGhwPXVsZWIoYixocCk7IF8saHA9dWxlYihiLGhwKQogICAgICAgICAgICAgICAgaWYgc3o8PTA6CiAgICAgICAgICAgICAgICAgICAgXyxocD11bGViKGIsaHApICAgIyBjYXRjaF9hbGxfYWRkcgogICAgICAgICAgICBoYW5kbGVycz1iW2hzdGFydDpocF0KICAgICAgICAgICAgdGFpbD10cmllc19ieXRlcytoYW5kbGVycwogICAgICAgICMgc3RhbmRhcmQgY29kZV9pdGVtIGhlYWRlciAoZGVidWdfaW5mb19vZmY9MCkKICAgICAgICBoZHI9c3RydWN0LnBhY2soJzxISEhISUknLHJlZyxpbnMsb3V0cyx0cmllcywwLGluc25zX2NvdW50KQogICAgICAgIG91dD1ieXRlYXJyYXkoaGRyK2luc25zKQogICAgICAgIGlmIHRyaWVzPjA6CiAgICAgICAgICAgIGlmIGluc25zX2NvdW50JjE6IG91dCs9YidceDAwXHgwMCcKICAgICAgICAgICAgb3V0Kz10YWlsCiAgICAgICAgcmV0dXJuIGJ5dGVzKG91dCkKCmRlZiB1bGViX3MoYixvKTogICAjIHNsZWIxMjgKICAgIHI9MDtzPTAKICAgIHdoaWxlIFRydWU6CiAgICAgICAgeD1iW29dO28rPTE7cnw9KHgmMHg3Zik8PHM7cys9NwogICAgICAgIGlmIHg8MHg4MDoKICAgICAgICAgICAgaWYgeCYweDQwOiByfD0tKDE8PHMpCiAgICAgICAgICAgIHJldHVybiByLG8KCgpkZWYgYWxpZ240KHgpOiByZXR1cm4gKHgrMykmfjMKCmRlZiBidWlsZChpbnBhdGgsIG91dHBhdGgpOgogICAgYz1Db252KGlucGF0aCk7IGI9Yy5iCiAgICAjIC0tLS0gaW5kZXggdGFibGVzIChyYXcpIC0tLS0KICAgIHN0cl9vZmY9W3N0cnVjdC51bnBhY2tfZnJvbSgnPEknLGIsYy5vX3N0citpKjQpWzBdIGZvciBpIGluIHJhbmdlKGMubl9zdHIpXQogICAgdHlwZV9pZHM9W3N0cnVjdC51bnBhY2tfZnJvbSgnPEknLGIsYy5vX3R5cCtpKjQpWzBdIGZvciBpIGluIHJhbmdlKGMubl90eXApXQogICAgcHJvdG89W2xpc3Qoc3RydWN0LnVucGFja19mcm9tKCc8SUlJJyxiLGMub19wcm8raSoxMikpIGZvciBpIGluIHJhbmdlKGMubl9wcm8pXQogICAgZmllbGRfaWRzPWJbYy5vX2ZsZDpjLm9fZmxkK2Mubl9mbGQqOF0KICAgIG1ldGhvZF9pZHM9YltjLm9fbXRoOmMub19tdGgrYy5uX210aCo4XQogICAgY2xhc3NkZWY9W2xpc3Qoc3RydWN0LnVucGFja19mcm9tKCc8OEknLGIsYy5vX2NscytpKjMyKSkgZm9yIGkgaW4gcmFuZ2UoYy5uX2NscyldCgogICAgIyAtLS0tIGNvbGxlY3QgZGF0YSBpdGVtcyAtLS0tCiAgICB0eXBlbGlzdHM9c2V0KCk7IGNvZGVfb2Zmcz1zZXQoKTsgY2xhc3NkYXRhX29mZnM9c2V0KCk7IHN0YXRpY19vZmZzPXNldCgpCiAgICBhbm5kaXJfb2Zmcz1zZXQoKQogICAgZm9yIHAgaW4gcHJvdG86CiAgICAgICAgaWYgcFsyXTogdHlwZWxpc3RzLmFkZChwWzJdKQogICAgZm9yIGNkIGluIGNsYXNzZGVmOgogICAgICAgIGlmIGNkWzNdOiB0eXBlbGlzdHMuYWRkKGNkWzNdKSAgICAgICAgICMgaW50ZXJmYWNlcwogICAgICAgIGlmIGNkWzVdOiBhbm5kaXJfb2Zmcy5hZGQoY2RbNV0pICAgICAgICMgYW5ub3RhdGlvbnNfZGlyZWN0b3J5CiAgICAgICAgaWYgY2RbNl06IGNsYXNzZGF0YV9vZmZzLmFkZChjZFs2XSkgICAgIyBjbGFzc19kYXRhCiAgICAgICAgaWYgY2RbN106IHN0YXRpY19vZmZzLmFkZChjZFs3XSkgICAgICAgIyBzdGF0aWMgdmFsdWVzCgogICAgIyBwYXJzZSBjbGFzc19kYXRhLCBnYXRoZXIgY29kZV9vZmZzCiAgICBjbGFzc2RhdGE9e30gICAjIG9sZF9vZmYgLT4gKHJhd19wcmVmaXhfbGlzdHMpIHdlIHN0b3JlIHBhcnNlZCBmb3IgcmUtZW5jb2RlCiAgICBmb3Igb2ZmIGluIGNsYXNzZGF0YV9vZmZzOgogICAgICAgIG89Yy5EKG9mZik7IHNmLG89dWxlYihiLG8pOyBpbmYsbz11bGViKGIsbyk7IGRtLG89dWxlYihiLG8pOyB2bSxvPXVsZWIoYixvKQogICAgICAgICMgc3RhdGljIGZpZWxkcwogICAgICAgIHNmbD1bXQogICAgICAgIGlkeD0wCiAgICAgICAgZm9yIF8gaW4gcmFuZ2Uoc2YpOgogICAgICAgICAgICBkLG89dWxlYihiLG8pOyBhLG89dWxlYihiLG8pOyBzZmwuYXBwZW5kKChkLGEpKQogICAgICAgIGlmbD1bXQogICAgICAgIGZvciBfIGluIHJhbmdlKGluZik6CiAgICAgICAgICAgIGQsbz11bGViKGIsbyk7IGEsbz11bGViKGIsbyk7IGlmbC5hcHBlbmQoKGQsYSkpCiAgICAgICAgZG1sPVtdCiAgICAgICAgZm9yIF8gaW4gcmFuZ2UoZG0pOgogICAgICAgICAgICBkLG89dWxlYihiLG8pOyBhLG89dWxlYihiLG8pOyBjbyxvPXVsZWIoYixvKTsgZG1sLmFwcGVuZChbZCxhLGNvXSkKICAgICAgICAgICAgaWYgY286IGNvZGVfb2Zmcy5hZGQoY28pCiAgICAgICAgdm1sPVtdCiAgICAgICAgZm9yIF8gaW4gcmFuZ2Uodm0pOgogICAgICAgICAgICBkLG89dWxlYihiLG8pOyBhLG89dWxlYihiLG8pOyBjbyxvPXVsZWIoYixvKTsgdm1sLmFwcGVuZChbZCxhLGNvXSkKICAgICAgICAgICAgaWYgY286IGNvZGVfb2Zmcy5hZGQoY28pCiAgICAgICAgY2xhc3NkYXRhW29mZl09KHNmbCxpZmwsZG1sLHZtbCkKCiAgICAjIHBhcnNlIGFubm90YXRpb25zX2RpcmVjdG9yeSBjaGFpbgogICAgYW5uc2V0X29mZnM9c2V0KCk7IGFubnJlZl9vZmZzPXNldCgpOyBhbm5pdGVtX29mZnM9c2V0KCkKICAgIGFubmRpcj17fQogICAgZm9yIG9mZiBpbiBhbm5kaXJfb2ZmczoKICAgICAgICBvPWMuRChvZmYpCiAgICAgICAgY2FvLGZzLG1zLHBzPXN0cnVjdC51bnBhY2tfZnJvbSgnPElJSUknLGIsbyk7IG8rPTE2CiAgICAgICAgaWYgY2FvOiBhbm5zZXRfb2Zmcy5hZGQoY2FvKQogICAgICAgIGZpZWxkcz1bXQogICAgICAgIGZvciBfIGluIHJhbmdlKGZzKToKICAgICAgICAgICAgZmksYW89c3RydWN0LnVucGFja19mcm9tKCc8SUknLGIsbyk7IG8rPTg7IGZpZWxkcy5hcHBlbmQoKGZpLGFvKSk7CiAgICAgICAgICAgIGlmIGFvOiBhbm5zZXRfb2Zmcy5hZGQoYW8pCiAgICAgICAgbWV0aG9kcz1bXQogICAgICAgIGZvciBfIGluIHJhbmdlKG1zKToKICAgICAgICAgICAgbWksYW89c3RydWN0LnVucGFja19mcm9tKCc8SUknLGIsbyk7IG8rPTg7IG1ldGhvZHMuYXBwZW5kKChtaSxhbykpCiAgICAgICAgICAgIGlmIGFvOiBhbm5zZXRfb2Zmcy5hZGQoYW8pCiAgICAgICAgcGFyYW1zPVtdCiAgICAgICAgZm9yIF8gaW4gcmFuZ2UocHMpOgogICAgICAgICAgICBtaSxhbz1zdHJ1Y3QudW5wYWNrX2Zyb20oJzxJSScsYixvKTsgbys9ODsgcGFyYW1zLmFwcGVuZCgobWksYW8pKQogICAgICAgICAgICBpZiBhbzogYW5ucmVmX29mZnMuYWRkKGFvKQogICAgICAgIGFubmRpcltvZmZdPShjYW8sZmllbGRzLG1ldGhvZHMscGFyYW1zKQogICAgYW5uc2V0PXt9CiAgICBmb3Igb2ZmIGluIGFubnNldF9vZmZzOgogICAgICAgIG89Yy5EKG9mZik7IG4sPXN0cnVjdC51bnBhY2tfZnJvbSgnPEknLGIsbyk7IG8rPTQKICAgICAgICBpdGVtcz1bXQogICAgICAgIGZvciBfIGluIHJhbmdlKG4pOgogICAgICAgICAgICBhaSw9c3RydWN0LnVucGFja19mcm9tKCc8SScsYixvKTsgbys9NDsgaXRlbXMuYXBwZW5kKGFpKQogICAgICAgICAgICBpZiBhaTogYW5uaXRlbV9vZmZzLmFkZChhaSkKICAgICAgICBhbm5zZXRbb2ZmXT1pdGVtcwogICAgYW5ucmVmPXt9CiAgICBmb3Igb2ZmIGluIGFubnJlZl9vZmZzOgogICAgICAgIG89Yy5EKG9mZik7IG4sPXN0cnVjdC51bnBhY2tfZnJvbSgnPEknLGIsbyk7IG8rPTQKICAgICAgICBpdGVtcz1bXQogICAgICAgIGZvciBfIGluIHJhbmdlKG4pOgogICAgICAgICAgICBzaSw9c3RydWN0LnVucGFja19mcm9tKCc8SScsYixvKTsgbys9NDsgaXRlbXMuYXBwZW5kKHNpKQogICAgICAgICAgICBpZiBzaTogYW5uc2V0X29mZnMuYWRkKHNpKSAgICMgKG1heSBhZGQgbmV3IHNldHMpCiAgICAgICAgYW5ucmVmW29mZl09aXRlbXMKICAgICMgc2Vjb25kIHBhc3M6IGFueSBuZXcgYW5uc2V0IGZyb20gcmVmcwogICAgZm9yIG9mZiBpbiBsaXN0KGFubnJlZi5rZXlzKCkpOgogICAgICAgIGZvciBzaSBpbiBhbm5yZWZbb2ZmXToKICAgICAgICAgICAgaWYgc2kgYW5kIHNpIG5vdCBpbiBhbm5zZXQ6CiAgICAgICAgICAgICAgICBvPWMuRChzaSk7IG4sPXN0cnVjdC51bnBhY2tfZnJvbSgnPEknLGIsbyk7IG8rPTQ7IGl0ZW1zPVtdCiAgICAgICAgICAgICAgICBmb3IgXyBpbiByYW5nZShuKToKICAgICAgICAgICAgICAgICAgICBhaSw9c3RydWN0LnVucGFja19mcm9tKCc8SScsYixvKTsgbys9NDsgaXRlbXMuYXBwZW5kKGFpKQogICAgICAgICAgICAgICAgICAgIGlmIGFpOiBhbm5pdGVtX29mZnMuYWRkKGFpKQogICAgICAgICAgICAgICAgYW5uc2V0W3NpXT1pdGVtcwoKICAgICMgLS0tLSBjb252ZXJ0IGNvZGUgaXRlbXMgLS0tLQogICAgY29kZV9ieXRlcz17b2ZmOmMuY29udl9jb2RlKG9mZikgZm9yIG9mZiBpbiBjb2RlX29mZnN9CgogICAgIyAtLS0tIGFzc2lnbiBuZXcgb2Zmc2V0cyAtLS0tCiAgICBwb3M9MHg3MAogICAgb19zdHJpZHM9cG9zOyBwb3MrPWMubl9zdHIqNAogICAgb190eXBpZHM9cG9zOyBwb3MrPWMubl90eXAqNAogICAgb19wcm9pZHM9cG9zOyBwb3MrPWMubl9wcm8qMTIKICAgIG9fZmxkaWRzPXBvczsgcG9zKz1jLm5fZmxkKjgKICAgIG9fbXRoaWRzPXBvczsgcG9zKz1jLm5fbXRoKjgKICAgIG9fY2xzZGVmPXBvczsgcG9zKz1jLm5fY2xzKjMyCiAgICBkYXRhX3N0YXJ0PXBvcwoKICAgIG5ldz17fSAgIyAoa2luZCxvbGQpLT5uZXdvZmYKICAgIGRlZiBwbGFjZShraW5kLCBvZmYsIGJsb2IsIGFsbjQpOgogICAgICAgIG5vbmxvY2FsIHBvcwogICAgICAgIGlmIGFsbjQ6IHBvcz1hbGlnbjQocG9zKQogICAgICAgIG5ld1soa2luZCxvZmYpXT1wb3M7IHBvcys9bGVuKGJsb2IpOyByZXR1cm4gcG9zCgogICAgIyBjb2RlIGl0ZW1zICg0LWFsaWduKQogICAgZm9yIG9mZiBpbiBzb3J0ZWQoY29kZV9vZmZzKTogcGxhY2UoJ2NvZGUnLG9mZixjb2RlX2J5dGVzW29mZl0sVHJ1ZSkKICAgICMgdHlwZV9saXN0cyAoNC1hbGlnbikKICAgIHRsX2J5dGVzPXt9CiAgICBmb3Igb2ZmIGluIHNvcnRlZCh0eXBlbGlzdHMpOgogICAgICAgIG89Yy5EKG9mZik7IG4sPXN0cnVjdC51bnBhY2tfZnJvbSgnPEknLGIsbyk7IHRsX2J5dGVzW29mZl09YltvOm8rNCtuKjRdICAjIHdhaXQ6IHR5cGVfaXRlbSBpcyB1MgogICAgIyBmaXg6IHR5cGVfbGlzdCBlbnRyaWVzIGFyZSB1MgogICAgdGxfYnl0ZXM9e30KICAgIGZvciBvZmYgaW4gc29ydGVkKHR5cGVsaXN0cyk6CiAgICAgICAgbz1jLkQob2ZmKTsgbiw9c3RydWN0LnVucGFja19mcm9tKCc8SScsYixvKTsgdGxfYnl0ZXNbb2ZmXT1iW286bys0K24qMl0KICAgICAgICBwbGFjZSgndGwnLG9mZix0bF9ieXRlc1tvZmZdLFRydWUpCiAgICAjIGFubm90YXRpb25faXRlbXMgKGJ5dGUgYWxpZ24pIHZlcmJhdGltCiAgICBhaV9ieXRlcz17fQogICAgZm9yIG9mZiBpbiBzb3J0ZWQoYW5uaXRlbV9vZmZzKToKICAgICAgICBMPWMuYW5ub3RhdGlvbl9pdGVtX2xlbihjLkQob2ZmKSk7IGFpX2J5dGVzW29mZl09YltjLkQob2ZmKTpjLkQob2ZmKStMXQogICAgICAgIHBsYWNlKCdhaScsb2ZmLGFpX2J5dGVzW29mZl0sRmFsc2UpCiAgICAjIGFubm90YXRpb25fc2V0X2l0ZW0gKDQtYWxpZ24pCiAgICBmb3Igb2ZmIGluIHNvcnRlZChhbm5zZXQua2V5cygpKToKICAgICAgICBibG9iPWInXHgwMCcqKDQrNCpsZW4oYW5uc2V0W29mZl0pKTsgcGxhY2UoJ2FzJyxvZmYsYmxvYixUcnVlKQogICAgIyBhbm5vdGF0aW9uX3NldF9yZWZfbGlzdCAoNC1hbGlnbikKICAgIGZvciBvZmYgaW4gc29ydGVkKGFubnJlZi5rZXlzKCkpOgogICAgICAgIGJsb2I9YidceDAwJyooNCs0Kmxlbihhbm5yZWZbb2ZmXSkpOyBwbGFjZSgnYXInLG9mZixibG9iLFRydWUpCiAgICAjIGFubm90YXRpb25zX2RpcmVjdG9yeSAoNC1hbGlnbikKICAgIGZvciBvZmYgaW4gc29ydGVkKGFubmRpci5rZXlzKCkpOgogICAgICAgIGNhbyxmaWVsZHMsbWV0aG9kcyxwYXJhbXM9YW5uZGlyW29mZl0KICAgICAgICBibG9iPWInXHgwMCcqKDE2KzgqKGxlbihmaWVsZHMpK2xlbihtZXRob2RzKStsZW4ocGFyYW1zKSkpOyBwbGFjZSgnYWQnLG9mZixibG9iLFRydWUpCiAgICAjIHN0YXRpYyB2YWx1ZXMgKGJ5dGUgYWxpZ24pIHZlcmJhdGltCiAgICBzdl9ieXRlcz17fQogICAgZm9yIG9mZiBpbiBzb3J0ZWQoc3RhdGljX29mZnMpOgogICAgICAgIGVuZD1jLmVuY19hcnJheV9sZW4oYy5EKG9mZikpOyBzdl9ieXRlc1tvZmZdPWJbYy5EKG9mZik6ZW5kXQogICAgICAgIHBsYWNlKCdzdicsb2ZmLHN2X2J5dGVzW29mZl0sRmFsc2UpCiAgICAjIGNsYXNzX2RhdGEgKGJ5dGUgYWxpZ24pIC0gc2l6ZSBkZXBlbmRzIG9uIG5ldyBjb2RlIG9mZnMKICAgIGRlZiBlbmNfY2xhc3NkYXRhKG9mZik6CiAgICAgICAgc2ZsLGlmbCxkbWwsdm1sPWNsYXNzZGF0YVtvZmZdCiAgICAgICAgb3V0PWJ5dGVhcnJheSgpCiAgICAgICAgb3V0Kz13dWxlYihsZW4oc2ZsKSkrd3VsZWIobGVuKGlmbCkpK3d1bGViKGxlbihkbWwpKSt3dWxlYihsZW4odm1sKSkKICAgICAgICBmb3IgZCxhIGluIHNmbDogb3V0Kz13dWxlYihkKSt3dWxlYihhKQogICAgICAgIGZvciBkLGEgaW4gaWZsOiBvdXQrPXd1bGViKGQpK3d1bGViKGEpCiAgICAgICAgZm9yIGdycCBpbiAoZG1sLHZtbCk6CiAgICAgICAgICAgIGZvciBkLGEsY28gaW4gZ3JwOgogICAgICAgICAgICAgICAgbmNvPW5ld1soJ2NvZGUnLGNvKV0gaWYgY28gZWxzZSAwCiAgICAgICAgICAgICAgICBvdXQrPXd1bGViKGQpK3d1bGViKGEpK3d1bGViKG5jbykKICAgICAgICByZXR1cm4gYnl0ZXMob3V0KQogICAgY2RfYnl0ZXM9e30KICAgIGZvciBvZmYgaW4gc29ydGVkKGNsYXNzZGF0YS5rZXlzKCkpOgogICAgICAgIGNkX2J5dGVzW29mZl09ZW5jX2NsYXNzZGF0YShvZmYpOyBwbGFjZSgnY2QnLG9mZixjZF9ieXRlc1tvZmZdLEZhbHNlKQogICAgIyBzdHJpbmdfZGF0YSAoYnl0ZSBhbGlnbikgdmVyYmF0aW0KICAgIHNkX2J5dGVzPXt9CiAgICBmb3Igb2ZmIGluIHNvcnRlZChzdHJfb2ZmKToKICAgICAgICBpZiBvZmY9PTA6IGNvbnRpbnVlCiAgICAgICAgTD1jLnN0cmluZ19kYXRhX2xlbihjLkQob2ZmKSk7IHNkX2J5dGVzW29mZl09YltjLkQob2ZmKTpjLkQob2ZmKStMXQogICAgICAgIHBsYWNlKCdzZCcsb2ZmLHNkX2J5dGVzW29mZl0sRmFsc2UpCiAgICAjIG1hcF9saXN0ICg0LWFsaWduKQogICAgcG9zPWFsaWduNChwb3MpOyBvX21hcD1wb3MKCiAgICBnbG9iYWxzKClbJ19jdHgnXT0oYyxuZXcsc3RyX29mZix0eXBlX2lkcyxwcm90byxmaWVsZF9pZHMsbWV0aG9kX2lkcyxjbGFzc2RlZiwKICAgICAgICAgICAgICAgICAgICAgICB0bF9ieXRlcyxhaV9ieXRlcyxhbm5zZXQsYW5ucmVmLGFubmRpcixzdl9ieXRlcyxjZF9ieXRlcyxzZF9ieXRlcywKICAgICAgICAgICAgICAgICAgICAgICBvX3N0cmlkcyxvX3R5cGlkcyxvX3Byb2lkcyxvX2ZsZGlkcyxvX210aGlkcyxvX2Nsc2RlZixvX21hcCxkYXRhX3N0YXJ0KQogICAgcmV0dXJuIF9jdHgsIHBvcwoKCmRlZiBzZXJpYWxpemUoaW5wYXRoLCBvdXRwYXRoKToKICAgIGN0eCx0b3RhbD1idWlsZChpbnBhdGgsTm9uZSkKICAgIChjLG5ldyxzdHJfb2ZmLHR5cGVfaWRzLHByb3RvLGZpZWxkX2lkcyxtZXRob2RfaWRzLGNsYXNzZGVmLAogICAgIHRsX2J5dGVzLGFpX2J5dGVzLGFubnNldCxhbm5yZWYsYW5uZGlyLHN2X2J5dGVzLGNkX2J5dGVzLHNkX2J5dGVzLAogICAgIG9fc3RyaWRzLG9fdHlwaWRzLG9fcHJvaWRzLG9fZmxkaWRzLG9fbXRoaWRzLG9fY2xzZGVmLG9fbWFwLGRhdGFfc3RhcnQpPWN0eAogICAgYj1jLmIKICAgIG91dD1ieXRlYXJyYXkodG90YWwrMTAyNCkgICAjIHJlc2VydmUgcm9vbSBmb3IgbWFwX2xpc3QKCiAgICAjIC0tLS0gY29kZSBpdGVtcyAtLS0tCiAgICBmb3Igb2ZmIGluIG5ldzoKICAgICAgICBwYXNzCiAgICAjIHdyaXRlIGRhdGEgaXRlbXMKICAgIGZvciBvZmYsYmxvYiBpbiBbKGtbMV0sdikgZm9yIGssdiBpbiBbXV06CiAgICAgICAgcGFzcwogICAgIyBjb2RlCiAgICBmb3Igb2ZmIGluIGNfY29kZV9vZmZzKGMpOgogICAgICAgIG5vPW5ld1soJ2NvZGUnLG9mZildOyBibG9iPWMuY29udl9jb2RlKG9mZik7IG91dFtubzpubytsZW4oYmxvYildPWJsb2IKICAgICMgdHlwZSBsaXN0cwogICAgZm9yIG9mZixibG9iIGluIHRsX2J5dGVzLml0ZW1zKCk6CiAgICAgICAgbm89bmV3WygndGwnLG9mZildOyBvdXRbbm86bm8rbGVuKGJsb2IpXT1ibG9iCiAgICAjIGFubm90YXRpb24gaXRlbXMKICAgIGZvciBvZmYsYmxvYiBpbiBhaV9ieXRlcy5pdGVtcygpOgogICAgICAgIG5vPW5ld1soJ2FpJyxvZmYpXTsgb3V0W25vOm5vK2xlbihibG9iKV09YmxvYgogICAgIyBhbm5vdGF0aW9uX3NldF9pdGVtCiAgICBmb3Igb2ZmLGl0ZW1zIGluIGFubnNldC5pdGVtcygpOgogICAgICAgIG5vPW5ld1soJ2FzJyxvZmYpXTsgc3RydWN0LnBhY2tfaW50bygnPEknLG91dCxubyxsZW4oaXRlbXMpKQogICAgICAgIGZvciBqLGFpIGluIGVudW1lcmF0ZShpdGVtcyk6CiAgICAgICAgICAgIHN0cnVjdC5wYWNrX2ludG8oJzxJJyxvdXQsbm8rNCtqKjQsIG5ld1soJ2FpJyxhaSldIGlmIGFpIGVsc2UgMCkKICAgICMgYW5ub3RhdGlvbl9zZXRfcmVmX2xpc3QKICAgIGZvciBvZmYsaXRlbXMgaW4gYW5ucmVmLml0ZW1zKCk6CiAgICAgICAgbm89bmV3WygnYXInLG9mZildOyBzdHJ1Y3QucGFja19pbnRvKCc8SScsb3V0LG5vLGxlbihpdGVtcykpCiAgICAgICAgZm9yIGosc2kgaW4gZW51bWVyYXRlKGl0ZW1zKToKICAgICAgICAgICAgc3RydWN0LnBhY2tfaW50bygnPEknLG91dCxubys0K2oqNCwgbmV3WygnYXMnLHNpKV0gaWYgc2kgZWxzZSAwKQogICAgIyBhbm5vdGF0aW9uc19kaXJlY3RvcnkKICAgIGZvciBvZmYsKGNhbyxmaWVsZHMsbWV0aG9kcyxwYXJhbXMpIGluIGFubmRpci5pdGVtcygpOgogICAgICAgIG5vPW5ld1soJ2FkJyxvZmYpXQogICAgICAgIHN0cnVjdC5wYWNrX2ludG8oJzxJSUlJJyxvdXQsbm8sIG5ld1soJ2FzJyxjYW8pXSBpZiBjYW8gZWxzZSAwLCBsZW4oZmllbGRzKSxsZW4obWV0aG9kcyksbGVuKHBhcmFtcykpCiAgICAgICAgcT1ubysxNgogICAgICAgIGZvciBmaSxhbyBpbiBmaWVsZHM6CiAgICAgICAgICAgIHN0cnVjdC5wYWNrX2ludG8oJzxJSScsb3V0LHEsIGZpLCBuZXdbKCdhcycsYW8pXSBpZiBhbyBlbHNlIDApOyBxKz04CiAgICAgICAgZm9yIG1pLGFvIGluIG1ldGhvZHM6CiAgICAgICAgICAgIHN0cnVjdC5wYWNrX2ludG8oJzxJSScsb3V0LHEsIG1pLCBuZXdbKCdhcycsYW8pXSBpZiBhbyBlbHNlIDApOyBxKz04CiAgICAgICAgZm9yIG1pLGFvIGluIHBhcmFtczoKICAgICAgICAgICAgc3RydWN0LnBhY2tfaW50bygnPElJJyxvdXQscSwgbWksIG5ld1soJ2FyJyxhbyldIGlmIGFvIGVsc2UgMCk7IHErPTgKICAgICMgc3RhdGljIHZhbHVlcwogICAgZm9yIG9mZixibG9iIGluIHN2X2J5dGVzLml0ZW1zKCk6CiAgICAgICAgbm89bmV3Wygnc3YnLG9mZildOyBvdXRbbm86bm8rbGVuKGJsb2IpXT1ibG9iCiAgICAjIGNsYXNzX2RhdGEKICAgIGZvciBvZmYsYmxvYiBpbiBjZF9ieXRlcy5pdGVtcygpOgogICAgICAgIG5vPW5ld1soJ2NkJyxvZmYpXTsgb3V0W25vOm5vK2xlbihibG9iKV09YmxvYgogICAgIyBzdHJpbmdfZGF0YQogICAgZm9yIG9mZixibG9iIGluIHNkX2J5dGVzLml0ZW1zKCk6CiAgICAgICAgbm89bmV3Wygnc2QnLG9mZildOyBvdXRbbm86bm8rbGVuKGJsb2IpXT1ibG9iCgogICAgIyAtLS0tIGluZGV4IHRhYmxlcyAtLS0tCiAgICBmb3IgaSxzbyBpbiBlbnVtZXJhdGUoc3RyX29mZik6CiAgICAgICAgc3RydWN0LnBhY2tfaW50bygnPEknLG91dCxvX3N0cmlkcytpKjQsIG5ld1soJ3NkJyxzbyldIGlmIHNvIGluIHNkX2J5dGVzIGVsc2UgMCkKICAgIGZvciBpLHQgaW4gZW51bWVyYXRlKHR5cGVfaWRzKToKICAgICAgICBzdHJ1Y3QucGFja19pbnRvKCc8SScsb3V0LG9fdHlwaWRzK2kqNCwgdCkKICAgIGZvciBpLChzaCxydCxwbykgaW4gZW51bWVyYXRlKHByb3RvKToKICAgICAgICBzdHJ1Y3QucGFja19pbnRvKCc8SUlJJyxvdXQsb19wcm9pZHMraSoxMiwgc2gsIHJ0LCBuZXdbKCd0bCcscG8pXSBpZiBwbyBlbHNlIDApCiAgICBvdXRbb19mbGRpZHM6b19mbGRpZHMrbGVuKGZpZWxkX2lkcyldPWZpZWxkX2lkcwogICAgb3V0W29fbXRoaWRzOm9fbXRoaWRzK2xlbihtZXRob2RfaWRzKV09bWV0aG9kX2lkcwogICAgZm9yIGksY2QgaW4gZW51bWVyYXRlKGNsYXNzZGVmKToKICAgICAgICB0LGFjYyxzdXAsaWZjLHNyYyxhbm4sY2RhLHN2PWNkCiAgICAgICAgc3RydWN0LnBhY2tfaW50bygnPDhJJyxvdXQsb19jbHNkZWYraSozMiwKICAgICAgICAgICAgdCxhY2Msc3VwLAogICAgICAgICAgICBuZXdbKCd0bCcsaWZjKV0gaWYgaWZjIGVsc2UgMCwgc3JjLAogICAgICAgICAgICBuZXdbKCdhZCcsYW5uKV0gaWYgYW5uIGVsc2UgMCwKICAgICAgICAgICAgbmV3WygnY2QnLGNkYSldIGlmIGNkYSBlbHNlIDAsCiAgICAgICAgICAgIG5ld1soJ3N2JyxzdildIGlmIHN2IGVsc2UgMCkKCiAgICAjIC0tLS0gbWFwX2xpc3QgLS0tLQogICAgIyBidWlsZCBzZWN0aW9ucyBsaXN0ICh0eXBlLCBzaXplLCBvZmZzZXQpCiAgICBkZWYgcm5nKGtpbmQsIGNudCk6CiAgICAgICAgb2Zmcz1bbmV3WyhraW5kLGspXSBmb3IgayBpbiBuZXdfa2V5cyhuZXcsa2luZCldCiAgICAgICAgcmV0dXJuIG1pbihvZmZzKSBpZiBvZmZzIGVsc2UgMAogICAgc2Vjcz1bXQogICAgc2Vjcy5hcHBlbmQoKDB4MDAwMCwxLDApKQogICAgc2Vjcy5hcHBlbmQoKDB4MDAwMSxjLm5fc3RyLG9fc3RyaWRzKSkKICAgIHNlY3MuYXBwZW5kKCgweDAwMDIsYy5uX3R5cCxvX3R5cGlkcykpCiAgICBzZWNzLmFwcGVuZCgoMHgwMDAzLGMubl9wcm8sb19wcm9pZHMpKQogICAgc2Vjcy5hcHBlbmQoKDB4MDAwNCxjLm5fZmxkLG9fZmxkaWRzKSkKICAgIHNlY3MuYXBwZW5kKCgweDAwMDUsYy5uX210aCxvX210aGlkcykpCiAgICBzZWNzLmFwcGVuZCgoMHgwMDA2LGMubl9jbHMsb19jbHNkZWYpKQogICAgZGVmIGFkZHNlYyh0YyxraW5kKToKICAgICAgICBrZXlzPW5ld19rZXlzKG5ldyxraW5kKQogICAgICAgIGlmIGtleXM6CiAgICAgICAgICAgIG9mZnM9W25ld1soa2luZCxrKV0gZm9yIGsgaW4ga2V5c10KICAgICAgICAgICAgc2Vjcy5hcHBlbmQoKHRjLGxlbihrZXlzKSxtaW4ob2ZmcykpKQogICAgYWRkc2VjKDB4MjAwMSwnY29kZScpOyBhZGRzZWMoMHgxMDAxLCd0bCcpOyBhZGRzZWMoMHgyMDA0LCdhaScpCiAgICBhZGRzZWMoMHgxMDAzLCdhcycpOyBhZGRzZWMoMHgxMDAyLCdhcicpOyBhZGRzZWMoMHgyMDA2LCdhZCcpCiAgICBhZGRzZWMoMHgyMDA1LCdzdicpOyBhZGRzZWMoMHgyMDAwLCdjZCcpOyBhZGRzZWMoMHgyMDAyLCdzZCcpCiAgICBzZWNzLmFwcGVuZCgoMHgxMDAwLDEsb19tYXApKQogICAgc2Vjcy5zb3J0KGtleT1sYW1iZGEgczpzWzJdKQogICAgc3RydWN0LnBhY2tfaW50bygnPEknLG91dCxvX21hcCxsZW4oc2VjcykpOyBxPW9fbWFwKzQKICAgIGZvciB0YyxzeixvZiBpbiBzZWNzOgogICAgICAgIHN0cnVjdC5wYWNrX2ludG8oJzxISElJJyxvdXQscSx0YywwLHN6LG9mKTsgcSs9MTIKICAgIGZpbmFsPXEKICAgIGRlbCBvdXRbZmluYWw6XSAgICAgICAgICAgIyB0cnVuY2F0ZSB0byByZWFsIGVuZAogICAgdG90YWw9ZmluYWwKCiAgICAjIC0tLS0gaGVhZGVyIC0tLS0KICAgIHN0cnVjdC5wYWNrX2ludG8oJzw4cycsb3V0LDAsYidkZXhcbjAzOVx4MDAnKQogICAgc3RydWN0LnBhY2tfaW50bygnPDIwSScsb3V0LDMyLAogICAgICAgIHRvdGFsLDB4NzAsMHgxMjM0NTY3OCwgMCwwLCBvX21hcCwKICAgICAgICBjLm5fc3RyLG9fc3RyaWRzLCBjLm5fdHlwLG9fdHlwaWRzLCBjLm5fcHJvLG9fcHJvaWRzLAogICAgICAgIGMubl9mbGQsb19mbGRpZHMsIGMubl9tdGgsb19tdGhpZHMsIGMubl9jbHMsb19jbHNkZWYsCiAgICAgICAgdG90YWwtZGF0YV9zdGFydCwgZGF0YV9zdGFydCkKICAgICMgc2lnbmF0dXJlIChzaGExIG9mIGJ5dGVzWzMyOl0pIHRoZW4gY2hlY2tzdW0gKGFkbGVyMzIgb2YgYnl0ZXNbMTI6XSkKICAgIHNpZz1oYXNobGliLnNoYTEoYnl0ZXMob3V0WzMyOl0pKS5kaWdlc3QoKQogICAgb3V0WzEyOjMyXT1zaWcKICAgIGNzPXpsaWIuYWRsZXIzMihieXRlcyhvdXRbMTI6XSkpJjB4ZmZmZmZmZmYKICAgIHN0cnVjdC5wYWNrX2ludG8oJzxJJyxvdXQsOCxjcykKICAgIG9wZW4ob3V0cGF0aCwnd2InKS53cml0ZShvdXQpCiAgICByZXR1cm4gdG90YWwKCmRlZiBjX2NvZGVfb2ZmcyhjKToKICAgIGI9Yy5iOyBvZmZzPXNldCgpCiAgICBmb3IgaSBpbiByYW5nZShjLm5fY2xzKToKICAgICAgICBjZD1zdHJ1Y3QudW5wYWNrX2Zyb20oJzw4SScsYixjLm9fY2xzK2kqMzIpWzZdCiAgICAgICAgaWYgbm90IGNkOiBjb250aW51ZQogICAgICAgIG89Yy5EKGNkKTsgc2Ysbz11bGViKGIsbyk7IGluZixvPXVsZWIoYixvKTsgZG0sbz11bGViKGIsbyk7IHZtLG89dWxlYihiLG8pCiAgICAgICAgZm9yIF8gaW4gcmFuZ2Uoc2YpOiBfLG89dWxlYihiLG8pOyBfLG89dWxlYihiLG8pCiAgICAgICAgZm9yIF8gaW4gcmFuZ2UoaW5mKTogXyxvPXVsZWIoYixvKTsgXyxvPXVsZWIoYixvKQogICAgICAgIGZvciBfIGluIHJhbmdlKGRtK3ZtKToKICAgICAgICAgICAgXyxvPXVsZWIoYixvKTsgXyxvPXVsZWIoYixvKTsgY28sbz11bGViKGIsbykKICAgICAgICAgICAgaWYgY286IG9mZnMuYWRkKGNvKQogICAgcmV0dXJuIG9mZnMKZGVmIG5ld19rZXlzKG5ldyxraW5kKToKICAgIHJldHVybiBba1sxXSBmb3IgayBpbiBuZXcgaWYga1swXT09a2luZF0KCgppZiBfX25hbWVfXz09J19fbWFpbl9fJzoKICAgIGltcG9ydCBzeXMKICAgIGlmIGxlbihzeXMuYXJndikhPTM6CiAgICAgICAgcHJpbnQoInVzYWdlOiBweXRob24zIGNkZXgyZGV4LnB5IDxpbnB1dC5jZGV4PiA8b3V0cHV0LmRleD4iKTsgc3lzLmV4aXQoMSkKICAgIHRvdGFsPXNlcmlhbGl6ZShzeXMuYXJndlsxXSxzeXMuYXJndlsyXSkKICAgIGM9Q29udihzeXMuYXJndlsxXSk7IG9rPW9wZW4oc3lzLmFyZ3ZbMl0sJ3JiJykucmVhZCg4KT09YidkZXhcbjAzOVx4MDAnCiAgICBwcmludCgiW09LXSB3cm90ZSAlcyAgKCVkIGJ5dGVzKSIlKHN5cy5hcmd2WzJdLHRvdGFsKSkKICAgIHByaW50KCIgICAgIG1hZ2ljIGRleCAwMzkgOiAlcyIlb2spCiAgICBwcmludCgiICAgICBjbGFzc2VzPSVkIG1ldGhvZHM9JWQgc3RyaW5ncz0lZCIlKGMubl9jbHMsYy5uX210aCxjLm5fc3RyKSkK | base64 -d > cdex2dex.py
wc -l cdex2dex.py
```

> [!TIP]
> The script parses the CompactDex container, converts every code item to the standard DEX layout, rebuilds the string/type/method tables and the map list, and recomputes the checksum and SHA-1 signature. Debug info (line numbers, local names) is dropped — not needed for execution, and it avoids CompactDex's trickiest section.

### 5.3 — vdex → cdex

vdexExtractor un-quickens the bytecode by default (turns ART's optimised opcodes back into standard ones), which is exactly what we need.

```bash
cd "$BASE/svc"
~/vdexExtractor/bin/vdexExtractor -i services.vdex -o . -f --ignore-crc-error
ls -lh "$BASE/svc"/*.cdex
```

> [!TIP]
> You should see `[INFO] 1 Dex files have been extracted in total` and a `services.vdex_classes.cdex` (~9.6MB). The `.cdex` extension is the giveaway — it is still CompactDex, not standard DEX. That is what the next step fixes.

### 5.4 — cdex → standard DEX

```bash
cd "$BASE/svc"
python3 cdex2dex.py services.vdex_classes.cdex services.dex
```

> [!TIP]
> Expected: `[OK] wrote services.dex (9214036 bytes)` and `magic dex 039 : True`. The numbers vary by firmware version. What matters is `magic dex 039 : True` — the output is now standard DEX.

> [!NOTE]
> The converter is verified against ground truth: for every method, the incoming-argument register count (`ins`) is fixed by the method signature. Its output matches the signature-derived value for all 55,217 methods, and no code item ends up with `registers < ins` — the exact error that would make reassembly fail.

### 5.5 — DEX → smali

```bash
cd "$BASE/svc"
rm -rf smali_out
baksmali d services.dex -o smali_out
ls smali_out/com/android/server/pm/PackageManagerServiceUtils.smali && echo "BAKSMALI OK"
```

No errors and the `.smali` file listed means the conversion round-trip is sound.

### 5.6 — Patch the 4 changes

Three of the changes replace a method body with a stub; the fourth rewrites a branch inside `installPackageLI`. All are in `PackageManagerService`:

| Method | Original | Patched to |
|---|---|---|
| `PackageManagerServiceUtils.compareSignatures([Signature;[Signature;)I` | compares two signature sets | `return 0` (SIGNATURE_MATCH) |
| `PackageManagerService.compareSignaturesWithAco([Signature;)I` | ECarX signature check | `return 0` |
| `PackageManagerService.reconcileApps(Ljava/lang/String;)V` | reconcile check on install | `return-void` (no-op) |
| `PackageManagerService.installPackageLI` — aco whitelist branch | rejects packages not on the whitelist | redirect all branches past the reject |

In Android, `compareSignatures` returning `0` means "signatures match". Force it to always return `0` and the installer stops rejecting foreign-signed APKs. The other changes are backups so nothing else re-blocks the install.

#### 5.6.1 — The three stubs

```bash
cd "$BASE/svc"
echo aW1wb3J0IHN5cwpTPSJzbWFsaV9vdXQvY29tL2FuZHJvaWQvc2VydmVyL3BtL1BhY2thZ2VNYW5hZ2VyU2VydmljZS5zbWFsaSIKVT0ic21hbGlfb3V0L2NvbS9hbmRyb2lkL3NlcnZlci9wbS9QYWNrYWdlTWFuYWdlclNlcnZpY2VVdGlscy5zbWFsaSIKZWRpdHM9WwogKFUsImNvbXBhcmVTaWduYXR1cmVzKFtMYW5kcm9pZC9jb250ZW50L3BtL1NpZ25hdHVyZTtbTGFuZHJvaWQvY29udGVudC9wbS9TaWduYXR1cmU7KUkiLAogICAgWyIgICAgLmxvY2FscyAxIiwiICAgIGNvbnN0LzQgdjAsIDB4MCIsIiAgICByZXR1cm4gdjAiXSksCiAoUywiY29tcGFyZVNpZ25hdHVyZXNXaXRoQWNvKFtMYW5kcm9pZC9jb250ZW50L3BtL1NpZ25hdHVyZTspSSIsCiAgICBbIiAgICAubG9jYWxzIDEiLCIgICAgY29uc3QvNCB2MCwgMHgwIiwiICAgIHJldHVybiB2MCJdKSwKIChTLCJyZWNvbmNpbGVBcHBzKExqYXZhL2xhbmcvU3RyaW5nOylWIiwKICAgIFsiICAgIC5sb2NhbHMgMCIsIiAgICByZXR1cm4tdm9pZCJdKSwKXQpmb3IgcGF0aCxzaWcsYm9keSBpbiBlZGl0czoKICAgIGxpbmVzPW9wZW4ocGF0aCkucmVhZCgpLnNwbGl0KCJcbiIpCiAgICBvdXQ9W107IGk9MDsgaGl0PUZhbHNlCiAgICB3aGlsZSBpPGxlbihsaW5lcyk6CiAgICAgICAgbG49bGluZXNbaV0KICAgICAgICBpZiBsbi5sc3RyaXAoKS5zdGFydHN3aXRoKCIubWV0aG9kIikgYW5kIGxuLnJzdHJpcCgpLmVuZHN3aXRoKHNpZyk6CiAgICAgICAgICAgIGo9aSsxCiAgICAgICAgICAgIHdoaWxlIGxpbmVzW2pdLnN0cmlwKCkhPSIuZW5kIG1ldGhvZCI6IGorPTEKICAgICAgICAgICAgb3V0LmFwcGVuZChsbik7IG91dCs9Ym9keTsgb3V0LmFwcGVuZCgiLmVuZCBtZXRob2QiKQogICAgICAgICAgICBpPWorMTsgaGl0PVRydWU7IHByaW50KCJQQVRDSEVEOiIsc2lnKTsgY29udGludWUKICAgICAgICBvdXQuYXBwZW5kKGxuKTsgaSs9MQogICAgb3BlbihwYXRoLCJ3Iikud3JpdGUoIlxuIi5qb2luKG91dCkpCiAgICBpZiBub3QgaGl0OiBwcmludCgiTk9UIEZPVU5EOiIsc2lnKQo= | base64 -d > patch_smali.py
cat patch_smali.py
python3 patch_smali.py
```

The script finds each method by its exact descriptor, deletes everything from `.method` to `.end method`, and writes the stub in its place. Expected:

```text
PATCHED: compareSignatures([Landroid/content/pm/Signature;[Landroid/content/pm/Signature;)I
PATCHED: compareSignaturesWithAco([Landroid/content/pm/Signature;)I
PATCHED: reconcileApps(Ljava/lang/String;)V
```

> [!CAUTION]
> If any line says `NOT FOUND`, the method descriptor differs in your firmware version. Run `grep -n "compareSignatures" "$BASE/svc/smali_out/com/android/server/pm/"*.smali` and adjust the descriptor in `patch_smali.py`.

#### 5.6.2 — The fourth change: bypass the aco whitelist

Stubbing the three signature methods is **not enough on their own**. `installPackageLI` holds a second, separate check: a whitelist of package names, and any package not on it is rejected with `"signatures do not match the aco version; ignoring!"`. Sideloaded apps (Magisk, etc.) are never on that list, so this check must be neutralised too. This script redirects all three branch paths to the safe continuation label:

```bash
cd "$BASE/svc"
cat > patch_aco.py <<'PY'
p="smali_out/com/android/server/pm/PackageManagerService.smali"
L=open(p).readlines()
# 1. find the aco reject block and the goto that jumps into it
err=next(i for i,l in enumerate(L) if 'do not match the aco version; ignoring!' in l)
j=err
while '-0x16' not in L[j]: j-=1
k=j
while not L[k].strip().startswith(':'): k-=1
elabel=L[k].strip()[1:]
g=next(i for i in range(k) if L[i].strip() in ('goto/16 :'+elabel,'goto :'+elabel))
# 2. the if- branch just above the jump -> make it an unconditional goto to the safe label
m=g-1
while L[m].strip()=='' or L[m].strip().startswith('move') or L[m].strip().startswith('.'):
    m-=1
br=L[m].strip()
assert br.startswith('if-'), 'unexpected: '+br
safe=br.split(':')[-1]
L[m]='    goto/16 :'+safe+'\n'
# 3. redirect the two whitelist guards above (they point at the reject-fallthrough label) to the same safe label
gl=None
for jj in range(m-1, max(0,m-18), -1):
    s=L[jj].strip()
    if s.startswith('if-') and ':' in s:
        t=s.split(':')[-1]
        if t!=safe: gl=t; break
c=0
if gl:
    for jj in range(m-1, max(0,m-18), -1):
        if L[jj].strip().startswith('if-') and L[jj].rstrip().endswith(':'+gl):
            L[jj]=L[jj].rsplit(':',1)[0]+':'+safe+'\n'; c+=1
open(p,'w').writelines(L)
print("aco branch -> goto :%s ; guard redirects: %d" % (safe,c))
PY
python3 patch_aco.py
```

> [!TIP]
> Expected: `aco branch -> goto :cond_XXXX ; guard redirects: 2`. All three paths (the two whitelist guards and the signature branch) now jump to the safe label, so the aco reject block can never run. Label names vary by firmware — the script finds them automatically.

> [!WARNING]
> If it prints `guard redirects: 0`, stop and re-check before deploying.

### 5.7 — smali → jar

```bash
cd "$BASE/svc"
(
set -e
rm -f classes.dex services.jar
smali a -a 28 smali_out -o classes.dex
[ -s classes.dex ]
rm -rf jar && mkdir -p jar/META-INF
cp classes.dex jar/
printf 'Manifest-Version: 1.0\r\nCreated-By: ArdentLab\r\n\r\n' > jar/META-INF/MANIFEST.MF
( cd jar && zip -rX ../services.jar classes.dex META-INF >/dev/null )
ls -lh services.jar
unzip -l services.jar
)
```

> [!WARNING]
> **The `-a 28` is not optional.** Without it, smali defaults to `dex 035`. Android 9 will load a 035 framework jar and boot, but under older verification rules the patches never take effect — installs still fail with the aco error. `-a 28` (API 28) produces `dex 039`, which the framework expects.

### 5.8 — Verify before deploying

A wrong `services.jar` can bootloop the IHU, so confirm the stubs are in the final DEX:

```bash
cd "$BASE/svc"
rm -rf verify && baksmali d classes.dex -o verify
for m in compareSignatures compareSignaturesWithAco reconcileApps; do
  echo "--- $m ---"
  grep -rA3 "\.method .*$m(" verify/com/android/server/pm/ | grep -E 'const/4|return' | head -2
done

echo "--- dex version (must be 039) ---"
unzip -p services.jar classes.dex | head -c 8 | xxd
echo "--- aco string still present (expected) — branches redirected in 5.6.2 ---"
grep -n "the aco version; ignoring" verify/com/android/server/pm/PackageManagerService.smali
```

> [!TIP]
> Each `compareSignatures*` method should show `const/4 v0, 0x0` then `return v0`; `reconcileApps` shows `return-void`. The `xxd` line must show `6465 780a 3033 3900` = `dex 039`. If it ends `3033 3500` (035), re-run the assemble step with `-a 28`.

### 5.9 — Stage the finished jar

```bash
cp "$BASE/svc/services.jar" "$BASE/MOD/services.jar"
ls -lh "$BASE/MOD/services.jar"
```

> [!CAUTION]
> It must be about **3.7MB**, definitely not 183 bytes (the stub). Confirm it is `dex 039`: `unzip -p "$BASE/MOD/services.jar" classes.dex | head -c 8 | xxd` must read `dex 039`. A `dex 035` build boots but leaves installs blocked.


> [!IMPORTANT]
> **Next physical step: jump to [Step 8.1–8.3 (Session A — read-only dump)](#83--session-a-back-up-everything-read-only) now.** Do **not** run Step 6 or Step 7 yet — `ORI/lk.bin` and `ORI/boot.bin` do not exist until you dump your unit's partitions in 8.3 and copy them into `ORI/`. Once 8.3 finishes and the unit is powered down, come back to Step 6.

---

## Step 6 — Patch lk.bin

`Shell`

> [!CAUTION]
> **Do the read-only dump first.** `ORI/lk.bin` and `ORI/boot.bin` must be copies of *your* unit's partitions, taken in [8.1–8.3](#83--session-a-back-up-everything-read-only). If `ORI/` is still empty, stop here, complete the dump and verification in [8.3](#83--session-a-back-up-everything-read-only), copy `DUMP/lk.bin` and `DUMP/boot.bin` into `ORI/`, and only then continue. Patching an image you did not dump from this unit is the fastest way to make an unrecoverable unit.

After the bootloader is unlocked, the IHU shows an **orange state** warning screen on every boot. This patch removes it for good. The script finds the offset by itself — nothing to edit.

> [!WARNING]
> **How it works:** The script scans `lk.bin` for the byte pattern `0E 4B 7B 44`, then confirms the right one by checking for the ARM Thumb function prologue `08 B5` (`PUSH {r3,lr}`) just before it. Those four bytes are replaced with `00 20 08 BD` (`MOVS r0,#0 / POP {r3,pc}`), which makes the check function return immediately.

> [!TIP]
> **Confirmed offsets:** X50 RC V144 `0x0003B392`; S70 V333 and X90 V735 `0x0003C0FE`. The offset differs per model — the script handles it.

**Scan before patching.** A pattern that matches more than once, or at an offset that is not in the confirmed list, means the firmware is not verified for this patch. The scan is read-only and refuses to continue in either case:

```bash
export BASE="${BASE:-$HOME/IHU_DEPLOY}"
python3 - "$BASE" <<'PY'
import sys, os
path = os.path.join(sys.argv[1], "ORI", "lk.bin")
if not os.path.isfile(path):
    print("STOP: ORI/lk.bin missing — do the read-only dump in 8.3 first")
    sys.exit(1)
b = open(path, "rb").read()
PAT, PRO = bytes([0x0E,0x4B,0x7B,0x44]), bytes([0x08,0xB5])
c = [i for i in range(len(b)) if b.startswith(PAT, i) and PRO in b[max(0,i-16):i]]
print("candidates:", [hex(x) for x in c])
if len(c) != 1:
    print("STOP: expected exactly one candidate. Do not patch or flash this lk.bin.")
    sys.exit(1)
if c[0] not in (0x3B392, 0x3C0FE):
    print("STOP: offset 0x%X is not in the confirmed list. Do not flash." % c[0])
    sys.exit(1)
print("OK: unique confirmed candidate at 0x%X" % c[0])
PY
```

> [!CAUTION]
> If the scan prints `STOP`, **do not run the next block and do not proceed**. A `STOP` from Python does not stop your shell, so the decision is yours: a second match or an unconfirmed offset means this `lk.bin` is not proven safe to patch.

**Make the copy, then patch only `MOD/lk.bin`:**

```bash
rm -f "$BASE/MOD/lk.bin"
cp "$BASE/ORI/lk.bin" "$BASE/MOD/lk.bin"
echo aW1wb3J0IHN5cywgb3MKYmFzZSA9IG9zLmVudmlyb25bIkJBU0UiXQpvcmlfcGF0aCA9IGJhc2UgKyAiL09SSS9say5iaW4iCm1vZF9wYXRoID0gYmFzZSArICIvTU9EL2xrLmJpbiIKZGVmIGZhaWwobXNnKToKICAgIHByaW50KG1zZykKICAgIGlmIG9zLnBhdGguZXhpc3RzKG1vZF9wYXRoKToKICAgICAgICBvcy5yZW1vdmUobW9kX3BhdGgpCiAgICBzeXMuZXhpdCgxKQppZiBub3Qgb3MucGF0aC5pc2ZpbGUob3JpX3BhdGgpOgogICAgZmFpbCgiRVJST1I6IE9SSS9say5iaW4gbWlzc2luZyDigJQgY29tcGxldGUgdGhlIHJlYWQtb25seSBkdW1wIGluIDguMyBmaXJzdC4iKQpvcmkgPSBvcGVuKG9yaV9wYXRoLCAicmIiKS5yZWFkKCkKUEFUVEVSTiAgPSBieXRlcyhbMHgwRSwgMHg0QiwgMHg3QiwgMHg0NF0pClBBVENIICAgID0gYnl0ZXMoWzB4MDAsIDB4MjAsIDB4MDgsIDB4QkRdKQpQUk9MT0dVRSA9IGJ5dGVzKFsweDA4LCAweEI1XSkKbWF0Y2hlcyA9IFtdCnBvcyA9IDAKd2hpbGUgVHJ1ZToKICAgIGlkeCA9IG9yaS5maW5kKFBBVFRFUk4sIHBvcykKICAgIGlmIGlkeCA9PSAtMToKICAgICAgICBicmVhawogICAgaWYgUFJPTE9HVUUgaW4gb3JpW21heCgwLCBpZHggLSAxNik6aWR4XToKICAgICAgICBtYXRjaGVzLmFwcGVuZChpZHgpCiAgICBwb3MgPSBpZHggKyAxCmlmIGxlbihtYXRjaGVzKSA9PSAwOgogICAgZmFpbCgiRVJST1I6IFBhdHRlcm4gbm90IGZvdW5kLiBUaGlzIGZpcm13YXJlIHZlcnNpb24gbWF5IG5vdCBiZSBzdXBwb3J0ZWQuXG4gICAgICAgRG8gbm90IGNvbnRpbnVlLiBNYWtlIHN1cmUgbGsuYmluIGNhbWUgZnJvbSB5b3VyIG93biB1bml0LiIpCmlmIGxlbihtYXRjaGVzKSAhPSAxOgogICAgZmFpbCgiRVJST1I6IE11bHRpcGxlIHBhdHRlcm4gbWF0Y2hlcyBmb3VuZDogJXNcbiAgICAgICBEbyBub3QgcGF0Y2ggb3IgZmxhc2ggdGhpcyBsay5iaW4uIiAlIFtoZXgoeCkgZm9yIHggaW4gbWF0Y2hlc10pCmZvdW5kID0gbWF0Y2hlc1swXQppZiBmb3VuZCBub3QgaW4gKDB4M0IzOTIsIDB4M0MwRkUpOgogICAgZmFpbCgiRVJST1I6IE9mZnNldCAweCUwOFggaXMgbm90IGluIHRoZSBjb25maXJtZWQgbGlzdCAoMHgwMDAzQjM5MiwgMHgwMDAzQzBGRSkuXG4gICAgICAgRG8gbm90IHBhdGNoIG9yIGZsYXNoIHRoaXMgbGsuYmluLiIgJSBmb3VuZCkKcHJpbnQoIlsrXSBGb3VuZCBhdCBvZmZzZXQgOiAweCUwOFggKGRlY2ltYWw6ICVkKSIgJSAoZm91bmQsIGZvdW5kKSkKcHJpbnQoIlsrXSBPcmlnaW5hbCBieXRlcyAgOiAiICsgb3JpW2ZvdW5kOmZvdW5kKzRdLmhleCgiICIpLnVwcGVyKCkpCndpdGggb3Blbihtb2RfcGF0aCwgInIrYiIpIGFzIGY6CiAgICBmLnNlZWsoZm91bmQpCiAgICBjdXIgPSBmLnJlYWQoNCkKICAgIGlmIGN1ciA9PSBQQVRDSDoKICAgICAgICBwcmludCgiWyFdIEFscmVhZHkgcGF0Y2hlZCAtIG5vdGhpbmcgY2hhbmdlZC4iKQogICAgZWxpZiBjdXIgIT0gUEFUVEVSTjoKICAgICAgICBmYWlsKCJFUlJPUjogVW5leHBlY3RlZCBieXRlcyBpbiBNT0QgZmlsZTogIiArIGN1ci5oZXgoIiAiKS51cHBlcigpICsgIlxuICAgICAgIERlbGV0ZSBNT0QvbGsuYmluLCBjb3B5IGl0IGFnYWluIGZyb20gT1JJLCB0aGVuIHJldHJ5LiIpCiAgICBlbHNlOgogICAgICAgIGYuc2Vlayhmb3VuZCkKICAgICAgICBmLndyaXRlKFBBVENIKQogICAgICAgIHByaW50KCJbK10gUGF0Y2ggd3JpdHRlbiAgIDogIiArIFBBVENILmhleCgiICIpLnVwcGVyKCkpCndpdGggb3Blbihtb2RfcGF0aCwgInJiIikgYXMgZjoKICAgIGYuc2Vlayhmb3VuZCkKICAgIHJlcyA9IGYucmVhZCg0KQppZiByZXMgPT0gUEFUQ0g6CiAgICBwcmludCgiIikKICAgIHByaW50KCJbT0tdIFBBVENIIFNVQ0NFU1NGVUwiKQogICAgcHJpbnQoIiAgICAgT2Zmc2V0ICAgOiAweCUwOFgiICUgZm91bmQpCiAgICBwcmludCgiICAgICBCeXRlcyAgICA6ICIgKyByZXMuaGV4KCIgIikudXBwZXIoKSkKICAgIHByaW50KCIgICAgIE9SSSBmaWxlIDogdW50b3VjaGVkIikKZWxzZToKICAgIGZhaWwoIkVSUk9SOiBWZXJpZmljYXRpb24gZmFpbGVkIC0gYnl0ZXMgYWZ0ZXIgd3JpdGU6ICIgKyByZXMuaGV4KCIgIikudXBwZXIoKSkK | base64 -d | python3
```

> [!TIP]
> Expected: `[+] Found at offset : 0x0003B392`, `[+] Patch written : 00 20 08 BD`, and `[OK] PATCH SUCCESSFUL`. The line that matters is `[OK] PATCH SUCCESSFUL`.

**Prove the patch is exactly four bytes and nothing else moved:**

```bash
python3 - "$BASE" <<'PY'
import sys, os
a = open(os.path.join(sys.argv[1], "ORI", "lk.bin"), "rb").read()
b = open(os.path.join(sys.argv[1], "MOD", "lk.bin"), "rb").read()
assert len(a) == len(b), "STOP: MOD/lk.bin size differs from ORI/lk.bin"
d = [i for i in range(len(a)) if a[i] != b[i]]
assert len(d) == 4, "STOP: expected 4 changed bytes, found %d" % len(d)
assert b[d[0]:d[0]+4] == bytes([0x00,0x20,0x08,0xBD]), "STOP: wrong replacement bytes"
print("lk diff OK: only 4 bytes changed at 0x%X" % d[0])
PY
```

> [!CAUTION]
> If it says "Pattern not found", "not in the confirmed list", or any `STOP`: stop. Either your firmware version is not confirmed yet, or that `lk.bin` did not come from your unit. Do not continue.

---

## Step 7 — Patch boot.bin (Magisk)

`Android phone`

Magisk gets in by patching the boot image with the Magisk app on a phone. This is what disables dm-verity — without it, `/system` will never become writable, even with root.

- Connect the phone to the PC, set USB mode to **File Transfer (MTP)**
- Copy `ORI/boot.bin` into the phone's `Downloads` folder
- Open **Magisk** → tap **Install** next to Magisk at the top

> [!CAUTION]
> **An Options screen appears here — before you tap Next:** Make sure `Preserve AVB 2.0 / dm-verity` is **NOT ticked**. Leave it ticked and `/system` will refuse to remount rw, and you will get stuck at [Step 10](#step-10--get-root--remount-rw).

- Tap **Next** → **Select and Patch a File** → go to **Downloads**, pick `boot.bin`
- Tap **LET'S GO** and wait for it to finish. Magisk saves `magisk_patched-XXXXX_YYYYY.img` (or `magisk_patched_XXXXX.img` on older Magisk releases) in `Download`
- Copy that file back to the PC (for example into `~/Downloads` or `$BASE`), or read it straight from the MTP mount, then verify and stage it as `MOD/boot.bin`:

```bash
export BASE="${BASE:-$HOME/IHU_DEPLOY}"
ls -lt "$HOME"/Downloads/magisk_patched*.img \
       "$HOME"/Download/magisk_patched*.img \
       "$BASE"/magisk_patched*.img \
       /run/user/"$UID"/gvfs/*/*/Download/magisk_patched*.img \
       /run/user/"$UID"/gvfs/*/Download/magisk_patched*.img \
       /media/"$USER"/*/Download/magisk_patched*.img \
       /run/media/"$USER"/*/Download/magisk_patched*.img 2>/dev/null
```

Pick the file the Magisk app just wrote. `ls -lt` above is newest-first; do not blindly take the top entry if more than one phone or patch has been used — check the timestamp for the patch you just ran. Then copy and verify it in a guarded subshell:

```bash
(
set -e
export BASE="${BASE:-$HOME/IHU_DEPLOY}"
[ -f "$BASE/ORI/boot.bin" ] || { echo "STOP: ORI/boot.bin missing — do the read-only dump in 8.3 first"; exit 1; }
src=$(ls -t "$HOME"/Downloads/magisk_patched*.img \
            "$HOME"/Download/magisk_patched*.img \
            "$BASE"/magisk_patched*.img \
            /run/user/"$UID"/gvfs/*/*/Download/magisk_patched*.img \
            /run/user/"$UID"/gvfs/*/Download/magisk_patched*.img \
            /media/"$USER"/*/Download/magisk_patched*.img \
            /run/media/"$USER"/*/Download/magisk_patched*.img 2>/dev/null | head -1)
[ -n "$src" ] && [ -f "$src" ] || { echo "STOP: no magisk_patched*.img found — copy it to ~/Downloads first"; exit 1; }
echo "using: $src"
ls -lh "$src"
rm -f "$BASE/MOD/boot.bin"
cp "$src" "$BASE/MOD/boot.bin"
[ -f "$BASE/MOD/boot.bin" ] || { echo "STOP: MOD/boot.bin was not created"; exit 1; }
[ "$(stat -c%s "$BASE/MOD/boot.bin")" -le "$(stat -c%s "$BASE/ORI/boot.bin")" ] \
  || { echo "STOP: patched boot is larger than the boot partition"; rm -f "$BASE/MOD/boot.bin"; exit 1; }
[ "$(stat -c%s "$BASE/MOD/boot.bin")" -ge $(( $(stat -c%s "$BASE/ORI/boot.bin") / 4 )) ] \
  || { echo "STOP: patched boot looks truncated — recopy it"; rm -f "$BASE/MOD/boot.bin"; exit 1; }
if cmp -s "$BASE/ORI/boot.bin" "$BASE/MOD/boot.bin"; then
  echo "STOP: MOD/boot.bin is identical to ORI — the Magisk patch did not happen"
  rm -f "$BASE/MOD/boot.bin"
  exit 1
fi
python3 - "$BASE/ORI/boot.bin" "$BASE/MOD/boot.bin" <<'PY'
import sys
for p in sys.argv[1:]:
    if open(p, "rb").read(8) != b"ANDROID!":
        print("STOP: %s does not start with ANDROID! boot magic" % p)
        sys.exit(1)
print("boot image magic OK")
PY
ls -lh "$BASE/MOD/"
)
```

> [!CAUTION]
> **The patched boot must never be larger than `ORI/boot.bin`.** mtkclient writes `min(file size, partition size)`; anything larger is silently truncated and the unit will not boot. If the size check above trips, redo the Magisk patch — usually the Magisk version is too new for Android 9, or the wrong file was selected.

> [!TIP]
> Renaming `.img` to `.bin` is completely safe — the extension is only a label. Your `MOD` folder should now hold `lk.bin` (~1MB), `boot.bin` (~32MB) and `services.jar` (~3.7MB). On Linux MTP usually appears under `/run/user/$UID/gvfs/` or in your file manager; use whichever route copies the two files.

> [!IMPORTANT]
> **Next physical step:** Now that `MOD/lk.bin`, `MOD/boot.bin` and `MOD/services.jar` are ready, continue to [Step 8.4 (Session B — preflight, erase & unlock)](#84--session-b-erase--unlock-no-image-written).

---

## Step 8 — BROM: back up, erase, unlock, flash

`mtkclient CLI`

This step has **three separate BROM sessions**: A (read-only dump — run it before Step 6), B (erase + unlock, nothing written), and C (write `lk` + `boot`, then read back). Between sessions the unit is powered off and the shorts stay in place; each power-on gives the BROM one fresh handshake. Nothing destructive runs until the files it needs have been verified.

> [!CAUTION]
> **Power must stay stable.** Never let the IHU lose power during a BROM operation. Unlocking the bootloader **wipes userdata** — settings and Bluetooth pairings are gone. That is expected and cannot be undone.

### 8.1 — What the CLI commands do

| Repo GUI action | CLI equivalent (used below) |
|---|---|
| Tick all partitions, untick userdata, Read partition(s) | `rl <dir> --skip userdata` (writes `gpt.bin` too) |
| Erase metadata / userdata / md_udc | `e metadata,userdata,md_udc` |
| Flash Tools → Unlock bootloader | `da seccfg unlock` |
| Add files manually → Write lk / boot | `w lk <file>` / `w boot <file>` |
| Read back a partition to verify | `r lk <file>` / `r boot <file>` |
| Back up the GPT area completely | `r gpt <file>` |

> [!CAUTION]
> **`mtk multi` splits on spaces, not shell quoting.** Commands are separated by `;` with **no space after it**, and each command's arguments are separated by single spaces. `w lk,<file>` is invalid in this version — `w` needs the partition and the file as two arguments. A path containing a space, a semicolon or a comma also breaks the split. This is not cosmetic: with the wrong spacing, `mtk multi` aborts after the commands that already ran.

`mtk multi "<cmd>;<cmd>;..."` connects to the device **once** and runs every command in the same DA session. That matters: the MediaTek BROM accepts its handshake exactly once per power-on, and the ECarX E02 does not re-enumerate its USB port — so one session per phase is the reliable approach.

### 8.2 — Enter BROM mode

| Pin | Colour | Function | When used |
|---|---|---|---|
| 1 | White | GND | Short to blue for BROM |
| 2 | Blue | Recovery | Short to white (GND) |
| 3 | Green | USB | Short to red (3.3V) to switch the USB port into device mode |
| 4 | Black | UART RX | Not used here |
| 5 | Yellow | UART TX | Not used here |
| 6 | Red | 3.3V | Short with green |

On the dedicated 4-wire JST jumper, short **white (GND) ↔ blue (Recovery)** and **green (USB) ↔ red (3.3V)**. Yellow and black are not fitted — there is no UART during a BROM session, and fewer wires means fewer chances for a stray short.

![Micro JST GH cable with white joined to blue and green joined to red, bare copper still showing](../images/jst-jumper-shorted.jpg)

*The two joins: white↔blue and green↔red. This is the only pair combination allowed.*

![The same jumper with each shorted pair sealed inside its own piece of heat shrink](../images/jst-jumper-shorted-insulated.jpg)

*Each join in its own sleeve, no copper visible. This is what goes into a powered unit.*

**Socket Block A — Power** is the black connector at the bottom of the ISO stack:

![ISO connector block with the black power socket highlighted](../images/power-socket-highlighted.jpg)

- **Disconnect IHU power.** Unplug **Socket Block A — Power**, the black connector sitting lowest in the ISO stack. Pulling the socket is what guarantees a true cold boot; standby is not enough.
- **Make both shorts on the JST jumper** — twisted and tinned, or soldered. A jumper you have to pinch by hand will let go at the worst possible moment, and a short that opens mid-session is exactly how a unit gets bricked. Sleeve the two joins **separately**.
- **Prove the joins with a multimeter before powering anything.** White↔blue must read a dead short; green↔red must read a dead short; every other pair must read open. If any other pair reads shorted, stop and rebuild the jumper.
- **Plug the JST jumper into the IHU port.** The CH340/UART cable is unplugged for the whole BROM phase — one connector, one purpose. Do not leave the adapter attached to the side of the unit.
- **Stop ModemManager and activate the mtkclient venv** before powering the unit:

```bash
sudo systemctl stop ModemManager brltty 2>/dev/null || true
source ~/mtkclient/.venv/bin/activate
```

- **Start the `mtk multi ...` command for your current session first** ([8.3](#83--session-a-back-up-everything-read-only), [8.4](#84--session-b-erase--unlock-no-image-written) or [8.5](#85--session-c-write-lk--boot-then-read-back)), and wait until it prints `Status: Waiting for PreLoader VCOM, please reconnect mobile/iot device to brom mode`. Having `mtk` already polling before power-on lets it claim the BROM USB interface the instant it enumerates, before the kernel's `cdc_acm` driver or any background service can touch it.
- **Plug the POWER socket back in.**
- **Immediately plug the USB A-to-A cable**: IHU USB port → a **USB 2.0** port on the PC.

  ![A laptop connected by a USB A-to-A cable to the car's centre-console USB socket, with the gear selector in P](../images/usb-a-to-a-laptop-to-ihu.jpg)

  *If working in the car rather than on a bench, use the **centre-console USB socket**. It is the **only** socket wired to the IHU data lines; every other USB port in the car is charge-only and will never enumerate.*

- `mtk` detects the BROM endpoint (vendor `0e8d`, product `0003`), completes the single-shot handshake, uploads the Download Agent, and runs the session automatically. (If you ever need to check enumeration from a second terminal: `dmesg | grep -i -E '0e8d|mediatek|mt6771'`.)

> [!CAUTION]
> **Two ways to get the shorts wrong, and both cost you hardware.** Red is a live 3.3 V rail whenever the unit is powered. There are exactly two joins — **white↔blue** and **green↔red** — and nothing else may touch. Red touching white is a dead short across the IHU's 3.3 V supply. Check the colours twice before the tube goes on, because once it is shrunk you cannot see what is underneath. Never pull the JST jumper while the unit is powered.

### 8.3 — Session A: back up everything (read-only)

The previous `DUMP` directory, if any, is renamed to `DUMP.old.<timestamp>` rather than overwritten — a failed new dump can never destroy an older good one.

> [!CAUTION]
> **Do this before you erase or unlock anything. It is the most important step on this page.** ECarX does not publish firmware for these units. There is no official image to download, no vendor recovery tool, and nobody else's dump will fully do — `nvdata` and `proinfo` carry calibration and serial data unique to your board. The dump you take right now is the only original copy of your unit that will ever exist.

Run one read-only session that prints the GPT and reads every partition except `userdata`. There is **no space after the semicolon** — `mtk multi` splits the string itself:

```bash
export BASE="${BASE:-$HOME/IHU_DEPLOY}"
[ -d "$BASE/DUMP" ] && mv "$BASE/DUMP" "$BASE/DUMP.old.$(date +%s)"
mkdir -p "$BASE/DUMP"
source ~/mtkclient/.venv/bin/activate
rm -f .state "$BASE/.state" "$BASE/DUMP/.state"
mtk multi "printgpt;r gpt $BASE/DUMP/gpt_full.bin;rl $BASE/DUMP --skip userdata"
```

> [!WARNING]
> **Why userdata is the exception.** It is by far the largest partition — tens of gigabytes against a few hundred MB for everything else — so reading it can take hours. It also holds nothing you need: settings, accounts and Bluetooth pairings, and unlocking wipes it anyway. Backing it up buys you nothing and costs most of an evening.

**Wait for every partition to report done. Do not touch the USB cable, the jumper or the power while it runs.** The run is only valid if the last line says:

```text
All Dumped partitions success.
```

If it says `Failed to dump all partitions`, at least one image is missing or truncated. **Do not power off yet** — power-cycle back into BROM (shorts stay in place) and repeat this session. A dump you did not verify is not a backup.

**Then verify the dump structurally.** Zero-size checks miss a truncated read; this compares every image against the GPT's own sector count and refuses to continue on any mismatch:

```bash
python3 - "$BASE/DUMP" <<'PY'
import sys, os, struct
d = sys.argv[1]
g = None
for fn in ("gpt_full.bin", "gpt.bin"):
    gp = os.path.join(d, fn)
    if os.path.exists(gp):
        cand = open(gp, "rb").read()
        if b"EFI PART" in cand:
            g, gp_name = cand, fn
            break
if g is None:
    print("STOP: no GPT header in gpt_full.bin or gpt.bin"); sys.exit(1)
hdr = g.find(b"EFI PART")
my_lba   = struct.unpack_from("<Q", g, hdr + 24)[0]
ent_lba  = struct.unpack_from("<Q", g, hdr + 72)[0]
n        = struct.unpack_from("<I", g, hdr + 80)[0]
sz       = struct.unpack_from("<I", g, hdr + 84)[0]
slice0   = my_lba - (hdr // 512)          # LBA of the first byte in the image
base     = (ent_lba - slice0) * 512
if base < 0 or base + n * sz > len(g):
    print("STOP: GPT entry table is not fully contained in", gp_name); sys.exit(1)
bad = []
for i in range(n):
    e = g[base + i*sz: base + (i+1)*sz]
    if len(e) < 128 or e[16:32] == b"\x00" * 16: break
    if e[:4] == b"\x00" * 4: continue
    name = e[56:128].decode("utf-16le", errors="ignore").split("\x00", 1)[0]
    if not name: continue
    first, last = struct.unpack_from("<QQ", e, 32)
    if first == 0 and last == 0: continue
    expected = (last - first + 1) * 512
    if name == "userdata": continue
    p = os.path.join(d, name + ".bin")
    if not os.path.exists(p):
        bad.append((name, "missing", expected, 0))
    elif os.path.getsize(p) != expected:
        bad.append((name, "size", expected, os.path.getsize(p)))
if bad:
    for b in bad: print("BAD:", b)
    sys.exit(1)
print("DUMP OK: every GPT partition except userdata has an exact-size image")
PY
find "$BASE/DUMP" -type f -size 0 -print    # must print nothing
```

> [!CAUTION]
> If the verifier prints `STOP` or any `BAD:` line, **do not copy anything into `ORI` and do not continue**. Re-run session A after a power-cycle; a missing or truncated partition is exactly the failure this check exists to catch.

**Tie the originals to this unit and save the hashes.** `ORI` is what every later step must match; a file from another unit has no business there:

```bash
cp "$BASE/DUMP/lk.bin"   "$BASE/ORI/lk.bin"
cp "$BASE/DUMP/boot.bin" "$BASE/ORI/boot.bin"
```

Before hashing, build a **minimal GPT restore image** — exactly the MBR, GPT header and entry table, with no partition data. Restoring the raw 0x16000 `gpt_full.bin` would also rewrite the first 90 KB after the table, which can belong to a real partition:

```bash
python3 - "$BASE/DUMP/gpt_full.bin" <<'PY'
import sys, struct
g = open(sys.argv[1], "rb").read()
h = g.find(b"EFI PART")
if h < 0:
    print("STOP: no GPT header"); sys.exit(1)
my_lba  = struct.unpack_from("<Q", g, h + 24)[0]
ent_lba = struct.unpack_from("<Q", g, h + 72)[0]
n       = struct.unpack_from("<I", g, h + 80)[0]
sz      = struct.unpack_from("<I", g, h + 84)[0]
ent_off = h + (ent_lba - my_lba) * 512
need    = ent_off + n * sz
if need > len(g):
    print("STOP: GPT table larger than the captured area"); sys.exit(1)
open(sys.argv[1].replace("gpt_full", "gpt_restore"), "wb").write(g[:need])
print("gpt_restore.bin:", need, "bytes")
PY
```

```bash
sha256sum "$BASE/ORI/lk.bin" "$BASE/ORI/boot.bin" "$BASE/DUMP/system.bin" \
          "$BASE/DUMP/seccfg.bin" "$BASE/DUMP/nvdata.bin" | tee "$BASE/HASHES.txt"
python3 - "$BASE/DUMP/seccfg.bin" <<'PY'
import sys
d = open(sys.argv[1], "rb").read(12)
if not (d.startswith(b"AND_SECCFG_v") or d.startswith(b"MMMM")):
    print("STOP: unknown seccfg header:", d.hex())
    sys.exit(1)
print("seccfg header OK")
PY
```

> [!CAUTION]
> **`seccfg` must start with `AND_SECCFG_v` or `MMMM`.** Anything else means the lock-state format is unknown; `mtk da seccfg unlock` would then prompt interactively and could write a guessed state. Stop, keep the dump, and do not run session B.

> [!TIP]
> A read that was interrupted still leaves files behind — finding that out later, when you need them, is too late. Then put the dump somewhere it cannot be lost: a separate drive, and a second copy elsewhere if you can. The ones that matter most for recovery are `lk`, `boot`, `system`, `seccfg`, `nvdata`, `nvcfg` and `proinfo`.

Power down **in this order**: unplug the **POWER socket first**, then the USB cable, then remove the JST jumper. Leave the unit unplugged and continue with [Step 6](#step-6--patch-lkbin).

> [!NOTE]
> `printgpt` in the output lists the partitions with sizes. Check whether `md_udc` appears — you need that for session B. It is normal for it to be absent on some builds.
>
> "Every partition" means every GPT partition. The preloader and the BROM live outside the GPT; this guide never writes them, so they do not need a dump.

### 8.4 — Session B: erase & unlock (no image written)

**Preflight first. This block hard-stops on any mismatch, and it runs in a subshell so it cannot kill your shell.** It proves the flashed files came from this unit's dump, that `lk` changed in exactly four bytes, that `boot` really is Magisk-patched and fits the partition, and that the jar is a real `dex 039`:

```bash
(
set -e
: "${BASE:?export BASE=$HOME/IHU_DEPLOY first}"
for d in ORI MOD DUMP; do [ -d "$BASE/$d" ] || { echo "STOP: $BASE/$d missing"; exit 1; }; done
for f in lk boot; do
  [ -f "$BASE/DUMP/$f.bin" ] && [ -f "$BASE/ORI/$f.bin" ] && [ -f "$BASE/MOD/$f.bin" ] \
    || { echo "STOP: $f.bin missing from DUMP/ORI/MOD"; exit 1; }
  cmp "$BASE/DUMP/$f.bin" "$BASE/ORI/$f.bin" \
    || { echo "STOP: ORI/$f.bin is not this unit's dump"; exit 1; }
done
[ "$(stat -c%s "$BASE/MOD/lk.bin")" -eq "$(stat -c%s "$BASE/ORI/lk.bin")" ] \
  || { echo "STOP: MOD/lk.bin size differs from ORI"; exit 1; }
[ "$(stat -c%s "$BASE/MOD/boot.bin")" -le "$(stat -c%s "$BASE/ORI/boot.bin")" ] \
  || { echo "STOP: MOD/boot.bin is larger than the boot partition"; exit 1; }
[ "$(stat -c%s "$BASE/MOD/boot.bin")" -ge $(( $(stat -c%s "$BASE/ORI/boot.bin") / 4 )) ] \
  || { echo "STOP: MOD/boot.bin looks truncated"; exit 1; }
if cmp -s "$BASE/ORI/boot.bin" "$BASE/MOD/boot.bin"; then
  echo "STOP: MOD/boot.bin is identical to ORI — Magisk patch missing"; exit 1
fi
python3 - "$BASE" <<'PY'
import sys, os
base = sys.argv[1]
a = open(base + "/ORI/lk.bin", "rb").read()
b = open(base + "/MOD/lk.bin", "rb").read()
assert len(a) == len(b), "STOP: lk size changed"
d = [i for i in range(len(a)) if a[i] != b[i]]
assert len(d) == 4 and b[d[0]:d[0]+4] == bytes([0x00,0x20,0x08,0xBD]), \
       "STOP: lk is not the expected 4-byte patch"
assert d[0] in (0x3B392, 0x3C0FE), "STOP: lk patch offset 0x%X not in confirmed list" % d[0]
print("lk patch proof OK at 0x%X" % d[0])
for p in (base + "/ORI/boot.bin", base + "/MOD/boot.bin"):
    assert open(p, "rb").read(8) == b"ANDROID!", "STOP: %s missing ANDROID! boot magic" % p
print("boot image magic OK")
PY
python3 - "$BASE/DUMP/seccfg.bin" <<'PY'
import sys
d = open(sys.argv[1], "rb").read(12)
if not (d.startswith(b"AND_SECCFG_v") or d.startswith(b"MMMM")):
    print("STOP: unknown seccfg header — do not unlock"); sys.exit(1)
print("seccfg header OK")
PY
[ "$(stat -c%s "$BASE/MOD/services.jar")" -gt 1000000 ] \
  || { echo "STOP: services.jar is the 183-byte stub or truncated"; exit 1; }
python3 - "$BASE/MOD/services.jar" <<'PY'
import sys, zipfile
try:
    h = zipfile.ZipFile(sys.argv[1]).read("classes.dex")[:8]
except Exception as e:
    print("STOP: services.jar has no readable classes.dex:", e); sys.exit(1)
if h != b"dex\n039\x00":
    print("STOP: classes.dex is not dex 039"); sys.exit(1)
print("dex 039 OK")
PY
echo "PREFLIGHT OK — record these:"
sha256sum "$BASE/MOD/lk.bin" "$BASE/MOD/boot.bin" "$BASE/MOD/services.jar"
)
```

Only when that prints `PREFLIGHT OK` should you continue. Re-enter BROM exactly as in [8.2](#82--enter-brom-mode) — the shorts stay on the jumper, power off, power on, connect USB A-to-A.

**Erase, then unlock. Nothing is written to `lk` or `boot` in this session.** Run the one line that matches the partition list you saw in `printgpt`:

```bash
export BASE="${BASE:-$HOME/IHU_DEPLOY}"
source ~/mtkclient/.venv/bin/activate
rm -f .state "$BASE/.state" "$BASE/DUMP/.state"
mtk multi "e metadata,userdata,md_udc;da seccfg unlock"
```

If `md_udc` was **not** in the GPT list, use this instead:

```bash
rm -f .state "$BASE/.state" "$BASE/DUMP/.state"
mtk multi "e metadata,userdata;da seccfg unlock"
```

> [!WARNING]
> **Erase first, unlock after.** Skipping the erase can leave the IHU bootlooping after the flash. The erase makes `userdata` and its encryption metadata match the new unlocked state.

Check the output before doing anything else. The erase must first print a `Formatted sector ...` line for **each** partition you named. The unlock runs immediately after the erase in this same session, so you cannot stop it mid-command; if an erase failed, **do not boot the unit** — power-cycle back into BROM and re-run the erase before session C. An unlocked unit with un-erased metadata can bootloop, but it is not bricked.

| Output | Meaning | Action |
|---|---|---|
| `Successfully wrote seccfg.` | Unlocked | Power off, go to 8.5 |
| `Device is already unlocked` | Already unlocked, nothing to do | Power off, go to 8.5 |
| `Couldn't detect partition: md_udc` | `md_udc` absent; the other partitions were still erased and the session continues | Expected on builds without `md_udc`; ignore it and confirm the next line says `Successfully wrote seccfg.` |
| No `Successfully wrote seccfg.` and no `already unlocked` | The unlock did not run (the session aborted after the erase) | Power-cycle back into BROM and repeat session B; do not go to session C. |
| `Unknown seccfg partition header` / `algo is unknown` / a `Type "v3" or "v4"` prompt | Lock-state format unknown or unparseable | **Do not guess, do not write anything.** Press Enter to cancel the prompt, power off, re-enter BROM, restore `seccfg` from the dump (`mtk multi "w seccfg $BASE/DUMP/seccfg.bin"`), and stop. |

> [!CAUTION]
> **Never answer the `v3`/`v4` prompt.** That writes a brand-new lock state from defaults. Your dump already contains the real one; restoring it is the only correct move.

Power down in the same order as 8.3. This is a safe checkpoint: userdata is erased and the unit is unlocked, but `lk` and `boot` are still stock, so it can still boot if you stop here.

### 8.5 — Session C: write lk + boot, then read back

Re-enter BROM ([8.2](#82--enter-brom-mode)) and run both writes plus a readback in **one** session. `w` takes the partition and the file as two separate arguments; `r` writes the readback image:

```bash
export BASE="${BASE:-$HOME/IHU_DEPLOY}"
source ~/mtkclient/.venv/bin/activate
mkdir -p "$BASE/VERIFY"
rm -f .state "$BASE/.state" "$BASE/DUMP/.state" "$BASE/VERIFY/.state" "$BASE/VERIFY/lk.bin" "$BASE/VERIFY/boot.bin"
mtk multi "w lk $BASE/MOD/lk.bin;w boot $BASE/MOD/boot.bin;r lk $BASE/VERIFY/lk.bin;r boot $BASE/VERIFY/boot.bin"
```

Expected, in order:

```text
Wrote .../MOD/lk.bin to sector ...
Wrote .../MOD/boot.bin to sector ...
```

If either line says `Failed to write` or `Filename doesn't exists`, **do not boot the unit**. Power-cycle back into BROM (unplug power, wait, plug back in with the shorts in place) and repeat this session; if it fails again, power-cycle once more and restore the stock pair from the dump:

```bash
export BASE="${BASE:-$HOME/IHU_DEPLOY}"
rm -f .state "$BASE/.state" "$BASE/DUMP/.state" "$BASE/VERIFY/.state"
mtk multi "w lk $BASE/ORI/lk.bin;w boot $BASE/ORI/boot.bin"
```

**Prove the flash landed.** The readback files contain the whole partition; compare only the bytes you wrote:

```bash
for f in lk boot; do
  n=$(stat -c%s "$BASE/MOD/$f.bin")
  cmp -n "$n" "$BASE/MOD/$f.bin" "$BASE/VERIFY/$f.bin" \
    && echo "$f: READBACK OK" || echo "$f: READBACK MISMATCH — reflash"
done
```

A mismatch means the image in flash is not the image you verified. Re-enter BROM and repeat this session before booting.

> [!NOTE]
> `w` writes exactly the file's byte count. `MOD/lk.bin` is byte-for-byte the dump size; `MOD/boot.bin` may be slightly smaller after Magisk repacks it, and the untouched tail of the boot partition is padding the bootloader ignores. Do not pad, truncate or concatenate these files.

Then shut down **in this order**: unplug the **POWER socket first**, then the USB cable, then remove the JST jumper. **Leave the power unplugged** — that is exactly where [Step 9](#step-9--connect-uart-again) begins.

> [!TIP]
> When the unit next boots, the orange state screen **will not** appear, because `lk.bin` is already patched. You do not need to boot it now to check.

### 8.6 — Optional: use the GUI instead

If you prefer the Windows guide's GUI workflow, it works on Linux too — with one patch. The double-init bug is in the Python source, not in any driver, so it applies here as well:

```bash
cd ~/mtkclient
cp mtk_gui.py mtk_gui.py.bak
python3 - <<'PY'
import re
from pathlib import Path
p = Path("mtk_gui.py")
s = p.read_text()
assert s.count("mtk_class.preloader.init()") == 1, "unexpected source — check manually"
s = re.sub(r"^(\s*)mtk_class\.preloader\.init\(\)$", r"\1pass", s, flags=re.M)
p.write_text(s)
PY
source .venv/bin/activate
python3 mtk_gui.py
```

> [!WARNING]
> The patch removes a **duplicate** initialisation; `da_handler.connect()` still calls `preloader.init()`, so the device is initialised exactly once — which is what the BROM expects. Indentation is load-bearing; the script above preserves it. Keep the patched copy safe: `git pull` will overwrite it. Using the CLI as in [8.3](#83--session-a-back-up-everything-read-only)/[8.5](#85--session-c-write-lk--boot-then-read-back) avoids this entirely.

---

## Step 9 — Connect UART again

`UART · picocom`

- Make sure the IHU **POWER socket is unplugged**.
- **Check blue, green and red are still covered** — heat shrink or tape intact, no copper showing.
- Yellow (IHU TX) → CH340G **RXD**; black (IHU RX) → CH340G **TXD**; white (GND) → CH340G **GND**. TX crosses to RX — wiring TX to TX gives silence.
- Open the console, then plug the POWER socket back in:

```bash
picocom -b 921600 /dev/ttyUSB0
```

Wait for boot to finish, press **Enter** once. You should get `console:/ $`.

> [!CAUTION]
> Unused wire ends are live next to your working wires whenever the unit is powered — same risks as [3.1](#31--insulate-the-unused-wires). Green must not be shorted to red, or the USB drive in [Step 11](#step-11--deploy-servicesjar) will not mount.

---

## Step 10 — Get root & remount rw

`UART · picocom`

Your root does **not** come from Magisk at this stage.

> [!TIP]
> Once Magisk is flashed, `/sbin` comes first in PATH. A plain `su` hits MagiskSU, which needs the Magisk app installed before it can approve the request — so it denies you with `Permission denied`. But ECarX leaves `/system/xbin/su` in stock firmware — `-rwsr-x--- root shell`. Setuid root, group shell. Your UART shell is uid 2000 (shell), so that binary hands you root with no conditions at all.

```bash
/system/xbin/su
mount -o rw,remount /system
mount | grep ' /system '
```

The prompt changes to `#` and the mount output must contain `rw`, e.g.:

```text
/dev/block/mmcblk0p42 on /system type ext4 (rw,seclabel,relatime,...)
```

> [!CAUTION]
> **Still shows `ro`?** Your `boot.bin` was patched with `Preserve AVB 2.0 / dm-verity` ticked. Go back to [Step 7](#step-7--patch-bootbin-magisk), patch again with that option **unticked**, then reflash boot through [8.5](#85--session-c-write-lk--boot-then-read-back). There is no command that works around this.

> [!WARNING]
> **Neither of these survives a reboot.** After every IHU restart you must run `/system/xbin/su` and `mount -o rw,remount /system` again.

---

## Step 11 — Deploy services.jar

`UART · picocom`

This is the core of the method. Swap `services.jar` and clear the old cache so the system actually loads the new jar.

> [!CAUTION]
> **Why the cache has to go:** Android loads the compiled `.odex` / `.vdex` / `.art` files, **not** the jar. Swap the jar but leave the old cache behind and the IHU keeps running the original code — your new jar just sits there unused. This is the number one reason for "I deployed it but installs are still blocked".

### 11.1 — Copy the files onto a USB drive

The drive must be **FAT32**. Create a folder called `mod` at the root and put these inside:

- `services.jar` — from `$BASE/MOD/`
- `Magisk.apk`
- `Files.apk`
- `SwipeBack.apk` (optional)

Plug the drive into the IHU USB port, then find where it mounted:

```bash
ls -la /mnt/media_rw/
mount | grep -i vfat
```

> [!TIP]
> It usually shows up as `usbotg-otg1`. You may also see an empty `usbotg` folder — that is a stale mount point, ignore it. The real one is whichever appears in the `mount` list.

> [!WARNING]
> Drive not showing up? Make sure the green pin is **not** shorted to red, and that the drive is FAT32 (exFAT and NTFS are not read).

### 11.2 — Stage the files to internal storage

Copy them inside first, because once you run `stop` the USB drive becomes unreachable:

```bash
mkdir -p /data/local/tmp/mod
cp /mnt/media_rw/usbotg-otg1/mod/* /data/local/tmp/mod/
ls -lh /data/local/tmp/mod/
```

**Verify the staged jar before touching `/system`.** A truncated copy on FAT32 is a guaranteed bootloop; this check is the only thing that catches it:

```bash
ls -l /data/local/tmp/mod/services.jar
md5sum /data/local/tmp/mod/services.jar
```

Compare that `md5sum` with the PC's value (they must be identical):

```bash
md5sum "$BASE/MOD/services.jar"
```

> [!CAUTION]
> If the hashes differ, the copy is bad. Recopy `services.jar` to the USB drive and repeat 11.2. Do **not** run `stop` with an unverified jar, and do not continue if the device-side size is 183 bytes.

Record what you are about to rename, so recovery does not depend on memory:

```bash
ls -l /system/framework/services.jar /system/framework/oat/arm64/services.*
```

### 11.3 — Back up, swap the jar, clear the cache

> [!WARNING]
> **Every original file is `renamed`, never deleted.** If it bootloops, you go back in over UART, rename them back, reboot, done. Never use `rm` here. Type these line by line — a terminal can mangle a long paste into the UART console.

```bash
cp /system/framework/services.jar /data/local/tmp/services.jar.orig
mount | grep ' /system '
```

The mount line must show `rw`. If it shows `ro`, stop and redo [Step 10](#step-10--get-root--remount-rw); renaming the jar on a read-only mount fails halfway and leaves you with no jar.

```bash
stop
```

Stage the new jar under a temporary name and prove it is complete **before** the swap. This keeps a valid `services.jar` present at every instant:

```bash
cp /data/local/tmp/mod/services.jar /system/framework/services.jar.new
chmod 644 /system/framework/services.jar.new
chown root:root /system/framework/services.jar.new
sync
ls -l /system/framework/services.jar.new
```

> [!CAUTION]
> The staged file must be ~3.7MB, not 183 bytes and not 0. If it is wrong, `/system` is out of space or the copy failed: **stop here**. The old jar is untouched, so a reboot returns a working unit. Free space with `df -h /system` before retrying; do not rename anything.

> [!NOTE]
> The two `mv` commands below are one atomic swap within the same filesystem. At every instant a complete `services.jar` exists — the old one until the first `mv`, the staged one after the second.

```bash
if [ -e /system/framework/services.jar.orig ]; then
  echo "STOP: services.jar.orig already exists — run the rollback in Panic recovery R1 first"
else
  mv /system/framework/services.jar /system/framework/services.jar.orig
  mv /system/framework/services.jar.new /system/framework/services.jar
  [ -e /system/framework/oat/arm64/services.odex ] && [ ! -e /system/framework/oat/arm64/services.odex.orig ] && \
    mv /system/framework/oat/arm64/services.odex /system/framework/oat/arm64/services.odex.orig
  [ -e /system/framework/oat/arm64/services.vdex ] && [ ! -e /system/framework/oat/arm64/services.vdex.orig ] && \
    mv /system/framework/oat/arm64/services.vdex /system/framework/oat/arm64/services.vdex.orig
  [ -e /system/framework/oat/arm64/services.art ]  && [ ! -e /system/framework/oat/arm64/services.art.orig ]  && \
    mv /system/framework/oat/arm64/services.art  /system/framework/oat/arm64/services.art.orig
  sync
fi
```

> [!CAUTION]
> **Never overwrite `.orig` files.** If the guard prints `STOP`, a previous deploy already left a `.orig` behind; those `.orig` files are your stock framework and cache. Roll back first ([R1](#r1--uart-works-undo-the-live-deploy)), then redeploy. Overwriting them after a failed boot destroys the only on-device copy of the originals.

> [!TIP]
> If `services.art` does not exist on your build, that `mv` prints `No such file or directory` and nothing else is affected — skip it and continue. The other two must rename.

```bash
ls -lh /system/framework/services.jar
ls -l /system/framework/oat/arm64/ | grep -i services
```

> [!TIP]
> The jar should now be about 3.7MB, and nothing should end in `.odex`, `.vdex` or `.art` any more — they all became `.orig`. That is exactly what you want.

> [!WARNING]
> **After `stop`, the IHU screen freezes and stops responding.** That is normal — the UI is shut down, but UART keeps working. Don't panic and don't cut power. If your console *does* die, power-cycle: the console comes back during boot, and if the new jar is bad the unit bootloops safely until you apply [R1](#r1--uart-works-undo-the-live-deploy). The atomic swap above means a valid jar always exists.

---

## Step 12 — Reboot & test

```bash
reboot
```

> [!WARNING]
> **This first boot is slow** — it can take several minutes, because the framework is recompiling from the new jar. **Do not cut power.** It only happens once; later boots are normal again.

Once it has booted, try installing Magisk. This proves whether the patch worked:

```bash
/system/xbin/su
pm install -r /data/local/tmp/mod/Magisk.apk
```

> [!TIP]
> If it says `Success`, the hard part is over. The `pm install` block is gone.

> [!CAUTION]
> If you get an `INSTALL_FAILED_...` other than `ALREADY_EXISTS`, or the IHU reboots the moment you install — see [Fix Fast](#fix-fast). If `services.jar` went back to 183 bytes after the reboot, dm-verity is still active: go back to [Step 7](#step-7--patch-bootbin-magisk), repatch without AVB, and reflash through [8.5](#85--session-c-write-lk--boot-then-read-back).

---

## Step 13 — Stabilise Magisk

`UART · picocom`

Magisk is installed but not yet stable. Two things to sort out: let the shell use MagiskSU, and remove the factory `su` that conflicts with it.

### 13.1 — Open the Magisk app on the IHU screen

Open the app list on the IHU screen and tap the Magisk icon. It may ask for extra setup and a reboot — just follow it. For now it will show **"Abnormal state"**; that is expected, and 13.3 fixes it. Tap OK and the IHU will reboot.

### 13.2 — Let the shell use MagiskSU

When you type `su`, a superuser permission dialog pops up on the IHU screen. Tap **Grant** before it closes — on most units it disappears far too quickly to catch. Setting the policy directly avoids the race:

```bash
/system/xbin/su
magisk --sqlite "REPLACE INTO policies (uid,policy,until,logging,notification) VALUES(2000,2,0,1,1)"
magisk --sqlite "SELECT * FROM policies"
```

> [!TIP]
> Expected: `uid=2000|policy=2|until=0|logging=1|notification=1`. `uid 2000` is the shell, `policy 2` means allow forever. Test it: exit, then type a plain `su` — you should get `#` straight away with no dialog.

### 13.3 — Remove the conflicting factory su

> [!CAUTION]
> **Do not do this until a plain `su` is confirmed working.** `/system/xbin/su` is your last safety net. If MagiskSU is not stable yet and you rename this binary, you lose root completely.

Run these one line at a time:

```bash
su
mount -o rw,remount /system
mv /system/xbin/su /system/xbin/su.bak
sync
ls -l /system/xbin/su*
```

After renaming, **do not reboot yet**. Test in a fresh shell first:

```bash
exit
su
```

> [!TIP]
> If you still get `#`, go ahead and `reboot`. After it comes back, the Magisk app should show **"Installed"** instead of "Abnormal state".

> [!CAUTION]
> **If the fresh `su` fails, you are not stuck.** `/system/xbin/su.bak` is still setuid root — run it by full path, then put the original back:
> ```bash
> /system/xbin/su.bak
> mount -o rw,remount /system
> mv /system/xbin/su.bak /system/xbin/su
> sync
> ```
> Reboot only after a plain `su` works, or leave the factory `su` in place.

---

## Step 14 — File Manager

`UART · picocom`

This is what removes the need for a PC from here on. File Manager+ can install APKs straight off a USB drive on the IHU screen.

```bash
su
pm install -r /data/local/tmp/mod/Files.apk
appops set com.alphainventor.filemanager REQUEST_INSTALL_PACKAGES allow
appops get com.alphainventor.filemanager REQUEST_INSTALL_PACKAGES
```

> [!TIP]
> Expected: `Success` and `REQUEST_INSTALL_PACKAGES: allow`.

> [!WARNING]
> **Do not use `pm grant ... android.permission.INSTALL_PACKAGES`.** That command fails with a SecurityException — File Manager+ does not declare that permission in its manifest. What it actually uses is `REQUEST_INSTALL_PACKAGES`, which the `appops` line above handles.

**Test it:** on the IHU screen, open File Manager+, browse to the USB drive, tap any APK and install. If it goes through without the IHU rebooting, **you are done** — apps can be installed from the head unit itself now, no PC involved.

---

## Step 15 — SwipeBack (optional)

`Optional`

SwipeBack adds a floating button for the "back" gesture on the IHU. It runs through an Accessibility Service, so it has to be registered first.

```bash
su
pm install -r /data/local/tmp/mod/SwipeBack.apk
```

Do not guess the service class name — ask the device:

```bash
pm dump ace.jun.simpleback | grep -iE 'accessibility|Service' | head -20
```

> [!TIP]
> Expected: `android.accessibilityservice.AccessibilityService: ace.jun.simpleback/.service.AccService`, so the full name is `ace.jun.simpleback/ace.jun.simpleback.service.AccService`.

```bash
settings put secure enabled_accessibility_services ace.jun.simpleback/ace.jun.simpleback.service.AccService
settings put secure accessibility_enabled 1
settings get secure enabled_accessibility_services
```

> [!TIP]
> The button appears on screen within a few seconds. This setting survives reboots — no Magisk boot script needed.

---

## Fix Fast

`Troubleshooting`

| Symptom | Cause | Fix |
|---|---|---|
| `mtk` does not detect the IHU | Missing udev rules / groups, ModemManager, or a charge-only cable | Install `Setup/Linux/*.rules`, add user to `dialout`/`plugdev`, reboot; stop ModemManager; use a USB 2.0 port and a known data cable; verify both JST shorts ([8.2](#82--enter-brom-mode)). |
| Chip info reads OK, then `Handshake failed, retrying` forever | GUI double-init bug (2.1.4) — BROM handshakes only once per power-on | Use the CLI (`mtk multi`) as in [8.3](#83--session-a-back-up-everything-read-only)/[8.5](#85--session-c-write-lk--boot-then-read-back), or apply the GUI patch in [8.6](#86--optional-use-the-gui-instead). |
| `error: externally-managed-environment` | PEP 668 blocks system pip | Use the venv from [2.4](#24--install-mtkclient): `source ~/mtkclient/.venv/bin/activate`. |
| `OSError: Unable to find libfuse` on startup | libfuse missing | `sudo apt install libfuse-dev` and reinstall the venv deps. |
| Second `mtk` command will not connect | BROM handshake already used this power-on | Power-cycle the IHU with the shorts in place; put all commands for one phase in a single `mtk multi` session. |
| `mtk multi` stops with `invalid choice: ''` | A space after a `;` in the `multi` string | Remove every space after `;`: `"cmd1;cmd2;cmd3"`. The CLI splits the string itself. |
| `the following arguments are required: filename` | `w lk,<file>` was used | `w` takes two arguments: `mtk multi "w lk $BASE/MOD/lk.bin"`; never join them with a comma. |
| `Filename doesn't exists: ...` during a write | `$BASE` unset or path typed wrong | Stop before rebooting; power-cycle to BROM and re-run the write with the checked path. Do not proceed on a failed write. |
| `READBACK MISMATCH` | Flash did not land as written | Re-enter BROM and repeat session C; if it repeats, restore `ORI/lk.bin` + `ORI/boot.bin` and investigate the cable/port first. |
| Dump lacks `All Dumped partitions success.` | Interrupted or failed read | Power-cycle with shorts in place, re-run session A; do not proceed until the line appears and the size verifier passes. |
| Size verifier prints `BAD: (...)` | Missing or truncated partition image | Re-run session A; if the same partition keeps failing, that partition has bad blocks — stop and do not flash. |
| `Unknown seccfg partition header` or a `v3`/`v4` prompt | Lock-state format not recognised | Cancel the prompt (Enter), power off, restore `seccfg` from `DUMP/seccfg.bin`, stop. Never invent a lock state. |
| No text at all in the serial console | TX/RX swapped, wrong device, ModemManager/brltty, or wrong baud | Swap TX/RX; check `/dev/ttyUSB*`; stop ModemManager/brltty; confirm `921600`. |
| picocom will not open the port | User not in `dialout`, or port held by another process | Re-login after `usermod`; check `sudo lsof /dev/ttyUSB0`. |
| `su` returns "Permission denied" | PATH picks `/sbin/su` (Magisk), and the Magisk app is not there to approve it | Use the full path `/system/xbin/su` ([Step 10](#step-10--get-root--remount-rw)). |
| `mount remount,rw` fails, `/system` stays `ro` | dm-verity still active — `Preserve AVB 2.0` was ticked | Repatch `boot.bin` with that option **unticked**, then reflash through [8.5](#85--session-c-write-lk--boot-then-read-back). No workaround command exists. |
| Jar swapped but `pm install` still blocked | The old `.odex` / `.vdex` / `.art` cache is still in place | Check `ls /system/framework/oat/arm64/ \| grep services` — if anything is not `.orig` yet, rename it and reboot. |
| `services.jar` back to 183 bytes after reboot | dm-verity restored the original partition | Same as the `ro` issue above — repatch boot without AVB. |
| IHU bootloops after deploying the jar | Wrong jar, or the cache was only half renamed | UART → `/system/xbin/su` → remount rw → rename the `.orig` files back → `reboot`. Full commands in [R1](#r1--uart-works-undo-the-live-deploy). This is why we rename instead of delete. |
| USB drive does not appear under `/mnt/media_rw/` | Green pin shorted to red, or the drive is not FAT32 | Remove the green-red short and reboot; format as FAT32. |
| `baksmali: not an apk, dex, odex or oat file` | You fed baksmali the `.cdex` directly | Run the cdex → dex conversion first ([5.4](#54--cdex--standard-dex)). |
| `smali: requires at least N registers` | CompactDex register-decode problem | Use the verified `cdex2dex.py` from [5.2](#52--create-the-cdex2dexpy-converter) and re-run from [5.4](#54--cdex--standard-dex). |
| Final DEX magic is `dex 035`, not 039 | `smali a` run without `-a 28` | Re-run `smali a -a 28 smali_out -o classes.dex`, repackage, redeploy. |
| Install still fails `"...aco version; ignoring!"` | Only the 3 stubs were applied — the `installPackageLI` whitelist still rejects | Apply [5.6.2](#562--the-fourth-change-bypass-the-aco-whitelist) as well, rebuild with `-a 28`, redeploy. |
| Patch script prints `NOT FOUND` | Method descriptor differs in your firmware | `grep -n compareSignatures .../*.smali` and update the descriptor in `patch_smali.py`. |
| IHU reboots instantly during `pm install` | ECarX security watchdog flagged that APK (confirmed with Termux) | That APK cannot go in through `pm install` or File Manager. The only route is injecting it into `system.bin` before flashing — do not attempt that without a verified full `system` dump and a tested BROM recovery path; it is outside this guide. |
| File Manager+ says "parse error" | APK is not arm64-v8a, or its minSdk is too high | Download an arm64-v8a build compatible with Android 9 (API 28). |

---

## Panic recovery — get back to stock

`Recovery`

Every destructive change in this guide is reversible **except the userdata erase**, which is data loss, not a brick. Decide by what you can still reach:

| What you have | What to do |
|---|---|
| UART prompt (`console:/ $` or `#`) | Roll back the jar or the framework cache — no BROM needed. |
| No UART, but BROM handshakes | Restore the damaged partition from `DUMP/` over BROM. |
| No UART and no BROM | Re-check the physical path first: both shorts, 12 V, USB 2.0 port, data cable, udev/ModemManager. Do not reflash blindly. |

### R1 — UART works: undo the live deploy

Bootloop after the jar swap, or installs stopped working after a bad build:

```bash
/system/xbin/su || /system/xbin/su.bak || su
mount -o rw,remount /system
rm -f /system/framework/services.jar.new
[ -e /system/framework/services.jar.orig ] && \
  mv /system/framework/services.jar.orig /system/framework/services.jar
[ -e /system/framework/oat/arm64/services.odex.orig ] && \
  mv /system/framework/oat/arm64/services.odex.orig /system/framework/oat/arm64/services.odex
[ -e /system/framework/oat/arm64/services.vdex.orig ] && \
  mv /system/framework/oat/arm64/services.vdex.orig /system/framework/oat/arm64/services.vdex
[ -e /system/framework/oat/arm64/services.art.orig ] && \
  mv /system/framework/oat/arm64/services.art.orig /system/framework/oat/arm64/services.art
sync
reboot
```

If a `.orig` name is missing, that file was not renamed — the `[ -e ... ]` guard skips it safely. Never `rm` any `.orig`, `.odex`, `.vdex` or `.art` file. If the console disappeared after `stop`, power-cycle: the console returns during boot and the `.orig` files are still there.

> [!TIP]
> **If you already renamed `/system/xbin/su` in [13.3](#133--remove-the-conflicting-factory-su):** `/system/xbin/su.bak` is still on `/system/xbin` with its setuid-root bit intact (`-rwsr-x--- root shell`). Even if MagiskSU (`su`) cannot answer during a framework bootloop, calling `/system/xbin/su.bak` by full path still hands you root directly over UART without needing BROM. Only if `/system` itself is corrupted should you restore `system.bin` over BROM ([R2](#r2--no-uart-brom-works-restore-from-the-dump)).

### R2 — No UART, BROM works: restore from the dump

Enter BROM exactly as in [8.2](#82--enter-brom-mode), then one session per group:

```bash
export BASE="${BASE:-$HOME/IHU_DEPLOY}"
source ~/mtkclient/.venv/bin/activate
rm -f .state "$BASE/.state" "$BASE/DUMP/.state" "$BASE/VERIFY/.state"
# Boot chain only (most common recovery)
mtk multi "w lk $BASE/ORI/lk.bin;w boot $BASE/ORI/boot.bin"

# If /system was damaged or a bad system image was flashed
rm -f .state "$BASE/.state" "$BASE/DUMP/.state" "$BASE/VERIFY/.state"
mtk multi "w system $BASE/DUMP/system.bin"

# If the lock state or unlock went wrong
rm -f .state "$BASE/.state" "$BASE/DUMP/.state" "$BASE/VERIFY/.state"
mtk multi "w seccfg $BASE/DUMP/seccfg.bin"

# If the partition table itself was touched (rare).
# gpt_restore.bin is only the MBR + GPT header + entry table,
# never partition data. Never use another unit's file.
rm -f .state "$BASE/.state" "$BASE/DUMP/.state" "$BASE/VERIFY/.state"
mtk multi "w gpt $BASE/DUMP/gpt_restore.bin"
```

Then power-cycle once and verify with UART. Restoring `lk`, `boot`, `system` and `seccfg` from your own dump returns the unit to the state it was in at 8.3, except `userdata` stays erased.

> [!CAUTION]
> **Restoring `seccfg` also restores the original lock state (usually locked).** If you restore `seccfg`, restore the stock `lk` and `boot` **first** — locking a bootloader that still has a patched `boot` is how a recoverable unit fails to boot. When in doubt, restore `lk` + `boot` and leave `seccfg` alone.

> [!WARNING]
> **Each `mtk multi` above is a separate BROM session.** After one finishes, unplug power, wait, plug it back in with the shorts still in place, then run the next. A second `mtk` process cannot reuse the handshake from the first.

> [!CAUTION]
> **Never restore `nvdata`, `nvcfg`, `proinfo` or `persist` from another unit.** Those carry per-board calibration, keys and serial data. Your own `DUMP/` copies are the only valid ones — and you should not need them unless something outside this guide wrote to those partitions.

### R3 — Relock (optional, and only after R2)

Relocking with modified images is how a recoverable unit becomes a boot failure. Restore `lk`, `boot` and `system` first, confirm the unit boots stock, and only then:

```bash
export BASE="${BASE:-$HOME/IHU_DEPLOY}"
source ~/mtkclient/.venv/bin/activate
rm -f .state "$BASE/.state" "$BASE/DUMP/.state" "$BASE/VERIFY/.state"
mtk multi "da seccfg lock"
```

The bootloader relocks; the orange-state warning returns because the stock `lk` is back. BROM remains available if the relock refuses to boot.

---

## Notes

### What was verified

- mtkclient is officially supported on Linux: libusb talks to `usbfs` directly, so there is no UsbDk equivalent and no kernel patch needed for MT6771. The kernel patch in upstream's docs is only for ancient kamakiri targets.
- The in-kernel Linux `ch341` driver handles the CH340G at 921600 baud; the serial console needs no vendor driver.
- All the jar-build tooling (Python 3, smali/baksmali, `make`/gcc, zlib headers, zip, vdexExtractor) is native on Linux — the WSL steps in the original guides are the same commands in a host shell.
- The two byte-level patchers (`cdex2dex.py`, the lk.bin patcher) are pure Python stdlib and OS-independent.

### Why the CLI is used

The GUI bug documented in the Windows install guide is in `mtk_gui.py` itself, not in a driver, so it applies on Linux too. The `mtk` CLI never enters that code path — it initialises the preloader exactly once per connection — so this guide uses it throughout. The optional [GUI patch](#86--optional-use-the-gui-instead) is included for completeness.

### One honest unknown

There is no public report of an ECarX E02 / MT6771 being flashed from Linux. The conclusion that it works rests on mtkclient using the same libusb code path on every OS and on the E02 being a standard MT6771 BROM device. The hardware procedure — JST shorts, power order, USB 2.0 port, backups — is identical regardless of host OS, so the risk is in setup, not in the method. If the handshake fails with correct udev rules and permissions, re-check the two shorts and the USB cable before anything else.

### Sources

- [bkerler/mtkclient](https://github.com/bkerler/mtkclient) — install instructions, udev rules and CLI commands
- [anestisb/vdexExtractor](https://github.com/anestisb/vdexExtractor) — vdex unpacking and un-quickening
- AOSP `art/libdexfile/dex/compact_dex_file.h` — the CompactDex layout implemented by `cdex2dex.py`
- The three Windows guides in this repository — hardware procedure, patch details and findings
