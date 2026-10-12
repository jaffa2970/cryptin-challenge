# Crypt-in extraction challenge

**Pull the identity seed out of a sealed Crypt-in dongle.** This repository holds the
firmware image for the challenge: a signed binary, its hashes, and the public key it is
signed with. **There is no source code here, and there will not be.**

Rules, submissions and the public list of claims:
**<https://cryptin.lake8.dev/security/extraction-challenge/>** — that page is the
authority. If it and this file ever disagree, the page wins.

---

# ⚠️ YOU WILL DESTROY THE CHIP. ⚠️

# THERE IS NO WAY BACK.

**The first boot of this image burns the chip's eFuses — permanently.** Secure Boot v2,
flash encryption in RELEASE mode, Secure Download Mode, JTAG off. From that moment:

- the chip runs **only firmware signed with the challenge key** — which you do not have;
- it **can never be reflashed** with anything that boots: not this, not the Crypt-in beta,
  not your own code, not Espressif's examples;
- the identity seed is generated **inside the chip** and **no command, no serial output
  and no recovery path ever returns it**. Nobody knows it — not you, not us;
- the dongle **cannot encrypt anything**: it has no licence and never will;
- **there is no `recover`, no 24 words, no escrow, no reset.**

**If you extract the seed, you win. If you don't, you own a paperweight.**
Use a board you are ready to throw away. Do not flash this on a dongle you use for
anything else — the Crypt-in beta included.

---

## Files (in the Release)

| file | what it is |
|---|---|
| `cryptin-CHALLENGE.factory.bin` | **the image you flash**: bootloader, partition table and application, written at `0x0` |
| `cryptin-CHALLENGE.bootloader.bin` | the signed bootloader, as it sits at `0x0` inside the image — for checking the signature, not for flashing |
| `cryptin-CHALLENGE.app.bin` | the signed application, as it sits at `0x20000` inside the image — same purpose |
| `k_challenge_signing_pub.pem` | the public half of the Secure Boot key that signs the image (RSA-3072) |
| `SHA256SUMS.txt` | the hashes — published as a separate file on purpose |

The image is **byte-for-byte the one we burned on our own reference chip**
(`14C19F200E48`, 11 October 2026). Every winning method is re-run on that chip before a
claim is confirmed (see the challenge page).

### Verify before you flash

```powershell
Select-String -Path .\SHA256SUMS.txt -Pattern '([0-9a-f]{64})\s+(.+)' | ForEach-Object {
  $h = $_.Matches[0].Groups[1].Value
  $f = $_.Matches[0].Groups[2].Value.Trim()
  $a = (Get-FileHash $f -Algorithm SHA256).Hash
  "{0}  {1}" -f $(if ($a -eq $h) { 'OK     ' } else { 'DIFFERS' }), $f
}
```

On Linux/macOS: `sha256sum -c SHA256SUMS.txt`. **Four `OK` lines.**

Then the signature, with Espressif's `espsecure` (it ships with `esptool`):

```
espsecure verify-signature --version 2 --keyfile k_challenge_signing_pub.pem cryptin-CHALLENGE.bootloader.bin
espsecure verify-signature --version 2 --keyfile k_challenge_signing_pub.pem cryptin-CHALLENGE.app.bin
```

Both must say `Signature block 0 verification successful`. The two parts are the same bytes
you find inside the factory image at `0x0` and `0x20000`.

---

## Hardware

- **ESP32-S3 with virgin eFuses.** A chip that has already had Secure Boot or flash
  encryption burned — by anyone, with any key — will not boot this image.
- **Flash ≥ 8 MB**, able to run QIO at 80 MHz. PSRAM is not needed.
- **Two USB ports**: one behind a USB-to-UART bridge (CH340/CH343/CP210x) for flashing,
  and the chip's **native USB** port, where the dongle appears as a HID device
  `303A:8123`. A board with only the native port is not covered by this procedure.
- Tested on an ESP32-S3 revision v0.2, N16R8-class board.

