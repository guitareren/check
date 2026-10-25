# check

A small terminal tool that shows your network and system status at a glance: WiFi, Ethernet, ping, internet speed, and hardware resources.

`check` doesn't need to be compiled and doesn't install any packages of its own. It's a single bash script that combines standard Linux tools (`nmcli`, `ip`, `ping`, `curl`, `/proc`, `/sys`) into one readable, color-coded view. Use it when your connection feels slow or your machine is under heavy load and you want instant diagnosis.

Available in **English** and **Turkish** (see [Language](#language)).

## Features

- **`check wifi`**: the connected network (SSID, BSSID), band (2.4 / 5 GHz), channel, frequency, link rate, signal strength (as a percentage bar and in dBm), local IP, gateway, DNS, and MAC. It also lists nearby networks and marks the ones on **your channel**, which can cause interference.
- **`check cable`**: whether a cable is plugged in, plus link speed (Mb/s), duplex, local IP, gateway, DNS, and MAC.
- **`check internet`**: your public IP, ping to the router, Cloudflare (1.1.1.1), and Google (8.8.8.8) with average latency, jitter, and packet loss, and DNS lookup time.
- **`check speed`**: everything in `internet`, plus a download/upload speed test via Cloudflare (about 20 seconds).
- **`check sys`** (or **`check sistem`**): real-time system and hardware monitor (~0.2s). CPU model, usage percentage bar, and temperature; RAM and Swap usage (in GB and percentage bar); GPU model, utilization %, VRAM, and temperature (supporting NVIDIA, AMD, and Intel; GPU lines are cleanly hidden on headless servers or VMs without a GPU).
- **`check`** with no arguments: wifi + cable + internet (no speed test).
- **`check -h`**: detailed help.

Values are colored **green** (good), **yellow** (fair), or **red** (bad).

## Usage

```
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

- Linux with **NetworkManager** (`nmcli`)
- A terminal with UTF-8 and ANSI color support
- Internet access for `internet` / `speed` (uses `1.1.1.1` and `speed.cloudflare.com`)
- System monitor (`sys`) uses `/proc` and `/sys` (standard Linux kernel interfaces); no extra tools required (optional `nvidia-smi` or `sensors` for enhanced metrics).

## How it works

- WiFi details come from `nmcli`, and signal in dBm from `/proc/net/wireless`.
- Ethernet details come from `/sys/class/net/<device>/`.
- Ping stats come from 10 quick pings per target.
- The speed test runs 4 parallel 25 MB downloads from Cloudflare and adds up their speeds, then does a 25 MB upload.
- System metrics (`sys`) utilize a multi-tier fallback architecture:
  - **CPU**: Model via `/proc/cpuinfo` (with ARM/Raspberry Pi and `lscpu` fallback). Real-time CPU % via 0.2s delta from `/proc/stat` (with `/proc/loadavg` fallback). Temperatures via hwmon drivers (`k10temp`, `zenpower`, `coretemp`, `cpu_thermal`), thermal labels (`Tctl`, `Package`, `CPU`), ACPI thermal zones, or `sensors` (silently omitted on VM/sensorless platforms).
  - **RAM & Swap**: `MemAvailable` from `/proc/meminfo` (with `MemFree + Buffers + Cached` fallback). Swap is displayed when configured (`SwapTotal > 0`).
  - **GPU**: NVIDIA detected via `nvidia-smi`; AMD detected via DRM sysfs (`gpu_busy_percent`, `mem_info_vram`) and `amdgpu` hwmon; Intel detected via DRM sysfs and `i915`/`xe` hwmon. If no GPU is present or on headless servers, GPU lines are automatically hidden.
- `LC_ALL=C.UTF-8` is set inside the script so `nmcli` output is the same in any system language.
- Colors are turned off when the output is piped or `NO_COLOR` is set. Errors go to stderr with exit code 1.

## License

[MIT](LICENSE)
