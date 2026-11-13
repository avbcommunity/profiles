# Voice Assistant Profile

A profile for audio endpoints that hold spoken conversations with a voice assistant on the network: how an endpoint describes what it captures, marks voice activity in its stream, reports events, takes commands, and exposes its settings.

Version: 1.0-draft

---

## 1. Scope

A voice assistant listens to people through the microphones of endpoints in their rooms and answers through the endpoints' speakers, in duplex, in several rooms at once. AVB gives it what a conversation needs and the open internet does not: one clock for every sample, a fixed latency both ways, and output buffers of milliseconds, so the assistant knows which word a person heard when they spoke over it, and stops speaking within a few milliseconds of deciding to.

This profile specifies what an endpoint adds to its ordinary streams for that:

- **Streams:** a capture stream and a playback stream that stay bound between conversations (§2).
- **Voice activity in the stream:** voice activity, double-talk and speech probability written into the low byte of each voice sample, where a 32-bit stream format leaves room for it (§3).
- **A vendor-unique protocol:** the endpoint's capabilities and capture layout, its state, the events it detects, and the commands it takes, in IEEE 1722.1 AECP (§4).
- **Controls:** its settings as AEM CONTROL descriptors of the standard's own types (§5).

How the assistant recognizes speech, decides when to talk, or what it says is out of scope.

### 1.1 Roles

- **Voice endpoint:** an ATDECC entity with at least one microphone and one speaker, a talker of its capture stream and a listener of its playback stream.
- **Assistant:** a system on the network that listens to the capture streams of one or more voice endpoints and talks into their playback streams. It runs an ATDECC controller, which binds the streams and registers for the endpoints' unsolicited notifications, and a talker and listener of its own.

Other controllers may enumerate a voice endpoint and set its controls as for any entity.

### 1.2 Relation to Other Profiles and Milan

All requirements of the Basic Audio Profile apply to a voice endpoint's streams. Where the endpoint uses them, the AVB Lite Profile and the AVB Wireless Profile apply as well (§6).

The vendor-unique messages of §4 follow the pattern of the AVB Lite status query (AVB Lite §2.4) and the AVB Wireless status query (AVB Wireless §5), and use the same MA-S.

An entity may conform to both Milan and this profile. The profile adds a stream format, a vendor-unique protocol and controls of standard types, all of which a Milan controller ignores or shows as it shows any other, and changes nothing Milan requires.

### 1.3 Draft Notes (informative)

Points open in this draft:

- A second playback stream mixed on the endpoint, for an assistant speaking over music from another talker, is reserved for a later revision (the MIX capability bit, §4.3).
- Settings with no standard control type, such as indicator brightness, are left to the endpoint's maker. A pool of vendor-defined control types would need values from the MA-S's EUI-64 space.

---

## 2. Streams

### 2.1 Voice Units

A voice endpoint offers at least one voice unit: one capture STREAM_OUTPUT and one playback STREAM_INPUT, paired. GET_VOICE_CAPABILITIES (§4.3) names the pair, and every message of §4 names its voice unit by the capture stream's STREAM_OUTPUT index.

Both streams meet the Basic Audio Profile: AAF at 48 kHz, Class B always and Class A where the endpoint can handle it, in the 2-channel and 8-channel stream sizes the Basic Audio Profile requires of every device.

### 2.2 Capture Stream

Channel 1 of the capture stream carries the voice the assistant listens to: the endpoint's processed voice where it cancels echo or suppresses noise, otherwise its microphone. The capture layout (§4.3) says what every channel carries. A channel carries one of:

| Kind | Value | Carries |
|------|-------|---------|
| Unused | 0 | Silence: every sample zero |
| Voice | 1 | The voice for the assistant, after whatever processing the endpoint applies. Channel 1 only |
| Microphone | 2 | One microphone, unprocessed, with its position |
| Beam | 3 | A beam formed from the endpoint's microphones |
| Played | 4 | A copy of what the endpoint played, as its echo reference (below) |

A 2-channel stream from an endpoint with one microphone and no played copy carries the voice on channel 1 and channel 2 unused. A microphone array uses the 8-channel stream size: the voice or a beam, the microphones, and a played channel.

**Played channel.** An endpoint that sets PLAYED_COPY (§4.3) carries in a played channel each sample it hands its digital-to-analog converter, after everything it mixes in, so the assistant's echo canceller has an exact reference for sound the assistant did not send itself. The sample the endpoint handed its converter at the instant it captured microphone frame *f* sits in frame *f* + `played_copy_offset` of the capture stream. The endpoint keeps that offset fixed while the stream runs and reports it, or reports `0x7FFF` where it cannot hold it fixed. COPY_BEFORE_VOLUME and COPY_EXCLUDES_LOCAL say what the copy leaves out.

