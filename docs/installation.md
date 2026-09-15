# Installation and verified runtime

This document describes the physically verified Cisco CP-9951 DOOM v.666 installation path.

## Verified configuration

| Setting | Tested value |
|---|---|
| Phone | Cisco CP-9951 |
| Firmware | `sip9951.9-2-1` |
| Phone address | `10.1.1.2` |
| Deployment host | `10.1.1.1` |
| FTP port | `2121` |
| Install root | `/mnt/flash2/doom-v666` |
| Applications URL | `http://127.0.0.1:8095/launch` |

The two phone-side values are centralized in `config/doom.env`:

```sh
DOOM_ROOT=/mnt/flash2/doom-v666
PHONE_IP=10.1.1.2
```

Change them before preparing the phone archive if your laboratory differs. The installer intentionally accepts only an installation root below `/mnt/flash2`.

## Prepare on the deployment host

The tested provisioning file was:

```text
/home/nebu/cisco9951-tftp/SEPC40ACB4D05D0.cnf.xml
```

Run:

```sh
cd Cisco9951-doom
host/prepare-on-cthulhu.sh
```

This validates the Doom executable, creates the FTP payloads, backs up the provisioning XML and replaces old Doom / Doom Beep entries with one local Doom entry.

The prepared files are:

```text
/tmp/cisco9951-ftp/Cisco9951-doom-phone.tar
/tmp/cisco9951-ftp/install-Cisco9951-doom.sh
```

Custom paths may be supplied as positional arguments:

```sh
host/prepare-on-cthulhu.sh DOOM_BINARY FTP_ROOT PROVISIONING_XML
```

Serve the selected FTP root on the deployment host. For the physically tested setup:

```sh
python -m pyftpdlib -i 10.1.1.1 -p 2121 -w -d /tmp/cisco9951-ftp
```

## Install on the phone

In the CP-9951 shell:

```sh
busybox ftpget -P 2121 -u anonymous -p cisco@ 10.1.1.1 \
  /tmp/install-Cisco9951-doom.sh install-Cisco9951-doom.sh
chmod 0755 /tmp/install-Cisco9951-doom.sh
/tmp/install-Cisco9951-doom.sh
```

Success ends with:

```text
CISCO9951-DOOM INSTALLED
```

Reboot the phone once so its existing xinetd startup path loads `/usr/local/etc/doom-local`. Reload the phone configuration if the Applications menu has not yet picked up its new entry, then choose:

**Applications → Doom**

## Controls

The package preserves the physically verified Cisco keypad mapping in the ARM binary.

Notable controls:

- OK confirms menu choices.
- `*` fires during the final SFX check.
- the red handset button exits Doom cleanly.

For the complete measured event map, see [`keypad-reverse-engineering.md`](keypad-reverse-engineering.md).

## Audio design

Doom writes compact events to file descriptor 9. The launcher connects that descriptor to a phone-local FIFO. A small relay forwards those events to the local service on UDP port 8096. The service emits 8 kHz G.711 μ-law RTP to the Cisco RTPRx endpoint.

Cisco's existing xinetd owns loopback port 8095; the installed bridge starts the backend once on port 8097 and proxies the Applications request to it.

The CP-9951-specific physical fix is important: RTP packets are explicitly bound to source `127.0.0.1` and normally sent to the phone address configured as `PHONE_IP` on port 20480. During a cable-free cold boot that Ethernet route does not exist, so the service falls back to a loopback destination instead of exiting. Both routes were physically verified with audible Doom SFX.

The implementation does **not** load `libms`, use `LD_PRELOAD`, kill Cisco media processes or write raw data to `/dev/dsplink`.

## Rollback

Phone files are backed up before replacement. To restore them:

```sh
/mnt/flash2/doom-v666/bin/rollback-Cisco9951-doom.sh
```

To restore the deployment host's provisioning XML:

```sh
host/rollback-applications-entry.sh
```

## Persistent cold-boot trigger

The program, WAD and helpers are installed under the single configurable `DOOM_ROOT`. The installer also adds the isolated xinetd service `/usr/local/etc/doom-local` on the phone's persistent `/mnt/flash` filesystem. Its generated `server` path comes from the same `DOOM_ROOT`; there is no second install-root setting. Cisco's unmodified boot sequence loads that service.

The completed physical acceptance test was:

1. disconnect LAN
2. power-cycle the phone
3. open **Applications**
4. select **Doom**
5. play with audible sound
6. exit with the red handset button

No shell command or deployment host was involved after the cold boot.

## Verification hashes

Physically verified payload hashes:

```text
Doom executable
67cd221941e64b1d8182b9b69aea991f4f8cd62c0aa1d10266aa0b7d8e1dc9bb

local service
92f83e055691624ccf021f1a6c8639bb426d7d2a261f0ac077a92d11fc9f2ea7

xinetd HTTP proxy
34006fa495cc867c583ed4234d4d140dadb85318b13d053ee8ae14319f8836d5

Doom sound relay
6d2fbe16a0ea9189f9ce3f28dbf037d110a80fc182958a95da94c6864eec1dfb
```

Run `sha256sum -c MANIFEST.sha256` from the package root to verify every distributed file.

## Licensing and warning

The package uses Freedoom Phase 2 as the redistributable IWAD and does not include Cisco firmware, proprietary Cisco libraries or a commercial Doom IWAD.

See the package license notices and source files for the applicable licenses. This is an experimental community project for privately owned hardware. It is not affiliated with or supported by Cisco. Use it at your own risk and keep backups of provisioning and phone-side files.
