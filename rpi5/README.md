# Raspberry Pi 5 build scaffold

This directory is the isolated ARM64 build entry point. It does not replace or modify the existing x86-64 `distro/` build.

## Target

- Raspberry Pi 5, initially 4 GB RAM
- Debian 13 (Trixie), ARM64
- microSD boot media
- Headless-first bring-up; the graphical Qt/touchscreen session is a later phase
- Lyrion Music Server plus local Squeezelite renderer and USB DAC are planned, but not yet integrated into this base image

## Build system

The candidate image generator is Raspberry Pi's official [rpi-image-gen](https://github.com/raspberrypi/rpi-image-gen). The configuration is deliberately based on its upstream Trixie-minbase example and the documented `rpi5` device layer. This keeps the Raspberry Pi firmware, kernel and boot layout under the Raspberry Pi-specific image-generation workflow rather than trying to reuse the PC installer.

Pin the image generator to the upstream **v2.8.0** release (commit `262d4df5a9f9d4133370465399a7958a7c22cdc7`) for this initial port. Do not silently build against a moving `master` branch. Use a native Debian Bookworm/Trixie ARM64 build host; upstream specifically recommends an up-to-date 64-bit Raspberry Pi OS host. amd64/QEMU and container builds may work but are not the formally supported path.

Example workflow (after installing the prerequisites described by the upstream project):

```sh
git clone --branch v2.8.0 --depth 1 https://github.com/raspberrypi/rpi-image-gen.git
cd rpi-image-gen
sudo ./install_deps.sh
./rpi-image-gen build -S /absolute/path/to/osmium-sound/rpi5 -c /absolute/path/to/osmium-sound/rpi5/osmium-rpi5.yaml
```

Run the build command from the rpi-image-gen checkout. Full host preparation, output handling and safe SD-card flashing notes are in [BUILD-HOST.md](BUILD-HOST.md). The secure account and SSH provisioning approach is documented in [PROVISIONING-DESIGN.md](PROVISIONING-DESIGN.md); it is a design only and is not enabled in the image yet.

## Current profile scope

The YAML profile selects the upstream `rpi5` device layer, the upstream `image-rpios` bootable image layout, and the `trixie-minbase` suite. It adds only the initial ALSA/Squeezelite diagnostic tools.

This is a **bring-up profile**, not an Osmium release image. It does not yet configure an account, remote administration, Lyrion, audio service policy, the Qt UI, read-only root, A/B updates, or touchscreen support. Do not flash it as a finished player image.

## Next engineering steps

1. Validate the configuration against the exact pinned rpi-image-gen revision and build a minimal image.
2. Boot on the Raspberry Pi 5 4 GB and record firmware, kernel, device-tree and package versions.
3. Validate Ethernet, USB enumeration and ALSA device discovery.
4. Add secure first-boot provisioning and remote administration.
5. Integrate Lyrion and Squeezelite with a persistent, stable ALSA device selection.
6. Verify PCM output and DoP separately with the target USB DAC.
7. Port the Qt 6/QML application and add display-independent graphical/headless mode selection.
8. Revisit read-only root and A/B updates only after the Pi boot and recovery model is understood.

## Important separation

The existing PC build assumes amd64 packages, GRUB EFI, PC-style disk partitioning and x86 bootloader installation. Those assumptions must not leak into this target. Keep Raspberry Pi boot, firmware, kernel and partition configuration in this directory and in the rpi-image-gen workflow.
