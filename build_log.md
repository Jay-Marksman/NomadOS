# NomadOS Phase 1 Build Log

Date: February 28, 2026  
Repo: `/home/jay/nomados`  
Scope: Phase 1 (Debian 13/trixie live-build ISO pipeline)

## 1) Phase 1 Requirements (from `AGENT.md`)

- Debian 13 (`trixie`)
- `amd64`
- `iso-hybrid`
- dual bootloader path: `grub-efi + syslinux`
- minimal live system
- reproducible build via `build/build.sh`

## 2) Repository Review (files/folders checked)

Top-level:
- `AGENT.md`
- `build/`
- `build_log.md`
- `release/`
- `docs/`
- `scripts/`
- `services/`
- `udev/`
- `wizard/`
- `caddy/`

Build pipeline and config reviewed:
- `build/build.sh`
- `build/live-build-config/auto/config`
- `build/live-build-config/config/binary`
- `build/live-build-config/config/chroot`
- `build/live-build-config/config/common`
- `build/live-build-config/config/archives/*.list.*`
- `build/live-build-config/config/package-lists/nomados-base.list.chroot`
- `build/live-build-config/config/hooks/*.chroot`
- `build/live-build-config/config/hooks/live/*.chroot` (duplicate tree)
- generated trees under `build/live-build-config/{binary,chroot,cache,.build,...}`

## 3) What We Did To Get Here

### A) Build script and artifact handling (`build/build.sh`)
- Added robust cleanup of stale live-build stage artifacts before every build.
- Fixed artifact copy logic so latest generated ISO is copied to:
  - `release/nomados-1.0-amd64.hybrid.iso`
- Fixed stale ISO selection bug (old `binary.hybrid.iso` no longer reused accidentally).
- Added live-build path compatibility logic for Debian packaging split.
- Added shim support at:
  - `build/live-build-config/.live-build-shim`

### B) Host/toolchain compatibility fixes (`auto/config`)
- Added fail-fast when host `lb config` does not support `--bootloaders`.
- Kept fallback for unsupported `--hostname` via `config/hostname`.
- Removed old `config/bootloaders` file conflict behavior.
- Switched deprecated installer setting to:
  - `--debian-installer none`
- Forced overlay union in generated config when CLI flag is unsupported:
  - `LB_UNION_FILESYSTEM="overlay"`

### C) Bootloader and syslinux reliability hooks
- Added/maintained syslinux compatibility hooks:
  - `005-syslinux-links.chroot`
  - `006-install-syslinux.chroot`
- Ensured required legacy files are available for binary syslinux stage.
- Added `rsvg` compatibility wrapper for live-build expectations.

### D) Service installation hooks
- Docker hook:
  - `010-docker.chroot` (official Docker repo)
- Caddy hook:
  - `030-caddy.chroot` (official Caddy repo)
- Ollama hook:
  - `020-ollama.chroot` (now pinned + checksum + local cache install path; see section 5)

### E) Login test hook
- Added `040-phase1-login.chroot` to create test users/passwords for Phase 1 login validation:
  - `user:nomados`
  - `nomad:nomados`
  - `root:nomados`

## 4) Critical Issues Encountered and Resolved

Resolved during this phase:
- unsupported live-build flags on Ubuntu/Mint host
- broken mirror/security suite paths
- sysvinit/systemd package conflicts
- missing syslinux assets (`isolinux.bin`, `vesamenu.c32`, `ldlinux.c32`)
- missing hook prerequisites (`curl`, `gpg`, `zstd`)
- old local `LIVE_BUILD` override breaking grub stage
- missing GRUB data path due to split live-build layout
- stale ISO file being copied instead of fresh output
- boot failure due to `aufs` (switched to overlay)

## 5) New Change: Ollama Cached + Pinned Install (Option 1)

Implemented to avoid re-downloading Ollama payload every `./build.sh clean`:

### In `build/build.sh`
- Added pinned version variable:
  - `OLLAMA_VERSION=0.17.4` (override via env if needed)
