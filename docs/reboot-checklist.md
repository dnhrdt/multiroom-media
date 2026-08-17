# Audio Upgrade, Reboot, Diagnosis, and Recovery Runbook

Version: 2026-08-17

This is the fixed production procedure for Unraid upgrades, ordinary reboots,
and UMC1820 disconnect recovery on Star Destroyer. It separates observation
from intervention and requires a recovery point before every production change.

Do not use package installation, service restarts, container restarts, module
reloads, USB resets, or mixer changes as diagnostic shortcuts.

## Known Baselines

| State | Unraid | Kernel | Required sound package |
|---|---|---|---|
| Current production | 7.2.5 | `6.12.85-Unraid` | `sound-20260430-6.12.85-Unraid-1.txz` |
| Evaluated candidate | 7.3.2 | `6.18.38-Unraid` | `sound-20260707-6.18.38-Unraid-1.txz` |

Candidate package evidence:

- Unraid 7.3.2 release notes:
  `https://docs.unraid.net/unraid-os/release-notes/7.3.2/`
- Matching sound-driver release:
  `https://github.com/ich777/unraid-sound-driver/releases/tag/6.18.38-Unraid`
- Package SHA-256:
  `cfafaffb4f84fd8d1085a9d23aa56e414b21eeae605e885516cacac630f03811`
- The package contains `snd-usb-audio.ko.xz` and `snd-usbmidi-lib.ko.xz` under
  `lib/modules/6.18.38-Unraid/`.

Unraid 7.3.2 is only a prepared candidate. Do not place its package in active
`/boot/extra` or update Star Destroyer outside an approved maintenance window.

## Why An Unraid Update Can Break Audio

There are three distinct failure classes.

### 1. Kernel and sound package mismatch

The ich777 sound package contains kernel modules built for one exact Unraid
kernel. A package for `6.12.85-Unraid` cannot load on `6.18.38-Unraid`.

Typical evidence:

- UMC1820 is visible in `lsusb`.
- `/proc/asound/cards` is empty or missing the UMC1820.
- `modprobe snd-usb-audio` reports a missing module or version problem.
- Kodi can read tracks but playback advances far too quickly because no real
  audio clock is available.

### 2. Overlapping `libasound` packages

Both the ich777 sound package and the pinned Slackware ALSA package contain
`/usr/lib64/libasound.so.2.0.0`.

The sound packages evaluated for both kernels contain the same stripped file:

- Size: 941,792 bytes
- SHA-256:
  `78fbd567234052902b97ab9fbeae126b8a10566d44a565304fcb166456c411a6`
- Missing symbol: `snd_use_case_mgr_open`

The known-good `alsa-lib-1.2.14-x86_64-1.txz` contains:

- Size: 1,143,568 bytes
- Runtime-library SHA-256:
  `92ac395d340e98797c2a15319c3e01400d5a69917b8b0e21947e1fae0048a59f`
- Required UCM symbol present

The current Unraid 7.2.5 boot path replays root-level `/boot/extra` files with
an unsorted `find` plus `upgradepkg --install-new`. The final owner of the
overlapping file is not a safe contract. A syslog line saying that alsa-lib was
installed is therefore not sufficient evidence. Re-check this boot logic on
every target release because Unraid 7.3.2 also updates `pkgtools`.

The hardened `start_pulseaudio.sh` is the required final guard. Immediately
before PulseAudio starts, it verifies the UCM symbol and reinstalls the pinned
ALSA package when necessary.

Do not replace 1.2.14 with a newer Slackware package merely because its version
number is higher. Version 1.2.15.3 was tested on 2026-05-09 and failed with the
same undefined-symbol error.

### 3. Live USB disconnect

A live power or USB disconnect is independent of an Unraid upgrade. The kernel
may enumerate the UMC1820 again while PulseAudio still owns a stale ALSA device
handle. A controlled PulseAudio and Kodi recovery may then be required.

If USB and ALSA are healthy but software output never reaches the Audac, a
physical UMC1820 power cycle may be required. This happened during the
2026-07-02 recovery.

## Phase 0: Candidate Preflight — No Production Changes

Complete this before scheduling an upgrade.

1. Read the official release notes and record the exact target kernel.
2. Confirm that an ich777 sound-driver release exists for that exact kernel.
3. Download the exact package locally or into a repository-owned WIP folder.
4. Verify the release-provided SHA-256.
5. Inspect the archive for the exact kernel module path and
   `snd-usb-audio.ko.xz`.
6. Inspect its embedded `libasound.so.2.0.0`; assume the pinned 1.2.14 overlay
   remains necessary unless an isolated test proves otherwise.
7. Review release notes for changes to the kernel, glibc, libusb, Docker,
   networking, and rollback restrictions.