**Format.** An endpoint that sets AUX (§4.3) offers on its capture stream, besides the Basic Audio Profile's 24-bit samples in 32-bit integers, AAF INT_32BIT at 48 kHz with a `bit_depth` of 32, in each stream size and class it offers. Only that format carries the voice activity byte (§3), so an assistant that wants the byte binds it.

**Timing.** As IEEE 1722 requires, each AVTPDU's timestamp is the sample's capture time plus the stream's maximum transit time. `capture_delay_ns` (§4.3) reports any fixed delay from sound reaching the microphone to the instant the endpoint takes as its capture time, such as an ADC's filter delay, so the assistant can align capture with playback to the sample.

### 2.3 Playback Stream

The endpoint presents each playback sample at its presentation time, with a fixed delay from that time to the sound leaving its speaker, which it reports as `playback_delay_ns` (§4.3) and keeps fixed while the stream runs. That delay and the capture delay let the assistant align its echo reference with what the microphone hears to within a sample, with no loopback.

A fixed delay the endpoint adds after the presentation time, such as a cushion a listener on a wireless hop always keeps, is part of `playback_delay_ns`. A delay it adds or removes as it runs, as such a listener may add after losses and then walk back over minutes, it reports in `playout_adjust_us` (§4.4), so the assistant's echo canceller follows the change at once rather than searching for it.

An assistant talking to an endpoint with one speaker sends the same signal on both channels of a 2-channel playback stream (informative).

### 2.4 Standing Connections

Binding a stream takes up to seconds where MSRP propagates slowly, so an assistant keeps its streams with an endpoint bound between conversations, and silences them with SET_VOICE_STATE's gates (§4.5) rather than unbinding. An endpoint should store its bindings and restore them after a restart without waiting for a controller.

---

## 3. Voice Activity in the Capture Stream

A voice endpoint that sets AUX (§4.3) writes a voice activity byte into the low 8 bits of each channel 1 sample while the capture stream's current format is INT_32BIT with a `bit_depth` of 32. In a stream whose `bit_depth` is 24 it writes none: IEEE 1722-2016 §7.3.4 requires every unused sample bit to be zero.

The voice then fills the sample's upper 24 bits, and the byte describes the very sample it rides in:

| Bits | Field | Value |
|------|-------|-------|
| 7–2 | `speech_probability` | 0 to 63, the endpoint's estimate that the sample is speech, 63 certain. 0 where the endpoint does not set SPEECH_PROBABILITY |
| 1 | `double_talk` | 1 while the endpoint hears speech in the room as it plays. 0 where it does not set DOUBLE_TALK |
| 0 | `voice` | 1 while the endpoint detects voice activity |

- The endpoint writes the byte after all gain and processing of channel 1, so no later stage alters it.
- The byte is zero while no one speaks. Where a wireless hop saves airtime by leaving out zero low bytes, in an encapsulation both ends support (AVB Wireless §3.7), it still does so in silence (informative).
- The low bytes of other channels stay zero unless their kind says otherwise.
- The assistant reads the byte from the raw 32-bit sample, before any conversion to floating point keeps only the upper 24 bits (informative).
- A listener that does not know this profile plays the byte as audio, at most 255 units of a 2^31 full scale, below −138 dBFS (informative).

Only what has to line up with the samples travels in the stream. Discrete events go in VOICE_EVENT (§4.6).

---

## 4. Voice Assistant Protocol

### 4.1 Protocol Identifier

The messages of this section are IEEE 1722.1 AECP `VENDOR_UNIQUE_COMMAND` and `VENDOR_UNIQUE_RESPONSE` PDUs (IEEE 1722.1-2021 §9.6), under the MA-S of the AVB Lite Profile with sub-protocol identifier `0x007`:

| Bits  | Field | Value |
|-------|-------|-------|
| 47–12 | MA-S OUI | `0x8C1F6436C` |
| 11–0  | Sub-protocol identifier | `0x007` |

Encoded as 6 bytes on the wire: `8C 1F 64 36 C0 07`.

### 4.2 Common Fields, Status and Timing

Every command and response starts with the same fields after `protocol_id`:

| Offset | Length | Field | Value |
|--------|--------|-------|-------|
| 0 | 2 | `u` (bit 15), `command_type` (bits 14–0) | `u` is 1 in an unsolicited response; `command_type` as below |
| 2 | 2 | `stream_output_index` | The STREAM_OUTPUT descriptor of the voice unit's capture stream (§2.1) |