- Added persistent host cache path:
  - `build/cache/ollama/`
- Downloads once from pinned release URL:
  - `ollama-linux-amd64.tgz`
  - release `sha256sum.txt`
- Extracts checksum line for the archive and verifies with `sha256sum -c`.
- Injects verified payload + checksum into chroot include path:
  - `build/live-build-config/config/includes.chroot/usr/local/src/ollama/`

### In `config/hooks/020-ollama.chroot`
- No longer uses `curl | sh`.
- Uses cached archive from `/usr/local/src/ollama`.
- Verifies checksum before install.
- Extracts and installs binary to `/usr/local/bin/ollama`.
- Installs bundled systemd service if present.
- Creates `ollama` system user if missing.

Result:
- clean rebuilds should reuse cached payload and avoid repeated network download.
- integrity improved by explicit checksum verification.
- reproducibility improved by pinned version.

## 6) Current Build State

Recent build completes and writes:
- `release/nomados-1.0-amd64.hybrid.iso`

ISO structure confirmed to include:
- BIOS path (`/isolinux/isolinux.bin`)
- UEFI path (`/boot/grub/efi.img` + `/EFI/boot/bootx64.efi`)
- `ldlinux.c32`
- kernel/initrd under `/live/`

## 7) Real Hardware USB Results (Final)

After writing `nomados-1.0-amd64.hybrid.iso` to USB:

UEFI observed messages included:
- firmware/microcode warnings
- missing bluetooth/realtek firmware notices
- previously: dropped to shell with `aufs not available` (now addressed by overlay switch)

CSM observed:
- reaches login prompt but authentication failure

Final verified outcome:
- boots in **UEFI**
- boots in **CSM/legacy BIOS**
- authenticates and reaches shell prompt:
  - `root@debian:~#`
- service binary checks pass in live session:
  - `Docker 29.2.1`
  - `Ollama 0.17.4`
  - `Caddy 2.11.1`

## 8) Login Failure Root Cause and Fix

Root cause:
- active build path executed hooks in `config/hooks/live/*.chroot`, while newer login hooks initially existed only in `config/hooks/*.chroot`.
- resulting squashfs had only locked `root` account and no usable test user credentials.

Fix applied:
- added mirrored login hooks in `config/hooks/live/`:
  - `040-phase1-login.chroot`
  - `050-phase1-autologin.chroot`
- validated final squashfs contains `root`, `user`, and `nomad` accounts with password hashes.

## 9) Suggested Next Steps

1. Optional hardware polish
- Add firmware packages for test hardware if desired:
- `firmware-realtek`
- `firmware-brcm80211`
- optionally `firmware-linux` for broader support

2. Config hygiene
- Keep only `config/hooks/*.chroot` and delete/stop maintaining `config/hooks/live/*` duplicates.

3. Add post-build validation script
- check ISO for BIOS+UEFI entries, `ldlinux.c32`, kernel/initrd, and optionally inspect default boot parameters.

## 10) Phase 1 Status

Status: **Accepted / Complete**

Done:
- reproducible build pipeline in Debian 13 container
- dual boot artifacts present in ISO
- overlay union fix applied
- Ollama cached/pinned/checksummed install path implemented
- boots in UEFI and CSM on real hardware
- successful shell login on real hardware
- runtime validation passed for Docker, Ollama, and Caddy versions

Blocking:
- none for Phase 1 acceptance criteria

## 11) Update: Ollama Cache 404 Fix (Feb 28, 2026)

Issue encountered:
- `./build.sh clean` failed during prefetch with:
  - `curl: (22) The requested URL returned error: 404`
- Cause: hardcoded GitHub archive filename did not match available release asset for the pinned version.

Fix implemented:
- `build/build.sh` now performs resilient source selection:
  - tries GitHub release `sha256sum.txt` for pinned `v${OLLAMA_VERSION}`
  - auto-detects matching `ollama-linux-amd64*` asset name
  - downloads that asset and rewrites checksum entry to local cache filename
  - if GitHub asset lookup fails, falls back to `https://ollama.com/download/ollama-linux-amd64.tar.zst`
  - fallback records `SOURCE` metadata and uses a local integrity hash for repeat builds