8. Do not combine the audio upgrade with internal-boot migration, TPM licensing
   migration, ZFS feature upgrades, or unrelated hardware changes. Those can
   invalidate the simple OS rollback path.

Preflight acceptance requires all of the following:

- Exact target kernel known.
- Exact matching sound package available and hashed.
- Pinned ALSA 1.2.14 artifact available and hashed.
- Current live configuration and package state understood.
- Backup and rollback procedure defined.
- Maintenance window includes physical access to the UMC1820 and Audac.

## Phase 1: Recovery Point Before Any Unraid Change

For an OS update, create a native Unraid boot-device backup first:

`Main -> Boot Device -> Boot Device Backup`

Also capture a repo-local evidence snapshot before touching `/boot`, packages,
processes, modules, mixer state, or containers. Include at minimum:

- `/boot/config/plugins/pulseaudio/`
- `/boot/config/plugins/user.scripts/scripts/pulseaudio_for_kodi/script`
- `/boot/extra/*alsa*`
- `/boot/extra/*sound*`
- File listings and SHA-256 hashes for all files above
- `/var/log/packages/alsa-lib*`
- `/var/log/packages/sound-*`
- `uname -r` and `/etc/unraid-version`
- `lsusb` and `/proc/asound/cards`
- PulseAudio sinks, clients, sink inputs, and master volume
- ALSA mixer state
- Relevant PCM `status` and `hw_params`
- Kodi container state, health, restart policy, and PulseAudio environment
- Current audio logs and relevant syslog lines

Record separately:

- The exact files and runtime state that will change.
- The exact rollback commands or restore operation.
- The expected acceptance evidence after the reboot.

## Phase 2: Stage the Target Driver Safely

Never place a future-kernel sound package beside the active package in
root-level `/boot/extra` days before the upgrade. An unexpected reboot on the
old kernel could leave the wrong package active.

Stage the candidate outside `/boot/extra`, for example under a dedicated
PulseAudio staging directory, and verify its hash there. Only during the
maintenance window should the old sound package be moved to a recovery folder
and the exact target package be placed in `/boot/extra` for the next boot.

Keep these artifacts available for rollback:

- Previous Unraid release files or the native Unraid rollback mechanism
- Previous kernel-matched sound package and its checksum
- Candidate kernel-matched sound package and its checksum
- Pinned ALSA 1.2.14 package
- Byte-exact live PulseAudio scripts and configuration

Do not mix the current manual `/boot/extra` method with the Sound Driver plugin
without a separate migration plan. Star Destroyer currently does not have the
Sound Driver plugin installed.

## Phase 3: Activation and Rollback Contract

Immediately before the upgrade reboot:

1. Confirm the Phase 1 snapshot and boot-device backup are readable.
2. Confirm that the target package hash still matches the accepted value.
3. Move the current sound package and checksum into the documented recovery
   location; do not delete them.
4. Place only the exact target-kernel sound package and checksum in active
   `/boot/extra`.
5. Keep the pinned ALSA package available.
6. Perform the Unraid update and reboot once.

Rollback must restore both halves of the version pair:

- Previous Unraid release/kernel
- Previous matching `sound-*.txz`

Never roll back only the OS or only the sound package.

If internal boot, TPM licensing, or ZFS feature flags were changed, stop and
review the release-specific rollback limitations instead of using this simple
pair rollback.

## Phase 4: Read-only Post-reboot Gate

Run the checks in this order. Do not intervene until the failing layer is
identified.

### Gate A — OS and kernel

```bash
cat /etc/unraid-version
uname -r
ls -l /boot/extra/*sound* /boot/extra/*alsa*
ls -l /var/log/packages/sound-* /var/log/packages/alsa-lib*
```

Acceptance:

- Expected Unraid release is running.
- `uname -r` exactly matches the active sound-package filename.

### Gate B — package contents and ALSA library

```bash
sha256sum /boot/extra/*sound* /boot/extra/*alsa*
ls -la /usr/lib64/libasound*
sha256sum /usr/lib64/libasound.so.2.0.0
grep -a -q snd_use_case_mgr_open /usr/lib64/libasound.so.2
echo $?
```

Acceptance:

- Package hashes match the preflight record.
- Runtime libasound contains `snd_use_case_mgr_open` after the hardened startup
  has run.

### Gate C — USB, driver, and ALSA card

```bash
lsusb | grep -Ei '1397:0503|Behringer|UMC1820'
modinfo -F vermagic snd_usb_audio
lsmod | grep -E '^snd_usb_audio|^snd_usbmidi_lib'
cat /proc/asound/cards
aplay -l
```

Acceptance:

- USB ID `1397:0503` is present.
- Driver vermagic matches the running kernel.
- UMC1820 exists as an ALSA playback card.

### Gate D — PulseAudio

