# NomadOS — Full Project Plan

---

## Project Goal

NomadOS is a custom bootable Linux ISO that allows any person to turn their own hardware into a private, portable personal cloud storage and AI assistant system — with no technical knowledge required. The user flashes the ISO to a USB stick, boots any x86_64 PC from it, and a guided setup wizard handles everything else. No subscriptions, no cloud providers, no data leaving the user's hardware.

The core design philosophy is a clean separation of identity and data:

- **The USB stick is the brain** — it holds the OS, the AI models, the assistant's persona and chat history, and all system services. It is fully self-contained and portable.
- **The external drive is the memory** — it holds Nextcloud cloud storage data, user files, photos, and documents. It can be swapped, replaced, or used on multiple machines.
- **The internal HDD is invisible** — it is never mounted, never written to, and never touched by NomadOS under any circumstances.

This means a user can boot NomadOS on any PC they have access to, plug in their external drive, and have their full personal cloud and their personalized AI assistant available immediately. When they return home, they reboot into their normal OS, plug in the external drive, and their files are there. The assistant remains consistent because it lives on the USB, not the drive.

---

## Canonical Specification

| Item | Value |
|---|---|
| Project name | NomadOS |
| ISO filename | `nomados-1.0-amd64.hybrid.iso` |
| Base OS | Debian 13 "Trixie" (stable, amd64) |
| Architecture | x86_64 only |
| Boot support | UEFI and Legacy BIOS (hybrid ISO) |
| Minimum USB size | 32 GB (64 GB recommended, 128 GB for power users) |
| External drive | Any USB drive, any size, formatted on first boot |
| Internal HDD | Suppressed by udev, never mounted |
| AI runtime | Ollama |
| AI interface | Open WebUI |
| Cloud storage | Nextcloud AIO (Docker) |
| Reverse proxy | Caddy |
| Setup wizard | Custom Flask web app |
| ISO build tool | live-build |
| Default model (32 GB) | Llama 3.2 3B (~2 GB) |

---

## Storage Architecture

### USB Stick (Brain)
- Debian 13 minimal OS
- Docker engine and images (Open WebUI, Nextcloud AIO)
- Ollama and AI models
- Open WebUI database (chat history, assistant persona, system prompts)
- Caddy and system service configs
- NomadOS wizard and update scripts
- `/etc/nomados/` — all NomadOS-specific configuration

### External Drive (Memory)
- Partition 1: ext4, labeled `NOMAD-STORAGE` — Nextcloud data directory, mounted at `/mnt/external`
- Partition 2: exFAT, labeled `NOMAD-DROP` — universal drop zone readable by Windows/macOS for easy file transfer

### Internal HDD
- Suppressed at boot via udev rules keyed to device type
- Never appears in `fstab`
- Never automounted under any circumstances

---

## Storage Tiers

| USB Size | Default Model | Additional Models Available | History Pruning |
|---|---|---|---|
| 32 GB | Llama 3.2 3B (~2 GB) | None | Enabled by default (90-day rolling window) |
| 64 GB | Gemma 3 9B (~5 GB) | Llama 3.2 3B as fallback | Optional |
| 128 GB+ | Phi-4 or Llama 3.1 8B | All tiers + second model | Optional |

RAM is a secondary gating factor — the wizard detects available RAM and downgrades the model recommendation if needed (systems under 8 GB are steered toward 3B models regardless of USB size).

---

## The Drive-Swap Workflow

This is a core NomadOS feature and must work reliably.

**Scenario A — Normal use with primary external drive:**
Boot USB + external drive → full cloud storage and AI assistant available on the local network.

**Scenario B — Using a different external drive (travel, work):**
Boot USB + different drive → system detects UUID mismatch in fstab → boots into a lightweight "New Drive" mode offering to set up the new drive or re-link an existing NOMAD-STORAGE drive.

**Scenario C — Returning home, booting normal OS:**
Reboot into Windows/macOS → plug in external drive → NOMAD-DROP partition (exFAT) is universally readable for file access. NOMAD-STORAGE partition (ext4) requires a driver (Paragon Linux File Systems for Windows) for full access. User documentation covers both options clearly.

**Scenario D — Files added to external drive from another OS:**
On next NomadOS boot, a systemd service automatically runs `occ files:scan --all` to re-index any new files into Nextcloud. Files placed in NOMAD-DROP are moved into Nextcloud automatically and the drop zone is cleared.

---

## Phase 1: Base ISO Construction

**Goal:** Produce a minimal, bootable NomadOS ISO using live-build with all core services installed but not yet configured. No user data, no wizard interaction, just a proven build pipeline.

**Tooling:** live-build on a Debian 13 build machine or Docker container. The entire build must be reproducible from a single `./build.sh` command.

**Steps:**

