# UTM (Apple Silicon) — x86_64 Ubuntu VM for flashing

This guide shows how to run an emulated x86_64 Ubuntu VM using UTM on Apple Silicon, install the MRS toolchain inside the VM, and flash the CH32x035 hardware.

**Overview**

- Emulate an x86_64 (amd64) Ubuntu VM with UTM (QEMU backend). Emulation is slower than native virtualization but runs the x86_64 MRS toolchain reliably on Apple Silicon.
- Use USB passthrough in UTM to give the VM access to your WCH debug probe so `openocd` can flash the board.

**Prerequisites (host macOS)**

- UTM app (recommended): install from https://mac.getutm.app or via Homebrew:

```bash
brew install --cask utm
```
- ~30 GB free disk space for the VM image and toolchain
- A downloaded Ubuntu amd64 ISO (instructions below)
- USB cable and probe plugged into the Mac when adding USB passthrough

**Download Ubuntu (amd64 ISO)**

Prefer Ubuntu 22.04 LTS (amd64). Example:

```bash
curl -LO https://releases.ubuntu.com/22.04/ubuntu-22.04.5-desktop-amd64.iso
```

You can use 24.04 as well; the instructions are the same.

**Create the VM in UTM (GUI steps)**

1. Open UTM → Click `Create a New Virtual Machine` → choose `Emulate` (important — this emulates x86_64 on Apple Silicon).
2. OS: Linux; Architecture: `x86_64` (amd64).
3. Drives: create a new disk — recommended size: 30 GB (or more).
4. CPU & Memory: 2 CPU cores (or 4 if you have spare cores), 4096 MB RAM (increase if you can).
5. Boot ISO: attach the Ubuntu ISO you downloaded as the CD image.
6. (Optional) Network: leave default NAT. If you want SSH from host → VM, enable port forwarding for guest port 22 to a host port (e.g. host 2222 → guest 22) in the NAT options.
7. USB: with the VM powered off, open the VM settings → `USB` (or `Devices`) → `+` → choose your WCH probe from the host device list. If the device does not appear, plug the probe into the Mac and re-open the dialog.
8. Save the VM and Start it. Install Ubuntu as normal from the ISO.

**Inside the VM — basic environment and MRS toolchain install**

After Ubuntu is installed and you have a working account, open a terminal in the VM and run:

```bash
sudo apt update
sudo apt install -y wget tar build-essential libusb-1.0-0 libncurses5 libtinfo5 libhidapi-dev udev minicom rsync usbutils

# Download & extract the MRS toolchain (adjust URL if needed)
MRS_URL="https://github.com/ch32-riscv-ug/MounRiver_Studio_Community_miror/releases/download/1.92-toolchain/MRS_Toolchain_Linux_x64_V1.92.tar.xz"
sudo mkdir -p /opt
sudo wget -O /tmp/mrs.tar.xz "$MRS_URL"
sudo tar -xf /tmp/mrs.tar.xz -C /opt
sudo chown -R $(whoami):$(whoami) /opt/MRS_Toolchain_Linux_x64_V1.92

# Run the bundled helper to install udev rules and libraries (this writes to /usr/lib and /etc/udev)
cd /opt/MRS_Toolchain_Linux_x64_V1.92/beforeinstall
sudo chmod +x start.sh || true
sudo ./start.sh
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Notes:
- The `start.sh` that comes with the MRS package copies libraries into `/usr/lib` and installs udev rules. Running it inside the VM is safe and required so OpenOCD can access the probe.
- If the script fails, inspect the `beforeinstall` directory for the exact library files and udev rules (they may already be present in the MRS package).

**Bring your project into the VM**

Options:
- Recommended: push your repo to a remote (GitHub) from the host and `git clone` or `git pull` inside the VM.
- Alternately, enable SSH port forwarding in the VM NAT settings and `scp`/`rsync` from host→VM, or use a temporary file share (less reliable with emulation).

Example (quick clone):

```bash
# inside VM
git clone https://github.com/yourname/your-repo.git ~/project
cd ~/project
make
```

**Flash the board**

With the probe passed into the VM and MRS toolchain installed, run inside the VM:

```bash
# build
make
# flash (OpenOCD in the MRS toolchain expects sudo / elevated access in many flows)
sudo make flash
```

If OpenOCD cannot see the probe, check:

- `lsusb` (install `usbutils` in the VM): `lsusb` should list the WCH probe
- `dmesg` for USB messages
- UTM USB settings (VM must be powered off to add/remove host USB devices)

**Workflow tips**

- Emulation is slower — increase CPUs/RAM if the host can spare them for faster builds.
- Edit on the macOS host and push to a remote (GitHub) or use `scp`/`rsync` to copy into the VM before building — this keeps your host files as the canonical source.
- If you plan repeated local transfers without a remote, set up an SSH server inside the VM and use port forwarding on the VM network to allow `scp` from host.

**Troubleshooting**

- Device not visible in VM: stop the VM, open UTM VM settings → USB → add the device, then start VM.
- udev rules not applied: re-run `sudo udevadm control --reload-rules && sudo udevadm trigger` inside the VM.
- OpenOCD errors: run OpenOCD from the MRS path to see vendor messages, e.g.:

```bash
/opt/MRS_Toolchain_Linux_x64_V1.92/OpenOCD/bin/openocd -f /opt/MRS_Toolchain_Linux_x64_V1.92/OpenOCD/bin/wch-riscv.cfg
```

**Alternatives**

- If UTM passthrough proves unreliable, use an Intel/Linux machine (physical or cloud) for flashing — it's faster and more stable.
- For builds only, Docker on macOS is fine (no USB passthrough) — use Docker on a Linux host for flashing.

**Files added by this repository**

- See [tools/vm/Vagrantfile](tools/vm/Vagrantfile) and [tools/vm/README.md](tools/vm/README.md) for the previously-added VM helper, which is only suitable for an x86_64 VirtualBox host.

test commit