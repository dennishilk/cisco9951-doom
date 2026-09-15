# Cisco CP-9951 — Local Keypad Input Reverse Engineering

**Device:** Cisco Unified IP Phone CP-9951-CL-K9  
**Firmware:** `sip9951.9-2-1`  
**Kernel:** Linux `2.6.18_pro500`, ARMv6, MontaVista Linux  
**Date:** 2026-09-13

---

**The CP-9951 exposes its physical keypad locally as Linux-style input events.**

The relevant character device is:

```text
/dev/input/keypad0
```

The device is backed by Cisco kernel modules:

```text
keyhandle 7684 1 - Live 0xbf03d000
keypad    6264 0 - Live 0xbf03a000
```

Relevant symbols from the running kernel include:

```text
keypadClassDev_fop_poll
keypadClassDev_fop_fasync
keypadClassDev_fop_release
keypadClassDev_fop_open
keypadClassDev_disconnect
keypadClassDev_event
keypadClassDev_fop_read
keypadClassDev_connect
keypadClassDev_fop_ioctl

ravenKeypad_open
ravenKeypad_close
ravenKeypad_pressedKeyIntr
ravenKeypad_freeKeyIntr
```

This establishes a clear path:

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

No remote key forwarding is required.

---

## `keyhandle.ko`

The original module was copied read-only from the phone:

```text
/lib/modules/2.6.18_pro500/extra/keyhandle.ko
```

File size:

```text
10276 bytes
```

On Cthulhu:

```text
ELF 32-bit LSB relocatable, ARM, EABI4 version 1 (SYSV), not stripped
```

The fact that it is **not stripped** made the internal keypad implementation directly inspectable.

---

## Event format

Reverse engineering of:

```text
keypadClassDev_event()
keypadClassDev_fop_read()
```

showed that each event is exactly **16 bytes**.

The structure matches the 32-bit Linux `struct input_event` layout:

```c
struct input_event_32 {
    uint32_t tv_sec;
    uint32_t tv_usec;
    uint16_t type;
    uint16_t code;
    int32_t  value;
};
```

Byte layout:

```text
+0x00  tv_sec       4 bytes
+0x04  tv_usec      4 bytes
+0x08  type         2 bytes
+0x0a  code         2 bytes
+0x0c  value        4 bytes
```

Observed event semantics:

```text
type = 0x0001  -> key event
type = 0x0000  -> SYN / synchronization event

value = 1      -> key press
value = 0      -> key release
```

Example for physical key `5`:

```text
... 01 00 06 02 01 00 00 00
```

Decoded:

```text
type  = 0x0001
code  = 0x0206
value = 1
```

Release:

```text
... 01 00 06 02 00 00 00 00
```

Decoded:

```text
type  = 0x0001
code  = 0x0206
value = 0
```

---

## Important read behavior

A fresh open of `/dev/input/keypad0` initially returns one all-zero 16-byte slot:

```text
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

Inspection of `keypadClassDev_fop_open()` showed that the per-client structure is allocated with:

```text
kmem_cache_zalloc()
```

The event ring therefore starts zero-initialized.

The key event path advances the ring and then stores real events. This explains why one-shot reads initially appeared to return only zeros.

Keeping the same open alive and reading multiple records exposes real key events normally.

---

## Ring buffer behavior

The Cisco key handler maintains a small per-client event ring.

Reverse engineering shows:

```text
16-byte records
16 ring slots
read() supports multiple complete records
poll()
fasync()
wake-up support
```

This is very suitable for a game input loop.

---

# Confirmed CP-9951 key map

All codes below were measured directly from the physical phone.

## Numeric keypad

| Physical key | Event code |
|---|---:|
| `0` | `0x0201` |
| `1` | `0x0202` |
| `2` | `0x0203` |
| `3` | `0x0204` |
| `4` | `0x0205` |
| `5` | `0x0206` |
| `6` | `0x0207` |
| `7` | `0x0208` |
| `8` | `0x0209` |
| `9` | `0x020a` |
| `*` | `0x020b` |
| `#` | `0x020c` |

The numeric block is therefore a clean contiguous range:

```text
0x0201 .. 0x020c
```

---

## Navigation cluster

| Physical key | Event code |
|---|---:|
| Up | `0x0220` |
| Center / OK | `0x0221` |
| Down | `0x0222` |
| Left | `0x0223` |
| Right | `0x0224` |

Contiguous range:

```text
0x0220 .. 0x0224
```

---

## Back and softkeys

| Physical key | Event code |
|---|---:|
| Back | `0x020d` |
| Softkey 1, leftmost | `0x020e` |
| Softkey 2 | `0x020f` |
| Softkey 3 | `0x0210` |
| Softkey 4, rightmost | `0x0211` |

This continues directly after the numeric keypad:

```text
0x0201 .. 0x0211
```

with `0x020d` as Back and `0x020e..0x0211` as the four display softkeys.

---

## Programmable line keys

Five physical programmable buttons exist on the right side of the display.

| Physical key | Event code |
|---|---:|
| Line key 1, top | `0x0218` |
| Line key 2 | `0x0219` |
| Line key 3 | `0x021a` |
| Line key 4 | `0x021b` |
| Line key 5, bottom | `0x021c` |

Contiguous range:

```text
0x0218 .. 0x021c
```

---

## Volume

| Physical key | Event code |
|---|---:|
| Volume + | `0x021e` |
| Volume - | `0x021f` |

---

## Feature / audio buttons

| Physical key | Event code |
|---|---:|
| Messages / envelope | `0x0225` |
| Contacts / address book | `0x0226` |
| Applications / gear | `0x0227` |
| Headset | `0x0228` |
| Speaker | `0x0229` |
| Mute | `0x022a` |

Again, a clean contiguous range:

```text
0x0225 .. 0x022a
```

---

# Consolidated map

```text
0x0201  0
0x0202  1
0x0203  2
0x0204  3
0x0205  4
0x0206  5
0x0207  6
0x0208  7
0x0209  8
0x020a  9
0x020b  *
0x020c  #
0x020d  Back
0x020e  Softkey 1
0x020f  Softkey 2
0x0210  Softkey 3
0x0211  Softkey 4

0x0218  Line key 1
0x0219  Line key 2
0x021a  Line key 3
0x021b  Line key 4
0x021c  Line key 5

0x021e  Volume +
0x021f  Volume -

0x0220  Up
0x0221  OK / Center
0x0222  Down
0x0223  Left
0x0224  Right
0x0225  Messages
0x0226  Contacts
0x0227  Applications
0x0228  Headset
0x0229  Speaker
0x022a  Mute
```

---

# Example raw capture

Physical `5` press and release:

```text
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
92 84 a6 6a 58 1c 01 00 01 00 06 02 01 00 00 00
92 84 a6 6a 65 1c 01 00 00 00 00 00 00 00 00 00
92 84 a6 6a 77 6a 02 00 01 00 06 02 00 00 00 00
```

Decoded:

```text
initial zero slot
EV_KEY  code=0x0206  value=1
EV_SYN
EV_KEY  code=0x0206  value=0
```

Physical Up press and release:

```text
EV_KEY  code=0x0220  value=1
EV_SYN
EV_KEY  code=0x0220  value=0
```

---

# Key technical conclusion

The Cisco CP-9951 does **not** require remote DTMF, SIP signaling, streamed controller input, or desktop-side key forwarding for game control.

The real physical keypad is exposed inside the phone through Cisco's Raven keypad driver and `keyhandle` module as Linux-style 16-byte input events.