1. Set up a Debian 13 build environment (native machine or Docker container running Debian Trixie).
2. Initialize live-build with the following key settings:
   - `--distribution trixie`
   - `--architecture amd64`
   - `--binary-images iso-hybrid` (enables both UEFI and legacy BIOS boot)
   - `--bootloaders grub-efi,syslinux` (covers both boot paths)
   - Minimal package set — no desktop environment, no unnecessary services
3. Write package lists for live-build hooks:
   - Base: `openssh-server` (disabled by default), `curl`, `wget`, `ufw`, `logrotate`, `udisks2`, `parted`, `util-linux`
   - Docker: installed via Docker's official Debian repo (not Debian's package), added as a live-build hook
   - Ollama: installed via official install script as a live-build hook
   - Caddy: installed via official Caddy Debian repo
   - Python3, pip, Flask: for the setup wizard
4. Write udev rules (`/etc/udev/rules.d/99-nomados-internal.rules`) to suppress internal HDDs. Rules should target `ID_BUS=ata` and `ID_BUS=nvme` devices that are not the boot device, blocking automount at the kernel level.
5. Write the NomadOS boot branding — replace Debian boot splash with NomadOS name and a clean minimal boot screen.
6. Produce the ISO and verify it boots correctly in both UEFI (test in QEMU with OVMF) and legacy BIOS mode (test in QEMU without OVMF).
7. Verify on real hardware that the internal HDD is completely invisible after boot.

**Deliverable:** `nomados-1.0-amd64.hybrid.iso` that boots, shows a terminal prompt, has Docker/Ollama/Caddy installed, and does not mount the internal HDD.

---

## Phase 2: Setup Wizard

**Goal:** A web-based first-boot setup wizard that a non-technical user can complete in under 10 minutes with no terminal interaction. This is the most important UX component of the entire project.

**Trigger mechanism:** A systemd service (`nomados-wizard.service`) checks on boot for the absence of `/etc/nomados/.setup_complete`. If the file does not exist, it starts the Flask wizard on port 80. The terminal displays one line: `NomadOS setup is ready. Open a browser and go to http://[IP-ADDRESS]`. Once setup completes, the service writes the flag file and disables itself permanently (`systemctl disable nomados-wizard`).

**Wizard Steps:**

**Step 1 — Welcome & Hardware Report**
- Display NomadOS name and version
- Show detected hardware: RAM, CPU cores, USB size, whether a second USB/external drive is detected
- Flag any concerns with plain-language warnings (e.g., "Only 4 GB RAM detected — we recommend a compact AI model")
- Single "Begin Setup" button

**Step 2 — External Drive Selection**
- List all block devices excluding the boot USB and suppressed internal HDD
- Show each drive's name, size, and current filesystem
- User selects their external drive
- Warn clearly: "This drive will be erased and formatted."
- Require the user to type `FORMAT MY DRIVE` to confirm
- On confirm: use `parted` to create two partitions — ext4 (`NOMAD-STORAGE`, 90% of drive) and exFAT (`NOMAD-DROP`, remaining 10%)
- Mount `NOMAD-STORAGE` at `/mnt/external` and write to `/etc/fstab` by UUID
- Show real-time progress

**Step 3 — Cloud Storage Setup**
- Ask for Nextcloud admin username and password (with confirmation and strength indicator)
- Launch Nextcloud AIO Docker container pointed at `/mnt/external/nextcloud-data`
- Run initial Nextcloud configuration non-interactively via `occ`
- Show progress indicator — this step takes 1–3 minutes
- On completion, confirm Nextcloud is accessible

**Step 4 — AI Assistant Setup**
- Ask user to name their assistant (text input, displayed throughout the UI)
- Show model selection cards, filtered by detected RAM and USB space:
  - Each card shows: model name, size on disk, RAM required, one-line capability description
  - Default selection is pre-highlighted based on hardware
- On model selection: run `ollama pull [model]` with a real-time download progress bar
- This is the longest step — show download speed and estimated time remaining

**Step 5 — Review & Finish**
- Summary of all choices made
- Configure Caddy to reverse proxy both services under HTTPS with a self-signed cert
- Start all Docker services
- Write `/etc/nomados/.setup_complete`
- Disable wizard service
- Display final screen with:
  - Nextcloud URL
  - AI Assistant URL
  - Brief "What's next" instructions (install Nextcloud sync client, customize your assistant)

**Deliverable:** Complete wizard flow that a non-technical user can run successfully on first boot, resulting in a fully operational NomadOS instance.

---

## Phase 3: Core Services & Boot Architecture

**Goal:** All services start correctly on every subsequent boot with no user interaction. The system is reliable and self-healing where possible.

