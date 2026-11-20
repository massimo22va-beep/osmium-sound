# Secure first-boot provisioning design

Status: design only. This document does not enable SSH or create accounts in the current image profile.

## Goals

The first test image must be manageable headlessly without embedding shared credentials in the public repository. The build should remain reproducible while each owner supplies their own account details and SSH public key locally.

## Upstream building blocks (rpi-image-gen v2.8.0)

The pinned upstream release provides these layers:

- `device-user-credentials`: declares the first account name, password/hash, supplementary groups and sudo policy.
- `device-user-admin`: creates the account and applies the credential and sudo settings; it requires `device-user-credentials`.
- `openssh-server`: installs OpenSSH and can install the first user's public key. It requires `device-user-admin`. With `ssh.pubkey_only: y`, SSH password authentication is disabled and public-key authentication is required.

These layer definitions were inspected in the pinned upstream v2.8.0 source. They are preferred over writing a bespoke account-creation script at this stage.

## Intended configuration shape

The Osmium profile will eventually include the upstream account and SSH layers, conceptually:

```yaml
layer:
  base: trixie-minbase
  account: device-user-admin
  ssh: openssh-server
```

The local provisioning values will supply:

- account name selected for this build: `osmium`;
- the path to that user's **public** SSH key (for example, an existing `.pub` file);
- public-key-only SSH authentication;
- a password hash for sudo authentication, generated locally with the upstream `genpasswd` helper.

No password, password hash, public key, Wi-Fi passphrase, or private key will be committed to the public repository. The private SSH key must never be copied to the Raspberry Pi image or to GitHub.

## Sudo policy

For a practical headless administration setup, the account should belong to the sudo group and sudo should require the account password. The password hash is therefore a local build input, not a value stored in the tracked YAML.

Do not use passwordless sudo as a shortcut for the first image. Do not set a shared default password. If a password hash is not supplied, the upstream account layer may leave the account locked or without sudo access, depending on the selected settings.

## Network scope for the first test

Use wired Ethernet first. Do not place home Wi-Fi credentials in the public build profile. Wi-Fi provisioning can be added later through a separate local-only configuration input once wired boot and remote access are verified.

## Implementation gates

Before adding these layers to `osmium-rpi5.yaml`:

1. Confirm the exact v2.8.0 variable names and layer requirements against the pinned source.
2. Account name selected: `osmium`. Generate a local password hash for sudo authentication.
3. Confirm the public SSH key file exists on the build host and is readable by the build process.
4. Add a local, untracked provisioning input or pass values through a secure build-time mechanism; do not put credentials in tracked files or shell history.
5. Build and inspect the image, then test first boot on a separate microSD.
6. Verify key-based SSH login, reject password-based SSH login, and verify sudo prompts for the account password.

The selected administrator username for this project is `osmium`. The current image profile remains unchanged until the SSH public key and local password-hash inputs are ready and the configuration has been validated.
