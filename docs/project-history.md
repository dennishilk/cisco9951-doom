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

## Stage 4 — first shell on the phone

Direct shell access changed the project completely.

The CP-9951 exposed an embedded Linux environment:

```text
MontaVista Linux Professional Edition Blackfoot
Linux 2.6.18_pro500
ARMv6 / ARMv6l
Hardware: raven
BusyBox
```

Local hardware interfaces included multiple framebuffers and Linux-style input devices such as:

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