- standardized cache filenames:
  - `build/cache/ollama/ollama-linux-amd64.archive`
  - `build/cache/ollama/ollama-linux-amd64.archive.sha256`

Hook update (`config/hooks/020-ollama.chroot`):
- switched to consuming standardized archive names from `/usr/local/src/ollama`
- supports tar.zst and tar.gz extraction, with raw-binary fallback
- verifies checksum before install
- validates installed version against `/usr/local/src/ollama/VERSION`

Net result:
- clean rebuilds should no longer fail on hardcoded filename drift
- one-time download remains cached
- repeated clean builds reuse cache, improving speed and reducing network-failure risk

## 12) Update: Phase 1 Step 5 Branding (Mar 1, 2026)

Implemented branding pipeline using source image:
- `splash.jpg` (repo root)
- guidance from `Branding.md`

### Branding automation added
- New generator script:
  - `build/generate_boot_branding.py`
- Script generates:
  - `build/branding/nomados-grub-1920x1080.png`
  - `build/branding/nomados-syslinux-640x480.png`
  - `build/branding/nomados-logo-512.png`

### Boot asset wiring
- Script writes branded splash assets into live-build includes:
  - `build/live-build-config/config/includes.binary/boot/grub/splash.png`
  - `build/live-build-config/config/includes.binary/isolinux/splash.png`
- `build/build.sh` now runs branding generation automatically before `lb build`.

### Result
- GRUB (UEFI) and ISOLINUX/SYSLINUX (legacy BIOS) now use generated NomadOS branding derived from `splash.jpg`.
- Branding step is now reproducible as part of standard build flow.

## 13) Phase 2 Kickoff (Mar 1, 2026)

Per `AGENT.md` + `style.md`, Phase 2 setup wizard implementation has started.

### Markdown review completed before implementation
- `AGENT.md` (Phase 2 functional requirements and flow)
- `style.md` (UI palette, typography, spacing, component style)
- `Branding.md` (cross-stage visual consistency reference)
- `build_log.md` (current baseline and known caveats)

### Implemented Phase 2 wizard scaffold

#### Backend
- Added Flask app:
  - `wizard/app.py`
- Added system operations module:
  - `wizard/scripts/system_ops.py`
  - `wizard/scripts/__init__.py`

Implemented routes and flow:
- `/` entry + setup-complete check (`/etc/nomados/.setup_complete`)
- Step 1: hardware report (RAM, CPU, external drive count)
- Step 2: external drive selection + destructive confirmation phrase
- Step 3: Nextcloud admin input + bootstrap command path
- Step 4: assistant name + model recommendation/pull path
- Step 5: review + finalize setup
- completion writes `/etc/nomados/setup.json` and setup-complete flag

Implemented backend operations:
- hardware and block-device discovery (`lsblk`, `findmnt`)
- external-drive partition/format/mount flow
- `/etc/fstab` UUID write for `/mnt/external`
- Nextcloud container bootstrap command
- Ollama model pull action
- setup completion and wizard service disable action

Safety behavior:
- supports `NOMADOS_WIZARD_DRY_RUN=1` for non-destructive test execution.

#### Frontend (style-aligned)
- Added templates:
  - `wizard/templates/base.html`
  - `wizard/templates/step1.html`
  - `wizard/templates/step2.html`
  - `wizard/templates/step3.html`
  - `wizard/templates/step4.html`
  - `wizard/templates/step5.html`
  - `wizard/templates/done.html`
- Added stylesheet with `style.md` color tokens and typography stack:
  - `wizard/static/style.css`

#### Service wiring
- Implemented wizard service unit:
  - `services/nomados-wizard.service`
- Service behavior:
  - starts only when `/etc/nomados/.setup_complete` is absent
  - runs Flask app on boot

