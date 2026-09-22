# Linux Mint 21.3 → 22.3 Upgrade Runbook

**Target:** Linux Mint 22.3 Cinnamon “Zena”
**Starting release:** Linux Mint 21.3 Cinnamon “Virginia”
**Companion:** [upgrade assessment](linuxmint-upgrade-assessment-2026-09-22.md)

> [!WARNING]
> Upgrade locally while on AC power and reliable networking. Do not use an SSH-only/fragile session. Do not manually change `jammy` to `noble`, interrupt `mintupgrade`, or run `autoremove` until final verification passes.

## Stop/go checks

Stop before the next phase if you lack an independent backup, a verified Timeshift snapshot, a boot-tested live USB, clean output from `dpkg --audit`, or an Ubuntu 24.04-compatible version of any business-critical third-party software.

## Phase 0 — schedule and recovery media

1. Reserve several uninterrupted hours and connect AC power.
2. Download the Mint 22.3 Cinnamon ISO and create a bootable USB.
3. Verify the ISO checksum against the value published by Linux Mint before writing it to USB. This avoids relying on a previous live image whose `casper-md5check` service failed; that historical failure was from a live session, not the installed OS.
4. Boot the USB once; confirm it sees the NVMe disk, display, keyboard, and Wi-Fi.
5. Confirm that the live session can access the Timeshift snapshot destination. This is the recovery environment if the installed system no longer boots.
6. Disconnect nonessential USB devices.

## Phase 1 — capture inventories and backups

```bash
mkdir -p ~/mint-upgrade-backup

# Full installed-package inventory: package, version, architecture.
dpkg-query -W -f='${binary:Package}\t${Version}\t${Architecture}\n' \
  > ~/mint-upgrade-backup/dpkg-packages.tsv

# Explicit/manual packages and Flatpak applications.
apt-mark showmanual > ~/mint-upgrade-backup/manual-packages.txt
flatpak list --app --columns=application,branch,origin \
  > ~/mint-upgrade-backup/flatpaks.tsv

# Repository definitions and essential mount/boot configuration.
sudo cp -a /etc/apt/sources.list.d ~/mint-upgrade-backup/
sudo cp -a /etc/fstab /etc/default/grub ~/mint-upgrade-backup/
uname -a > ~/mint-upgrade-backup/kernel-before.txt
dpkg --audit > ~/mint-upgrade-backup/dpkg-audit-before.txt
apt-mark showhold > ~/mint-upgrade-backup/apt-holds-before.txt
```

Independently back up projects, Documents, SSH/GPG keys, needed browser data, Docker compose files/volumes/bind mounts, database data, VM definitions, and VM disks. Identify active workloads first:

```bash
docker ps -a
docker volume ls
virsh list --all
systemctl status mongod
```

Use application-specific, credential-aware backup procedures for MongoDB and any encrypted data. Do not retain the only copy on the upgraded NVMe disk.

## Phase 2 — create a rollback point

1. Open **Menu → Timeshift**.
2. Configure a destination with adequate capacity.
3. Create a named pre-upgrade snapshot and verify it appears in Timeshift.
4. Keep the tested live USB available: it can run Timeshift restore or Boot Repair.

## Phase 3 — fully update Mint 21.3

```bash
sudo apt update
sudo apt full-upgrade
sudo reboot
```

After reboot, proceed only when checks are clean:

```bash
cat /etc/os-release
uname -r
dpkg --audit
apt-mark showhold
systemctl --failed
```

The local boot-chain assessment found a healthy UEFI/GRUB configuration, ample EFI space, and multiple existing fallback kernels. It also found historical NVIDIA Xid 31 and PRIME/DRM errors. Before continuing, confirm that the currently updated NVIDIA driver and its DKMS module are healthy:

```bash
nvidia-smi
dkms status
```

Do not use force options, pin changes, or package removals to work around an update problem. If NVIDIA reports an error or DKMS has no installed module for the running kernel, stop and resolve that on Mint 21.3 before attempting the release upgrade.

## Phase 4 — temporarily disable external APT sources

These files are currently compatible with Ubuntu 22.04/Jammy or external channels, not necessarily the Ubuntu 24.04/Noble base used by Mint 22. Move them aside reversibly:

