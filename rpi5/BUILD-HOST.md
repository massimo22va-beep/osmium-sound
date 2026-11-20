# Raspberry Pi 5 build host and first image procedure

This guide prepares a **native ARM64** build environment for the Osmium Sound Raspberry Pi 5 bring-up profile. It does not claim that the image has already been built or tested.

## 1. Recommended build host

Use the Raspberry Pi 5 (4 GB) with an up-to-date **64-bit Raspberry Pi OS** installation. The upstream rpi-image-gen project documents Raspberry Pi OS as its preferred host and Debian Bookworm/Trixie ARM64 as supported native environments.

Do not use the existing Windows 11 + WSL2 x86-64 installation as the first build environment. Upstream notes that amd64/QEMU and containers can work in some configurations, but are not formally supported. The build uses private mount namespaces and needs `CAP_SYS_ADMIN` or an equivalent privileged environment.

For the first pass, keep the build host separate from the microSD card that will receive the Osmium test image.

## 2. Prepare storage

A clean Raspberry Pi OS 64-bit installation is recommended. Complete its normal first-boot setup and update it:

```sh
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

After reboot, attach a separate USB 3 SSD if available and use it for source checkouts and build workspace. This avoids filling the system microSD with package caches and image build output.

Inspect storage before mounting anything:

```sh
lsblk -o NAME,SIZE,MODEL,TRAN,FSTYPE,MOUNTPOINTS
df -h
```

Create and mount a workspace on the SSD using its actual partition and filesystem. Do not copy a guessed device name from an example: identify it with `lsblk` first. Keep the target Osmium microSD disconnected during host setup and image generation.

## 3. Install the pinned image generator

The initial project profile is pinned to upstream rpi-image-gen **v2.8.0**, commit:

```
262d4df5a9f9d4133370465399a7958a7c22cdc7
```

Clone that exact release:

```sh
git clone --branch v2.8.0 --depth 1 https://github.com/raspberrypi/rpi-image-gen.git
cd rpi-image-gen
sudo ./install_deps.sh
```

Confirm the checkout:

```sh
git describe --tags --exact-match
git rev-parse HEAD
```

The first command should report `v2.8.0`; the second should report the commit above. If either differs, stop and resolve the version mismatch before building.

## 4. Build the Osmium profile

Clone the user's fork and select the porting branch:

```sh
git clone --branch rpi5-arm64 https://github.com/massimo22va-beep/osmium-sound.git
```

From the rpi-image-gen checkout, run the build using absolute paths. Replace `/path/to` with the real location of the checkout:

```sh
./rpi-image-gen build \
  -S /path/to/osmium-sound/rpi5 \
  -c /path/to/osmium-sound/rpi5/osmium-rpi5.yaml
```

The output location is reported by the build tool. Do not assume a filename or mark the build successful until the command exits successfully and the resulting image file can be inspected.

The initial profile is intentionally small: Raspberry Pi 5 boot layer, Debian Trixie ARM64 base, and basic ALSA/Squeezelite diagnostic packages. It is not yet a complete player.

## 5. Flashing safety

Use Raspberry Pi Imager's **Use custom** option to write the resulting image to a separate test microSD card. Before writing, check the selected target's model and capacity in the Imager UI.

If using a command-line imaging tool instead, first identify the exact removable device with:

```sh
lsblk -o NAME,SIZE,MODEL,TRAN,FSTYPE,MOUNTPOINTS
```

Unmount only the partitions belonging to the verified target card. Never use a command copied from an example with an assumed `/dev/mmcblk0`, `/dev/sdX` or `/dev/nvme...` path. A wrong target can overwrite the build host's operating system or data SSD.

Keep the known-good Raspberry Pi OS card untouched so the Pi can be returned to a working state.

## 6. First-boot security and validation

The current profile does **not** define a user password, SSH key, or first-boot provisioning workflow. Do not expose the test image to an untrusted network or assume remote login is available. The next engineering change must add secure provisioning without baking a shared password or private SSH key into the public repository.

On the first successful boot, record at least:

```sh
uname -a
cat /etc/os-release
cat /proc/device-tree/model
lsusb
aplay -l
aplay -L
```

Then validate Ethernet, USB enumeration and ALSA device visibility before adding Lyrion, service automation or the Qt display stack.

## 7. Scope deliberately deferred

- Lyrion Music Server integration and library configuration
- Squeezelite service tuning and stable USB DAC selection
- PCM bit-perfect verification and separate DoP testing
- Qt 6/QML, DRM/KMS/EGLFS and touchscreen support
- Graphical/headless mode switching
- Read-only root filesystem, RAUC and A/B updates

These are separate milestones. No bit-perfect, DoP, touchscreen, update or recovery behaviour is claimed until tested on the actual Raspberry Pi 5 and USB DAC.