## Flashing — follow it exactly

Two traps, both of which leave you with a dead chip:

1. **Never use `--flash-size detect`** (esptool's default on some paths). It rewrites
   bytes of the bootloader header **after** the signature, and Secure Boot then refuses
   it. Always pass `keep` for size, mode and frequency.
2. **Never reset the chip during the first ~6 seconds of the first boot.** That is when it
   burns the eFuses and encrypts its own flash in place. A reset in that window — a
   button, a replug, or a serial monitor that toggles DTR/RTS when it opens the port —
   leaves a half-encrypted application that never boots again.

With `esptool` v5 on the **UART** port:

```
esptool --chip esp32s3 --port <UART_PORT> erase-flash
esptool --chip esp32s3 --port <UART_PORT> --after no-reset write-flash --flash-mode keep --flash-freq keep --flash-size keep 0x0 cryptin-CHALLENGE.factory.bin
```

Then **one** reset (the board's reset button, or one replug) and **only listen**. If you
open a serial monitor, open it with DTR and RTS held low, or don't open it at all. The
chip resets itself once by design, halfway through. It is done when the UART console
prints:

```
Ready (CHALLENGE image). HID challenge/response is live. The identity seed is never exported by any command or serial path.
```

and the native USB port enumerates the HID device `303A:8123`.

From now on `esptool` finds the chip "in Secure Download Mode", and `espefuse` can no
longer read the eFuses. That is expected. What still answers is
`esptool --chip esp32s3 --port <UART_PORT> get-security-info`: it must show
`Secure Boot: Enabled`, `Flash Encryption: Enabled`, `SPI_BOOT_CRYPT_CNT: 0x7` and
`JTAG: Permanently Disabled`. **Keep that output** — it is your evidence of the burn.

## Reading your public key `P`

`P` is the 32-byte Ed25519 public key of the seed you are trying to extract — it is what
you commit to. The command needs no PIN on this image (the PIN is unknowable, it derives
from the seed).

The protocol is public: <https://cryptin.lake8.dev/docs/protocol/>. In short: vendor HID,
**report ID 1**, 64-byte reports (report ID + 63 bytes).

```
send:     01 02 00 00 … (64 bytes, zero-padded)      # CMD_GET_PUBKEY
receive:  01 02 00 <32 bytes of P> 00 …               # opcode echoed, STATUS_OK, P
```

`CMD_CHALLENGE` (`0x03`) signs a 32-byte nonce with the same key, so `P` can be bound to a
live chip. Both are described on the protocol page.

## What the challenge allows

The Crypt-in threat model excludes laboratory attacks on the chip — side channels and
fault injection. **This challenge allows them anyway**, because we want to know where the
barrier really ends.

- A win **through software** — USB, HID, the ROM in Secure Download Mode, the console — is
  inside what the product claims to resist. It is a finding on the product, and the
  problem is ours.
- A win **through the lab** — glitching, power analysis, opening the package — reproduced
  on our reference chip, is confirmed on the challenge, and it **confirms** the limit we
  declare: the product never promised to hold against that attacker.

Notes for whoever starts:

- Flash encryption uses the ESP-IDF default, **XTS-AES-128**.
- **In scope: the dongle in your own hand, and nothing else.** Our servers,
  `license.lake8.dev` and the address `5.157.106.226` are **out of scope** — touching them
  is not part of this challenge and is not covered by its safe harbour.

## Submitting

Through the channel on the challenge page only: `security@lake8.dev`, encrypted to our PGP
key — <https://cryptin.lake8.dev/security/pgp/>. Commit `P`, with the
`get-security-info` output above, **before** you attack, then send the 32 bytes and the method. A claim
without a reproducible method stays open until it has one.

## Third-party software

The image contains open-source components (ESP-IDF, arduino-esp32, TinyUSB, Monocypher and
others). The list with versions and licences is the SBOM:
<https://cryptin.lake8.dev/docs/sbom/>.
