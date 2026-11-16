# Raspberry-Pi-NAS

The progress of building my own ARM based NAS with everything I have downloaded in it.

Before all, I know using a raspberry for a stationary device like this is not the brightest idea, but I had it resting without a usecase for a long time in my room, and building a NAS out of it made the most sense.

Based on a 8gb Pi 5.

> **Build status — 2026-11-16:** Base OS, SSH, primary HDD mount, Docker Engine and Docker Compose are working. Tailscale, Nextcloud and other server applications are not installed yet.

Currently trying to fix the issues that's been occuring with the SATA driver.

## Hardware

| Part | Specification | Current status |
| --- | --- | --- |
| Computer | Raspberry Pi 5, 8 GB RAM | Running |
| System storage | Original 64 GB A1 microSD card | Raspberry Pi OS installed |
| Power | Original Raspberry Pi 27 W USB-C adapter | In use |
| Primary drive | Seagate ST1000LM049, 1 TB, 2.5" SATA | Direct USB 3.0; ext4; mounted |
| Secondary drive | Hitachi HTS545050B9A300, 500 GB, 2.5" SATA | Available; not configured yet |
| SATA bridges | USB 3.0 to 2.5" SATA adapters | UGREEN bridge tested with Seagate |
| USB hub | 4-port USB 3.0 hub with external power input | Correct power adapter still needed; not in regular use |
| Wired networking | Gigabit switch, existing router-to-room Ethernet cable | Planned; Pi currently connected by Wi-Fi |
| Case | Custom enclosure | Planned |

## Network design

**Currently:** Wi-Fi with router DHCP reservation `192.168.1.30`; password-based SSH access working. Local administration from Windows using MobaXterm.

**Planned wired network:**

```text
Router
  |
  | Existing Ethernet cable to room
  |
Gigabit switch
  |-- Raspberry Pi 5 NAS
  |-- Laptop
  +-- Other devices
```

The switch is intended to keep laptop-to-NAS traffic on a local Gigabit link. The Pi has not yet been migrated from Wi-Fi to Ethernet.

## Software foundation

- **Operating system:** Raspberry Pi OS Lite 64-bit, Debian 13 (Trixie), ARM64 / `aarch64`.
- **Remote administration:** SSH, password authentication for now; SSH keys planned.
- **Containers:** Docker Engine installed and confirmed with `hello-world`.
- **Orchestration:** Docker Compose v5.7.1 confirmed.
- **VPN:** Tailscale planned, not yet installed.

Observed during setup: approximately 7.9 GiB system RAM, idle CPU temperature around 44 °C, and `vcgencmd get_throttled` returned `0x0`. These are basic checks, not long-duration load tests.

## Storage

### Seagate 1 TB — primary storage

The previous NTFS filesystem was erased and replaced with **ext4**.

| Property | Value |
| --- | --- |
| Model | ST1000LM049-2GH172 |
| Label | `NAS_STORAGE` |
| UUID | `59d5e1ef-ce51-426c-a4da-c1f8fbc61062` |
| Mount point | `/mnt/storage` |
| Filesystem | ext4 |
| Filesystem size | ~916 GiB |
| Initially available | ~870 GiB (ext4 reserved blocks account for part of the difference) |
| Persistent mount | Verified with `findmnt /mnt/storage` after reboot |

Proposed application layout (directories and services may not all exist yet):

```text
/mnt/storage/
├── nextcloud/
│   ├── data/
│   └── database/
├── jellyfin/
│   └── media/
└── shared/
```

The system OS and Docker engine currently reside on the microSD. The intention is to place persistent application data on the HDD, and potentially move database-heavy workloads to an SSD in the future.

### Hitachi 500 GB — secondary storage

Not yet mounted or prepared. Intended for selected backups, important configuration backups, and potential game-server world backups. It cannot back up a completely full 1 TB primary disk.

## Disk and USB diagnostics

