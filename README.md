# Cisco CP-9951 DOOM v.666

<p align="center">
  <img alt="release" src="https://img.shields.io/badge/release-v.666-8B0000?style=for-the-badge">
  <img alt="platform" src="https://img.shields.io/badge/platform-Cisco%20CP--9951-1f6feb?style=for-the-badge">
  <img alt="cpu" src="https://img.shields.io/badge/CPU-ARMv6-444?style=for-the-badge">
  <img alt="kernel" src="https://img.shields.io/badge/kernel-2.6.18__pro500-555?style=for-the-badge">
</p>

<p align="center">
  <img alt="video" src="https://img.shields.io/badge/video-%2Fdev%2Ffb1-success?style=flat-square">
  <img alt="input" src="https://img.shields.io/badge/input-%2Fdev%2Finput%2Fkeypad0-success?style=flat-square">
  <img alt="sound" src="https://img.shields.io/badge/sound-local%20G.711%20%C2%B5--law-success?style=flat-square">
  <img alt="offline" src="https://img.shields.io/badge/offline%20cold%20boot-PROVEN-success?style=flat-square">
  <img alt="iwad" src="https://img.shields.io/badge/IWAD-Freedoom%20Phase%202-blueviolet?style=flat-square">
  <img alt="cat" src="https://img.shields.io/badge/cat%20wallpaper-MANDATORY-ff69b4?style=flat-square">
</p>

> **Native DOOM on a €35 Cisco IP phone.**  
> Not streamed. Not remotely controlled. Not dependent on Ethernet after installation.

The Cisco CP-9951 runs DOOM on its own ARM CPU, renders directly through `/dev/fb1`, reads its real physical keypad through `/dev/input/keypad0`, produces sound through the phone's local media path, survives reboot and launches from **Applications → Doom**.

On **15 September 2026**, the final build was physically verified after a cold boot **with the Ethernet cable disconnected**.

<p align="center">
  <b>Stage 1:</b> Cisco displays DOOM from Cthulhu<br>
  <b>Stage 2:</b> Cisco controls DOOM on Cthulhu<br>
  <b>v.666:</b> <strong>the Cisco itself runs DOOM</strong>
</p>

---

## Final verified result

| | Physically verified state |
|---|---|
| Phone | Cisco Unified IP Phone CP-9951 |
| Firmware | `sip9951.9-2-1` |
| OS | MontaVista Linux Professional Edition Blackfoot |
| Kernel | `Linux 2.6.18_pro500` |
| CPU | ARMv6 / ARMv6l |
| Cisco hardware ID | `raven` |
| Rendering | `/dev/fb1` |
| Physical input | `/dev/input/keypad0` |
| Persistent install | `/mnt/flash2/doom-v666` |
| Launch | **Applications → Doom** |
| Sound | local 8 kHz G.711 µ-law path |
| Network after install | **not required** |
| Exit | red handset button :D |
| IWAD | Freedoom Phase 2 |

The final acceptance test was intentionally simple:

```text
unplug Ethernet
remove power
cold boot with power only
open Applications
select Doom
play with sound
exit with red handset button
```

No Cthulhu service, external HTTP/TFTP/FTP, SSH launch or Internet connection is needed for normal final operation.

---

## This is not the original streamed version

The project began with the CP-9951 acting as a SIP/H.264 thin client. DOOM ran on the Linux workstation **Cthulhu**, while Asterisk and Baresip delivered live video and audio to the phone.

The next version used the physical Cisco keypad as a controller through RFC4733 DTMF → Asterisk AMI → Python → Linux `uinput`.

Then direct shell access changed the project completely.

The phone turned out to expose a small embedded ARM Linux environment with local framebuffer and input interfaces. That made a native port possible.

The complete chronology is in [`docs/project-history.md`](docs/project-history.md).

The longer workshop story, photos and video belong to:

### [Field Note #007 — I Bought a €35 Cisco Phone. Naturally, I Put DOOM on It.](https://www.dennishilk.com/museum/home-computing-lab/field-notes/field-note-7/)

---

## Under the hood

The first direct shell login exposed:

```text
MontaVista(R) Linux(R) Professional Edition Blackfoot
Cisco IP Phone 9951 9-2-1
```

