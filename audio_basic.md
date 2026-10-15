# Basic Audio Profile

This AVB community profile specifies the basic requirements for audio streaming compatibility.

Version: 1.1-draft

---

## Stream Format

Talker and listeners must use the following format:

| Parameter | Value |
|-----------|-------|
| Format | AAF (AVTP Audio Format) |
| Bit depth | 24-bit samples packed in 32-bit integers |
| Sample rate | 48 kHz |


---

## SR Class

Talkers and listeners must support SR Class B. Those that can handle Class A's frame rate, 8000 frames per second for each of their streams, must support SR Class A as well.

| SR class | Required | Frames per second | Samples per frame at 48 kHz |
|----------|----------|-------------------|-----------------------------|
| B | Always | 4000, one each 250 µs interval | 12 |
| A | Where the device can handle 8000 frames per second | 8000, one each 125 µs interval | 6 |

Supporting Class B means sending and receiving Class B's own frames, 12 samples at 48 kHz, one each interval. Class A's 6-sample frames sent two to a Class B interval are required of no device, and talkers should not send them: they take twice the frames, and some bridges reserve too little for them (AVB Wireless, §4).

Since every device supports Class B, any talker and listener can always meet in it.

Class A carries a stream with less latency, 2 ms across seven hops against Class B's 50 ms (IEEE 802.1BA). Milan requires Class A and not Class B, so a device that supports Class B alone reaches only those Milan devices that also support Class B (informative).

---

## Channel Layout

Talkers and listeners must be able to transport audio channels using the following stream sizes: a 2-channel stream, then doubling from 8 up to the largest stream one frame carries:

| Channels to transport | Stream size to use |
|-----------------------|--------------------|
| 1–2 | 2-channel stream |
| 3–8 | 8-channel stream |
| 9–16 | 16-channel stream |
| 17–32 | 32-channel stream (Class A only) |

Each of a stream's frames fits one Ethernet frame, its AVTPDU at most 1500 octets, so a stream size is required in a class only where its frame fits. At 48 kHz, Class B's frames carry up to 30 channels and Class A's up to 61: the 32-channel stream is Class A's alone, and no larger stream size fits in either class.

Channels are always populated starting from the lowest channel number. Any unused slots in the stream are left empty.

Every talker and listener must support both the 2-channel and the 8-channel stream, in each SR class it supports, whatever its channel count, so a stereo device and an 8-channel device can always reach each other. If a device supports a larger stream size, it must also support all smaller stream sizes in the table above. Which channels are used within those smaller streams is up to the implementer.

Support for other channel counts (e.g. 1, 4, 6) is optional.
