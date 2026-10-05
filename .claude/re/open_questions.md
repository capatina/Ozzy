# Ploytec RE — Open Questions and Leads

## Open Questions

### Sample rate change: 5-call dance vs official driver
Our driver sends SET_CUR 5 times alternating 0x86/0x05 (from Windows USB capture). The official
macOS driver only calls `setFrequency` once per pipe (2 total: input + output). The 5-call
pattern may be a Windows driver quirk, or redundant. Need to test if 2 calls works for all devices.
Xone:4D (fw 1.4.1): 2 calls work at init and for live rate changes, as long as no PCM URB was
cancelled mid-packet just before (see SET_CUR in ploytec_vendor_requests.md).

### Junction encoder vs our bit-interleaved codec
`pcmTo24Junction6CH` does **channel reordering** (0→0, 3→1, 1→2, 4→3, 2→4, 5→5) and
extracts 3 bytes from 4-byte int samples. It does NOT do bit-level interleaving.
Our codec (`ploytec_codec.h`) does bit-interleaved encoding. Need to verify these produce
the same wire format, or if our codec is wrong.

### Index 0 bits 3-7 (0x08–0x80)
- For product 0x644: bits 3-4 (mask `0x18`) come from `this[0xd2]`, rest preserved with `& 0xE7`
- ESU uses fixed composite states (0x32, 0xB0, 0xB2) rather than individual flags
- Bits 0-1 now understood (USB1 mode and clock source), but upper bits still device-dependent

### PGNoiseGenerator anti-tamper
`onServiceStart` uses a PRNG seeded with VID/PID to verify expected values. This is a
license/anti-clone check. Not relevant to our driver but interesting.

### AJ subsystem
`AJ::configurationDone`, `AJ::sendCPort`, `AJ::getLevelInfo` — purpose unclear. May be
related to level metering or some control protocol. Has its own pipe assignment via `AJ::assignPipes`.

## Resolved (moved from questions)

- ~~Vendor request 'A'~~ — Now understood: ESU clock config (wIndex=0x101) and Wolfson codec (wIndex=0x102/0x106)
- ~~Index 2 digital output selector~~ — Write-only, `AjExtData[4] & 0xFFFF`, only for devices with `this[0x7AC]`
- ~~setEsuCpldByte~~ — Writes to 'I' wIndex=1 with `| 0xE7` mask
- ~~this[0x1888] meaning~~ — USB 2.0 High Speed flag, NOT bulk device flag
- ~~wValue high byte~~ — Sign extension artifact from `(short)(char)byte` cast
- ~~Xone:4D streams nothing on Linux (#75)~~ — Needs fw 1.4.1 plus CLEAR_FEATURE(ENDPOINT_HALT) on
  EP 0x86/0x05 after the handshake (Windows driver capture). It also needs the interrupt OUT packet
  encoded in full: 8 x 482-byte sub-packets, MIDI at 482k + 432. The old table covered 40 of 80
  frames and left 4 MIDI slots at 0x00, so the stream ran at ~40% speed and stalled after ~7 s.

## Leads

- **pcmTo24Junction8CH** — decompile to verify our 8-channel codec against the official encoder
- **USB::buildDummyDescriptor** — how the driver synthesizes USB audio descriptors for Ploytec devices
- **sendElektronChannelMap** — USB transactions for Elektron channel config (for Elektron support)
- **rtsProcessBulkOut** — the actual real-time audio processing loop (partially decompiled, shows MIDI embedding)
- **Real-time service functions**: `rtsBulkOutBulkIn`, `rtsInterruptOutInterruptIn` — the streaming state machines
- Windows driver RE for sample rate change pattern comparison (init captured on Xone:4D; a
  runtime rate change has not been captured yet)
