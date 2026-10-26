# check

[![ShellCheck & Syntax](https://github.com/guitareren/check/actions/workflows/shellcheck.yml/badge.svg)](https://github.com/guitareren/check/actions/workflows/shellcheck.yml)

A small terminal tool that shows your network and system status at a glance: WiFi, Ethernet, ping, internet speed, and hardware resources.

`check` doesn't need to be compiled and doesn't install any packages of its own. It's a single bash script that combines standard Linux tools (`nmcli`, `ip`, `ping`, `curl`, `/proc`, `/sys`) into one readable, color-coded view. Use it when your connection feels slow or your machine is under heavy load and you want instant diagnosis.

Available in **English** and **Turkish** (see [Language](#language)).

## Features

- **`check wifi`**: the connected network (SSID, BSSID), band (2.4 / 5 / 6 GHz for WiFi 6E/7), channel, frequency, real-time link bitrate (via `iw`, with AP max rate fallback), signal strength (as a percentage bar and in dBm), local IP, gateway, DNS, and MAC. It also lists nearby networks and marks networks on **your channel** and **adjacent channels** (in 2.4 GHz, within ±4 channels) that can cause interference.
- **`check cable`**: whether a cable is plugged in, plus link speed (Mb/s; displays "unknown" if unnegotiated or in VMs), duplex, local IP, gateway, DNS (resolved via fallback: `nmcli` → `resolvectl` → `/etc/resolv.conf`), and MAC. Runs completely without NetworkManager.
- **`check internet`**: your public IP, ping to the router, Cloudflare (1.1.1.1), and Google (8.8.8.8) with average latency, jitter (ping mdev, approximate), and packet loss, and DNS lookup time.
- **`check speed`**: everything in `internet`, plus a download/upload speed test via Cloudflare (about 20 seconds).
- **`check sys`** (or **`check sistem`**): real-time system and hardware monitor (~0.2s). CPU model, usage percentage bar, and temperature; RAM and Swap usage (in GiB and percentage bar); multi-GPU support for NVIDIA, AMD, and Intel (GPU model and temperature; utilization % and VRAM for NVIDIA/AMD only; numbered `GPU 0`, `GPU 1`, etc. when multiple GPUs are present, or simply `GPU` for a single card; GPU lines are cleanly hidden on headless servers or VMs without a GPU).
- **`check`** with no arguments: wifi + cable + internet (no speed test).
- **`check -h`**: detailed help.

Values are colored **green** (good), **yellow** (fair), or **red** (bad).

## Usage

```bash
check wifi
check cable              # or: check kablo
check internet           # or: check net
check speed              # or: check internet speed
check sys                # or: check sistem
check -h                 # or: check --help
```

### Language

By default `check` follows your system locale: Turkish if it starts with `tr`, English otherwise. To override it:

```bash
check --lang en wifi     # just this run (also: --lang=tr, -l tr)
export CHECK_LANG=en     # permanent: add to ~/.bashrc or ~/.zshrc
```

Priority: `--lang` > `CHECK_LANG` > system locale.

### Terminal & ASCII Mode

`check` automatically detects whether your terminal/locale supports UTF-8. If UTF-8 is not available, when `LANG` is undefined (such as in minimal Docker containers, cron jobs, or certain SSH sessions), or if you force ASCII mode:

```bash
CHECK_ASCII=1 check sys
```

It replaces Unicode box-drawing characters and symbols with ASCII equivalents (`#` and `-` for bars, `==` for titles, `C` for temperatures).

### Reading the results

| Symptom | Likely cause |
|---|---|
| High or unstable ping to the router | WiFi problem: weak signal, crowded channel, interference |
| Router ping is fine, external ping is bad | Internet provider problem |
| Many networks on the same channel | Switch the router to a less crowded channel |
| High CPU, RAM, or GPU usage (>85%) | Resource bottleneck or runaway process |

- **Signal guide**: **-50 dBm** excellent · **-67 dBm** good · **-75 dBm** or below weak.
- **Typical latency**: router ping under 5 ms, external ping around 50 ms or lower.
- **Resource usage**: **<70%** green (good) · **70%–85%** yellow (moderate) · **>85%** red (high load).
- **Temperatures**: **<60°C** green (cool) · **60°C–80°C** yellow (warm) · **>80°C** red (hot).

## Installation

Install the dependencies (see [`requirements.txt`](requirements.txt)):

```bash
# openSUSE
sudo zypper install bash NetworkManager iproute2 iputils curl gawk grep sed coreutils
# Debian / Ubuntu
sudo apt install bash network-manager iproute2 iputils-ping curl gawk grep sed coreutils
```

Clone the repo and link the script into your `PATH`:

```bash
git clone https://github.com/guitareren/check.git
cd check
chmod +x check
mkdir -p ~/.local/bin
ln -s "$PWD/check" ~/.local/bin/check
```

Make sure `~/.local/bin` is in your `PATH`, then run `check -h`.

## Requirements

- **NetworkManager** (`nmcli`): required **only** for `check wifi`. `check cable`, `internet`, `speed`, and `sys` do not require NetworkManager. In `check` (all mode), WiFi is gracefully skipped if NetworkManager is absent while cable and internet diagnostics run normally.
- **iw** (optional): used by `check wifi` to display actual real-time rx/tx link bitrates. If `iw` is not installed or returns no link data, `check wifi` falls back to showing the AP's theoretical maximum rate (`AP max rate` / `AP maks. hız`).
- A terminal with ANSI color support (automatically disabled when piping or when `NO_COLOR` is set; stderr error messages never leak escape codes into redirected logs).
- Internet access for `internet` / `speed` (uses `1.1.1.1` and `speed.cloudflare.com`).
- System monitor (`sys`) uses `/proc` and `/sys` (standard Linux kernel interfaces); no extra tools required (optional `nvidia-smi` or `sensors` for enhanced metrics).

## How it works

- WiFi details come from `nmcli` and `iw` (real rx/tx bitrates, falling back to AP max rate if `iw` is absent), frequency is evaluated to determine 2.4 GHz, 5 GHz, or 6 GHz (WiFi 6E/7), nearby networks highlight same and adjacent (±4 channels on 2.4 GHz) channel interference, and signal in dBm from `/proc/net/wireless`.
- Ethernet details come from sysfs (`/sys/class/net/<device>/`), without requiring NetworkManager. Interface discovery filters physical interfaces from virtual/WiFi ones. DNS resolution resolves via fallback chain (`nmcli` → `resolvectl` → `/etc/resolv.conf`). Unknown link speeds (such as in VMs with `-1`) and unknown duplex states display cleanly as "unknown" / "bilinmiyor".
- Ping stats come from 10 quick pings per target; jitter (ping mdev, approximate) is displayed with clean dash (`—` / `-`) fallback when unavailable.
- The speed test runs 4 parallel 25 MB downloads from Cloudflare and adds up their speeds, then does a 25 MB upload.
- System metrics (`sys`) utilize a modular multi-tier fallback architecture:
  - **CPU**: Model via `/proc/cpuinfo` (with ARM/Raspberry Pi and `lscpu` fallback). Real-time CPU % via 0.2s delta from `/proc/stat` (with `/proc/loadavg` fallback). Temperatures via hwmon drivers (`k10temp`, `zenpower`, `coretemp`, `cpu_thermal`), thermal labels (`Tctl`, `Package`, `CPU`), ACPI thermal zones, or `sensors` (silently omitted on VM/sensorless platforms).
  - **RAM & Swap**: `MemAvailable` from `/proc/meminfo` (with `MemFree + Buffers + Cached` fallback) in GiB. Swap is displayed when configured (`SwapTotal > 0`).
  - **GPU**: Multi-GPU support covering NVIDIA (via `nvidia-smi`), AMD (via DRM sysfs and `amdgpu` hwmon), and Intel (via DRM sysfs and `i915`/`xe` hwmon; utilization % and VRAM are available for NVIDIA and AMD, while Intel reports model and temperature). When multiple GPUs are detected, each card is cleanly enumerated (`GPU 0`, `GPU 1`, ...). If no GPU is present or on headless servers, GPU lines are automatically hidden.
- External commands whose outputs are parsed are executed via a C-locale wrapper (`lc`), preventing locale mismatches across different system languages while preserving the user's native UTF-8 locale for terminal display and padding.
- Colors are turned off when the output is piped or `NO_COLOR` is set. Errors go to stderr with exit code 1.

## License

[MIT](LICENSE)
