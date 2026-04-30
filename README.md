# Adafruit-PN532 — fork with Mifare Ultralight EV1 / NTAG21x secure command set

A fork of [adafruit/Adafruit-PN532](https://github.com/adafruit/Adafruit-PN532)
that adds native support for the Mifare Ultralight EV1 / NTAG21x security
features, which the upstream library does not expose. Everything else —
the base PN532 driver, the Mifare Classic helpers, the Ultralight
read/write helpers, the I2C / SPI / HSU transports and the existing
examples — is left untouched.

## Why this fork

The upstream library wraps every card command in `InDataExchange` (0x40),
which imposes ISO14443A error handling on the response. The EV1 / NTAG21x
secure commands have non-standard reply shapes — a 2-byte `PACK` on
`PWD_AUTH`, a 4-bit `ACK`/`NAK` on `INCR_CNT`, a 32-byte ECC signature on
`READ_SIG`, a 3-byte counter on `READ_CNT` — that `InDataExchange` cannot
deliver cleanly. NXP's recommended workaround on the PN532 is to dispatch
them via `InCommunicateThru` (0x42), a raw passthrough, and parse the
response frame on the host side.

This fork adds a small private helper (`_ev1RawCommand`) that does
exactly that, plus six public methods that build on it.

## What's added

All new methods live on the existing `Adafruit_PN532` class.

| Method | Card command | What it gives you |
|---|---|---|
| `mifareultralightev1_GetVersion(version[8])` | `GET_VERSION` (0x60) | Vendor / product type / subtype / major / minor / storage / protocol. Distinguishes EV1 sub-variants (MF0UL11 / MF0UL21) and NTAG213/215/216 from plain Ultralight. Useful as a pre-flight check before issuing EV1-only commands, since plain Ultralight returns a NAK on those. |
| `mifareultralightev1_PwdAuth(pwd[4], pack_out[2])` | `PWD_AUTH` (0x1B) | Unlocks pages protected by `AUTH0` / `ACCESS`. Returns the 2-byte `PACK` so the caller can verify the tag's authenticity (mutual auth), not just the auth status. |
| `mifareultralightev1_ReadSignature(signature[32])` | `READ_SIG` (0x3C) | 32-byte ECC signature over the UID, signed by NXP at manufacturing with NXP's private key. Verify it offline (ECDSA) against the corresponding NXP public key for anti-clone / originality checking. `READ_SIG` does not authenticate the tag by itself — it provides the material for the verification step. |
| `mifareultralightev1_ReadCounter(num, counter_out[3])` | `READ_CNT` (0x39) | Reads one of the three 24-bit one-way counters (single counter on NTAG21x). Counters can only be incremented, which makes them a building block for anti-replay schemes. |
| `mifareultralightev1_IncrementCounter(num, increment[3])` | `INCR_CNT` (0xA5) | Adds a 24-bit value to the selected counter. Non-reversible; saturates at 0xFFFFFF. The required RFU 4th payload byte is sent as 0x00 by the function. |
| `mifareultralightev1_CheckTearingEvent(num, flag_out)` | `CHECK_TEARING_EVENT` (0x3E) | Returns the per-counter tearing flag (0xBD = counter intact; any other value means an increment was interrupted by tag removal and the value cannot be trusted). |

A new `MIFARE_ULTRALIGHTEV1_CMD_*` group of constants is also added in
`Adafruit_PN532.h`, alongside the existing `MIFARE_CMD_*` group, so
callers that want to send their own raw `InCommunicateThru` commands can
reuse them.

Validated on an ESP32 + PN532 (SPI) bench against MF0UL11 / MF0UL21 tags.

## Quick example

```cpp
#include <Adafruit_PN532.h>

Adafruit_PN532 nfc(/* SS pin */ 5);

void setup() {
  Serial.begin(115200);
  nfc.begin();
  nfc.SAMConfig();

  uint8_t uid[7], uidLen;
  if (!nfc.readPassiveTargetID(PN532_MIFARE_ISO14443A, uid, &uidLen)) return;

  // Pre-flight: confirm we're talking to an EV1 / NTAG21x
  uint8_t version[8];
  if (!nfc.mifareultralightev1_GetVersion(version)) {
    Serial.println("Not an EV1 / NTAG21x");
    return;
  }

  // Originality check (offline ECDSA against NXP public key, not shown)
  uint8_t sig[32];
  nfc.mifareultralightev1_ReadSignature(sig);

  // Authenticate
  uint8_t pwd[4]  = { 0xFF, 0xFF, 0xFF, 0xFF }; // factory default
  uint8_t pack[2] = { 0 };
  if (nfc.mifareultralightev1_PwdAuth(pwd, pack)) {
    // Compare `pack` with the value you provisioned at personalisation.
  }

  // Anti-replay: read counter 0
  uint8_t cnt[3];
  if (nfc.mifareultralightev1_ReadCounter(0, cnt)) {
    uint32_t value = cnt[0] | (uint32_t)cnt[1] << 8 | (uint32_t)cnt[2] << 16;
    (void)value;
  }
}

void loop() {}
```

## Installation

Not published in the Arduino Library Manager. Install one of the
following ways:

* Clone into your sketchbook libraries folder:
  ```bash
  git clone https://github.com/tbringuier/Adafruit-PN532.git \
    "$HOME/Arduino/libraries/Adafruit-PN532"
  ```
* Or download a release zip from the
  [Releases](https://github.com/tbringuier/Adafruit-PN532/releases) page
  and use *Sketch → Include Library → Add .ZIP Library…* in the Arduino
  IDE.

Dependency: [Adafruit BusIO](https://github.com/adafruit/Adafruit_BusIO),
installable from the Arduino Library Manager.

## Versioning

This fork ships its own SemVer line, advertised through
`library.properties`:

* `1.4.0` — `mifareultralightev1_PwdAuth()` added.
* `1.5.0` — `GetVersion`, `ReadSignature`, `ReadCounter`,
  `IncrementCounter`, `CheckTearingEvent` added; `PwdAuth` refactored
  onto the shared `_ev1RawCommand` helper (no API change).

## License and credit

All pre-existing code is © Adafruit Industries, BSD-licensed (see
`license.txt`). The PN532 driver was originally written by Limor Fried /
Ladyada and Kevin Townsend for Adafruit. The additions in this fork
follow the same BSD license and the same Doxygen / clang-format
conventions as the upstream library.

## Contributing

Contributions are welcome. Please:

* Document new public methods with Doxygen, matching the style of the
  surrounding code.
* Run `clang-format -i *.cpp *.h` before submitting — the CI rejects
  unformatted diffs.

See the upstream
[Code of Conduct](https://github.com/adafruit/Adafruit-PN532/blob/master/CODE_OF_CONDUCT.md).
