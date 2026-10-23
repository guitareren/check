# check

A small terminal tool that shows your network status at a glance: WiFi, Ethernet, ping, and internet speed.

`check` doesn't need to be compiled and doesn't install any packages of its own. It's a single bash script that combines standard Linux tools (`nmcli`, `ip`, `ping`, `curl`) into one readable, color-coded view. Use it when your connection feels slow and you want to know whether the problem is your WiFi signal, a crowded channel, the router, or your internet provider.

Available in **English** and **Turkish** (see [Language](#language)).

## Features

- **`check wifi`**: the connected network (SSID, BSSID), band (2.4 / 5 GHz), channel, frequency, link rate, signal strength (as a percentage bar and in dBm), local IP, gateway, DNS, and MAC. It also lists nearby networks and marks the ones on **your channel**, which can cause interference.
- **`check cable`**: whether a cable is plugged in, plus link speed (Mb/s), duplex, local IP, gateway, DNS, and MAC.
- **`check internet`**: your public IP, ping to the router, Cloudflare (1.1.1.1), and Google (8.8.8.8) with average latency, jitter, and packet loss, and DNS lookup time.
- **`check speed`**: everything in `internet`, plus a download/upload speed test via Cloudflare (about 20 seconds).
- **`check`** with no arguments: wifi + cable + internet (no speed test).
- **`check -h`**: detailed help.

Values are colored **green** (good), **yellow** (fair), or **red** (bad).

## Usage

```
check wifi
check cable              # or: check kablo
check internet           # or: check net
check speed              # or: check internet speed
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

Signal guide: **-50 dBm** excellent · **-67 dBm** good · **-75 dBm** or below weak.
Typical values: router ping under 5 ms, external ping around 50 ms or lower.

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
git clone <repo-url> check
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

## How it works

- WiFi details come from `nmcli`, and signal in dBm from `/proc/net/wireless`.
- Ethernet details come from `/sys/class/net/<device>/`.
- Ping stats come from 10 quick pings per target.
- The speed test runs 4 parallel 25 MB downloads from Cloudflare and adds up their speeds, then does a 25 MB upload.
- `LC_ALL=C.UTF-8` is set inside the script so `nmcli` output is the same in any system language.
- Colors are turned off when the output is piped or `NO_COLOR` is set. Errors go to stderr with exit code 1.

## License

[MIT](LICENSE)
