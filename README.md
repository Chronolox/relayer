<div align="center">

<a href="https://chronolox.com">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/chronolox-logo-dark.png">
    <img src="assets/chronolox-logo-light.png" alt="Chronolox — hourglass emblem and wordmark" width="360">
  </picture>
</a>

</div>

# Chronolox Relayer — Service & Appliance Distributions

This repository hosts official, pre-compiled binary distributions, container specifications, appliance ISO releases, release signatures, and integrity verification materials for **Chronolox Relayer** (`chronolox-relayer`).

Chronolox Relayer is the backend relay and transaction submission daemon: gasless transaction relaying, prepaid voucher redemption, autonomous journal records, and on-chain timestamp notary services.

Pre-built binaries and appliance ISO distributions are available on the [Releases](https://github.com/Chronolox/relayer/releases) page.

---

## Capabilities & Architecture

- **Gasless Transaction Relay:** Facilitate gasless protocol interactions for recipients and creators using multi-chain gas reserve pools.
- **Voucher Redemption:** Settle prepaid voucher tokens on-chain against protocol fee schedules.
- **Secret Journal Vault:** Monotonically record on-chain capsule metadata and notarization anchors.
- **Standalone Daemon & Appliance ISO:** Ships as both a standalone Linux `amd64` service binary and a hardened bootable appliance ISO.

---

## Release Verification Keys

All official binaries are cryptographically signed before release. Never execute binaries whose SHA-256 digests or GPG signatures do not match the published records.

### Public Signing Keys

- **Release Signing GPG Key ID:** `0xF873EA8F770A9136`
- **Key Fingerprint:** `CE23 439F 2EA0 9A0C 445A  F720 F873 EA8F 770A 9136`
- **User ID:** `Chronolox APT Signing Key <apt@chronolox.com>`
- **Public Key File:** Included in repository root as [`chronolox-release-key.pub`](./chronolox-release-key.pub) — always verify the fingerprint above after importing; trust the fingerprint, not the transport.

Import the key and verify a release:

```bash
gpg --import chronolox-release-key.pub
gpg --fingerprint 0xF873EA8F770A9136          # must match the fingerprint above
gpg --verify SHA256SUMS.sig SHA256SUMS        # against a release's checksum ledger
sha256sum --check SHA256SUMS --ignore-missing
```

---

## Integrity & Authenticity Verification

Each release on GitHub includes:

1. The service binary archive (`chronolox-relayer-<network>-linux-amd64.tar.gz`)
2. `SHA256SUMS` (Plaintext hash ledger)
3. `SHA256SUMS.sig` (Detached GPG signature of the hash ledger)

### 1. Verify Checksum Ledger Signature

Verify that the checksum ledger itself has not been tampered with:

```bash
gpg --verify SHA256SUMS.sig SHA256SUMS
```

*Expected output:* `gpg: Good signature from "Chronolox APT Signing Key <apt@chronolox.com>"`

### 2. Verify Binary Integrity

Run the checksum check against your downloaded archive:

```bash
sha256sum --check SHA256SUMS --ignore-missing
```

---

## Deployment & Running Notes

### Linux (`amd64` Service Binary)

Chronolox Relayer targets dedicated Linux x86_64 server environments.

```bash
tar -xzf chronolox-relayer-<network>-linux-amd64.tar.gz
chmod +x chronolox-relayer-amd64
./chronolox-relayer-amd64 --help
```

> **Systemd Integration:** Operators typically run the service under a dedicated system user (e.g., `relayer`) managed by systemd with environment configuration loaded from a secured configuration file.

### Appliance ISO Image

Official releases also publish hardened bootable ISO images for bare-metal servers or hypervisors (Proxmox, VMware ESXi, KVM), providing a zero-touch pre-configured relayer appliance.

For complete technical specifications on relayer node operations and protocol mechanics, consult the [Chronolox Architecture & Security Model](https://github.com/Chronolox/specs).

---

## Security & Bug Reporting

If you observe a checksum mismatch, corrupted binary artifact, or unexpected network behavior:

* **Do NOT file a public issue or pull request for security vulnerabilities.**
* Review our disclosure procedure in [`SECURITY.md`](./SECURITY.md).
* Contact our engineering security desk directly at **`security@chronolox.com`**.

---

## Contributing

To report packaging bugs, operating system compatibility issues, or hash verification errors, please see [`CONTRIBUTING.md`](./CONTRIBUTING.md).

---

## License

The binary releases and verification materials in this repository are distributed under the [Chronolox Binary Release License](./LICENSE).

Source code for the underlying applications is protected during commercial beta by the on-chain Cryptographic Source Escrow Failsafe documented in [`specs`](https://github.com/Chronolox/specs).