and later inspection showed the local hardware interfaces needed for a real native game port:

```text
/dev/fb0
/dev/fb1
/dev/fb2
/dev/fb3

/dev/input/keypad0
/dev/input/touchscreen0
/dev/input/hookswitch0
```

`/proc/bus/input/devices` identifies the important local interfaces as:

```text
Raven Keypad
Raven Touchscreen
Raven Hookswitch
```

### Local keypad path

The physical input path was reverse engineered as:

```text
physical key
    ↓
Raven keypad hardware driver
    ↓
Cisco keyhandle layer
    ↓
/dev/input/keypad0
    ↓
local userspace application
```

Each keypad event is a 16-byte Linux-style input record. The numeric keypad, navigation cluster, softkeys, line keys, volume keys and feature buttons were measured directly on the phone.

The complete map and event format are documented in [`docs/keypad-reverse-engineering.md`](docs/keypad-reverse-engineering.md).

---

## Native video and controls

DOOM executes directly on the CP-9951 CPU and renders to `/dev/fb1`.

The final binary reads the Cisco keypad locally rather than using DTMF or desktop-side forwarding.

Notable controls include:

- navigation cluster for movement/menu navigation
- OK for menu confirmation
- `*` for fire in the verified mapping
- `#` for use/open in the verified mapping
- **red handset button to exit DOOM cleanly**

The normal Cisco UI returns after exit.

---

## Local sound

Sound was the last major runtime problem.

The final implementation emits compact DOOM sound events to a small phone-local relay/service path. The service produces **8 kHz G.711 µ-law RTP** for the Cisco media subsystem.

The important physical fix is that RTP packets are explicitly bound to source `127.0.0.1`. With Ethernet present the configured phone address can be used; after a cable-free cold boot, the service falls back to loopback rather than failing because the Ethernet route does not exist.

Both paths were physically verified with audible DOOM sound effects.

The final implementation does **not**:

- ship Cisco firmware
- ship proprietary Cisco libraries
- start a second private `libms` client
- use `LD_PRELOAD`
- kill/replace the Cisco media service
- blindly write raw audio to `/dev/dsplink`

---

## Persistence and Applications launch

The persistent installation root is:

```text
/mnt/flash2/doom-v666
```

The Cisco Applications menu launches the local endpoint:

```text
http://127.0.0.1:8095/launch
```

An isolated persistent xinetd service is loaded by Cisco's existing unmodified boot sequence. The program, WAD and helpers remain on the phone across reboot.

Full installation, rollback and verified hash information is in [`docs/installation.md`](docs/installation.md).

---

## Distribution

The physically verified distribution is named:

```text
Cisco9951-doom.doompkg
```

The intended public GitHub release is **v.666 — Cisco CP-9951 DOOM Edition**.

The package contains the physically tested ARM build, Freedoom Phase 2, local helpers, installer/rollback material, source/build support files, license notices and SHA-256 verification data.

It does **not** contain:

- commercial Doom IWADs
- Cisco firmware
- proprietary Cisco libraries

See [`release/README.md`](release/README.md) for the publication layout.

---

## Repository docs

- [`docs/project-history.md`](docs/project-history.md) — the full escalation from SIP/H.264 streaming to native offline v.666
- [`docs/keypad-reverse-engineering.md`](docs/keypad-reverse-engineering.md) — Raven/keyhandle event format and measured CP-9951 key map
- [`docs/installation.md`](docs/installation.md) — installation, sound path, persistence, rollback and hashes
- [`docs/media-notes.md`](docs/media-notes.md) — publication/media plan

---

## Safety / scope

This is an experimental community project for privately owned hardware. It is not affiliated with or supported by Cisco.

The final design deliberately avoids bootloader modification, firmware replacement, raw NAND/block writes and replacement of Cisco's core media service. Keep backups of provisioning and phone-side files and use the rollback path if needed.

---

## The most important design requirement

> **The cat wallpaper stays.**

That requirement was established before any of the useful engineering decisions and has therefore been preserved throughout the project.

<p align="center">
  <img alt="questionable engineering" src="https://img.shields.io/badge/questionable%20engineering%20decisions-physically%20verified-ff8c00?style=for-the-badge">
</p>
