# Linux Mint Upgrade Assessment

**Assessment date:** 2026-09-22
**Method:** read-only local inspection; no operating-system changes were made.

## Recommendation

Upgrade from **Linux Mint 21.3 Cinnamon (Virginia)** to **Linux Mint 22.3 Cinnamon (Zena)** through Mint’s supported stages:

```text
Mint 21.3 → Mint 22 → Mint 22.3
```

Mint 22.3 is supported until April 2029. This computer has compatible hardware and 200 GB free on `/`, so an in-place upgrade is viable. Because it also has Docker, Percona MongoDB, QEMU/libvirt, NVIDIA, and many external repositories, a clean Mint 22.3 installation remains the lower-risk alternative if its applications and data can be restored from backups.

Follow the companion [upgrade runbook](linuxmint-22.3-upgrade-runbook.md) for the in-place route. Never upgrade by manually replacing `jammy` with `noble` in APT source files.

## Observed system state

| Item | Result |
| --- | --- |
| OS / desktop | Linux Mint 21.3 “Virginia”, Cinnamon |
| Ubuntu base | 22.04 “Jammy” |
| Architecture | amd64 / x86_64 |
| Running kernel | 5.15.0-151-generic |
| Root filesystem | ext4 on `/dev/nvme0n1p2` |
| Root disk | 468 GB total; 200 GB available; 56% used |
| EFI partition | 511 MB total; 502 MB available |
| Secure Boot | Disabled |
| DPKG integrity | No output from `dpkg --audit` |
| Explicit APT holds | None (`apt-mark showhold` empty) |
| Installed Debian packages | 3,056 |
| Manually marked packages | 2,114 |
| Flatpak apps | Whaler; SQuirreL SQL Client |

The simulated update showed a large normal pending update set. It listed NVIDIA 535, `fwupd`, `netplan`, and `xreader` packages as retained, but that is not a manual hold. Apply ordinary Mint 21.3 updates before starting the major upgrade; do not force packages or remove them to bypass a preflight check.

## Hardware and driver compatibility

| Component | Detected state | Required post-upgrade test |
| --- | --- | --- |
| Integrated graphics | Intel UHD 620; `i915` | Login, desktop, external display |
| Discrete graphics | NVIDIA GeForce MX150; NVIDIA 535 | `nvidia-smi`, desktop, suspend/resume |
| Wi-Fi | Intel Wireless-AC 7265; `iwlwifi` | Wi-Fi and Bluetooth |
| Ethernet | Realtek RTL8111/8168; `r8169` | Wired network |
| Storage | WD Blue SN500/SN520 NVMe | Boot and filesystem health |

Mint 22.3 has an HWE 6.14 kernel option. Initially retain the Mint 22 6.8 LTS kernel: the laptop is stable, uses NVIDIA, and has virtualization tooling. The MX150 is on driver 535—not Mint’s documented legacy NVIDIA 470 concern—but graphics must still be tested after every release stage.

## Important installed workloads

| Area | Relevant installed software | Required precaution |
| --- | --- | --- |
| Containers | Docker Engine 29, Docker Desktop, Compose, Buildx, containerd | Back up compose files, bind mounts, volumes, and required images/configuration |
| Database | Percona Server for MongoDB 8.0, mongosh, telemetry | Make a tested database backup and confirm Ubuntu 24.04 vendor support |
| Virtualization | QEMU 6.2, libvirt 8.0 | Back up guest XML definitions, VM disks, ISOs, networks |
| VPN/security | FortiClient 7.4 | Obtain vendor-supported Ubuntu 24.04 release before restoring |
| Developer tools | Node.js 20, Bazel 9, VS Code, Sublime | Restore only vendor-supported repositories/packages |
| Browsers | Chrome and Brave | Reinstall current vendor repository definitions after base upgrade |

No matching Docker, MongoDB, libvirt, FortiClient, OpenVPN, or SSH service was running during inspection. This does not prove that their data is absent.

## External repository inventory

Disable all listed sources during the base-system upgrade; keep `official-package-repositories.list` enabled.

| File | Source/channel |
| --- | --- |
| `mongodb-org-8.0.list` | MongoDB upstream, Jammy |
| `percona-tools-release.list` | Percona tools, Jammy |
| `percona-telemetry-release.list` | Percona telemetry, Jammy |
| `percona-psmdb-80-release.list` | Percona Server for MongoDB, Jammy |
| `percona-pbm-release.list` | Percona Backup for MongoDB, Jammy |
| `brave-browser-release.sources` | Brave stable |
| `google-chrome.sources` | Google Chrome stable |
| `vscode.sources` | Visual Studio Code stable |
| `docker.list` | Docker, Jammy |
| `nodesource.list` | NodeSource 20 |
| `bazel.list` | Bazel stable |
| `repo.fortinet.com.list` | FortiClient 7.4, Ubuntu 22 |
| `sublime-text.list` | Sublime stable |
| `antigravity.list` | Antigravity updater |
| `mwt-desktop.list` | Shiftkey/MWT Desktop |

Percona Server for MongoDB is installed, but `mongodb-org` is not. Do not reactivate both server ecosystems by default. Choose the required distribution and install only its vendor-supported Ubuntu 24.04 source.

## Recovery and package inventory

Timeshift is installed, but its configured destination and snapshots could not be checked without administrator access. Before upgrading, configure and verify a Timeshift snapshot and an independent data backup. Keep a tested Mint 22.3 live USB available for recovery.

The full package listing is not embedded here because it contains 3,056 entries and becomes stale with each update. Phase 1 of the runbook creates dated manifests. At assessment time, the package listing SHA-256 was `03fbf3fee20e4983aaacb3d645c982986691ef8a091ebd57dadd4c5a54ddb1f3`; the manual-package list SHA-256 was `eb68ea2f784d3ef8fa412483a7ba2c83a14045af26b4b911f0c625b4597263a6`.

## Official sources

- [Mint 21.3 → 22 upgrade guide](https://linuxmint-user-guide.readthedocs.io/en/latest/upgrade-to-mint-22.html)
- [Mint 22 → 22.3 upgrade guide](https://blog.linuxmint.com/?p=4980)
- [Mint 22.3 release notes](https://linuxmint.com/rel_zena.php)
