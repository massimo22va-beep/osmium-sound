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

Use a native Debian Trixie ARM64 build host where possible. Upstream documents amd64 cross-building with QEMU as possible but not formally supported; native ARM64 is the preferred first validation environment.

Example workflow (after installing the prerequisites described by the upstream project):

```sh
git clone https://github.com/raspberrypi/rpi-image-gen.git
cd rpi-image-gen
sudo ./install_deps.sh
cd -
rpi-image-gen build -S ./rpi5 -c ./rpi5/osmium-rpi5.yaml
```

Run the build command from the rpi-image-gen checkout, replacing the final source path with the absolute path to this repository's `rpi5/` directory if necessary.

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
