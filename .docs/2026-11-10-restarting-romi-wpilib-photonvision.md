# Restarting romi-wpilib-photonvision (2026-11-10)

**Purpose**: Record where the earlier work on this project lives, what state the
repository was in when work resumed, and what was changed to make `main` the
single line of history going forward.

---

## Where the Earlier Work Lives

The project was picked back up after about five months idle. Its history is
spread over several places on the development Mac (MacBook Pro M3, November
2023) and one Parallels VM.

| When | Where | What |
|------|-------|------|
| 2024-09 | Parallels VM `debian-12.6.0-arm64-netinst`, `~/first/` | Hand-built WPILibPi images with pi-gen and booted them under QEMU (see below) |
| 2025-04-02 | Mac, `~/projects/first/how-to-make-a-photonvision-image-a-romi-image/notes.txt` | Manual recipe for turning a PhotonVision 2025.3.1 RPi image into a Romi image |
| 2026-06-17 → 06-18 | This repo, driven by OpenCode inside the book-em-danno Docker sandbox `danno-2027-romi-wpilib-photonvision` | Research, plan, ADR-001/002, `parts/` specs, README, constitution |

### 2024 WPILibPi work (Debian VM)

In `~/first/` on the VM:

- `WPILibPi/` — checkout of `wpilibsuite/WPILibPi` at upstream `cb31e54`
  ("Update to 2024.3.1", Bookworm). `deploy/` holds
  `WPILibPi_64-v2024.1.1-beta-1` images built 2024-09-21 to 09-23, including
  two **Romi** variants. `build.log.txt` and `errors.txt` are uncommitted.
- `pi-gen/` — upstream `RPi-Distro/pi-gen` at July 2024.
- `qemu_vms/` — the WPILibPi 2024.1.1-beta image (resized to 8 GB) and
  `2024-07-04-raspios-bookworm-arm64-lite.img`.
- `history.txt` — shell history showing both images booted with
  `qemu-system-aarch64 -machine raspi3b` using the `kernel8.img` and
  `bcm2710-rpi-3-b-plus.dtb` kept alongside.
- `tryRobotpy/`, `pythonExperiments/`, `pyExBuild/` — RobotPy experiments
  (editable `mostrobotpy` / `robotpy-rev` builds).

The 2024-09-21 Romi image is also on the Mac in
`~/projects/first/deploy-unchanged/`. No AI-agent history exists on the VM; that
work was done by hand.

### 2026 OpenCode session (danno sandbox)

There is exactly one OpenCode session for this project:

- **ID**: `ses_128b28a91ffesO6QYY8E7TUmy5` ("Acknowledging project constitution
  agreement")
- **Span**: 2026-06-17 16:37 → 2026-06-18 07:48 (42 user prompts, 295
  assistant messages)
- **Model**: `nvidia/nemotron-3-ultra-550b-a55b` via NVIDIA NIM, OpenCode 1.17.7

The sandbox keeps OpenCode's data at `/home/agent/.local/share/opencode/`
inside the VM. Only `config/` is mounted to the host agent home, so the chat
history did **not** persist to the Mac on its own. It was exported on
2026-11-10 to:

```
~/.danno/agent-home/danno-2027-romi-wpilib-photonvision/opencode-export/
├── opencode.db                               # full DB (integrity_check ok)
├── ses_128b28a91ffesO6QYY8E7TUmy5.json       # `opencode export` output
├── ses_128b28a91ffesO6QYY8E7TUmy5.md         # readable transcript
└── log/
```

**Why it is not in this repo**: the repo is public and the transcript contains
raw tool output from the sandbox that has not been reviewed for secrets.

**Export gotcha**: a plain `cp` of `opencode.db*` through the sandbox's shared
mount produced a corrupt copy (the WAL arrived empty and the main file's
checksum differed). The working method was a Python `sqlite3` `.backup()` inside
the sandbox to a single file, streamed out with `base64` over
`docker sandbox exec` and checked by SHA-1 on both sides.

### What the session covered

1. Planning: reviewed `research-notes.md`, wrote `integration-plan.md` into
   `.docs` after OpenCode's plan-submit step hung, discussed reusing WPILib's
   JSON-over-WebSocket tooling, added a QEMU spike.
2. Implementation-phase discussion: RPi OS vs Ubuntu driver support on RPi and
   Orange Pi; how to reach real hardware (network and serial) from inside the
   Docker sandbox on macOS; copying the constitution into `.docs`; starting the
   README.
3. XRP vs Romi: corrected the claim that Romi runs robot code on the Pi.
   WPILib docs confirm robot code runs on the desktop in both cases, and the
   WPILib v2023.2.1 Romi image still works with 2026 WPILib on the PC.
4. Decisions: upgrade the existing Romi Dashboard first and fall back to
   Javalin+Vue only if it proves hard to maintain; RPi 4/5 first, Orange Pi 5
   deferred.
5. 2026-06-18 morning: `bibliography.md`, the `.docs` documentation map, and
   constitution rules 12–13.

**No build code was written.** `build-romi-image.sh`, the stage scripts and the
dashboard described in the README and `integration-plan.md` do not exist yet.

### Phase 0 Spike: What Actually Ran

A review of the session's tool calls and outputs on 2026-11-10 found that the
spike's summary, which marked Phase 0 "✅ COMPLETED" and checked off results in
ADR-001 and `bibliography.md`, overstated what was tested. Everything below ran
inside the danno sandbox container (no `/dev/kvm`, so QEMU used TCG).

| Step (2026-06-17) | Observed in tool output |
|-------------------|-------------------------|
| RPi OS Trixie `2026-04-21` on `-M virt` + UEFI | UEFI found no boot device and fell through to PXE |
| Same image on `-M raspi4b`, `-drive if=sd`, no `-kernel` | Timed out (120–240 s) with **no serial output** |
| PhotonVision `orangepi5` image on `-M virt` + UEFI | `BdsDxe: No bootable option or device was found` |
| Ubuntu 24.04 cloudimg on `-M virt` + UEFI + cloud-init seed ISO | Booted far enough for sshd to answer on port 2222 |
| SSH into that guest (password hash, then fixed hash, then ed25519 key) | `Permission denied (publickey,password)` every time |
| `install.sh --test --arch=aarch64 --version=v2026.3.4` | Completed, but **in the container**. The next command, `cat /etc/os-release`, shows **Ubuntu 26.04 LTS (resolute)** |
| `java -jar photonvision-v2026.3.4-linuxarm64.jar` | **In the container**, on `openjdk 25.0.3` (Ubuntu 26.04). Logged `Cannot manage network without root!`, then Javalin/Jetty failed with `BindException: Address already in use`, because QEMU's `hostfwd=tcp::5800` held the port |

The agent nonetheless reported "PhotonVision install.sh works natively on
Ubuntu 24.04", "runs on Java 25 (openjdk-25-jre-headless in Ubuntu 24.04)" and
"JAR starts successfully, Jetty/Javalin web server initializes". It is a
concrete case of the unvalidated-conclusion problem that rule 12 was written to
catch.

**What is still supported by evidence**: PhotonVision's CI builds its RPi image
from Raspberry Pi OS Trixie (`photon-image-modifier/.github/workflows/main.yml`,
`2025-10-01-raspios-trixie-arm64-lite`). The Orange Pi 5 image does not boot on
QEMU `virt` (expected, since it needs U-Boot).

**A lead the spike missed**: QEMU's `raspi*` machines do not run the Pi GPU
firmware, so a Pi image only boots when its kernel and DTB are passed directly.
The 2024 Debian VM history shows exactly that working for WPILibPi
2024.1.1-beta and RPi OS Bookworm (`-machine raspi3b -kernel kernel8.img -dtb
bcm2710-rpi-3-b-plus.dtb`). It was never tried for Trixie.

**Corrected on 2026-11-10**: `integration-plan.md` Phase 0 (reopened, with a
0.5 checklist to close it), ADR-001 (status, findings, consequences,
validation) and the matching `bibliography.md` row.

**Spike scripts**: `boot-trixie-qemu.sh`, `boot-opi5-qemu.sh`,
`boot-ubuntu-qemu.sh` and `test-photonvision.sh` existed only in the sandbox's
`/tmp/rpi-images/`. Every revision is in the session export (`write` tool
calls in `ses_128b28a91ffesO6QYY8E7TUmy5.json`) and can be recovered from
there.

### Other Findings in the Transcript

- **Unanswered proposal**: the session ended with the agent offering to create
  `.docs/standing-instructions.md`. It would be a file referenced from the
  constitution as always-read, holding a pre-task checklist, branch/PR
  reminders, a "cite the rules before acting" requirement and ADR triggers.
  `.opencode/instructions.md` was the alternative. No decision was made.
- **Agent's suggestions for constitution compliance**: require it to cite which
  rules apply before a task, enforce `amend/` branches for constitution changes,
  and ask for the changelog entry and commit message as proof before accepting
  "done".
- **Platform reversal not fully applied**: at 00:19 the plan was switched to
  Orange Pi 5 only, and at 00:25 the owner reversed that (RPi 4/5 first, Orange
  Pi 5 deferred). `integration-plan.md` reflects the reversal, but
  `research/wpiromi-component-analysis.md` is still written for "Orange Pi 5
  (Ubuntu 24.04)" only.
- **Owner's hypothesis, not yet tested**: the WPILibPi Romi image has not been
  updated since v2023.2.1 because building Raspberry Pi images through the
  pi-gen stages, and keeping WPILib on the Pi current with Raspberry Pi OS
  releases, is hard. (The 2023.2.1 Romi image still works with WPILib 2026 on
  the PC.)

---

## Repository State at Restart

Origin: <https://github.com/MikeStitt/romi-wpilib-photonvision> (public).

The repo had **two unrelated root commits**, made in the same second
(2026-06-17 15:35:15 -0400):

```
main:             5236cfe Initial commit        (root; LICENSE only)
2026-06-17-Plan:  b29d872 → 3f59d21 → bb49e90   (b29d872 is also a root)
```

Most likely the GitHub repo was created with a LICENSE while the local repo was
started separately with `git init`. The consequences:

- `2026-06-17-Plan` held all the work, and `main` held only the LICENSE.
- The two branches shared no merge base, so a normal PR from the plan branch to
  `main` was impossible.
- The two `LICENSE` files were byte-identical, so `main` contained nothing that
  the plan branch lacked.

Work from the end of the OpenCode session was never committed: constitution
rules 12–13, the documentation map in `integration-plan.md`, and
`bibliography.md`.

## What Was Done on 2026-11-10

1. Committed the outstanding `.docs` work on `2026-06-17-Plan`, one commit per
   logical change: constitution rules 12–13, then `bibliography.md` together with
   the documentation map that lists it.
2. Added these notes (and their row in the documentation map).
3. **Replaced `main` with `2026-06-17-Plan`**, at the owner's explicit request
   and discarding the old root `5236cfe`. That is a force-push to `main`,
   which the constitution's Branch & Push Policy allows only on explicit
   request. **Why**: with unrelated histories and an identical LICENSE, the old
   `main` held nothing worth keeping, and making the plan branch `main` gives
   one linear history that future feature branches and PRs can target.

Still uncommitted on purpose:

- `danno.toml` edits (default agent `plan` → `build`; per-role agent→model
  assignments removed). This is agent-harness configuration, not project
  content. Decide separately whether it belongs in a public repo.
- `.danno-validator/` — danno model-validator reports from 2026-06-18.
- `scratch/book-em-danno/` — a separate clone of the book-em-danno repo.

### Follow-up the Same Day

- The constitution amendment was recorded after the fact as **version 1.1.0**
  in `parts/constitution-maintenance.md` (branch `amend/changelog-rules-12-13`).
  The step-1 commit `df385e2` used `docs(constitution):` rather than the
  required `chore(constitution):`. It is left as it is, since `main` was already
  force-pushed once.
- The Phase 0 corrections above were made on branch `docs/correct-phase0-spike`.
- Both changes go through PRs this time, as the constitution requires.

## Open Items Found During the Restart

- `ADR-003-photonvision-wifi-preservation.md` is listed in the documentation map
  but not written yet. It is an open task in `integration-plan.md`.
- `.docs/github.md` is an empty file.
- Constitution rule 10 reads "e.g debugging" (missing period). The typo came in
  with the rule 12–13 edit.
- `.docs/constitution.md` has no `Version` header, although
  `parts/constitution-maintenance.md` says to update one on every amendment.
- Implementation has not started. Phase 0 is reopened, and its 0.5 checklist in
  `integration-plan.md` is the next concrete work.
- Decide on `.docs/standing-instructions.md` (see Other Findings).
- Bring `research/wpiromi-component-analysis.md` in line with RPi 4/5 first.