**Boot sequence:**
1. Debian 13 boots from USB
2. udev suppresses internal HDD
3. `fstab` mounts external drive by UUID
4. If UUID not found (different drive plugged in) → trigger New Drive mode (see Phase 4)
5. Docker daemon starts
6. Nextcloud AIO and Open WebUI containers start (Docker restart policy: `always`)
7. Ollama starts as a systemd service (`ollama.service`)
8. Caddy starts and serves both interfaces
9. `nomados-filescan.service` runs `occ files:scan --all` in the background
10. `nomados-dropzone.service` moves any files from NOMAD-DROP into Nextcloud and clears the drop zone

**Service definitions to write:**
- `ollama.service` — starts Ollama, binds to localhost:11434 only
- `nomados-filescan.service` — oneshot, runs after Docker is ready, scans Nextcloud
- `nomados-dropzone.service` — oneshot, processes NOMAD-DROP contents into Nextcloud
- `nomados-wizard.service` — first-boot only, disabled after setup
- `nomados-newdrive.service` — triggered when fstab UUID is not found

**Port layout (all via Caddy):**
- `443` → Caddy (HTTPS)
  - `/` → Nextcloud
  - `/ai` → Open WebUI
- `80` → redirect to `443`
- `11434` → Ollama API (localhost only, never exposed)

**Security baseline:**
- UFW enabled, only ports 80 and 443 open externally
- Ollama bound to `127.0.0.1` only
- SSH disabled by default
- Caddy handles HTTPS with self-signed cert for LAN use
- No root password set — sudo only via a created `nomad` user during wizard

**Deliverable:** A NomadOS instance that boots reliably into a fully operational state with all services running, from cold boot in under 90 seconds.

---

## Phase 4: Drive-Swap Logic

**Goal:** The drive-swap workflow is seamless and handles all realistic scenarios without requiring the user to understand filesystems, UUIDs, or Linux.

**New Drive Detection:**
A boot-time script checks whether the UUID stored in `/etc/fstab` for `/mnt/external` is present in the system. If not, it sets a flag that triggers `nomados-newdrive.service`, which starts a lightweight web UI on port 80 (similar in style to the setup wizard) offering two options:

1. **Set up a new drive** — runs the external drive setup portion of the wizard (Steps 2–3) for the new drive
2. **This is my existing NomadOS drive** — attempts to locate a NOMAD-STORAGE labeled partition, re-links it in fstab by its UUID, and reboots cleanly

**Drop Zone Processing (`nomados-dropzone.service`):**
On every boot, if NOMAD-DROP contains files:
- Move files into `/mnt/external/nextcloud-data/[admin-user]/files/Drop Zone/`
- Run `occ files:scan` to index them into Nextcloud
- Clear NOMAD-DROP
- Log all moved files to `/var/log/nomados/dropzone.log`

Users on Windows/macOS simply drag files into the NOMAD-DROP drive letter and they appear in Nextcloud on next NomadOS boot. This is the primary cross-OS file transfer mechanism and should be documented prominently.

**Deliverable:** Drive-swap scenarios A through D all work correctly and gracefully. No scenario leaves the system in an unrecoverable state without user guidance.

---

## Phase 5: Storage Management & Pruning

**Goal:** NomadOS stays healthy on a 32 GB USB indefinitely without requiring manual maintenance.

**Automatic maintenance (systemd timers):**

- **Weekly — Chat history pruning:** Delete Open WebUI conversation records older than 90 days. Users on 64 GB+ USB can disable this in the dashboard settings. Log pruning activity.
- **Weekly — Log rotation:** Aggressive logrotate config — weekly rotation, 2-week retention, immediate compression. Applied to all NomadOS-specific logs.
- **Weekly — Docker cleanup:** Run `docker image prune -f` and `docker container prune -f` to reclaim space from stale layers.
- **Monthly — Service updates:** Pull updated Docker images for Nextcloud AIO and Open WebUI. Restart containers. Log changes.

**What never goes on the USB:**
- Swap file (write amplification risk — if needed, place on external drive)
- Nextcloud data files (always on external drive)
- User-uploaded documents to Open WebUI that exceed a size threshold (warn user and suggest saving to Nextcloud instead)

**Storage health monitoring:**
The NomadOS dashboard (accessible at the root of the AI assistant URL) shows:
- USB used/free space with a visual indicator
- External drive used/free space
- Last backup date (future feature)
- Running services status

**Deliverable:** A 32 GB USB running NomadOS for 12+ months without requiring manual cleanup or running out of space under normal usage patterns.

---

## Phase 6: Update System

**Goal:** Users can keep NomadOS current without reflashing and without risking their configuration or data.

**Update layers:**

1. **OS security updates** — Weekly systemd timer: `apt update && apt upgrade -y`. Non-interactive, logged to `/var/log/nomados/updates.log`.
2. **Docker image updates** — Monthly timer: pull latest images, restart containers gracefully.
3. **Ollama updates** — Monthly timer: `ollama upgrade` (or equivalent for the installed version).
4. **Model updates** — User-triggered from dashboard: "Check for model updates" button runs `ollama pull [current-model]` to fetch latest version.
5. **NomadOS system updates** — A `nomados-update` script handles migration of system-level configs when new NomadOS versions are released. Updates are announced on the dashboard with a one-click apply button that pulls and runs the update script.

