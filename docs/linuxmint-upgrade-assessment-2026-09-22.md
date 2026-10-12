# Linux Mint Upgrade Assessment

**Initial inspection:** 2026-09-22; **refreshed:** 2026-10-12
**Method:** read-only local inspection; no operating-system changes were made. Package, update, disk, and repository state were rechecked on October 12. The companion [runbook](linuxmint-22.3-upgrade-runbook.md) was reviewed against current Mint guidance on that date.

## Recommendation

Upgrade from **Linux Mint 21.3 Cinnamon (Virginia)** to **Linux Mint 22.3 Cinnamon (Zena)** through Mint’s supported stages:

```text
Mint 21.3 → Mint 22 → Mint 22.3
```

As of October 12, 2026, Mint lists 22.3 as its latest recommended release. Mint 21.3 remains supported until April 2027; Mint 22.3 is supported until April 2029. This computer has compatible hardware and 198 GB free on `/`, so an in-place upgrade is viable after the preflight checks below pass. Because it also has Docker, Percona MongoDB, QEMU/libvirt, NVIDIA, and many external repositories, a clean Mint 22.3 installation remains the lower-risk alternative if its applications and data can be restored from backups. Recheck the [current supported-release list](https://linuxmint.com/download_all.php) before starting in case Mint has published a newer version or changed its upgrade route.

Follow the companion [upgrade runbook](linuxmint-22.3-upgrade-runbook.md) for the in-place route. Never upgrade by manually replacing `jammy` with `noble` in APT source files.

## Observed system state

| Item | Result |
| --- | --- |
| OS / desktop | Linux Mint 21.3 “Virginia”, Cinnamon |
| Ubuntu base | 22.04 “Jammy” |
| Architecture | amd64 / x86_64 |
| Running kernel | 5.15.0-151-generic |
| Root filesystem | ext4 on `/dev/nvme0n1p2` |
| Root disk | 468 GB total; 198 GB available; 56% used (rechecked 2026-10-12) |
| EFI partition | 511 MB total; 502 MB available |
| Secure Boot | Disabled |
| DPKG integrity | No output from `dpkg --audit` |
| Explicit APT holds | None (`apt-mark showhold` empty) |
| Installed Debian packages | 3,056 (rechecked 2026-10-12) |
| Manually marked packages | 2,114 (rechecked 2026-10-12) |
| Flatpak apps | Whaler; SQuirreL SQL Client |

The October 12 simulated update showed a large pending update set. It listed `fwupd`, Go 1.24, `netplan`, NVIDIA 535, and `xreader` packages as retained, but no packages are manually held and `dpkg --audit` is clean. Apply ordinary Mint 21.3 updates before starting the major upgrade; do not force packages or remove them to bypass a preflight check.

## Hardware and driver compatibility

| Component | Detected state | Required post-upgrade test |
| --- | --- | --- |
| Integrated graphics | Intel UHD 620; `i915` | Login, desktop, external display |
| Discrete graphics | NVIDIA GeForce MX150; NVIDIA 535 | `nvidia-smi`, desktop, suspend/resume |
| Wi-Fi | Intel Wireless-AC 7265; `iwlwifi` | Wi-Fi and Bluetooth |
| Ethernet | Realtek RTL8111/8168; `r8169` | Wired network |
| Storage | WD Blue SN500/SN520 NVMe | Boot and filesystem health |

Mint 22.3 has an HWE 6.14 kernel option. Initially retain the Mint 22 6.8 LTS kernel: the laptop is stable, uses NVIDIA, and has virtualization tooling. The MX150 is on driver 535—not Mint’s documented legacy NVIDIA 470 concern—but graphics must still be tested after every release stage.

## Boot and filesystem readiness

The machine uses UEFI with firmware configured to start Mint’s shim. GRUB contains entries for the running 5.15.0-151 kernel, the previous 5.15.0-141 kernel, and an older 5.4 kernel. The EFI partition is 511 MB with 502 MB free, so EFI capacity is not a concern. Secure Boot is disabled. These findings indicate a usable current boot chain, though the recovery media and Timeshift snapshot still need to be verified before upgrading.

The NVIDIA 535 DKMS module was installed for the running 5.15.0-151 kernel only. Logs contain historical NVIDIA Xid 31 and PRIME/DRM errors. The kernel log check found no ext4 or NVMe I/O errors, but the inspection shell reported both `/` and `/boot/efi` as read-only. That may reflect the inspection environment rather than the installed desktop’s mount state, so this is **unresolved**. From a terminal opened in the installed desktop, check `findmnt -T /` and `findmnt -T /boot/efi`; both should include `rw` in their options. If either reports `ro`, stop before upgrading and investigate it from the actual local session. Do not try to remount a possibly damaged root filesystem read-write as a workaround. The [runbook](linuxmint-22.3-upgrade-runbook.md) includes the stop condition and recovery checks.

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

There are 15 external APT source files (rechecked 2026-10-12). Disable all listed sources during the base-system upgrade; keep `official-package-repositories.list` enabled.

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

Timeshift is installed, but its configured destination and snapshots could not be checked without administrator access. Before upgrading, configure and verify a Timeshift snapshot and an independent data backup. Keep a checksum-verified, boot-tested Mint 22.3 live USB available for recovery.

The full package listing is not embedded here because it contains 3,056 entries and becomes stale with each update. Phase 1 of the runbook creates dated manifests. For reference, the package listing SHA-256 was `03fbf3fee20e4983aaacb3d645c982986691ef8a091ebd57dadd4c5a54ddb1f3` on September 22 and `522d02a4c60a2520c0dbb09bc612fc4afcb9e942a67ef9ed0a8e426a4e4a1509` on October 12. The manual-package list hash remained `eb68ea2f784d3ef8fa412483a7ba2c83a14045af26b4b911f0c625b4597263a6`.

## Official sources

- [Currently supported Mint releases and recommendation](https://linuxmint.com/download_all.php)
- [Mint 21.3 → 22 upgrade guide](https://linuxmint-user-guide.readthedocs.io/en/latest/upgrade-to-mint-22.html)
- [Mint 22 → 22.3 upgrade guide](https://blog.linuxmint.com/?p=4980)
- [Mint 22.3 release notes](https://linuxmint.com/rel_zena.php)