#### Build integration for ISO
- `build/build.sh` now syncs wizard assets into chroot includes before build:
  - `config/includes.chroot/opt/nomados/wizard`
  - `config/includes.chroot/etc/systemd/system/nomados-wizard.service`
- Added enable hooks so wizard service is enabled in live image:
  - `config/hooks/070-enable-wizard.chroot`
  - `config/hooks/live/070-enable-wizard.chroot`

### Notes / current limitations
- This is a functional scaffold for Phase 2, not full production hardening yet.
- Nextcloud/Open WebUI deep post-install automation (`occ` full non-interactive provisioning, complete Caddy final wiring, progress streaming, rich validation UX) is partially stubbed and should be completed in subsequent Phase 2 iterations.
- Duplicate hook tree (`config/hooks` and `config/hooks/live`) still exists; currently mirrored intentionally for compatibility with observed live-build behavior.

## 14) Phase 2 Decisions Applied (Mar 1, 2026)

User-confirmed implementation choices:
- destructive disk actions enabled by default
- Nextcloud AIO defaults in initial Phase 2 flow
- IP-based URLs first (instead of local DNS hostname)

Changes applied:
- `services/nomados-wizard.service` already defaults to `NOMADOS_WIZARD_DRY_RUN=0` (destructive mode enabled)
- backend URL output now resolves to host IP:
  - `wizard/scripts/system_ops.py` now derives primary IPv4 via `hostname -I`
  - completion URLs are emitted as `https://<ip>/` and `https://<ip>/ai`
  - Step 3 Nextcloud URL output also uses `https://<ip>/`

## 15) Phase 2 Runtime Validation Updates (Mar 1, 2026)

Recent live-session findings and fixes before next rebuild:

1. Wizard service crash loop (port 80 conflict)
- Symptom:
  - `Address already in use` / `Port 80 is in use by another program`
- Cause:
  - `caddy` occupying port 80 while wizard also binds to 80.
- Fix applied in service unit (`services/nomados-wizard.service`):
  - `Conflicts=caddy.service`
  - `Before=caddy.service`
  - `ExecStartPre=/bin/systemctl stop caddy.service`

2. Step 2 formatting failure
- Symptom:
  - `FileNotFoundError: mkfs.exfat`
- Cause:
  - exFAT tools missing in live image.
- Fixes:
  - Added `exfatprogs` to `build/live-build-config/config/package-lists/nomados-base.list.chroot`
  - Added explicit wizard precheck with actionable error in `wizard/scripts/system_ops.py`

3. Step 4 model pull failure
- Symptom:
  - `ollama pull` panic: `$HOME is not defined`
- Cause:
  - service/process environment missing required HOME/PATH.
- Fixes:
  - Service env now sets `HOME=/root` and full PATH in `services/nomados-wizard.service`
  - command runner `_run()` now sets fallback HOME/PATH in `wizard/scripts/system_ops.py`
  - Step 4 now starts ollama service before pulling model

4. Completion URL bug fixed
- Bug found in `wizard/scripts/system_ops.py`:
  - `host_ip` assignment was after return in `complete_setup()`.
- Fix:
  - moved `host_ip = _primary_ip()` before return.

5. Login status checkpoint
- User confirmed boot/login works with `root / nomados`.

Notes for next build/test cycle:
- Rebuild required so all Phase 2 fixes land in ISO.
- Confirm CSM behavior again after rebuild (prior report: UEFI boots; CSM blinking cursor).
- Re-test wizard flow end-to-end: Step 1→5, then verify `/etc/nomados/.setup_complete` and service disable behavior.

## 16) Phase 2 Validation Progress (Mar 1, 2026, later)

Latest hardware/session outcomes:
- UEFI boot: pass
- CSM boot: pass
- login: pass with `root / nomados`

Wizard runtime observations:
1. Wizard did not start automatically in this session; manual start used:
- `systemctl start nomados-wizard`
- Wizard reachable at `http://10.0.0.75/step/1`

2. Wizard reached Step 5 and displayed "Setup Complete".
- Displayed URLs at that time did not load as expected when using `/` and `/ai` paths.

