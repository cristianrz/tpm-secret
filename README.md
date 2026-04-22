# tpm-secret

Store secrets in a LUKS volume unlocked by your TPM2 chip.

The secret can only be retrieved on **this specific machine** with **Secure Boot
intact** and the correct **PIN**. Unlike password managers or encrypted files,
there is no master password to steal — the hardware is the key.

Useful for:
- Unlocking LUKS disks without typing a passphrase on boot
- Storing SSH key passphrases that auto-fill only on trusted hardware
- Keeping API keys and tokens on a homelab server without storing them in plaintext

## How it works

`tpm-secret` creates a 32MB LUKS container at `/var/secret.enc`. The container
is enrolled with:

- **TPM2 + PCR7** — binds the secret to this machine and Secure Boot state. If
  the firmware or boot chain changes, the TPM will refuse to unlock.
- **PIN** — a second factor so physical access alone is not enough.
- **Recovery key** — a backup printed once at setup time. Store it somewhere safe.

The original password used during LUKS format is wiped immediately, leaving only
the TPM slot and the recovery key.

## Dependencies

- `cryptsetup`
- `systemd-cryptenroll` (systemd ≥ 248)
- A TPM2 chip with Secure Boot enabled

## Installation

```sh
git clone https://github.com/cristianrz/tpm-secret.git
cd tpm-secret
sudo cp tpm-secret-* /usr/local/bin/
```

## Usage

### 1. Set up the secret store (once)

```sh
sudo tpm-secret-make
```

This creates `/var/secret.enc`, enrolls your TPM and PIN, and prints a recovery
key. Store the recovery key somewhere safe — it is the only way to access the
secret if the TPM seal breaks (e.g. after a firmware update).

### 2. Store a secret

```sh
echo "my-api-key" | sudo tpm-secret-set
# or interactively:
sudo tpm-secret-set
```

### 3. Retrieve the secret

`tpm-secret-get` refuses to output to a terminal — it is designed to be piped
directly into another program to prevent accidental exposure in logs or on screen.

```sh
sudo tpm-secret-get | your-program --password-stdin

# Example: unlock an SSH key
sudo tpm-secret-get | ssh-add -
```

You can add it to sudoers so it can be called without a password:

```
your-username ALL=(ALL) NOPASSWD: /usr/local/bin/tpm-secret-get
```

## Security notes

- The secret is hardware-bound: it will not unlock on a different machine or
  if Secure Boot is disabled or the boot chain is modified.
- If a firmware update breaks the TPM seal, use the recovery key to regain
  access and re-enroll with `tpm-secret-make`.
- `tpm-secret-get` requires root by default. Scope sudoers access carefully.

## License

MIT
