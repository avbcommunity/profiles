**Version: 1.0-draft**

# Extended Audio Profile

This AVB community profile specifies the extended requirements for audio streaming compatibility. All requirements from the [Basic Audio Profile](audio_basic.md) apply, with the following additions.

---

## Stream Format

All formats required by the Basic Audio Profile remain required. In addition, talkers and listeners must support the following sample rates:

| Parameter | Value |
|-----------|-------|
| Format | AAF (AVTP Audio Format) |
| Bit depth | 24-bit samples packed in 32-bit integers |
| Sample rate | 96 kHz and 192 kHz |

Both talkers (senders) and listeners (receivers) must support both additional sample rates.

---

## Channel Layout

The channel layout rules from the Basic Audio Profile apply unchanged at all supported sample rates.
