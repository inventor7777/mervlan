# MerVLAN

MerVLAN is an addon for Asuswrt‑Merlin that adds a **graphical VLAN manager directly inside the stock Asus/Merlin web UI**.

It is designed for AP‑mode deployments and lets you:

- Assign VLANs per SSID (Wi‑Fi network)
- Assign VLANs per physical LAN port
- (Experimental) Configure trunk ports for nodes connected directly to the main unit
- Synchronize VLAN config to other Asuswrt‑Merlin nodes over SSH

The addon installs under the normal Merlin web interface (LAN section) and handles the low‑level bridge/VLAN wiring for you.

> [!WARNING]
> **MerVLAN is not a router or managed switch.** It tags and bridges traffic at the AP; you still need a VLAN-aware upstream switch/firewall for routing, DHCP, and policy.

> [!TIP]
> New here or looking for setup details? Read the full [MerVLAN Help Guide](docs/HELP.md) for topology examples, requirements, supported devices, troubleshooting, and mapper instructions.  
> The full help guide is also available offline from within the MerVLAN UI. Click **INFO** → **Help**.

---

<a id="index"></a>

## Index

1. [Status / Beta Notes](#status-beta-notes)
2. [What MerVLAN Actually Does](#what-mervlan-actually-does)
3. [Key Features](#key-features)
4. [Requirements](#requirements)
5. [Limitations](#limitations)
6. [Install](#install)
7. [Uninstall](#uninstall)
8. [Update](#update)
9. [Logs & Debugging](#logs-debugging)
10. [Development / Testing Notes](#development-testing-notes)
11. [Changelog](#changelog)
12. [Help wanted: LAN/ETH port mapping (device support)](#help-wanted)
13. [Branches, Releases and Contributions](#branches-releases-and-contributions)
14. [Community Contributors](#community-contributors)
15. [License](#license)

---

<h2 id="status-beta-notes">Status / Beta Notes</h2>

- **[View the complete list of supported devices](docs/HELP.md#8-device-support)**
- **Status:** Public beta – expect bugs and breaking changes.
- **Mode:** **AP‑mode only** (main and nodes must be running as APs, not routers).
- For setup questions and general discussion, use [Discord](https://discord.com/invite/8c3C8q54hn) or [snbforums.com](https://www.snbforums.com/threads/mervlan-v0-52-1-dev-0-52-7-simple-and-powerful-vlan-management-beta.95936/).
- If you hit issues, reproducible bugs or broken features, open a GitHub Issue and include the relevant logs:
  - CLI output:
    - `/tmp/mervlan_tmp/logs/cli_output.log` (also visible via the UI)
  - Main log:
    - `/tmp/mervlan_tmp/logs/vlan_manager.log` (also visible via the UI)

---

<h2 id="what-mervlan-actually-does">What MerVLAN Actually Does <sub><sup><a href="#index">. . . [back to index]</a></sup></sub></h2>

High level:

- Adds a **VLAN configuration UI** into Asuswrt‑Merlin so you don’t need to maintain custom scripts by hand.
- Converts your choices into the right mix of **SSID ↔ interface mapping, bridges, and VLAN interfaces** for your device.
- Keeps configuration **persistent across reboots** and **repairs it if something breaks**.

Under the hood (simplified):

- Detects hardware capabilities (SSIDs, LAN ports, guest slots, etc.) via `functions/hw_probe.sh` and stores them in `settings/settings.json`.
- Maps each SSID and LAN port to VLANs based on your UI selections and writes a canonical JSON config.
- Applies VLAN tagging/bridging via `functions/mervlan_manager.sh` and friends.
- Hooks into `services-start` and `service-event` using templates in `templates/mervlan_templates.sh` so VLANs re‑apply automatically on boot and certain system events.
- Uses a health‑check/cron‑style script (`functions/heal_event.sh`) to detect if VLAN bridges go missing and re‑apply them.

This gives you a repeatable, UI‑driven way to deploy and maintain VLANs on Asuswrt‑Merlin APs.

---

<h2 id="key-features">Key Features <sub><sup><a href="#index">. . . [back to index]</a></sup></sub></h2>

**UI‑driven VLAN management**

- Per‑SSID VLAN tagging (up to the number of SSIDs supported by your device).
- Per‑LAN‑port VLAN tagging for access ports.
- **Experimental MAIN → NODE trunk support** for nodes connected directly to selected LAN ports on the main unit.
- Built‑in “Clients Overview” panel to see which VLAN clients are active on each node.

**Multi‑AP / Multi‑node aware**

- Syncs configuration and scripts to other Asuswrt‑Merlin APs/nodes over SSH using `functions/sync_nodes.sh`.
- Supports mixed models as long as they run Asuswrt‑Merlin (or compatible) with addon support.
- Optional modes to run VLAN manager locally, on nodes only, or on both.

**Self‑healing behavior**

- `functions/heal_event.sh` and service hooks monitor VLAN bridges.
- If VLAN bridges disappear (e.g., you changed LAN/Wi‑Fi settings and Merlin wiped them), MerVLAN re‑applies the expected configuration.
- Health check runs on a short interval (worst‑case downtime roughly a few minutes); in testing, stable setups run for weeks without observed VLAN drops.

**Safe integration with Merlin**

- Uses templates in `templates/mervlan_templates.sh` instead of blindly overwriting `services-start`/`service-event`.
- Hooks are injected in a variant‑aware way and can be removed cleanly by the uninstall script.

**Logging and debugging**

- Structured logs in `/tmp/mervlan_tmp/logs/`:
  - `vlan_manager.log` – core apply pipeline
  - `cli_output.log` – what the UI shows in the command output panel
  - Additional logs for node sync, hardware probe, etc.
- Logs are also exposed via the UI under `/www/user/mervlan/tmp/logs`.

**Install/Update lifecycle**

- First‑install script lays out directories, installs hooks, and provisions the UI.
- Update script (`functions/update_mervlan.sh`) can refresh the addon in‑place while preserving `settings/settings.json` and SSH keys.
- A public copy of settings is kept under `/www/user/mervlan/settings/settings.json` for the SPA to read.

---

<h2 id="requirements">Requirements <sub><sup><a href="#index">. . . [back to index]</a></sup></sub></h2>

- **Asuswrt‑Merlin firmware** with addon support on every device that will tag VLANs.
- **AP‑mode only** on all participating routers/APs.
- **JFFS enabled** for persistent storage.
- **SSH enabled** on the main AP and any standalone APs/nodes (AiMesh nodes share SSH keys).
- **Ethernet backhaul only** between the main unit, upstream switch and nodes:
  - Wi-Fi backhaul cannot preserve VLAN tags on Asus hardware or driver stacks.
  - **MAIN → NODE trunking is supported experimentally:** a downstream node may connect directly to a selected LAN port on the main unit when that port is configured as an 802.1Q trunk.
  - **NODE → NODE trunking is not supported:** a node cannot provide a MerVLAN trunk to another downstream node. Each directly connected node must connect to the main unit.
- **VLAN‑aware upstream device** (mandatory):
  - Managed switch and VLAN‑aware router/firewall (e.g., OPNsense, pfSense, Asus Pro, etc.).
  - MerVLAN does **not** provide routing, firewalling, or DHCP; those must be handled upstream.

Multi‑AP notes:

- Each node must either connect to a VLAN-aware switch or directly to a trunk-enabled LAN port on the main unit using the experimental MAIN → NODE topology.
- LAN port VLAN tagging is currently **global** – the same per‑port mapping is applied to all synced APs.
  - Per‑device LAN port settings are planned but not yet available; for now, any per‑device tweaks must be applied manually via SSH.

SSH key behavior:

- On typical AiMesh setups, the **main AP’s SSH key** (installed via MerVLAN’s “SSH Key Install” flow) is shared with AiMesh nodes by the firmware.
- For **standalone APs used as nodes** (non‑AiMesh), you must **manually install the same public key** on each unit, just as you did on the main AP, before MerVLAN can sync and execute remotely on them.

---

<h2 id="limitations">Limitations <sub><sup><a href="#index">. . . [back to index]</a></sup></sub></h2>

- The number of wireless VLAN assignments is bounded by the number of usable SSID slots supported by the device. For example, a device with five usable SSID slots can have up to five active wireless VLAN assignments. Additional wired-only VLANs may be assigned to physical LAN ports, although the number of assignable ports varies by device.
- Mesh behavior is constrained by Asus firmware:
  - Some models support more guest SSIDs than they can actually mesh; non‑mesh SSIDs will only broadcast from the main node.
  - Devices on VLANs use standard band steering; per‑VLAN steering is not supported.
- Wi‑Fi backhaul cannot carry VLAN tags; only Ethernet backhaul is supported for VLAN‑aware nodes.
- MerVLAN does not: route traffic, run DHCP, or replace a firewall.

---

<h2 id="install">Install <sub><sup><a href="#index">. . . [back to index]</a></sup></sub></h2>

Only install if you are comfortable with **beta software** and have a way to recover (including factory reset) if something goes wrong.

SSH into the AP and run this command. The addon will be placed under **LAN → MerVLAN** in the GUI:

Before installing, skim the [MerVLAN Help Guide](docs/HELP.md), especially the topology and requirements sections.

```sh
mkdir -p /jffs/addons/mervlan && /usr/sbin/curl -fsL --retry 3 "https://raw.githubusercontent.com/r80xcore/mervlan/refs/heads/main/install.sh" -o "/jffs/addons/mervlan/install.sh" && chmod 0755 /jffs/addons/mervlan/install.sh && /jffs/addons/mervlan/install.sh full
```

### Development install

Use this if you want to test the latest development build. It may contain unfinished changes and can be less stable than the `main` branch, but may include support for additional devices, security improvements, and newer implementations.

```sh
mkdir -p /jffs/addons/mervlan && /usr/sbin/curl -fsL --retry 3 "https://raw.githubusercontent.com/r80xcore/mervlan/refs/heads/dev/install.sh" -o "/jffs/addons/mervlan/install.sh" && chmod 0755 /jffs/addons/mervlan/install.sh && /jffs/addons/mervlan/install.sh full dev
```

This will:

- Create `/jffs/addons/mervlan` and required subdirectories.
- Install the core scripts under `functions/` and configs under `settings/`.
- Install the web UI under `/www/user/mervlan`.
- Inject the required `services-start` / `service-event` hooks.

If the web UI ever looks out of sync or partially broken after manual file changes, you can use a quick uninstall + reinstall as a **manual flush/refresh** of the public UI and addon files:

```sh
/jffs/addons/mervlan/uninstall.sh && /jffs/addons/mervlan/install.sh
```

---

<h2 id="uninstall">Uninstall <sub><sup><a href="#index">. . . [back to index]</a></sup></sub></h2>

From `/jffs/addons/mervlan` on the AP:

- **Standard uninstall** (leave addon data directories in place):

  ```sh
  ./uninstall.sh
  ```

- **Full uninstall** (also removes addon directories and temp workspace):

  ```sh
  ./uninstall.sh full
  ```

  This will remove `/jffs/addons/mervlan` and `/tmp/mervlan_tmp` and attempt to clean up the service hooks.

---

<h2 id="update">Update <sub><sup><a href="#index">. . . [back to index]</a></sup></sub></h2>

> [!NOTE]
> Updates preserve your settings, SSH keys, MAC Shield databases, and local backups whenever possible. After updating, MerVLAN refreshes the public web UI files and reapplies the required service hooks. Refresh your browser after the update to load the latest web interface.

MerVLAN can be updated in place without losing its existing configuration or SSH keys.

The recommended method is through the web UI. Click the version button in the bottom-right corner, select the branch you want to use, check for updates, and install the available version.

- **`main`** is the recommended public beta and release branch.
- **`dev`** is the active development branch and may contain less-tested changes.

The web UI supports switching between `main` and `dev` when the selected branch contains a newer version. Downgrading through the web UI is not currently supported, but can be done manually over SSH.

### Manual Update Commands

| Command | What it does |
| --- | --- |
| `sh functions/update_mervlan.sh` | Update to the latest version from the `main` public beta channel. |
| `sh functions/update_mervlan.sh dev` | Update to the latest version from the `dev` development channel. |
| `sh functions/update_mervlan.sh restore` | Open the restore menu and restore a previously created local MerVLAN backup. |

The manual updater supports switching between the `main` and `dev` branches. If an update does not behave as expected, or the configuration becomes corrupted, the built-in restore function can return MerVLAN to one of its locally stored backups.

The updater will:

- Download and validate the selected version from GitHub.
- Stage the new files and perform an atomic replacement.
- Preserve `settings/settings.json`, SSH keys, MAC Shield databases, and local backups whenever possible.
- Re-run the hardware probe.
- Refresh the public web UI files and reinstall the required service hooks.
- Synchronize configured remote nodes when SSH is enabled and the nodes are reachable.

> [!TIP]
> For custom branches, manual downgrades, restore instructions, and additional update commands, see [Updating MerVLAN in the Help Guide](docs/HELP.md#updating-mervlan).

---

<h2 id="logs-debugging">Logs & Debugging <sub><sup><a href="#index">. . . [back to index]</a></sup></sub></h2>

Primary log directory:

- `/tmp/mervlan_tmp/logs`

Common logs:

- `vlan_manager.log` – VLAN apply pipeline and health checks.
- `cli_output.log` – mirrored output of commands run from the UI.
- Additional logs for node sync, hardware probe, and other helpers.

You can tail these over SSH, for example:

```sh
tail -f /tmp/mervlan_tmp/logs/cli_output.log
tail -f /tmp/mervlan_tmp/logs/vlan_manager.log
```

These same logs are exposed via the web UI using symlinks under:

- `/www/user/mervlan/tmp/logs`

Log formatting, colors, and syslog tagging are configurable in:

- `settings/log_settings.sh`

---

<h2 id="development-testing-notes">Development / Testing Notes <sub><sup><a href="#index">. . . [back to index]</a></sup></sub></h2>

- Developed on an **ASUS XT8** mesh system in AP‑mode.
- Intended to work with most newer Asuswrt‑Merlin / Gnuton‑supported routers and mesh AP systems when used as APs.
- Experimental MAIN → NODE trunk support is under active testing and can be configured through the trunk options in the UI.

For structured beta testing and discussion, see the SNBForums thread and Discord (links below)

- **MerVLAN on Discord:** <https://discord.gg/8c3C8q54hn>
- **MerVLAN on SNBForums:** <https://www.snbforums.com/threads/mervlan-v0-50-simple-and-powerful-vlan-management-beta.95936/#post-972292>

---

<h2 id="changelog">Changelog <sub><sup><a href="#index">. . . [back to index]</a></sup></sub></h2>

See the **[`changelog.txt`](changelog.txt)** in this repository for detailed version history and notes.

### Current development fixes (v0.53.21-dev)

- Apply-on-boot actions use compact verified-action tokens so ASUS service-event
  handling cannot truncate the correlation value before completion is reported.
- `BOOT_ENABLED` updates refresh legacy regular-file web settings copies while
  retaining the persistent-settings symlink as the preferred source of truth.
- The Settings modal remains open after a successful save, shows a green
  `Settings Saved!` confirmation, and re-enables interaction.
- Boot propagation failures on configured nodes are reported as partial success
  while preserving the successfully applied state on the main router.

---

<h2 id="help-wanted">Help wanted: LAN/ETH port mapping (device support) <sub><sup><a href="#index">. . . [back to index]</a></sup></sub></h2>

To add official support for more routers, we need accurate LAN port mapping (LAN1 → LANX → ethX). The helper script below walks you through mapping and creates everything needed for upstream support.

For the current supported-device list, see [Device Support in HELP.md](docs/HELP.md#8-device-support).

### What the mapper does

- Detects the WAN/uplink interface.
- Guides you through mapping each physical LAN port.
- Generates a ready‑to‑use `hw_probe.sh` case snippet.
- Writes a full report to `/tmp/mervlan_tmp/results`.
- Provides a pre‑filled GitHub issue link for submission.
- Optionally patches a local MerVLAN install for temporary support.

### Run the mapper (one‑liner)

```sh
mkdir -p /tmp/mervlan_tmp && /usr/sbin/curl -fsL --retry 3 "https://raw.githubusercontent.com/r80xcore/mervlan/dev/functions/device_support_mapper.sh" -o "/tmp/mervlan_tmp/device_support_mapper.sh" && chmod 0755 /tmp/mervlan_tmp/device_support_mapper.sh && sh /tmp/mervlan_tmp/device_support_mapper.sh
```

### How to use it

1. **Start with only the WAN cable connected.**
2. **Unplug all LAN cables** before running the script.
3. **WAN detection (Step 1/2):** the script detects the WAN/uplink interface.
4. **LAN mapping (Step 2/2):**
   - Enter the number of physical LAN ports (excluding WAN).
   - For each LAN port (LAN1 → LANX):
     - Unplug the cable when prompted.
     - Plug into the requested LAN port.
     - Press Enter and confirm the detected interface.
   - You can retry, skip, or quit at any step.
5. **Report generation:** submit the pre‑filled GitHub issue link (add extra notes if needed).

### Important notes

- MerVLAN does **not** need to be installed to run the mapper.
- `/tmp` is cleared on reboot—save the report or submit the issue.
- Local patching is a stopgap; please submit the report for official support.
- Primary testing target is AP mode, but router‑mode validation is helpful too.


### Manual template (if you already know the mapping)

Use the template below (text in brackets is informational):

```sh
RT-AX86U) MODEL="RT-AX86U"; ETH_PORTS="eth4 eth3 eth2 eth1 eth5"; LAN_PORT_LABELS="LAN1 LAN2 LAN3 LAN4 LAN5"; MAX_ETH_PORTS=5; WAN_IF="eth0" ;;
[nvramname]     [ model  ]            [        interface       ]                  [        LAN ports       ] [ Max LAN ports ]    [wan port]
```

Example with a different nvram name than the commonly used name:

```sh
RT-AX95Q) MODEL="XT8"; ETH_PORTS="eth1 eth2 eth3"; LAN_PORT_LABELS="LAN1 LAN2 LAN3"; MAX_ETH_PORTS=3; WAN_IF="eth0" ;;
```

Example where WAN is not `eth0`:

```sh
RT-AX58U) MODEL="RT-AX58U"; ETH_PORTS="eth3 eth2 eth1 eth0"; LAN_PORT_LABELS="LAN1 LAN2 LAN3 LAN4"; MAX_ETH_PORTS=4; WAN_IF="eth4" ;;
```

Find your nvramname with:

```sh
nvram get productid
```

`MODEL` can use the same value as the NVRAM product ID or a different display name if the device is commonly known by another model name, as shown by the XT8 example.

### Models requiring testing

WiFi 7 / BE Series:

- RT‑BE58 Go
- RT‑BE86U
- RT‑BE88U
- RT‑BE96U
- GT‑BE98 Pro
- GT‑BE19000AI

ROG & high‑performance series:

- GT‑AX11000
- GT‑AXE11000
- GT‑AXE16000

TUF Gaming series:

- TUF‑AX3000 v1
- TUF‑AX5400 v1

Standard RT‑AX series:

- RT‑AX68U
- DSL‑AX82U
- DSL‑AX5400

Models added to the support table are excluded from this list. Any help testing is appreciated.

---

<h2 id="branches-releases-and-contributions">Branches, Releases and Contributions <sub><sup><a href="#index">. . . [back to index]</a></sup></sub></h2>

MerVLAN uses two primary branches:

- **`main`** is the recommended public beta and release branch.
  - Normal installations and updates use this branch.
  - Tagged GitHub pre-releases are created from commits on `main`.
  - Development changes reach `main` through controlled merges from `dev`.

- **`dev`** is the active development and integration branch.
  - It may contain unfinished, experimental, or less-tested changes.
  - It remains available directly through GitHub but does not receive tagged releases.
  - Development installations may use this branch to test upcoming changes.

For development and testing, temporary branches may also be created from `dev`. These are normally named using the `dev-test<number>` format, such as `dev-test1` or `dev-test2`.

These branches are temporary and are only intended for active development and targeted testing. Unless you are directly involved in testing a specific branch, using one is strongly discouraged unless requested by the maintainer. They may contain incomplete, experimental, or untested code and should not be considered release versions.

MerVLAN can be manually updated to one of these branches through SSH. See the [MerVLAN Help Guide, CLI Usage](docs/HELP.md#7-cli-usage) for instructions.

### Contributing

Contributors should normally:

1. Create feature or fix branches from `dev`.
2. Submit pull requests back into `dev`.
3. Avoid targeting `main` directly unless requested by the maintainer.
4. Ensure that emergency fixes made against `main` are also merged or cherry-picked back into `dev`.

When a development version is considered ready, `dev` is merged into `main` and a new tagged GitHub pre-release is published.

---

<h2 id="community-contributors">Community Contributors ⭐ <sub><sup><a href="#index">. . . [back to index]</a></sup></sub></h2>

MerVLAN wouldn't support as many devices as it does today without the community members who took the time to run the hardware mapper on their own routers and submit their device mappings. Every submission helps expand compatibility, improve detection, and make MerVLAN available to more users. Thank you to everyone who contributed!

**Model collection from GitHub:**

bieniu, pxdl, davittoncat, RikshaDriver, Mudcrab353, franzatkiermeyereu, mdraco11, tooty-1135, getBoolean, piratak, kashif789us, bigadron, MathNerd28

**Model collection from SNBForums:**

mistermoonlight1, kstamand, commodoro, amplatfus, jksmurf, brzd, ika

---

**Extra Special Thanks** to **agnithin** and **inventor7777**. Thank you for jumping in, contributing code, sharing ideas, and helping improve MerVLAN for everyone. Open-source projects thrive because people like you choose to spend your time helping others, and I genuinely appreciate everything you've done for this project.


---

<h2 id="license">License <sub><sup><a href="#index">. . . [back to index]</a></sup></sub></h2>

See `LICENSE` for full license details.