| `command_type` | Name | Section |
|----------------|------|---------|
| `0x0000` | GET_VOICE_CAPABILITIES | §4.3 |
| `0x0001` | GET_VOICE_STATUS | §4.4 |
| `0x0002` | SET_VOICE_STATE | §4.5 |
| `0x0003` | VOICE_EVENT | §4.6 |

A response's `status` takes the values of IEEE 1722.1-2021 Table 7-141 by the same names, which from 2 up lie in the vendor-defined range of Table 9-6: SUCCESS; NOT_IMPLEMENTED for a `command_type` the endpoint does not support; NO_SUCH_DESCRIPTOR for a `stream_output_index` the entity does not have; NOT_SUPPORTED for a STREAM_OUTPUT that is not a voice unit's capture stream; BAD_ARGUMENTS when a reserved bit is set or a value is out of range; ENTITY_ACQUIRED or ENTITY_LOCKED for SET_VOICE_STATE when another controller has acquired or locked the entity, as for AEM's SET commands.

An endpoint shall answer every command within 240 ms, or extend that with IN_PROGRESS responses every 120 ms, as IEEE 1722.1-2021 §9.3.2.6 requires of ATDECC commands.

Unsolicited responses go to each controller registered for unsolicited notifications (IEEE 1722.1-2021 §7.4.37 REGISTER_UNSOLICITED_NOTIFICATION). An assistant registers with every voice endpoint it serves.

### 4.3 GET_VOICE_CAPABILITIES

What a voice unit can do and what its capture stream carries. The command carries only the common fields. The response continues:

| Offset | Length | Field | Value |
|--------|--------|-------|-------|
| 4 | 1 | `profile_version` | 1 |
| 5 | 1 | `channel_count` | The capture stream's channels in its current format, each described below |
| 6 | 2 | `flags` | Bit 0 AUX: writes the voice activity byte (§3). Bit 1 DOUBLE_TALK: sets its `double_talk` bit. Bit 2 SPEECH_PROBABILITY: sets its `speech_probability` field. Bit 3 AEC: cancels echo before channel 1. Bit 4 NOISE_SUPPRESSION: suppresses noise before channel 1. Bit 5 WAKE_WORD: detects wake words. Bit 6 BUTTON: has buttons. Bit 7 INDICATOR: shows the indicator states of §4.5. Bit 8 PLAYED_COPY: carries a played channel (§2.2). Bit 9 COPY_BEFORE_VOLUME: the played copy is taken before the volume control, which the assistant then applies to it itself. Bit 10 COPY_EXCLUDES_LOCAL: sounds the endpoint plays of its own accord are not in the copy. Bit 11 LOCAL_PLAYBACK: plays sounds of its own, which DUCK_LOCAL and STOP_LOCAL act on. Bit 12 MIX: reserved for a later revision, 0. Other bits zero |
| 8 | 2 | `stream_input_index` | The STREAM_INPUT descriptor of the voice unit's playback stream |
| 10 | 2 | `played_copy_offset` | Signed, in frames (§2.2); `0x7FFF` where not fixed; 0 without PLAYED_COPY |
| 12 | 4 | `capture_delay_ns` | Fixed delay from sound at the microphone to the sample's capture time (§2.2) |
| 16 | 4 | `playback_delay_ns` | Fixed delay from a sample's presentation time to its sound leaving the speaker (§2.3) |
| 20 | 2 | `volume_control` | The CONTROL descriptor of the speaker volume (§5), `0xFFFF` if none |
| 22 | 2 | `mic_gain_control` | The CONTROL descriptor of the microphone gain, `0xFFFF` if none |
| 24 | 2 | `aec_control` | The CONTROL descriptor enabling echo cancellation, `0xFFFF` if none |
| 26 | 2 | `noise_suppression_control` | The CONTROL descriptor enabling noise suppression, `0xFFFF` if none |
| 28 | 2 | `wake_word_control` | The CONTROL descriptor enabling wake word detection, `0xFFFF` if none |
| 30 | 2 | `duck_control` | The CONTROL descriptor of the ducking level, `0xFFFF` if none |
| 32 | 8 × `channel_count` | Channel entries | One per capture channel, from channel 1 |

Each channel entry:

| Offset | Length | Field | Value |
|--------|--------|-------|-------|
| 0 | 1 | `kind` | §2.2's kinds: 0 unused, 1 voice, 2 microphone, 3 beam, 4 played |
| 1 | 1 | `number` | Which microphone or beam, from 0; 0 for other kinds |
| 2 | 2 | `x_mm` | Signed. A microphone's position in millimetres from the endpoint's reference point, positive toward its front; 0 for other kinds |
| 4 | 2 | `y_mm` | Signed, positive toward its left |
| 6 | 2 | `z_mm` | Signed, positive upward |

The capabilities of an endpoint with eight channels take 96 octets, well within the 508 of a vendor-unique payload. An endpoint sends the response unsolicited whenever it changes, as when the capture stream's format changes its channel count. The capabilities are not configured, so nothing persists.

### 4.4 GET_VOICE_STATUS

The voice unit's state. The command carries only the common fields. The response continues:

| Offset | Length | Field | Value |
|--------|--------|-------|-------|
| 4 | 1 | `state_flags` | Bits 0–3 as SET_VOICE_STATE last set them (§4.5). Bit 4 LOCAL_PLAYING: the endpoint is playing a sound of its own. Bit 5 MIC_MUTED: a switch or setting on the endpoint has its microphones off. Bit 6 CAPTURE_RUNNING: the capture stream is being sent. Bit 7 PLAYBACK_RUNNING: the playback stream is being received |
| 5 | 1 | `indicator` | The indicator state last set (§4.5) |
| 6 | 1 | `erle_db` | Signed: the echo return loss enhancement the endpoint's echo canceller achieves now, in dB; −128 where unknown or without AEC |
| 7 | 1 | reserved | 0 |
| 8 | 2 | `last_event_sequence` | The `event_sequence` of the latest VOICE_EVENT (§4.6), 0 before the first |
| 10 | 2 | reserved | 0 |
| 12 | 4 | `playout_adjust_us` | Signed: how far, in microseconds, the endpoint now presents playback later than its presentation time plus `playback_delay_ns` (§2.3), 0 while it presents on time |

An endpoint sends the response unsolicited whenever any field other than `erle_db` and `playout_adjust_us` changes, and whenever `playout_adjust_us` has moved 1 ms or more from the value it last sent. Controllers that follow `erle_db` poll; once a second is enough.

### 4.5 SET_VOICE_STATE

The assistant's commands to a voice unit. The command and the response carry:

| Offset | Length | Field | Value |
|--------|--------|-------|-------|
| 4 | 1 | `state_flags` | Bit 0 GATE_CAPTURE: send silence in the capture stream, its binding and reservation kept. Bit 1 GATE_PLAYBACK: play silence in place of the playback stream, its binding and reservation kept. Bit 2 DUCK_LOCAL: lower the endpoint's own sounds by the ducking level (§5). Bit 3 STOP_LOCAL: stop the endpoint's own sounds. Other bits zero |
| 5 | 1 | `indicator` | What the endpoint shows: 0 idle, 1 listening, 2 thinking, 3 speaking, 4 muted, 5 attention. Other values reserved |
| 6 | 2 | reserved | 0 |

- The response carries the state after the command.
- An endpoint should act on the command within 5 ms of receiving it, and apply its gates at a block boundary.
- An endpoint without INDICATOR or LOCAL_PLAYBACK takes the fields it cannot act on and reports them as set.
- The state does not persist: after a restart both gates are open, nothing is ducked, and the indicator is idle.
- A change is reported with an unsolicited GET_VOICE_STATUS response.

A gate is the way to silence a room without unbinding (§2.4), and the way a person's request for privacy reaches the endpoint as well as the assistant (informative).

### 4.6 VOICE_EVENT

What the endpoint detects, timed on the network's clock. The endpoint sends each event as an unsolicited response. A controller's command, carrying only the common fields, returns the latest event, or `event_type` 0 before the first. The response carries:

| Offset | Length | Field | Value |
|--------|--------|-------|-------|
| 4 | 2 | `event_sequence` | One more than the previous event's, from 1, wrapping from `0xFFFF` to 1 |
| 6 | 1 | `event_type` | 0 none, 1 WAKE_WORD, 2 BUTTON_DOWN, 3 BUTTON_UP, 4 LOCAL_PLAYBACK_START, 5 LOCAL_PLAYBACK_STOP, 6 MIC_MUTED, 7 MIC_UNMUTED. Other values reserved |
| 7 | 1 | `event_detail` | The wake word's or button's number, from 0; 0 for other types |
| 8 | 8 | `event_time` | When the event happened, in nanoseconds of the PTP timescale the endpoint follows (IEEE 802.1AS, or the AVB Lite PTP profile); for a wake word, when it ended |
| 16 | 4 | `event_span_ns` | How long before `event_time` the event began, as for a wake word; 0 otherwise |
| 20 | 4 | reserved | 0 |