3. Step 4 model pull issue was reproduced and handled:
- Error: `ollama pull ... panic: $HOME is not defined`
- Service and command environment fixes were already applied in section 15.

4. Additional runtime gap found:
- `systemctl start ollama` failed with:
  - `Unit ollama.service not found`
- Cause:
  - Ollama binary existed, but no systemd unit was included in ISO.

Fixes now implemented in source:
- Added `services/ollama.service` (binds Ollama to `127.0.0.1:11434`, runs as `ollama` user).
- Updated `build/build.sh` service sync to include all service units from repo:
  - `cp -f services/*.service .../config/includes.chroot/etc/systemd/system/`
- Updated wizard completion URLs to IP-first direct ports for current Phase 2 scope:
  - Nextcloud: `http://<ip>:8080/`
  - Assistant (Open WebUI): `http://<ip>:3000/`
- Updated completion flow to attempt Open WebUI container start for Phase 2 testing.

Current status before next build:
- Source-side fixes are in place.
- New ISO build required to validate these fixes in live environment.

Next immediate action:
- Run `./build.sh clean`
- Re-test full flow: boot (UEFI+CSM), login, wizard auto-start behavior, Step 1→5, and final URLs on ports 8080/3000.

## 17) Pre-Build Review Before Next Clean Build (Mar 7, 2026)

Performed source review and sanity checks before next `./build.sh clean`.

### Review scope
- `build/build.sh`
- `build/live-build-config/auto/config`
- `build/live-build-config/config/package-lists/nomados-base.list.chroot`
- `services/nomados-wizard.service`
- `services/ollama.service`
- `wizard/app.py`
- `wizard/scripts/system_ops.py`
- hook inventory under `build/live-build-config/config/hooks` and `.../hooks/live`

### Findings
1. No syntax/parsing blockers
- `python3 -m py_compile` passed for wizard backend modules.
- `bash -n` passed for `build/build.sh`.

2. Step 5 ENOSPC root cause identified and addressed in source
- Earlier internal server error was due to no space left on USB/root FS while completing setup.
- Current logic now supports temporary model staging on external storage when USB free space is insufficient, with required migration back to USB at finalize stage.

3. Architecture alignment retained
- Final model location is enforced on USB (`/var/lib/ollama/models`) via migration + systemd override.
- This preserves the project rule that the assistant must not depend on the same external drive after reboot.

4. Runtime service coverage improved
- Added `services/ollama.service` and build-time copy of all service unit files into chroot includes.

### Current expected behavior after rebuild
- Boot/login works (UEFI + CSM, `root/nomados` confirmed previously).
- Wizard runs on port 80 (with caddy conflict handling in service unit).
- Step 2 can format exFAT (`exfatprogs` included).
- Step 4 can pull model without `$HOME` panic.
- Step 5 should return direct IP-first URLs:
  - Nextcloud: `http://<ip>:8080/`
  - Assistant: `http://<ip>:3000/`

Ready for next full build validation cycle.

## 18) Source Review + Error Check Before Next Build (Mar 7, 2026)

Performed a repo-wide source sanity pass before the next `./build.sh clean`.

Checks run:
- `python3 -m py_compile wizard/app.py wizard/scripts/system_ops.py build/generate_boot_branding.py`
- `bash -n` on:
  - `build/build.sh`
  - `build/live-build-config/auto/config`
  - all `build/live-build-config/config/hooks/*.chroot`
  - all `build/live-build-config/config/hooks/live/*.chroot`
  - `scripts/nomados-prune`
  - `scripts/nomados-update`

Results:
- No Python syntax errors.
- No shell syntax errors.
- Hook inventory is present in both paths (`config/hooks` and `config/hooks/live`) for current live-build compatibility.

Fix applied during review:
- Wizard Step 4 radio default selection logic was improved.
- Previous template condition could leave no default model selected.
- Updated files:
  - `wizard/app.py` now computes and passes `default_model`.
  - `wizard/templates/step4.html` now checks radio by `m.id == default_model`.

