# Crypto Clock V3.1 — OTA Release Feed

This folder is the Over-The-Air (OTA) update delivery channel for the
**Crypto Clock V3.1** device (ESP32-S3 + BT817 EVE4 display controller).

All files are **AES-256-GCM encrypted**. They are only useful to a device
that holds the matching decryption key. Downloading them directly gives you
unreadable ciphertext.

The firmware source code lives in a separate private repository.

---

## Files

| File | Description |
|------|-------------|
| `update.json` | Release manifest — version strings, file sizes, SHA-256 hashes, GCM IVs and authentication tags |
| `firmware.bin.enc` | Encrypted ESP32 application firmware (flashed via `Update.h`, then device reboots) |
| `blob_part_0.enc` | Encrypted BT817 external-flash asset blob, part 0 |
| `blob_part_N.enc` | Additional blob parts when the blob exceeds 6 MB (PSRAM limit per part) |

---

## `update.json` format

```json
{
  "version": "3.1.1",
  "blob_version": 37,
  "fw": {
    "enc_size": 1302456,
    "sha256": "abc123...",
    "iv":  "<base64 12-byte GCM nonce>",
    "tag": "<base64 16-byte GCM auth tag>"
  },
  "blob": [
    {
      "enc_size": 6221824,
      "sha256": "def456...",
      "iv":  "<base64 12-byte GCM nonce>",
      "tag": "<base64 16-byte GCM auth tag>"
    }
  ]
}
```

- **`version`** — compared against `FW_VERSION` in the running firmware. A mismatch triggers an ESP32 firmware update.
- **`blob_version`** — compared against the value stored in ESP32 NVS (`phenix/blobVer`). A mismatch triggers a BT817 flash update.
- **`fw`** — metadata for `firmware.bin.enc`. Omit or set `enc_size: 0` to skip firmware update.
- **`blob`** — array of parts for the display firmware blob. Each part must fit within the device PSRAM limit (6 MB).

---

## Update flow

```
Device boot / midnight check
        │
        ▼
  Fetch update.json
        │
  ┌─────┴──────┐
  │            │
fw newer?   blob newer?
  │            │
  ▼            ▼
Download    Download
& decrypt   & decrypt
  │         part 0..N
  ▼            │
Update.h    BT817 CMD_FLASHWRITE
  │            │
Reboot      Reboot
```

1. The device fetches `update.json` over HTTPS (certificate verification: `setInsecure()` — public repo, blobs are authenticated by GCM tag).
2. Each file is downloaded into PSRAM, its SHA-256 hash is verified, then AES-256-GCM decryption is performed. The GCM authentication tag acts as a second layer of integrity and authenticity verification.
3. A corrupt or tampered file is rejected **before** any flash erase occurs.
4. Blob updates erase the full BT817 external flash on the first part only; subsequent parts are appended.

---

## Releasing a new version

Files are generated automatically by `scripts/release.py` on every PlatformIO
build and pushed to this repository by the GitHub Actions workflow
(`.github/workflows/ota-release.yml`) whenever the commit message contains
`#Release`.

To release manually:

```bash
pip install cryptography   # one-time
python scripts/release.py  # run from the firmware source root
git add firmware/
git commit -m "ota: release 3.1.1"
git push
```

---

## Encryption details

| Parameter | Value |
|-----------|-------|
| Algorithm | AES-256-GCM |
| Key size  | 256 bit (32 bytes) |
| Nonce     | 96 bit (12 bytes), random per file |
| Tag       | 128 bit (16 bytes) |
| Key source | `FW_AES_KEY_B64` in `include/secrets.h` (base64-encoded, never committed) |

---

## Licence

MIT — see repository root for the full licence text.
The encrypted binaries contain proprietary compiled code; this licence covers
only the release manifest format and tooling scripts.
