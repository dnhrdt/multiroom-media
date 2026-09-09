# Repository Instructions

Conversation with Michael: German, informal "du".
Code, comments, commits, and persistent docs: English unless Michael asks otherwise.

## Project Memory

Follow the MB3 contract in Fleet global instructions. Start with
`memory-bank/projectBrief.md` and load the canonical roles relevant to the task:
`techContext.md`, `systemPatterns.md`, `tasks.yaml`, `decisions.yaml`, and
`lessons-learned.md`. The explicit migration and retirement of `activeContext.md`
are recorded in `memory-bank/decisions.yaml` (UMA-D012).

The Memory Bank is private and intentionally Git-ignored in this public
repository. Never force-add it. Its pre-adoption recovery copy and adoption
record remain local under `memory-bank/archive/pre-adoption-2026-09-09/` and
`memory-bank/ADOPTION-2026-09-09.md`. Named WIP evidence and actionable work are
linked from the canonical roles; loading Memory does not resume production work.

## System Criticality

This repo operates a production Unraid multi-room audio system. Treat Unraid
("Star Destroyer") as a 24/7 customer production system, not as a lab machine.

The PulseAudio/ALSA/UMC1820 stack is known fragile. It took substantial work to
make it reliable, and kernel or package changes can break audio in non-obvious
ways.

## Unraid SSH Access

Star Destroyer's private SSH key exists only as an attachment in Michael's
KeePass/KeeAgent. No private key file exists on disk. Access is not standing
authorization: use it only after Michael explicitly grants the current
instance/session access. Do not delegate live Unraid access to subagents unless
Michael explicitly authorizes that delegation.

Use Windows OpenSSH with KeeAgent through KeeAgent's native Windows OpenSSH
endpoint. Do not set an agent socket or alter `PATH`:

```powershell
Get-Command ssh
ssh-add -l
ssh -p 22 -o BatchMode=yes -o ForwardAgent=no root@10.0.0.44 '<command>'
```

- `Get-Command ssh` must resolve to
  `C:\Windows\System32\OpenSSH\ssh.exe`.
- Expected Unraid key fingerprint:
  `SHA256:R28Naod49ShXisLLLd3w49g/EQnckqMP6I+jASH7Qp4`.
- If absent from `ssh-add -l`, ask Michael to unlock KeePass and press `Ctrl+M`.
  Do not recreate, export, or search for a private key file.
- KeeAgent access currently expires 1,800 seconds after loading. Closing
  KeePass removes access immediately.
- Configure Git independently of shell `PATH` with
  `git config --global core.sshCommand "C:/Windows/System32/OpenSSH/ssh.exe"`.
- Do not use Git-for-Windows OpenSSH, Plink, Cygwin/MSYS agent sockets,
  `SSH_AUTH_SOCK`, `PATH` changes, or SSH wrappers.
- Do not bypass host-key checking. Star Destroyer ED25519 host fingerprint:
  `SHA256:smPlsHHU7+M3k2GKfXfHd1ktjCJjpQ8gC2kTmMIdnoE`.

## Unraid Change Policy

Read-only diagnostics are allowed.

Before making any write/change on Unraid, including but not limited to package
install/removal, editing `/boot`, restarting services, restarting containers,
loading/unloading kernel modules, USB reset/rebind, or changing mixer state:

1. Identify the exact current state that will be touched.
2. Back up that state first, on Unraid or in the repo as appropriate.
3. Define the exact rollback command or restore procedure before changing it.
4. State the planned change and rollback path to Michael.
5. Proceed only when the change is necessary and the rollback is credible.

Do not "just install" packages, reinstall libraries, restart PulseAudio, restart
Kodi containers, reload `snd-usb-audio`, or rewrite files on `/boot` as a quick
diagnostic shortcut. If a change is needed for diagnosis, treat it as a
production change and follow the policy above.

If a live Unraid state may differ from the repo, preserve the live state before
deploying repo files over it. Never assume the repo is the live source of truth
unless verified.

## Backup Before Unraid Changes

Before any Unraid write/change, create a recovery point appropriate to the
change scope.

For broad or uncertain changes, prefer a native Unraid boot-device backup first:

- WebGUI: Main -> Boot Device -> Boot Device Backup.
- If Unraid Connect flash backup is enabled, verify a recent flash backup
  exists and understand that it is configuration-focused, not a full runtime
  system snapshot.

For audio-stack work, also capture a repo-local, timestamped evidence snapshot
before changing anything. Include at minimum:

- `/boot/config/plugins/pulseaudio/`
- `/boot/config/plugins/user.scripts/scripts/pulseaudio_for_kodi/script`
- `/boot/extra/*alsa*`
- `/boot/extra/*sound*`
- `ls -l` and hashes for the files above
- `/var/log/packages/alsa-lib*`
- `/var/log/packages/sound-*`
- current `uname -r`, `/proc/asound/cards`, `pactl` sink/client/input state,
  ALSA mixer state, relevant `/proc/asound/card*/pcm*/sub*/status` and
  `hw_params`, Docker status/env for `kodi-*`, and recent audio logs

The snapshot must be stored under a repo-owned topic directory, not in bare
temporary storage. Do not rely on Unraid runtime files under `/usr`, `/etc`, or
`/var` as persistent rollback sources unless they were explicitly captured.

## Recovery Discipline

When investigating audio failures:

- Prefer evidence from logs, `/proc`, `/sys`, `pactl`, `aplay`, `amixer`,
  Docker inspect, and Kodi JSON-RPC before changing runtime state.
- Separate observation from intervention in the notes.
- Record every production change made during the session, including timestamps
  and touched paths.
- If a change is later reverted, verify byte-level or command-level equivalence
  where practical.

## Known Production Paths

- PulseAudio config: `/boot/config/plugins/pulseaudio/`
- User Scripts orchestrator:
  `/boot/config/plugins/user.scripts/scripts/pulseaudio_for_kodi/script`
- Persistent packages: `/boot/extra/`
- Kodi appdata: `/mnt/user0/appdata/kodi-*`
