# Basic Video Profile

This AVB community profile specifies the basic requirements for video streaming compatibility: unprotected video up to HD, at up to 30 frames per second.

Version: 1.0-draft

---

## Stream Formats

Every talker and listener must support MJPEG, so any talker and listener can always meet in it. H.264 is optional: talkers should offer it as well, and listeners may accept it.

| Format | Talkers | Listeners |
|--------|---------|-----------|
| MJPEG | Must offer | Must accept |
| H.264 | Should offer | May accept |

In either format, video is progressive, at up to 30 frames per second, with no content protection.

MJPEG asks least of a listener, and H.264 takes a fraction of its bandwidth, so a talker that offers both lets each listener choose (informative).

### MJPEG

| Parameter | Value |
|-----------|-------|
| Format | CVF (Compressed Video Format, IEEE 1722-2016 Clause 8), `format_subtype` MJPEG |
| Payload | JPEG as RFC 2435 carries it: baseline, sequential DCT, the standard Huffman tables |
| Sampling | Type 0 (4:2:2) or type 1 (4:2:0), RFC 2435 §4.1 |

Talkers send either type, and listeners must accept both. Types 64 and 65, the same with restart markers, are optional: a talker sends them only to a listener whose formats list includes them.

Talkers carry the quantization tables in the first AVTPDU of every frame, with a Q from 128 to 255 and a non-zero Length (RFC 2435 §3.1.8 and §4.2), so a listener can decode the first whole frame it receives. Listeners must also accept a Q from 1 to 99, whose tables RFC 2435 §4.2 computes.

Samples are YCbCr as JFIF (ITU-T T.871) defines them, full range, which is what JPEG encoders and decoders assume when nothing says otherwise.

Each JPEG frame decodes on its own, so a lost AVTPDU costs one frame, not the frames after it (informative).

### H.264

| Parameter | Value |
|-----------|-------|
| Format | CVF, `format_subtype` H264 (IEEE 1722-2016 §8.5) |
| Payload | H.264 as RFC 6184 carries it, in its non-interleaved mode (Sending a Stream, below) |
| Profile | Constrained Baseline (ITU-T H.264 §A.2.1.1) |
| Level | At most 4.0, which 1080p at 30 frames per second needs; 720p needs only 3.1 |
| Sampling | 4:2:0, 8 bits |

A listener that accepts H.264 must decode every stream this profile allows: Constrained Baseline up to level 4.0, at either picture size. The H.264 stream format carries no profile, level or picture size (Discovery, below), so a listener cannot accept some of these and refuse the rest.

Every H.264 decoder, whether built for the Baseline, Main or High profile, decodes Constrained Baseline, and Constrained Baseline has no B pictures, so pictures arrive in the order they are shown (informative).

A talker sends its sequence and picture parameter sets in the stream before every IDR picture, and an IDR picture at least once a second, so a listener that joins a stream, or loses part of it, decodes again within a second. IEEE 1722.1-2021 gives a listener no way to ask for an IDR picture (informative).

A talker sends BT.709 limited range, the usual for HD, or describes its colour in the stream's VUI (ITU-T H.264 Annex E). A listener follows the description where there is one.

A lost AVTPDU can spoil the pictures that follow it, up to the next IDR picture, where in MJPEG it costs one frame (informative).

---

## Picture Sizes

Video is sent in the following picture sizes, with square pixels:

| Picture size | Width × height | RFC 2435 width, height |
|--------------|----------------|------------------------|
| 720p | 1280 × 720 | 160, 90 |
| 1080p | 1920 × 1080 | 240, 135 |

**MJPEG.** Every talker and listener must support 720p, so any talker and listener can always meet in it. A talker that offers 1080p must also offer 720p, and a listener that accepts 1080p must also accept 720p. Other sizes are optional.

**H.264.** A talker sends 720p or 1080p, and a listener that accepts H.264 decodes both. A 1080p picture is coded as 1920 × 1088 and cropped to 1080 lines by the sequence parameter set, as H.264 encoders do.

A listener presents each frame at its picture size, scaling it to its display as it sees fit.

---

## SR Class

Talkers and listeners must support SR Class B. SR Class A is optional.

