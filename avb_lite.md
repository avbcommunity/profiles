# AVB Lite Profile

This AVB community profile specifies the AVB Lite requirements for audio streaming compatibility.

Version: 1.0-draft

---

## Stream Format

Talker and listeners must use the following format:

| Parameter | Value |
|-----------|-------|
| Format | AAF (AVTP Audio Format) |
| Bit depth | 24-bit samples packed in 32-bit integers |
| Sample rate | 48 kHz |


---

## ATDECC Discovery, Control, and Connection Management

Devices must support IEEE 1722.1 ATDECC for discovery, enumeration, and stream connection management.

Minimum required support:

| Function | Requirement |
|----------|-------------|
| Discovery | ADP entity discovery and advertisement |
| Control and enumeration | AECP using the AEM command set |
| Connection management | ACMP for connecting and disconnecting AVTP streams |
| Descriptors | Entity, Configuration, Audio Unit, Stream Input/Output, Clock Source, and Clock Domain descriptors as applicable |
| Stream formats | Supported and current stream formats must be exposed through AEM |
| Identity | Entity IDs and Stream IDs must be stable across reboots unless the hardware configuration changes |

Devices must reject unsupported stream formats or invalid connections using standard ATDECC error responses.

---

## Channel Layout

Talkers and listeners must be able to transport audio channels using the following stream sizes, doubling from 8 up to the maximum permitted by AVTP:

| Channels to transport | Stream size to use |
|-----------------------|--------------------|
| 1–8 | 8-channel stream |
| 9–16 | 16-channel stream |
| 17–32 | 32-channel stream |
| 33–64 | 64-channel stream |
| … and so on | next doubling up |

Channels are always populated starting from the lowest channel number. Any unused slots in the stream are left empty.

If a device supports a given stream size, it must also support all smaller stream sizes in the table above. Which channels are used within those smaller streams is up to the implementer.

Support for other channel counts (e.g. 1, 2, 4, 6) is optional.
