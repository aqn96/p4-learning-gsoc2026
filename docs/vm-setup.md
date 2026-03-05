# Environment Setup: VirtualBox + Ubuntu 24.04 ARM64 on Apple Silicon

## Overview

This guide documents setting up a P4 development environment on an Apple Silicon Mac (M3 Pro) using VirtualBox and Ubuntu Server 24.04 LTS ARM64. This setup follows the official [p4lang/tutorials](https://github.com/p4lang/tutorials/blob/master/vm-ubuntu-24.04/README.md) documentation.

## Prerequisites

- macOS with Apple Silicon (M1/M2/M3/M4)
- At least 18GB RAM (6GB allocated to VM)
- At least 40-50GB free disk space
- Stable internet connection (install downloads ~2-3GB)

## Step 1: Install VirtualBox

Download VirtualBox 7.1.0+ from [virtualbox.org](https://www.virtualbox.org/wiki/Downloads). Apple Silicon support was added in version 7.1.0 — earlier versions will not work.

Install the DMG and grant all permissions macOS requests in System Settings → Privacy & Security.

## Step 2: Download Ubuntu Server ARM64 ISO

Download Ubuntu Server 24.04 LTS ARM64 from [ubuntu.com/download/server/arm](https://ubuntu.com/download/server/arm).

The file will be named something like `ubuntu-24.04.4-live-server-arm64.iso`. There is no Ubuntu Desktop ISO for ARM64, so we install Server and add the desktop GUI afterwards.

## Step 3: Create the Virtual Machine

Open VirtualBox and click **New**:

- **VM Name:** P4 Tutorial Ubuntu 24.04
- **ISO Image:** select the downloaded ARM64 ISO
- **Skip Unattended Installation:** checked (important — do manual install)
- **RAM:** 6144 MB (6GB). 4GB may cause the GUI to freeze.
- **CPUs:** 4
- **Disk:** 60GB with "Pre-allocate Full Size" unchecked
- **Use EFI:** checked (required for ARM64)

Before starting, go to **Settings**:
- General → Advanced → Shared Clipboard: Bidirectional
- Display → Screen → Video Memory: 32 MB

## Step 4: Install Ubuntu Server

Start the VM and walk through the Ubuntu Server installer:

- **Language/Keyboard:** defaults
- **Network:** accept defaults (VirtualBox handles networking automatically)
- **Proxy:** leave blank
- **Mirror:** accept default
- **Storage:** "Use an entire disk" with LVM. Important: expand the root partition to use the full disk (~56GB) instead of the default half allocation.
- **Profile:** username `p4`, password `p4` (matches p4lang tutorial convention)
- **Ubuntu Pro:** skip
- **SSH:** optionally install OpenSSH server
- **Featured snaps:** skip all

After installation, reboot. The "Failed unmounting cdrom" message is normal — just press Enter.

### Note on LVM default behavior

The installer defaults to allocating only ~50% of disk space to the root partition when LVM is selected. For a P4 dev environment, expand it to use the full available space. Select `ubuntu-lv` in the storage configuration screen and resize it before proceeding.

## Step 5: Install Desktop GUI

Ubuntu Server has no graphical interface by default. The P4 tutorials require one for tools like `xterm` and `wireshark`.

```bash
sudo apt update && sudo apt install -y ubuntu-desktop
sudo reboot
```

This installs the GNOME desktop environment. The first boot after installation may take 5-8 minutes.

### Performance note

GNOME can be heavy on VirtualBox ARM64. If the GUI freezes or is slow:
- Increase VM RAM to 6GB or more
- Consider installing Xfce as a lighter alternative: `sudo apt install -y xfce4 xfce4-goodies`
- Or work primarily through SSH from the host Mac terminal

## Step 6: Install P4 Development Tools

Open a terminal in the VM and run:

```bash
cd
git clone https://github.com/p4lang/tutorials
mkdir src
cd src
../tutorials/vm-ubuntu-24.04/install.sh |& tee log.txt
```

This compiles and installs from source: p4c (P4 compiler), BMv2 (behavioral model software switch), Mininet (network emulator), PI (P4Runtime), gRPC, Protobuf, Scapy, and Python dependencies.

**This takes several hours.** The tools are compiled from source because pre-built ARM64 binaries are not readily available, and all dependencies need to be version-matched for the specific OS and architecture.

The `|& tee log.txt` saves all output (stdout and stderr) to a log file for troubleshooting while still displaying progress on screen.

## Step 7: Verify Installation

```bash
p4c --version          # Expected: 1.2.5.10 or similar
simple_switch --version # Expected: 1.15.0 or similar
```

## Step 8: Test the Environment

```bash
cd ~/tutorials/exercises/basic
make run
```

This compiles `basic.p4`, starts Mininet with a pod-topo topology, and opens a Mininet prompt. Running `pingall` should show 100% packet loss — this is expected because the starter code drops all packets.

## Known Issues on Apple Silicon

- **VirtualBox Guest Additions:** clipboard sharing between host and VM may not work on ARM64. Guest Additions reports "unsupported arm64 machine type." Workaround: use SSH from host terminal for copy/paste.
- **GNOME performance:** the desktop can be slow or freeze on first boot. Increasing RAM to 6GB helps. Xfce is a lighter alternative.
- **Terminal bell:** the VM may produce annoying beep sounds. Disable via VirtualBox menu: Devices → Audio → disable audio output.

## Useful Commands

| Command | Purpose |
|---|---|
| `Ctrl+Alt+F3` | Switch to text terminal if GUI freezes |
| `Ctrl+Shift+C` / `Ctrl+Shift+V` | Copy/paste within Linux terminal |
| `Left ⌘ + F` | Toggle VirtualBox fullscreen |
| `sudo reboot` | Reboot the VM |
| `df -h /` | Check disk space |
| `ip addr \| grep inet` | Check VM IP address (for SSH) |
