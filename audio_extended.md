# Extended Audio Profile

This AVB community profile specifies the extended requirements for audio streaming compatibility. All requirements from the Basic Audio Profile apply, with the following additions.

Version: 1.1-draft

---

## Stream Format

Talkers and listeners must support the following additional formats:

| Parameter | Value |
|-----------|-------|
| Format | AAF (AVTP Audio Format) |
| Bit depth | 24-bit samples packed in 32-bit integers |
| Sample rate | 96 kHz and 192 kHz |

---

## SR Class

Talkers and listeners must support SR Class A as well as Class B, at every sample rate: the Basic Audio Profile's exception for devices that cannot handle 8000 frames per second does not apply.

Each class sends one frame each interval of its class, so a frame holds more samples at the higher rates:

| SR class | Samples per frame at 48 kHz | at 96 kHz | at 192 kHz |
|----------|-----------------------------|-----------|------------|
| A | 6 | 12 | 24 |
| B | 12 | 24 | 48 |

As in the Basic Audio Profile, a stream size is required in a class only where its frame fits one Ethernet frame. The largest stream size each class carries is:

| Sample rate | Class A | Class B |
|-------------|---------|---------|
| 48 kHz | 32 channels | 16 channels |
| 96 kHz | 16 channels | 8 channels |
| 192 kHz | 8 channels | 2 channels |