```bash
export XDG_RUNTIME_DIR=/run/user/0
export PULSE_SERVER=unix:/run/user/0/pulse/native
pgrep -a pulseaudio
pactl info
pactl list sinks short
pactl get-sink-volume umc1820_master
pactl list clients short
pactl list sink-inputs short
ss -ltnp | grep ':4713'
```

Acceptance:

- Exactly one intended PulseAudio process.
- Six sinks: master plus five room remaps.
- Master volume is 63% / -12.04 dB and balanced.
- TCP 4713 is listening.

A listening TCP port alone is not success. PulseAudio can listen while the
hardware sink failed to load.

### Gate E — Kodi

```bash
docker ps --format '{{.Names}}|{{.Status}}' | grep '^kodi-'
```

Acceptance:

- All five Kodi containers are healthy.
- Five `kodi-x11` PulseAudio clients and five Kodi sink inputs appear when all
  players are connected.
- JSON-RPC `JSONRPC.Ping` returns `pong` from every Kodi.

### Gate F — physical signal path

```bash
cat /proc/asound/card*/pcm*/sub*/status
cat /proc/asound/card*/pcm*/sub*/hw_params
```

During active playback, acceptance is:

- ALSA playback `RUNNING`
- `S24_3LE`, 12 channels, 48 kHz
- Audac VU activity consistent with the selected routing
- Audible confirmation in at least one room

## Diagnosis Matrix

| Observation | Primary diagnosis | Next action |
|---|---|---|
| UMC1820 absent from `lsusb` | Physical power/cable/USB path | Restore physical connection; do not change packages |
| Present in `lsusb`, absent from `/proc/asound/cards` | Missing or mismatched kernel sound package/module | Compare kernel, package name, module path, and vermagic |
| ALSA card present, `snd_use_case_mgr_open` missing | Overlapping stripped libasound won boot replay | Use the hardened PulseAudio start only after snapshot/rollback preparation |
| PulseAudio listens on 4713 but has no six sinks | `module-alsa-sink` or config startup failure | Read syslog and `pa_start.log`; do not start Kodi |
| Six sinks exist, Kodi missing | Kodi startup was gated or failed | Verify sinks first, then use staggered Kodi startup |
| Software and PCM healthy, no Audac signal | USB interface state or downstream analog path | Check Audac VU/routing; consider controlled UMC1820 power cycle |
| One channel weak and moving Audac input gain restores it | Audac analog input potentiometer contact fault | Treat as hardware fault, not software balance |

## Controlled Repair Procedures

Every repair below is a production change. Capture the failing state and define
rollback first.

### Kernel package mismatch

1. Verify the exact running kernel.
2. Verify the exact matching sound package and checksum.
3. Preserve the currently active package and package logs.
4. Install only the matching package.
5. Run `depmod` for the exact kernel.
6. Load `snd-usb-audio` and verify `/proc/asound/cards` before touching
   PulseAudio.

### Stripped ALSA library or sinkless PulseAudio

The deployed hardened script is the canonical repair entry point:

```bash
bash /boot/config/plugins/pulseaudio/start_pulseaudio.sh
```

It verifies/reinstalls the ALSA library, waits for the UMC1820, replaces stale
PulseAudio processes, requires `umc1820_master`, and restores calibrated volume.

Verify six sinks before starting or restarting Kodi.

### Live UMC1820 disconnect

1. Confirm the UMC1820 is powered and present in `lsusb`.
2. Confirm it has returned to `/proc/asound/cards`.
3. If it has not, do not restart software repeatedly; resolve the physical or
   driver layer first.
4. Capture PulseAudio, ALSA, PCM, Docker, and log state.
5. Run the hardened PulseAudio start.
6. Verify six sinks.
7. Restart the Kodi containers with five-second staggering.
8. Verify PCM and audible output.

The repository contains `recover_audio_after_usb_reconnect.sh`, but it is not
currently deployed on Star Destroyer. Do not invoke a presumed live copy. Its
deployment should be a separate reviewed production change.

### Software healthy but no physical output

If PulseAudio, Kodi, and PCM all verify but the Audac has no signal:

1. Stop making software/package changes.
2. Record the verified healthy software state.
3. Stop PulseAudio cleanly.
4. Power-cycle the UMC1820 physically for approximately one minute.
5. Start through the hardened PulseAudio script.
6. Verify six sinks and calibrated volume.
7. Restart Kodi with staggering and verify Audac VU plus audible output.

## Completion Record

An upgrade or recovery is complete only when the session record includes:

- Before and after timestamps
- Old and new Unraid versions and kernels
- Old and new sound-package names and hashes
- Runtime libasound size, hash, and UCM-symbol result
- USB, ALSA, PulseAudio, Kodi, PCM, Audac, and audible evidence
- Every production change and touched path
- Rollback procedure and whether it was used
- Any deviation from this runbook

Persist the accepted final state in the repository Memory Bank.
