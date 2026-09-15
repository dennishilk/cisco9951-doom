# Project history — from €35 office phone to DOOM v.666

## Stage 1 — normal Cisco phone, then Werner

The experiment started with a used Cisco Unified IP Phone CP-9951 that cost about €35. The phone remained on its original Cisco firmware (`sip9951.9-2-1`) and was provisioned in the homelab with DHCP/TFTP and Asterisk.

The first deliberately unnecessary use was a media endpoint called **Werner** on extension `111`, proving that the phone could receive suitable H.264 video and PCMU audio.

The cat wallpaper was kept from the beginning. This turned out to be the correct priority.

## Stage 2 — streamed DOOM on 666

The first DOOM implementation did **not** run on the Cisco itself. DOOM ran on the Linux workstation **Cthulhu** and the Cisco acted as a SIP/H.264 thin client.

The useful low-latency path became:

```text
DOOM / Sway
  ↓
wf-recorder
  ↓
v4l2loopback (/dev/video42)
  ↓
Baresip v4l2 / H.264 RTP
  ↓
Asterisk
  ↓
Cisco CP-9951
```

Audio came from Cthulhu and was sent as PCMU over SIP/RTP.

Dialing `666` therefore showed the same live DOOM session to the phone; it did not create a new game instance per caller.

## Stage 3 — the Cisco becomes the controller

The phone keypad was next used as a controller for the still-remote game.

The early path was:

```text
Cisco keypad
  ↓
RFC4733 DTMF
  ↓
Asterisk AMI
  ↓
Python bridge
  ↓
Linux uinput
  ↓
DOOM on Cthulhu
```

At this point the Cisco was both display and controller, but the game still executed elsewhere.

## Stage 4 — the turning point: getting a shell on the phone

Until this point the CP-9951 was still, for practical purposes, a very unusual SIP/H.264 client.

Then we found that Cisco had provided an official SSH diagnostic-access path for this generation of 89xx/99xx phones.

This was **not an exploit**, not a bootloader modification and not a modified firmware image.

Cthulhu was already supplying the phone's provisioning through DHCP/TFTP, so the existing provisioning path could be used:

```text
Cthulhu
  ↓
dnsmasq / TFTP
  ↓
SEPC40ACB4D05D0.cnf.xml
  ↓
Cisco CP-9951
```

Before changing anything, the working provisioning file was backed up as:

```text
SEPC40ACB4D05D0.cnf.xml.pre-ssh
```

SSH credentials for the first authentication stage were then added through the SEP provisioning file. After the phone reloaded the configuration, TCP port 22 was actually reachable.

There was one more time-travel problem: the phone's SSH server is old enough that a current OpenSSH client on Cthulhu would not negotiate with it cleanly.

The practical solution was an appropriately old client:

```text
PuTTY 0.60 / plink
```

The command that became the normal connection method was:

```text
/tmp/putty-0.60/unix/plink -ssh nebu@10.1.1.2
```

After that first SSH authentication, the phone presented a **second login**. On this Cisco generation, the outer SSH login and the internal Linux shell login are separate stages.

Using Cisco's internal `default` login finally landed on the real Linux shell of the phone.

And that was the moment the project changed completely:

```text
Welcome to MontaVista Linux Professional Edition Blackfoot
```

The shell user was not root:

```text
uid=65533(default)
gid=100(users)
```

But it was more than sufficient to inspect the running system directly.

`uname`, `/proc`, `/dev` and the available BusyBox tools revealed:

- Linux `2.6.18_pro500`
- ARMv6 / ARMv6TEJ
- platform `raven`
- roughly 244 MB RAM
- Cisco Enhanced BusyBox 1.9.1
- multiple framebuffer devices
- local input devices
- writable or otherwise useful runtime storage areas

Important local interfaces included:

```text
/dev/fb0
/dev/fb1
/dev/fb2
/dev/fb3

/dev/input/keypad0
/dev/input/touchscreen0
/dev/input/hookswitch0
```

The phone was no longer merely a proprietary SIP appliance. It was an embedded ARM Linux computer that could be investigated directly.

The question changed from:

> **How far can we abuse this Cisco phone as a display and controller?**

into:

> **Wait. If this thing runs Linux on ARM... can we just run DOOM on the phone itself?**

A few days later, the answer was yes. :D

## Stage 5 — Raven keypad reverse engineering

The physical keypad path was reverse engineered from Cisco's local input stack.

The important result was:

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

The events are 16-byte Linux-style input records. A complete useful key map was measured directly on the physical phone.

This removed the need for DTMF, SIP signaling or Cthulhu-side input forwarding for a native application.

See [`keypad-reverse-engineering.md`](keypad-reverse-engineering.md) for the full event format and measured key map.

## Stage 6 — native DOOM

DOOM was then built for the CP-9951's ARM CPU.

The final native backend:

- executes directly on the phone CPU
- renders locally through `/dev/fb1`
- reads the real Cisco keypad locally
- supports complete gameplay
- uses OK for confirmations and menu interaction
- exits cleanly with the red handset button

At this point the distinction became important:

> **DOOM is no longer streamed from Cthulhu.**

## Stage 7 — local sound

Sound was the difficult final runtime problem.

The completed implementation emits compact DOOM sound events and uses a small local relay/service path that produces 8 kHz G.711 μ-law RTP for the existing Cisco media system.

The implementation does not load a second private Cisco `libms` client, use `LD_PRELOAD`, kill Cisco media processes or blindly write raw audio to `/dev/dsplink`.

A key physical fix binds RTP packets to source `127.0.0.1`. If a cold boot occurs without Ethernet, the service falls back to loopback instead of failing because the Ethernet route is absent.

This was physically verified with audible DOOM sound.

## Stage 8 — persistence and Applications menu

The game, Freedoom IWAD and helpers were installed persistently under:

```text
/mnt/flash2/doom-v666
```

A local application entry launches DOOM through:

```text
Applications → Doom
```

using the phone-local launch URL:

```text
http://127.0.0.1:8095/launch
```

The existing Cisco xinetd startup path is used with an isolated persistent service. Installer and rollback paths are part of the package.

## Final physical acceptance test — 15 September 2026

The decisive test was performed on the real CP-9951:

- Ethernet cable disconnected
- phone power-cycled from cold
- no Cthulhu service available
- no manually started shell service
- **Applications → Doom** after boot
- DOOM launches locally
- framebuffer video works
- physical Cisco controls work
- sound is audible from the phone
- the game is fully playable
- the red handset button exits cleanly

The final answer therefore changed from:

> "Yes — as a near-realtime SIP/H.264 thin client."

into:

> **Yes. The Cisco CP-9951 runs DOOM natively, locally, persistently and offline.**

## Distribution

The final distribution is named:

```text
Cisco9951-doom.doompkg
```

It uses **Freedoom Phase 2** as the redistributable IWAD. Cisco firmware, proprietary Cisco libraries and commercial Doom IWADs are not distributed.

The public presentation name is:

# Cisco CP-9951 DOOM v.666

And the cat wallpaper stays.