## 19) Gaps Not Blocking Current Build (Mar 7, 2026)

The following files currently exist as empty placeholders and should be implemented as Phase 3/4 hardening work:
- `udev/99-nomados-internal.rules` (currently 0 bytes)
- `caddy/Caddyfile.template` (currently 0 bytes)
- `scripts/nomados-update` (currently 0 bytes)
- `scripts/nomados-prune` (currently 0 bytes)

Impact:
- Not a blocker for current Phase 2 wizard flow validation.
- But they are required to fully meet AGENT.md service/security/drive-suppression goals in later phases.

## 20) Installer Image Mode Added (Mar 8, 2026)

Implemented a dedicated installer build mode so NomadOS can produce an ISO with Debian Installer entries.

Changes:
- `build/live-build-config/auto/config`
  - added `NOMADOS_IMAGE_MODE` support:
    - `live` (default) -> `--debian-installer none`
    - `installer` -> `--debian-installer cdrom`
- `build/build.sh`
  - added argument parsing for `clean`, `installer`, and `live`
  - exports `NOMADOS_IMAGE_MODE` to `auto/config`
  - output filenames:
    - live: `release/nomados-1.0-amd64.hybrid.iso`
    - installer: `release/nomados-1.0-installer-amd64.hybrid.iso`

Build examples:
- Live ISO (default): `./build.sh clean`
- Installer ISO: `./build.sh installer clean`

## 21) Live-Persistence Direction Reconfirmed (Mar 8, 2026)

Decision:
- Keep installer image generation available for dedicated-host deployments.
- Primary project path remains live boot portability across multiple host devices.

Change applied for live boot:
- Updated kernel boot args in `build/live-build-config/auto/config` to enable persistence discovery by default:
  - `boot=live components persistence persistence-label=persistence noautologin`

Purpose:
- Allow NomadOS live USB to automatically use a persistence partition labeled `persistence`.
- This is required to avoid transient overlay-only behavior that causes Step 5 failures (`No space left on device`) during model/container setup.

## 22) Installer ISO + Persistence Partition Notes (Mar 8, 2026)

### Installer ISO track (kept, but not primary mission)
- Added optional installer build mode while keeping live ISO as default portable path.
- Build controls now support:
  - live (default): `./build.sh clean`
  - installer: `./build.sh installer clean`
- Installer output path:
  - `release/nomados-1.0-installer-amd64.hybrid.iso`
- Use case:
  - dedicated-host deployments where full disk install is preferred.
- Project direction confirmed:
  - core mission remains live USB portability across different host devices.

### Live persistence setup work (for portable mission)
- Updated live kernel args to enable persistence discovery automatically:
  - `boot=live components persistence persistence-label=persistence noautologin`
- Practical finding:
  - persistence partition cannot be created while booted from the same USB (disk in use lock).
  - `sfdisk` correctly blocked repartition with:
    - "This disk is currently in use ... Use the --force flag to overrule all checks."
- Required procedure:
  - boot from another environment (host OS or separate live media)
  - then create `/dev/sdb3` in free space and label it `persistence`
  - add `/ union` in `persistence.conf`

### Partitioning diagnostics observed
- `fdisk -l /dev/sdb` showed usable DOS/MBR layout and free space after ISO partitions:
  - `/dev/sdb1` (ISO)
  - `/dev/sdb2` (EFI)
  - large free space available for `/dev/sdb3`
- `parted` output was inconsistent on hybrid media (reported mac label + block-size warning), so `fdisk/sfdisk` is the reliable path for persistence partitioning.

### Next action queued
- From a non-booted context for `/dev/sdb`, create and configure persistence partition:
  1. `echo ',,83' | sudo sfdisk -N 3 --force /dev/sdb`
  2. `sudo mkfs.ext4 -F -L persistence /dev/sdb3`
  3. mount `/dev/sdb3`, write `persistence.conf` with `/ union`, unmount
- Reboot NomadOS USB and verify persistence is active via:
  - `cat /proc/cmdline`
  - `mount | grep -i persistence`