**Configuration safety contract:**
No update script ever modifies:
- `/etc/nomados/` — NomadOS config
- Open WebUI's database (chat history, personas)
- `/etc/fstab` entries for the external drive
- Any Nextcloud user data

**Major version upgrades** (e.g., 1.x → 2.0) that require reflashing are clearly communicated with a migration guide for transferring Open WebUI config to the new USB.

**Deliverable:** Update system that keeps all components current with zero risk to user data or assistant configuration.

---

## Phase 7: Distribution & Documentation

**Goal:** A non-technical user can find NomadOS, flash it, and complete setup successfully with no outside help.

**Distribution:**
- GitHub repository for source (build configs, wizard code, service files)
- GitHub Releases for ISO downloads with SHA256 checksums
- Simple static project website with download button, hardware requirements, and flashing instructions
- Flashing guide covers: Balena Etcher (recommended, cross-platform), Rufus (Windows alternative)

**Minimum hardware documentation:**

| Requirement | Minimum | Recommended |
|---|---|---|
| RAM | 4 GB | 8 GB+ |
| CPU | Any x86_64, 2 cores | 4+ cores |
| USB stick (brain) | 32 GB USB 3.0 | 64–128 GB, fast drive |
| External drive (memory) | Any USB drive | USB 3.0, 500 GB+ |
| Network | Wired or WiFi | Wired for stability |

**Recommended USB drives to document by name:**
- Samsung Fit Plus 32/64 GB (fast, physically small, reliable)
- SanDisk Ultra Fit 32/64 GB (budget-friendly, USB 3.1)
- External SSD in USB enclosure (premium option, best longevity)

**User documentation structure:**
1. What is NomadOS (one paragraph, non-technical)
2. What you need (hardware list)
3. How to flash the ISO (step by step with screenshots)
4. First boot and setup wizard walkthrough
5. Using your cloud storage (Nextcloud basics)
6. Using your AI assistant (Open WebUI basics, customization)
7. The drive-swap workflow (how to use on the go)
8. Accessing your files from Windows/macOS
9. Keeping NomadOS updated
10. Troubleshooting

**Deliverable:** A complete, downloadable ISO with documentation sufficient for a non-technical user to get fully operational independently.

---

## Repository Structure

```
nomados/
├── build/
│   ├── build.sh                  # Single command to produce ISO
│   ├── live-build-config/        # All live-build configuration
│   │   ├── package-lists/
│   │   ├── hooks/                # Install Docker, Ollama, Caddy etc.
│   │   └── includes.chroot/      # Files copied into the ISO filesystem
├── wizard/
│   ├── app.py                    # Flask setup wizard
│   ├── templates/                # HTML templates for each wizard step
│   ├── static/                   # CSS, JS, NomadOS branding
│   └── scripts/                  # Shell scripts called by wizard (partition, install)
├── services/
│   ├── nomados-wizard.service
│   ├── nomados-filescan.service
│   ├── nomados-dropzone.service
│   ├── nomados-newdrive.service
│   └── nomados-update.timer
├── caddy/
│   └── Caddyfile.template        # Populated during wizard
├── udev/
│   └── 99-nomados-internal.rules
├── scripts/
│   ├── nomados-update            # System update script
│   └── nomados-prune             # Storage pruning script
├── docs/
│   └── (user documentation source)
└── release/
    └── (built ISOs, checksums)
```

---

## Development Sequence

| Phase | Task | Dependencies |
|---|---|---|
| 1 | live-build pipeline → bootable minimal ISO | Build environment |
| 1 | udev internal HDD suppression | Phase 1 ISO |
| 1 | Verify UEFI + legacy BIOS boot on real hardware | Phase 1 ISO |
| 2 | Setup wizard (Flask app, all 5 steps) | Phase 1 ISO |
| 2 | Wizard systemd integration + first-boot trigger | Wizard app |
| 3 | All core systemd services | Phase 2 complete |
| 3 | Caddy reverse proxy config | Phase 3 services |
| 3 | Full boot-to-operational test | All Phase 3 |
| 4 | Drive-swap detection and new drive mode | Phase 3 |
| 4 | Drop zone processing service | Phase 3 |
| 5 | Storage pruning timers | Phase 3 |
| 5 | Dashboard storage health display | Phase 4 |
| 6 | Update scripts and timers | Phase 5 |
| 7 | Documentation | Phase 6 |
| 7 | Public beta — broad real hardware testing | All phases |
| 7 | 1.0 release | Beta feedback resolved |
