# Basic Audio Profile

This AVB community profile specifies the basic requirements for audio streaming compatibility.

---

## Stream Format

All audio streams must use the following format:

| Parameter | Value |
|-----------|-------|
| Format | AAF (AVTP Audio Format) |
| Bit depth | 24-bit samples packed in 32-bit integers |
| Sample rate | 48 kHz |

Both talkers (senders) and listeners (receivers) must support this format.

---

## Channel Layout Rules

### Single Audio Channel

When carrying a single channel of audio, a device must be able to send and receive it in each of the following stream sizes:

- **1-channel stream** — the signal is in channel 1
- **2-channel stream** — the signal is in channel 1 (channel 2 may carry the same signal or be unused)
- **8-channel stream** — the signal is in channel 1 (other channels may carry the same signal or be unused)

### Multiple Discrete Channels

When carrying two or more independent channels of audio:

- Channels are filled starting from the lowest channel number first.
- Up to **8 channels** may be carried in a single stream.
- If more than 8 channels are needed, they must be grouped into streams of **8 channels each**, with the lowest channel numbers in the first stream.

**Example — 12 discrete channels:**
- Stream 1 carries channels 1–8
- Stream 2 carries channels 9–12 (channels 13–16 are unused)
