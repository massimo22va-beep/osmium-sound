# Raspberry Pi 5 ARM64 port

## Status

Planning and bring-up scaffold only. This branch does not yet contain a bootable image, a validated Raspberry Pi 5 build, or hardware-tested audio/display support.

Target hardware for the first validation cycle: Raspberry Pi 5 (4 GB RAM), booting from microSD.

## Goals

- Add an ARM64 target while keeping the existing x86-64 target unchanged.
- Retain Osmium Sound's Lyrion Music Server-based playback architecture.
- Support a USB DAC through ALSA and an ARM64-compatible Squeezelite build.
- Offer both graphical/touchscreen and headless operation from the same installation.
- Allow the graphical mode to start when a supported display is connected, without making audio playback depend on the display being present.
- Keep the system suitable for appliance-style operation, while validating the Raspberry Pi boot/update model before adopting the existing x86-64 image layout.

## Initial platform direction

The first candidate is Debian 13 (Trixie) ARM64, using the Raspberry Pi 5 boot firmware, kernel, device-tree and graphics stack. The image-generation path will be evaluated against Raspberry Pi's `rpi-image-gen` project before any build profile is committed.

Do not assume that the current x86-64 `live-build` scripts, GRUB/UEFI installer, kernel package selection, or disk partitioning can be reused unchanged. In particular, the current x86-64 GRUB installation path is not a Raspberry Pi 5 boot path.

## Audio architecture

1. Lyrion Music Server provides the library/server and control-plane functionality.
2. Squeezelite (ARM64) acts as the local player/renderer.
3. Squeezelite outputs to the selected ALSA device, initially the USB DAC.
4. The default configuration must avoid software volume changes, equalisation, DSP, and sample-rate conversion unless explicitly enabled by the user or required by a documented compatibility fallback.
5. Validate PCM formats and rates against the actual DAC/ALSA device. Validate DoP/DSD separately; do not advertise native DSD or any maximum rate before hardware tests.

The first hardware audio target is a Denafrips Pontus 15th connected over USB. This is a test target, not a claim that every DAC behaves identically.

## Display and operating modes

The implementation must not depend on a specific display model.

- `headless`: no local graphical session; audio and network control services continue normally.
- `graphical`: start the local Qt/QML interface on the active DRM/KMS display.
- `auto`: select graphical mode when a supported display is available, otherwise remain headless.

Display discovery must not assume that the desired connector is always `/dev/dri/card0`. Raspberry Pi systems can expose multiple DRM devices, and device numbering can vary. Select the DRM device/connector by probing capabilities and connected outputs.

Touch input should use the Linux input stack (libinput/evdev as appropriate), with rotation and calibration handled as a coherent display/input configuration. HDMI displays with USB touch are the initial generic compatibility target; official DSI display overlays can be added after testing. No display purchase or model selection is required for the initial software work.

A display or Qt startup failure must never stop the audio services or prevent headless administration.

## Proposed implementation phases

### Phase 0 — source and build audit
- Map architecture assumptions in the current distro scripts, package lists, systemd units and Qt build.
- Identify x86-64-only packages, paths, boot assumptions and binaries.
- Confirm licensing and upstream contribution requirements before redistributing modified artifacts.

### Phase 1 — minimal Pi 5 ARM64 boot
- Build a minimal Debian Trixie ARM64 image using a documented Raspberry Pi 5-capable image-generation workflow.
- Boot from microSD on the target Pi 5.
- Verify Ethernet, USB, storage, temperature and clean shutdown/reboot.
- Record exact kernel, firmware, bootloader and image-generator versions.

### Phase 2 — headless audio player
- Install and configure Lyrion Music Server and ARM64 Squeezelite.
- Enumerate ALSA devices reliably; do not hardcode a card number.
- Add a persistent player identity and safe service ordering.
- Verify bit-perfect PCM output with loopback/diagnostic tools where possible and with DAC-reported format indicators where available.
- Test sample-rate switching, channel mapping, underruns, restart and USB reconnection.
- Test DoP/DSD independently before enabling it in the UI.

### Phase 3 — Osmium services and appliance behavior
- Port required backend services and package dependencies to ARM64.
- Separate persistent user data from the read-only operating-system area.
- Decide whether the existing A/B/RAUC update design can be adapted safely to the Pi boot chain; do not copy the x86 partition scheme without validation.
- Add recovery and update tests.

### Phase 4 — graphical/touchscreen mode
- Build the Qt 6/QML UI for ARM64 (native build or verified cross-build).
- Validate DRM/KMS and EGL integration on the Pi 5.
- Add dynamic DRM device/connector discovery and touch configuration.
- Test graphical, headless and auto modes on the same microSD installation.

### Phase 5 — release validation
- Test cold boot, repeated reboot, power recovery, long playback, library scanning, network loss and USB DAC reconnect.
- Publish image checksums, build instructions, known limitations and a tested hardware/software matrix.
- Do not label the image stable until the target hardware tests pass.

## Initial acceptance criteria

- Boots reliably on Raspberry Pi 5 4 GB from microSD.
- Headless mode is usable without a display attached.
- Lyrion control and Squeezelite playback work after reboot.
- USB DAC selection is stable across reboot and USB reconnection.
- PCM playback is verified without unintended DSP, volume adjustment or resampling.
- Graphical mode can be enabled later without reinstalling the system.
- A failed display/UI startup does not interrupt playback.

## Open technical questions

- Which `rpi-image-gen` release/profile is the best-maintained fit for the chosen Debian Trixie ARM64 base?
- Which kernel/firmware combination provides the most reliable Pi 5 USB audio and DRM/KMS behavior?
- Which Squeezelite package/build and options provide the required ALSA and DoP support on ARM64?
- Which parts of the current RAUC/A-B update and read-only-root design can be retained on Raspberry Pi boot media?
- Does the current Qt UI build cleanly for ARM64, and which Qt platform/EGL path is most reliable on Pi 5?

## Development rules

- Keep Raspberry Pi-specific changes isolated from the existing x86-64 build until the port is proven.
- Prefer feature detection over hardcoded device names and numbering.
- Document every hardware-tested claim with the tested model, software versions and procedure.
- Keep audio playback independent from optional display components.
- Treat this branch as experimental until the release validation phase is complete.