A talker spreads each frame across up to one frame period (Sending a Stream, below), far beyond Class A's 2 ms latency, so Class A gives video little (informative).

---

## Sending a Stream

Each AVTPDU is at most 1500 octets (IEEE 1722-2016 §4.4.4.10) and carries data of one frame only.

**MJPEG.** A talker fragments each JPEG frame as RFC 2435 does, in order (§8.4.4). It sets M on the frame's last AVTPDU (§8.4.2.3), and `tv` and the presentation time on that AVTPDU only (§8.4.2.2).

**H.264.** A talker packetizes each access unit, one frame, in RFC 6184's non-interleaved mode (§6.3): single NAL unit packets, STAP-A and FU-A, in decoding order. Listeners must accept all three. A talker does not use STAP-B, MTAP or FU-B, the interleaved mode's packets, which IEEE 1722-2016 §8.5.4 also allows. It sets M on the access unit's last AVTPDU (§8.5.2.4), and `tv` and the presentation time on each AVTPDU that ends a NAL unit (§8.5.2.2), the same time for every NAL unit of the access unit. It sets `ptv` and the `h264_timestamp`, RFC 6184's 90 kHz timestamp, in every AVTPDU (§8.5.2.3, §8.5.3.1).

**Reservation.** A video stream reserves at most one AVTPDU of at most 1500 octets each Class B interval: a `MaxIntervalFrames` of 1 and a `MaxFrameSize` of at most 1500, about 49 Mb/s on the wire. A talker may reserve less, and should at 720p. Listeners must accept any stream within this reservation.

**Pacing.** A talker sends no more AVTPDUs in each Class B interval than its reservation allows, so it spreads each frame across the frame period instead of sending it in a burst (IEEE 1722-2016 §8.1). It keeps each frame within what its reservation carries in one frame period, lowering the quality where it must: at 30 frames per second and the largest reservation, about 195,000 octets of video data. In H.264 this holds for IDR pictures too, the largest.

**Presentation time.** A talker sets a frame's presentation time to the time it has the complete frame from its encoder (§8.2.1.1), plus one frame period for sending it, plus the stream's maximum transit time, by default 50 ms for Class B (IEEE 1722-2016 Table 3). A listener presents each frame at its presentation time, so it needs no knowledge of the frame rate, and drops a frame that arrives incomplete.

A 1080p MJPEG stream at the largest reservation takes about two-thirds of the 75% of a 100 Mb/s link that streams may use: room is left for audio, but not for a second such stream. An H.264 stream's reservation is set by its largest picture, usually an IDR picture, and is often far less (informative).

---

## Discovery

A talker lists each format it offers in its STREAM_OUTPUT formats, and a listener each format it accepts in its STREAM_INPUT formats, in the stream formats of IEEE 1722-2016 Annex I.2.5.

The MJPEG stream format (Annex I.2.5.2) carries the type, and the width and height in the 8-pixel units of RFC 2435. A listener therefore lists both types at each picture size it accepts.

The H.264 stream format (Annex I.2.5.3) carries nothing beyond the format itself: no profile, level or picture size. A talker that lists it sends only streams this profile allows, and a listener that lists it decodes all of them, learning the picture size from the sequence parameter set.

Neither stream format carries the frame rate. A talker reports its frame rate as its VIDEO_CLUSTER's current sampling rate (IEEE 1722.1-2021 §7.2.17), and its content protection as NONE (Table 7-131).

Audio that goes with a video stream travels in its own streams under the Basic Audio Profile. A talker sets the presentation times of both from the same capture instant, so listeners on the same time base keep them in sync (informative).

---

## Beyond This Profile (informative)

The Extended Video Profile is expected to add:

- **More of H.264:** the Main and High profiles and levels above 4.0, with a way to advertise them, since the IEEE 1722-2016 H.264 stream format has no fields for them, and a way for a listener to ask for an IDR picture.
- **H.265 and AV1**, which the 2025 revision of IEEE 1722 was chartered to add.
- **Raw video** in RVF (IEEE 1722-2016 Clause 12) on gigabit links, where 720p at 30 frames per second fits and 1080p does not, with the component order and sample packing that Clause 12 leaves open.
- **60 frames per second**, and interlaced video where a source needs it.