- Seagate was detected through a UGREEN USB–SATA adapter using ASMedia bridge ID `174c:1153`.
- USB negotiated **5000M (USB 3.0)** and initially used the `uas` driver.
- A SMART/ATA pass-through attempt (`smartctl -a -d sat /dev/sda`) caused a timeout/reset sequence; the disk temporarily disappeared, and the USB bridge re-enumerated with `usb-storage`.
- A full power cycle restored the disk and `uas` operation.
- A read-only test of 1 GiB using `dd` completed at **134 MB/s**, with no new USB or I/O errors in the inspected logs.
- The unpowered hub had separate USB enumeration/reset trouble while connecting drives. Its correct external 5 V adapter is still needed before the two-disk setup is used.

**Caution:** A successful 1 GiB read test is not proof of long-term HDD health. SMART pass-through compatibility and sustained read/write reliability still need testing. Avoid relying on reused laptop HDDs as the only copy of important data.

## Planned services

| Service | Purpose | Status |
| --- | --- | --- |
| Tailscale | VPN access to Pi and potentially the rest of the home LAN | Pending |
| Nextcloud | Google Drive-like file access and synchronization | Pending |
| Jellyfin | Media streaming, preferably direct-play | Pending |
| Pi-hole | Network-wide DNS filtering | Optional / pending |
| Terraria | Small multiplayer server; ARM compatibility needs testing | Optional / pending |
| Portainer / monitoring | Container and service administration | Optional / pending |

## Build checklist

### Phase 1 — Base system
- [x] Flash Raspberry Pi OS Lite 64-bit onto the 64 GB microSD
- [x] Configure Wi-Fi, SSH and DHCP reservation
- [x] Update OS
- [x] Detect primary HDD via USB 3.0
- [x] Format Seagate as ext4
- [x] Configure and verify persistent mount at `/mnt/storage`
- [x] Install Docker Engine and verify `hello-world`
- [x] Install Docker Compose and verify version

### Phase 2 — Reliability and network
- [ ] Obtain correct external power adapter for USB hub (voltage, current, plug dimensions, polarity)
- [ ] Test powered hub and both HDDs together
- [ ] Resolve or work around USB–SATA SMART/UAS issue
- [ ] Perform sustained read/write and reconnection tests
- [ ] Prepare Hitachi 500 GB as secondary storage
- [ ] Migrate Pi to wired Gigabit networking
- [ ] Establish and test an independent backup strategy

### Phase 3 — Remote access and cloud storage
- [ ] Install Tailscale and authorize Pi
- [ ] Test remote SSH through Tailscale
- [ ] Optionally advertise home subnet through the Pi
- [ ] Deploy Nextcloud, database and Redis using Docker Compose
- [ ] Ensure services fail safely when the primary HDD is not mounted
- [ ] Configure HTTPS, sync clients and backup/restore tests

### Phase 4 — Additional services
- [ ] Deploy Jellyfin and verify direct-play streaming
- [ ] Evaluate Pi-hole DNS filtering before changing router DNS
- [ ] Evaluate a Terraria dedicated server on ARM64
- [ ] Add monitoring or Portainer as needed
- [ ] Design and print enclosure with proper drive mounting and airflow

## Useful read-only checks

```bash
uname -m
free -h
vcgencmd measure_temp
vcgencmd get_throttled
lsusb -t
lsblk -o NAME,SIZE,FSTYPE,LABEL,MODEL,MOUNTPOINTS
findmnt /mnt/storage
df -h /mnt/storage
sudo dmesg | tail -n 60
docker compose version
```

**Safety:** Disk device names such as `/dev/sda` can change. Verify the exact target before destructive operations. Never commit passwords, SSH private keys, access tokens, or service secrets to this repository.

## Progress log

### 2026-11-16
- Verified Raspberry Pi OS Lite ARM64, Wi-Fi and SSH.
- Prepared primary Seagate HDD as ext4 and confirmed persistent mounting after reboot.
- Investigated UGREEN/ASMedia USB bridge reset during SMART access; recovered after power cycle.
- Completed a 1 GiB sequential read test at 134 MB/s.
- Installed Docker and Docker Compose; `hello-world` ran successfully and Compose reported v5.7.1.
- Next planned steps: supply power to USB hub, install Tailscale, then build Nextcloud stack.
