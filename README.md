# Root Access — an I2C app for the DEF CON 34 badge

Prebuilt firmware for the DC34 badge (baosec-lite / bao1x) that adds a **Root
Access** entry to the badge's own menu: scan the SAO connector's I2C bus, poke
at whatever answers, and drive a **Root Access** SAO through its 50 animations —
all from the badge's OLED and nav switch, with no computer attached.

<img alt="" src="https://img.shields.io/badge/status-experimental-orange"> <img alt="" src="https://img.shields.io/badge/target-bao1x%20%2F%20baosec--lite-blue">

---

## ⚠️ Flashing this permanently erases your badge's light key

This image is signed with the public **development key**, so the badge enters
developer mode on boot — and the DEF CON 34 key scheme wipes the shared light
key `Ko` on that transition. That is the anti-cheat mechanism working as
designed, not a bug in this app.

**What you lose:** the QR-code light-pattern trading game with other badges.
Permanently. Reflashing bunnie's official firmware afterwards does **not** bring
it back.

**What still works:** everything else — the console, the vault, FIDO2, TOTP,
image upload, and the badge's menu.

If the light game matters to you, stop here.

## ⚠️ This is experimental and it is glitchy

These exact images have been flashed onto a real badge and run — they boot, and
the app works. But treat it as a toy, not a tool: it is a hobby build, unsigned
by anyone official, and unsupported. Known rough edges are listed in [Known
issues](#known-issues--why-it-feels-glitchy) below — read them before you
conclude something is broken. Recovery is covered in
[If it goes wrong](#if-it-goes-wrong).

---

## The files

All three must be copied. They are one image split three ways, not
alternatives — a mismatched set will not boot.

| file | size | SHA-256 |
|---|---|---|
| `loader.uf2` | 345 KB | `1FAFFB05357F9FDA0F23302C436233166DB6DDA4E392D7B853CC1EBC6D8437FC` |
| `swap.uf2` | 2.26 MB | `C023F71C0C559385B789946C8A7FD517813225B6EB021E4E266089A0AC7B1F17` |
| `xous.uf2` | 6.08 MB | `2464F3A6E14AEA461242CB989415C28988363A1E6EC0D72B11416FBB099B12E1` |

Verify before flashing:

```powershell
Get-FileHash .\loader.uf2, .\swap.uf2, .\xous.uf2 -Algorithm SHA256
```

```bash
shasum -a 256 loader.uf2 swap.uf2 xous.uf2
```

## Getting them onto the badge

The badge's bootloader lives in ROM and presents itself as a USB drive, so no
flashing tool, driver, or serial port is involved — it is drag-and-drop.

1. **Hold any button down while plugging the module in.** Keep holding until it
   enumerates. A removable volume named **BAOCHIP** appears.
2. **Copy all three `.uf2` files onto it.** Order does not matter. The drive may
   appear to "reject" each file by making it vanish after the copy — that is
   normal UF2 behaviour, the bootloader consumes the file rather than storing
   it.
3. **Wait for the copies to finish flushing**, then **press any button to boot.**

Step 3 is not optional. Unplugging or powering off instead of pressing a button
can leave the last sector half-written, which produces a badge that does not
boot and has to be reflashed.

**On Windows,** Explorer will happily keep showing the `D:` drive letter after
the badge has already left the bootloader. Do not trust it — run `Get-Volume`
to see whether BAOCHIP is genuinely still there.

**On macOS,** eject the volume before pressing the button so the OS actually
flushes its write cache.

## How the app works

### Finding it

Open the badge menu and choose **Root Access**. Nothing else is removed — Help,
About, Token Mode, Close Menu and Power Off are all still where they were.

The bus hardware is only claimed when you open the app and is released as soon
as you leave it, so it does not interfere with the rest of the badge in between.

### Controls

Everything is the nav switch:

| | |
|---|---|
| **up / down** | move the `>` cursor, or change a digit |
| **press** | select / confirm |

There is no back button. Every screen has a **Back** or **Exit** row instead.

### Screens

```
ROOT ACCESS                ROOT ACCESS            DEVICE 0x50

> Scan                     Scanning...            > Send
  Send  00->50                                      Quick Commands
  Address: 0x50            > 0x50 Root Access       Probe
  Data:    0x00              0x68                   Rescan
  Quick Commands             Back                   Back
  Exit
```

- **Scan** walks the bus from `0x08` to `0x77` and lists everything that
  answers. Selecting a result opens it. `0x50` is the Root Access SAO, which is
  what this app was built for, so it is called out by name in the list.
- **Send** writes the current **Data** byte to the current **Address** — one
  byte, no register prefix. That is exactly the Root Access mode-select command.
- **Address** and **Data** open the byte editor.
- **Probe** re-tests a single device and reports `ACK`, `no response`, or a bus
  error.

### Editing a byte

```
SET VALUE

   0x5A
     ^
 up/down: digit
 press:   next
```

Up and down change **only the digit under the caret**; press moves to the low
nibble, and pressing again commits. Two nibbles is at most 32 presses instead of
the 255 that stepping a whole byte would take with three buttons.

### Quick Commands

```
QUICK 0x50

 anim 0x00/0x31
> Next
  Previous
  Jump to...
  Restart at 0x00
  Back
```

Root Access has 50 built-in animations, selected by a single byte `0x00`–`0x31`.
**Next** and **Previous** step through them and wrap around; **Jump to...**
opens the byte editor bounded to `0x31`.

### Where to plug things in

Both SAO headers are wired to the same I2C bus, so a device works in either:

```
J3 (right)   3.0V   SDA   SCL   GND   SAO_GPIO1   SAO_GPIO2
J4 (left)    3.0V   SDA   SCL   GND   SAO_GPIO3   SAO_GPIO4
```

Note that `SAO_GPIO1..4` are **not** SDA/SCL — they are four spare GPIOs. This
app talks to the header's real `SDA`/`SCL` pins through the SoC's hardware I2C
block.

### Two addresses are refused on purpose

`0x19` and `0x3C` are the badge's own accelerometer and display controller,
sharing the bus with the SAO headers. The app refuses them outright and reports
`badge device - refused`.

This is deliberate and conservative. The I2C interface this app is built on
cannot express an address-only probe — every transaction it can issue puts at
least one byte on the wire — so "just probing" the badge's own display
controller would mean writing to its register pointer. Refusing is cheaper than
finding out what that does.

## Known issues / why it feels glitchy

| | |
|---|---|
| **The badge freezes during a scan** | The scan is synchronous: ~112 addresses, each a real bus transaction. `Scanning...` is drawn first, then nothing redraws until it finishes. It is working, not hung. |
| **A "scan" is really a read** | Because there is no address-only probe (see above), each check is a one-byte read, which sets the target's register pointer to 0. Well-behaved devices do not care. Unusual ones might. |
| **Devices show up that are not really there** | Anything that ACKs a one-byte read is listed. Some chips ACK addresses they do not own. |
| **A device answers on one visit and not the next** | Nothing here retries or recovers the bus. A device mid-transaction from a previous poke can NAK until it is power-cycled. |
| **Exit drops you to the idle screen** | Not to the menu you came from. Open the menu again to re-enter. |
| **Only single-byte writes** | There is no register-write, no block read, no repeated start. This browses a bus; it does not speak any device's protocol beyond Root Access's one-byte mode select. |

### Boot fragility

The build this image came from sits close to a known, unresolved bug: on this
project's Windows toolchain, adding roughly 2.5 KB of app code produced an image
whose **loader** hangs at about half of the boot progress bar, before any app
code ever runs. The cause looks like an assertion in the loader about how
process images are laid out in the swap area, not a defect in the app itself.

The images in this repo are not affected — they have been flashed and booted.
The reason to mention it at all is that it is sensitive to how much code the app
contains, so a *future* build could hit it again, and the symptom is easy to
mistake for a bad copy: if the progress bar stops halfway and the BAOCHIP drive
does not come back, that is this bug, not something you did. See
[If it goes wrong](#if-it-goes-wrong).

## If it goes wrong

The bootloader is in ROM and cannot be overwritten by any of this, so the badge
is always recoverable:

1. Hold any button while plugging the module in — BAOCHIP comes back.
2. Copy a known-good image over.

bunnie's official firmware is the safe fallback:

**https://ci.betrusted.io/releases/latest/baochip/dc34-badge/latest.zip**

Reflashing that undoes everything here except the erased `Ko` key, which is
gone for good.

## Building it yourself

These images are built from bunnie's badge software and the Xous OS, with local
patches adding the app:

- [bunnie/dc34-vault](https://github.com/bunnie/dc34-vault) — the badge app the
  Root Access screens live in
- [bunnie/dc34-console](https://github.com/bunnie/dc34-console) — the badge
  shell
- [betrusted-io/xous-core](https://github.com/betrusted-io/xous-core) — the OS
  and the image packer

The patches are not in this repo — it holds only the built images. Open an issue
if you want the source.

## Credits

The badge, its hardware, and all of the firmware these images are built on are
[bunnie](https://github.com/bunnie)'s and the
[betrusted-io](https://github.com/betrusted-io) project's work; `xous-core` is
Apache-2.0. The Root Access app is the only part added here.

Not affiliated with or endorsed by DEF CON, bunnie, or the badge team.
