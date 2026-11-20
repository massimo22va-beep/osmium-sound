# Raspberry Pi 5 porting audit

Audit basis: the current `main` branch of Osmium Sound, inspected while preparing the ARM64 target. This is a source-level audit; no Raspberry Pi image has been built or boot-tested by this work.

## Findings

### 1. Distribution build is PC-specific

- `distro/build-distro.sh` defaults `ARCH=amd64`.
- It invokes Debian `live-build` with the `syslinux,grub-efi` bootloader set.
- It injects PC installer hooks and an EFI repair hook.
- The build script seeds a live ISO and then optionally hands the chroot to the A/B image post-processor.

Conclusion: do not reuse this script as the Pi 5 image entry point. Keep it as the x86-64 build.

### 2. Package manifest contains x86-only boot and diagnostic packages

`distro/config/package-lists/hifi.list.chroot` includes `linux-image-amd64`, `grub-efi-amd64-signed`, `shim-signed`, `grub-pc-bin`, `efibootmgr`, `intel-gpu-tools`, and `radeontop`, alongside many architecture-neutral services.

Conclusion: create a separately curated ARM64 package set. Do not copy the complete PC manifest and merely remove the kernel line; bootloader, GPU diagnostics and display dependencies need deliberate selection.

### 3. Disk installer and boot finalisation are not Pi boot flows

`distro/config/includes.chroot/usr/local/sbin/hifi-disk-install.sh` describes and implements PC disk layouts with BIOS boot and EFI System partitions, GRUB installation and an A/B slot layout. The image post-processor also manages GRUB state on an EFI partition.

Conclusion: Raspberry Pi firmware boot, FAT boot partition contents, kernel/device-tree selection, root partition discovery and recovery must be designed and tested independently. Do not invoke the existing PC disk installer on the Pi.

### 4. Audio components are a promising reuse boundary

The PC manifest already installs `alsa-utils` and `squeezelite`; the service hook enables Squeezelite and Lyrion. The existing backend includes `hifi_squeezelite.py` and Lyrion integration.

Conclusion: reuse the application-level audio configuration and control logic where architecture-neutral, but create ARM64-specific packaging, service defaults and ALSA-device discovery. The initial Pi profile installs ALSA diagnostics and Squeezelite only; it does not yet install or configure Lyrion.

### 5. Qt application is not a simple copy of the PC display stack

- `native-ui-qt/Makefile` builds with Qt 6 Quick/QML, Qt DBus and libdrm.
- `native-ui-qt/src/main.cpp` defaults `QT_QPA_PLATFORM` to `eglfs` and configures the display before creating the Qt application.
- The existing Qt code has device-specific display selection logic; the Pi port must validate DRM card and connector discovery on real Pi 5 hardware rather than assuming a stable `/dev/dri/card0`.

Conclusion: first establish a headless boot and audio baseline. Then validate Qt/EGLFS/DRM on Pi 5 and adapt display selection without making the audio services depend on the GUI.

## ARM64 implementation boundary

| Area | Strategy |
|---|---|
| Image generation and Pi boot | Raspberry Pi `rpi-image-gen` profile |
| Debian base | Trixie ARM64 |
| Kernel, firmware, device tree | Pi 5 device layer and upstream supported boot stack |
| Audio | ARM64 Squeezelite + ALSA; validate PCM and DoP separately |
| Lyrion | Integrate after minimal image and network access are validated |
| Osmium Python services | Reuse after dependency and architecture audit |
| Qt UI | ARM64 build, then Pi 5 DRM/EGLFS and touchscreen validation |
| Read-only root and A/B updates | Defer until boot/recovery and persistent-data layout are proven |

## Current status

The initial profile in `osmium-rpi5.yaml` follows the upstream rpi-image-gen Trixie-minbase example and selects the documented `rpi5` device layer. It is only a build scaffold. The profile still needs a secure provisioning strategy, a tested image-generator revision, a successful build, and hardware validation.

No bootable image, audio bit-perfect result, DoP result, display result, or OTA result is claimed at this stage.