```bash
sudo mkdir -p /etc/apt/sources.list.d/disabled-for-mint-upgrade
sudo mv \
  /etc/apt/sources.list.d/mongodb-org-8.0.list \
  /etc/apt/sources.list.d/percona-tools-release.list \
  /etc/apt/sources.list.d/percona-telemetry-release.list \
  /etc/apt/sources.list.d/percona-psmdb-80-release.list \
  /etc/apt/sources.list.d/percona-pbm-release.list \
  /etc/apt/sources.list.d/brave-browser-release.sources \
  /etc/apt/sources.list.d/google-chrome.sources \
  /etc/apt/sources.list.d/vscode.sources \
  /etc/apt/sources.list.d/docker.list \
  /etc/apt/sources.list.d/nodesource.list \
  /etc/apt/sources.list.d/bazel.list \
  /etc/apt/sources.list.d/repo.fortinet.com.list \
  /etc/apt/sources.list.d/sublime-text.list \
  /etc/apt/sources.list.d/antigravity.list \
  /etc/apt/sources.list.d/mwt-desktop.list \
  /etc/apt/sources.list.d/disabled-for-mint-upgrade/
sudo apt update
```

Do **not** move `/etc/apt/sources.list.d/official-package-repositories.list`. If any listed file is absent, stop, inspect `ls /etc/apt/sources.list.d/`, and move the remainder individually—never use a wildcard.

## Phase 5 — upgrade Mint 21.3 to Mint 22

```bash
sudo apt update
sudo apt install mintupgrade
sudo mintupgrade
```

In Mint’s tool, resolve every reported blocker, leave the external repositories disabled, and keep the terminal open until success. Temporary unreadable fonts are documented during this upgrade; do not interrupt it. Note packages removed by the tool for deliberate later restoration.

When it completes successfully:

```bash
sudo apt remove mintupgrade
sudo reboot
```

## Phase 6 — validate Mint 22

```bash
cat /etc/os-release
uname -r
dpkg --audit
systemctl --failed
nvidia-smi
dkms status
inxi -A
nmcli general status
```

Test login, Cinnamon, Wi-Fi, Bluetooth, sound, suspend/resume, external displays, NVIDIA graphics, VPN, representative Docker workloads, and a QEMU/libvirt guest. Confirm that `dkms status` lists the NVIDIA module as installed for the newly booted kernel. Initially keep the stable 6.8 kernel rather than manually selecting HWE 6.14.

If the graphical session fails but the computer reaches GRUB, select **Advanced options for Linux Mint** and boot the earlier working kernel or its recovery entry. If the computer cannot boot successfully, start the verified Mint live USB and restore the pre-upgrade Timeshift snapshot; use Boot Repair only if the firmware/GRUB boot sequence itself is damaged. Do not re-enable third-party sources to work around a graphics or boot issue.

## Phase 7 — upgrade Mint 22 to Mint 22.3

1. Open **Update Manager** and press **Refresh**.
2. Install available updates to `mintupdate` and `mint-upgrade-info`.
3. Select **Edit → Upgrade to Linux Mint 22.3 Zena**.
4. Follow the prompts; choose **replace** if asked whether to keep or replace configuration files.
5. Reboot.

`mintupgrade` is for the major 21.3 → 22 stage. Use Update Manager for the 22 → 22.3 stage.

## Phase 8 — restore external applications one at a time

Do not move old source files back into place. For every required product, use its official current instructions for Ubuntu 24.04/Noble, then verify it before proceeding to the next product.

Suggested order:

1. FortiClient and required networking/VPN software.
2. Docker Engine/Desktop; test containers and volumes.
3. Percona Server for MongoDB **or** upstream MongoDB—select one server ecosystem.
4. Node.js, Bazel, and developer tools.
5. Browsers, VS Code, Sublime, and remaining desktop software.
6. Antigravity and MWT/Shiftkey only after confirming vendor support.

After restoring each vendor source:

```bash
sudo apt update
apt-cache policy <package-name>
sudo apt install <package-name>
```

If no Ubuntu 24.04 support exists, keep that repository disabled or isolate the application in a VM/container.

## Phase 9 — final verification and cleanup

```bash
sudo apt update
sudo apt full-upgrade
sudo reboot

cat /etc/os-release
uname -r
dpkg --audit
systemctl --failed
apt-mark showhold
nvidia-smi
docker version
virsh list --all
```

After several normal boots and successful application/data tests, review cleanup candidates:

```bash
sudo apt autoremove --dry-run
```

Only if every proposed removal is acceptable:

```bash
sudo apt autoremove --purge
sudo apt clean
```

Keep the Timeshift snapshot and `~/mint-upgrade-backup` until the machine has remained stable through normal work, reboot, graphics, networking, container/database, and VM tests.

## Official sources

- [Mint 21.3 → 22 upgrade guide](https://linuxmint-user-guide.readthedocs.io/en/latest/upgrade-to-mint-22.html)
- [Mint 22 → 22.3 upgrade guide](https://blog.linuxmint.com/?p=4980)
- [Mint 22.3 release notes](https://linuxmint.com/rel_zena.php)