- An endpoint should send an event within 10 ms of detecting it.
- Unsolicited responses are not acknowledged. An endpoint may send an event again with the same `event_sequence`; receivers drop repeats. A gap in `event_sequence` tells the assistant it missed one.
- `event_time` is a capture time, so the samples a wake word spans are presented from `event_time` − `event_span_ns` + the stream's maximum transit time (informative).

---

## 5. Controls

A voice endpoint exposes its settings as AEM CONTROL descriptors (IEEE 1722.1-2021 §7.2.22) of the standard control types of Table 7-98, set and read with SET_CONTROL and GET_CONTROL (§7.4.25 and §7.4.26), and sends an unsolicited notification whenever one changes. GET_VOICE_CAPABILITIES (§4.3) names each by descriptor index, so the assistant finds them without reading their names:

| Setting | Control type | Value |
|---------|--------------|-------|
| Speaker volume | GAIN `0x90E0F00000000004` | In dB, a level unit of §7.3.4.23 as §7.3.5.5 allows, on the playback path |
| Microphone gain | GAIN | In dB, on the capture path |
| Echo cancellation | ENABLE `0x90E0F00000000000` | LINEAR_UINT8, 0 off and 255 on (§7.3.5.1) |
| Noise suppression | ENABLE | 0 off, 255 on |
| Wake word detection | ENABLE | 0 off, 255 on |
| Ducking level | GAIN | In dB, 0 or less: how far DUCK_LOCAL lowers the endpoint's own sounds |

An endpoint names each control in its descriptor so other controllers show it plainly, and keeps its settings across restarts. A control that changes the level of channel 1 also changes what the assistant hears; the voice activity byte is written after it (§3).

---

## 6. Wireless Endpoints

A voice endpoint on an 802.11 hop meets the AVB Wireless Profile, and its streams run in Class B, the hop's own class (AVB Wireless §3.6, §4).

- Messages of §4 cross the hop like other AECP. An endpoint may repeat its events once over the hop (§4.6), and controllers poll no more often than §4.4 allows.
- The voice activity byte is zero in silence (§3), so it costs airtime only while someone speaks (informative).
- A wireless hop adds its own transit time to both streams; the assistant takes it from the streams' presentation offsets and counts it in its timing (informative).

---

## 7. Conformance

| Requirement | Reference |
|-------------|-----------|
| **Voice endpoint** | |
| Offers a voice unit, a capture STREAM_OUTPUT and a playback STREAM_INPUT that meet the Basic Audio Profile | §2.1 |
| Carries the voice on capture channel 1, and every capture channel as one of §2.2's kinds | §2.2 |
| With AUX, offers INT_32BIT with a `bit_depth` of 32 on its capture stream | §2.2 |
| With PLAYED_COPY, carries what it hands its DAC in a played channel, at a fixed offset it reports, or reports it unfixed | §2.2 |
| Reports its fixed capture and playback delays, and keeps them fixed while its streams run | §2.2, §2.3 |
| Presents playback at its presentation time, or reports in `playout_adjust_us` how much later it presents | §2.3, §4.4 |
| With AUX, writes the voice activity byte after all gain, only at a `bit_depth` of 32, and leaves other low bytes zero | §3 |
| Leaves unused sample bits zero at a `bit_depth` of 24 | §3 |
| Answers GET_VOICE_CAPABILITIES, GET_VOICE_STATUS, SET_VOICE_STATE and VOICE_EVENT within 240 ms, with the status codes of §4.2 | §4.2 to §4.6 |
| Sends GET_VOICE_CAPABILITIES and GET_VOICE_STATUS unsolicited on change, and each event as it happens, to registered controllers | §4.3 to §4.6 |
| Gates its streams without unbinding them, and keeps no gate across a restart | §4.5 |
| Exposes its settings as CONTROL descriptors of standard types, named in its capabilities, with unsolicited notifications | §5 |
| On a wireless hop, meets the AVB Wireless Profile | §6 |
| **Assistant** | |
| Runs an ATDECC controller and registers for unsolicited notifications with each voice endpoint it serves | §4.2 |
| Keeps its streams with an endpoint bound between conversations, and silences them with gates | §2.4 |
| Reads the voice activity byte from the raw 32-bit sample | §3 |
| Polls `erle_db` no more often than once a second | §4.4 |
