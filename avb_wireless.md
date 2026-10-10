# AVB Wireless Specification

A profile for extending AVB and AVB Lite media streaming across an IEEE 802.11 hop: a wireless access point acting as a bridge to a wired AVB or AVB Lite domain, and Wi-Fi station endpoints acting as talkers and listeners.

Version: 1.1-draft

---

## 1. Scope

This profile covers a single 802.11 infrastructure hop: one access point that is also an AVB (or AVB Lite) bridge on its wired port, and the stations associated with it. It defines how stations get the network's time, how streams are addressed on the air, what is admitted, and how controllers discover and watch the wireless hop:

1. **Time synchronization** (§2) in one of two modes. Mode B is IEEE 802.1AS-2020 §12, gPTP over Fine Timing Measurement or Timing Measurement with the §12.7 element in those frames, for platforms that can put it there. Mode A is an interim carrier for platforms whose Wi-Fi stacks cannot: the grandmaster's time in the access point's beacons, mapped to the access point's TSF, with FTM measuring only the link delay.
2. **Unicast stream translation** (§3), because 802.11 delivers group addressed frames roughly an order of magnitude slower than individually addressed frames, in both directions.
3. **Admission** (§4): Class B only over the wireless hop.
4. **Discovery and status** (§5): a status query that tells controllers which interfaces are wireless, which access point a station uses, and how the hop's link and time hold.

Mode A and unicast translation are interim: when a platform exposes the standard carriers (802.1AS-2020 §12 over FTM or TM, 802.11aa GCR/DMS for group delivery), implementations should migrate to them. The wire formats are chosen so that migration changes the carrier, not the payload.

Only SR Class B streams are admitted over the wireless hop. IEEE 802.1BA excludes Class A over 802.11 media, whose per-hop latency contribution cannot meet the 2 ms budget; implementations may offer a bench-test opt-in for Class A but must not admit it by default.

---

## 2. Time Synchronization

### 2.1 Two Modes

A station recovers grandmaster time in one of two modes, never both at once:

| | Mode A, beacon carrier | Mode B, 802.1AS over FTM or TM |
|---|---|---|
| Grandmaster time | The access point's beacons, mapped to its TSF (§2.3) | The §12.7 element in the access point's FTM or TM frames (§2.4) |
| FTM or TM | FTM measures the link delay only | Carries time and measures the link delay |
| Platform needs | Writable Beacon and Probe Response vendor elements | Vendor elements in FTM or TM frames, and the counter their timestamps run on mapped to gPTP time in hardware |
| IEEE 802.1AS port | No: asCapable is FALSE (§2.3) | Yes, asCapable as §12.4 sets it |
| Typical accuracy | About a millisecond, bounded by the access point's capture of its two clocks | Within about a microsecond |

The two cannot be combined by converting FTM timestamps through the beacon's TSF mapping. FTM's timestamps run on the responder's FTM counter, in picoseconds, 48 bits wide on the wire and wrapping every 281.475 s; the TSF is a different clock with a different epoch. A station that treated them as one put itself 47,243 s off once the TSF grew past the FTM counter's range. FTM's `t1` reaches grandmaster time only through the §12.7 element in the FTM frame itself, or a mapping from gPTP time to the FTM counter made in hardware at the access point: that is Mode B.

### 2.2 Announce and Signaling, Both Modes

Neither the beacon element nor FTM carries the grandmaster's priority, clock quality or `stepsRemoved`. The access point shall send §12.2 Announce, individually addressed, and gPTP-capable Signaling to each associated station, and shall process the message interval request TLV in the station's Signaling as §12.8.2 specifies: `logTimeSyncInterval` -3 by default, 127 to stop, 126 for the initial value, -128 for no change, and unsupported values ignored, with `logLinkDelayInterval` sent as -128 and ignored on 802.11. A station sends standard Signaling; it needs no custom frames or elements of its own.

### 2.3 Mode A: Beacon Carrier

A Mode A station's Wi-Fi port is not an IEEE 802.1AS port: its time comes from this profile, not from §12, so it reports asCapable FALSE for the port in GET_AVB_INFO, and GET_WIRELESS_STATUS (§5) shows the mode and whether its time is locked.

#### 2.3.1 The Beacon Time Element

The access point shall publish one 802.11 Vendor Specific element in its Beacon and Probe Response frames:

| Field | Width | Value |
|-------|-------|-------|
| Element ID | 1 byte | 221 |
| Length | 1 byte | 90, or 92 with the capture bracket |
| Organization identifier and sub-ID | 6 bytes | `8C:1F:64:36:C0:06`: the MA-S `8C-1F-64-36-C` in the first 36 bits and AVB Wireless sub-ID `0x006` in the last 12, as the profile family's other identifiers are built |
| Follow_Up | 76 bytes | The content of the IEEE 802.1AS-2020 §12.7 element: the PTP common header, `preciseOriginTimestamp`, and the Follow_Up information TLV |
| AP TSF | 8 bytes | The access point's TSF in microseconds, unsigned, little-endian |
| Capture bracket | 2 bytes, optional | Microseconds between the two captures, unsigned, little-endian, `0xFFFF` when not known |

In the beacon element, `preciseOriginTimestamp` plus `correctionField` give the grandmaster's time at the instant the access point sampled its TSF, not at the departure of an FTM or TM frame as in §12.7; the Follow_Up's other fields keep their §12.7 meaning. The access point shall sample the gPTP time and the TSF as nearly simultaneously as the platform allows. Any residual gap should be one-sided, gPTP sampled first, so stations can follow the envelope of the samples (§2.3.3). When the beacon goes out does not matter: the station's TSF follows the access point's through each beacon's Timestamp field. Where the platform installs vendor elements by a call, both values are refreshed in the same call, so a beacon never pairs one refresh's gPTP time with another's TSF.

The access point refreshes the element from its own gPTP state at its Sync interval, 125 ms, without blocking its wired gPTP: a refresh that stalls the wired port's Pdelay exchange costs its asCapable. It shall keep the TLV's `gmTimeBaseIndicator` and `lastGmPhaseChange` current, so stations see a change of grandmaster rather than inferring one from a jump.

Earlier drafts published the §12.7 element unchanged with the TSF in a second element, `8C:1F:64` subtype `0x01`, and a first implementation joined them as `8C:1F:64` subtype `0x00`. `8C-1F-64` is the IEEE block shared by every MA-S assignment in it, so the identifier now carries the full MA-S and a sub-ID, as this profile family's other identifiers do. Stations may accept the earlier forms during migration.

#### 2.3.2 Link Delay by FTM

Where the access point runs an FTM responder (§2.5), the station measures the link delay with FTM, for example a burst of 16 frames every 200 ms. In Mode A, FTM timestamps never become time samples.

- Below about a metre the true round trip is about zero, and the computed one lands at or below zero. A link delay limited to zero there is acceptable.
- A small negative round trip can arrive as two's complement in an unsigned field: filter on the signed value being positive, not on it being neither zero nor all ones.
- An invalid measurement never becomes a sample. On other failures the station keeps the last good link delay.

Without FTM the station assumes a fixed link delay; sub-microsecond phase is then not to be expected.

#### 2.3.3 Station Recovery

1. The rate comes from the station's own TSF, which its hardware keeps locked to the access point's TSF through beacon Timestamp fields, and which it reads locally.
2. Each beacon element gives an offset `O = gPTP_marker - TSF_marker × 1000 + link_delay`, in nanoseconds, where `gPTP_marker` is its `preciseOriginTimestamp` plus `correctionField` and `TSF_marker` its AP TSF; grandmaster time is `O_filtered + STA_TSF_now × 1000`.
3. Beacons go out every 102.4 ms while the element refreshes every 125 ms, so about a fifth repeat the last element. The station deduplicates on `preciseOriginTimestamp`; processing a repeat again aliases the rate.
4. The access point's capture gap is one-sided, so the station tracks the upper envelope of `O`, not its mean, which sits tens of milliseconds off.

An example envelope tracker (informative), as a reference implementation used: acquire with 16 samples of peak hold, then one step; then rise toward higher samples by at most 0.4 ms a sample, and otherwise decay by 0.1 ms a sample; step again on a jump of more than 100 ms up, or eight samples in a row more than 350 ms down. With an access point whose gPTP and TSF are on separate chips, it locked about 28 s after association, its own servo error at a median of 0.53 ms, 95% within 1.45 ms and 99% within 1.85 ms, and its rate within ±10 ppm. An access point sampling both clocks on one chip should do much better.

### 2.4 Mode B: 802.1AS over FTM or TM

Mode B is IEEE 802.1AS-2020 §12, over Fine Timing Measurement or, where both ends support it, Timing Measurement (§12.3). The access point fills the §12.7 element in its FTM or TM frames from a coherent snapshot in its transmit path, without blocking. Over TM, asCapable does not depend on burst parameters (§12.4 c) 1)). Points that implementations found over FTM:

- **Timestamps.** All four are in picoseconds. `t1`, FTM departure, and `t4`, the acknowledgement's arrival, are on the responder's FTM counter, 48 bits on the wire: compare and difference them modulo 2^48, or a wrap looks like a step back. `t2` and `t3` are on the initiator's counter.
- **Pairing.** The element's Follow_Up describes the previous measurement's `t1`, with `correctionField` = rateRatio × (`t1` − the upstream transmit time) + the upstream correction (§12.5.1.4.6). It pairs with its measurement by the FTM dialog token; its `sequenceId` is the port's own counter.
- **The station's reference.** `preciseOriginTimestamp` + `correctionField` + (RTT ÷ 2) × rateRatio, against its local `t2`.
- **Round trip.** The neighbor rate comes from the first and last `t1` against `t2` in a burst; RTT = (`t4` − `t1`) − (`t3` − `t2`) × rate, which also corrects the turnaround for the clocks' relative rate (−53 ppm over a 104 µs turnaround is −5.5 ns); the quartet with the lowest RTT wins. An entry with an RTT below zero or above 2 µs, a response or turnaround above 1 ms, or no single matching element is discarded.
- **Mapping at the access point.** The access point's FTM counter shall be mapped to its gPTP time with hardware precision. With gPTP and the radio on separate chips, a mapping by message round trip left about 6.4 ms of ambiguity; a shared hardware edge, a PTP pulse latched by the radio, was needed.
- **Burst parameters.** §12.6 requires the station to ask for one burst per request, of three frames and then two on retry (Table 12-2), with a Burst Duration and Min Delta FTM that follow the sync interval, 64 ms and 10 ms at the default of log -3 (Table 12-3). Where a platform grants other parameters, such as one burst of eight frames every 125 ms with frames 2 ms apart, stations may use them as an interim measure. An FTM link is asCapable only when three- or two-frame bursts are granted (§12.4), so with larger bursts asCapable stays FALSE; implementations should report the granted parameters and why asCapable is FALSE (§5).
- **Channel width.** The access point keeps a fixed channel width: a responder recalibrating as the width changed stepped the round trip by 12.5 ns.

A reference implementation reached, between bridge and station, all edges typically within 1 µs (95% within 0.48 to 0.88 µs, worst 1.06 µs), and within −480 to +643 ns while carrying a Class B stream at 4000 frames a second. It recovered in about 1 s after losing Sync, 2 s after a reset, and 8 s after losing Announce. These are informative, not bounds.

### 2.5 Access Point Requirements

The access point:

1. Runs an FTM responder on its access point interface and sets the FTM Responder Extended Capability.
2. Keeps a fixed channel width.
3. Sends individually addressed Announce and gPTP-capable Signaling to each station (§2.2).
4. In Mode A, publishes the beacon time element (§2.3.1), refreshed without blocking its wired gPTP.
5. In Mode B, fills the §12.7 element in its FTM or TM frames from a coherent snapshot, maps the counter their timestamps run on to gPTP time in hardware, and over FTM grants the burst parameters of §12.6 (Tables 12-2 and 12-3), reporting what it grants.

### 2.6 Station Behavior, Both Modes

- **Power save.** A station stays in active mode while it is synchronized or streaming: power save drops beacons and FTM frames, and the radio's clock may not be readable while it sleeps.
- **A new time source.** A new BSSID, or the access point's TSF going backward by more than a second on three beacons in a row, which marks an access point reset, is a new time source: the station discards its markers and fits and acquires again. The TSF is 64 bits in microseconds and does not wrap in practice.
- **Until locked.** A station should not present media until its time is locked: a listener mutes, and a talker holds off or sends with presentation times marked invalid (`tv` 0). On loss of lock it gates again.

### 2.7 Migration

When a platform can put vendor elements in FTM or TM frames and map their timestamps' counter in hardware, it moves from Mode A to Mode B, and may stop publishing the beacon time element. Parsers treat the beacon element as optional from the start.

---

## 3. Stream Transport Addressing over Wi-Fi

802.11 penalizes group addressed data frames in both directions. Downlink, they are buffered to the DTIM beacon cycle and are neither acknowledged nor retried (IEEE 802.11 10.3.6). Uplink, embedded drivers commonly throttle them regardless of the standard's allowance. Measured on a representative embedded platform with 234-byte stream frames, group addressing delivered roughly 400 packets per second downlink and 320 uplink, against 8300 and 4800 individually addressed, below the 4000 packets per second one Class B stream requires. Every serious WLAN vendor converts multicast to unicast for exactly this reason, and RFC 9119 documents the problem as inherent to 802.11.

The rules below are DMS-equivalent behavior, not IEEE 802.11 DMS: real DMS wraps the group addressed MSDU in an individually addressed A-MSDU and is negotiated with Action frames. Where 802.11aa DMS/GCR is actually available, it satisfies this section.

### 3.1 General Rule

Stream data frames on the WLAN shall be individually addressed, in both directions. Stream identity is not affected: the AVTP `stream_id` rides in every stream AVTPDU (IEEE 1722 4.4.4.8) and remains the stream's identity end to end. The destination MAC on the wireless hop is transport addressing only, exactly as in AVB Lite unicast transport.

### 3.2 Bridge (AP) Downlink Translation

For streams flowing wired-to-wireless, the bridge shall re-address each stream frame from its advertised (group) destination address to the individual address of the listening station:

1. The mapping is derived entirely from SRP state the bridge already holds: the Talker Advertise gives `stream_id` to advertised destination address; the Listener Ready received on the wireless port gives `stream_id` to the declaring station's MAC (taken from the MRPDU source address, since the Listener attribute itself carries only a StreamID). IEEE 802.1Q 35.2.2.8.3 guarantees one stream per destination address, so the advertised DA is a valid lookup key at forwarding time.
2. Wired-side SRP and ATDECC state are not modified: upstream devices, and Milan's (Stream ID, Stream Destination MAC Address, Stream VLAN ID) triple, continue to see the advertised MAAP destination address.
3. Multiple wireless listeners on one stream receive per-listener unicast copies, up to a bridge-defined fan-out limit. Beyond the limit the bridge shall not escalate to group addressing on the wireless port (that would degrade every listener below a single stream's rate); additional listeners are simply not served, and the condition should be made observable (§5).
4. A group addressed stream frame with no resolved listener mapping may be forwarded unchanged as a compatibility fallback.

### 3.3 Wireless Talker Targeting

A station acting as a talker shall address its stream frames as follows:

1. **In SRP and ATDECC declarations**, the talker advertises a normal MAAP-allocated (or locally administered) multicast destination address. A station's globally administered factory MAC is not a legal SRP `destination_address` (802.1Q 35.2.2.8.3 permits only multicast or locally administered addresses), and the advertised DA is what wired listeners and bridges must continue to see.
2. **On the air**, the talker transmits stream frames individually addressed using a carrier destination: the BSSID with the locally administered (U/L) bit set. Any individual MSDU destination earns the unicast air treatment, since the 802.11 receiver address is the BSSID for every uplink frame regardless, but split AP architectures (host processor plus radio coprocessor) commonly consume an MSDU addressed to the AP's own MAC inside the radio and never deliver it to the bridge host, so the plain BSSID must not be used. Flipping the U/L bit yields a deterministic locally administered unicast carrier that needs no discovery and collides with no real device. The bridge restores the talker's advertised destination address on wired egress, keyed by `stream_id`, so the carrier value is otherwise opaque.
3. Where the listener is another station in the same BSS, per-listener unicast copies addressed per AVB Lite unicast transport apply; the AP relays individually addressed frames without the group addressed penalty.
4. Group addressed stream transmission by a station is permitted only as a last-resort compatibility fallback and should be expected to carry no more than a few hundred packets per second.

### 3.4 Listener Behavior

A wireless listener shall accept stream frames whose destination is either its own MAC or the advertised destination address, and shall match streams by `stream_id`, mirroring AVB Lite listener behavior.

---

## 4. Admission

The bridge applies SRP admission on the wireless port against a nominal link-rate cap (75% per SR class), and shall declare Talker Failed with `insufficient_bandwidth_for_traffic_class` for Class A toward the wireless port unless a non-default opt-in is set (§5.2). Implementations should log the admission basis and every rejection, so a relayed upstream failure can be told apart from a local decision.

Note that the deliverable unicast rate bounds the practical stream count: with roughly 8000 individually addressed packets per second available on the air, two Class B streams (or their fan-out equivalent) is a realistic ceiling per BSS, and the fan-out limit of §3.2 should be set accordingly.

---

## 5. Discovery and Status

A controller learns that an interface is wireless, which access point a station uses, and how the hop's link and time hold, through the AVB Wireless status query: an IEEE 1722.1 AECP `VENDOR_UNIQUE_COMMAND` / `VENDOR_UNIQUE_RESPONSE` exchange under the MA-S of the AVB Lite profile, with sub-protocol identifier `0x005`, encoded as `8C 1F 64 36 C0 05`. Stations answer it, and so do access points that run an ATDECC entity, for their wireless port. What IEEE 1722.1 GET_AVB_INFO already reports, the grandmaster, the link's propagation delay and asCapable, it does not repeat.

This is a first pass: enough to place stations and watch the hop, not to manage the Wi-Fi network.

### 5.1 GET_WIRELESS_STATUS

The command and the response start with the same fields after `protocol_id`:

| Offset | Length | Field | Value |
|--------|--------|-------|-------|
| 0 | 2 | `u` (bit 15), `command_type` (bits 14–0) | `u` is 1 in an unsolicited response; `command_type` is `0x0000`, GET_WIRELESS_STATUS (`0x0001` is SET_WIRELESS_CONFIG, §5.2) |
| 2 | 2 | `avb_interface_index` | The AVB_INTERFACE descriptor the status is for |

The command carries nothing more. The response continues:

| Offset | Length | Field | Value |
|--------|--------|-------|-------|
| 4 | 1 | `flags` | Bit 0 ACCESS_POINT: the interface is an access point's wireless port, else a station's. Bit 1 LOCKED: a station's time is locked (§2.6). Bit 2 HOLDOVER: a station is holding over without updates. Bit 3 SERVO_ERROR_VALID. Bit 4 RTT_VALID. Bit 5 CLASS_A_ALLOWED (§5.2). Other bits zero. |
| 5 | 1 | `time_mode` | 0 none, 1 Mode A, beacon carrier (§2.3), 2 Mode B over FTM, 3 Mode B over TM (§2.4) |
| 6 | 1 | `band` | 0 not known, 1 2.4 GHz, 2 5 GHz, 3 6 GHz |
| 7 | 1 | `channel` | The primary channel number, 0 when not known |
| 8 | 2 | `channel_width` | In MHz, 0 when not known |
| 10 | 1 | `phy_generation` | The Wi-Fi generation of the negotiated PHY: 4 802.11n, 5 802.11ac, 6 802.11ax, 7 802.11be; 0 when not known |
| 11 | 1 | `rssi` | A station's RSSI from its access point in dBm, signed; `0x7F` on an access point, whose per-station RSSI is in the station list, or when not known |
| 12 | 2 | `phy_rate` | A station's current transmit PHY rate in Mb/s, 0 when not known |
| 14 | 1 | `ftm_success` | Percent of FTM measurements valid over the last 10 s; `0xFF` without FTM |
| 15 | 1 | `as_capable_reason` | Why asCapable is FALSE on a Mode B station, 0 when it is TRUE, or in Mode A, whose port is not an IEEE 802.1AS port: 1 FTM frames per burst granted other than three or two (§12.4), 2 neither FTM nor TM with the other end, 3 no gPTP-capable Signaling from it |
| 16 | 1 | `ftm_burst_frames` | Frames per burst granted, 0 without FTM |
| 17 | 1 | `listeners_unserved` | An access point's listeners Ready on its wireless port that it does not serve, beyond its fan-out limit (§3.2); 0 on a station, `0xFF` when not known |
| 18 | 1 | `ftm_burst_duration` | The granted Burst Duration as IEEE 802.11 encodes it (2 for 250 µs up to 11 for 128 ms, 15 for no preference), 0 without FTM |
| 19 | 1 | `ftm_min_delta` | The granted Min Delta FTM in units of 100 µs, as IEEE 802.11 encodes it, 0 without FTM |
| 20 | 4 | `ftm_rtt` | The last valid FTM round trip in nanoseconds, signed (RTT_VALID) |
| 24 | 4 | `servo_error` | A station's servo error estimate in nanoseconds, signed (SERVO_ERROR_VALID); not an absolute measurement against the grandmaster |
| 28 | 4 | `association_age` | Seconds since a station associated, or since an access point started its BSS |
| 32 | 2 | `time_age` | Milliseconds since a station last applied a time element, beacon or FTM; `0xFFFF` for none or beyond the range |
| 34 | 2 | `ap_resets` | Access point resets a station has seen (§2.6) since it started |
| 36 | 6 | `bssid` | A station's access point BSSID, or an access point's own |
| 42 | 10 | `ap_port_identity` | The gPTP port identity of the access point's wireless port, as a station follows it, or as the access point's own |
| 52 | 4 | `downlink_readdressed` | An access point's stream frames re-addressed to a station (§3.2), counted since it started; 0 on a station |
| 56 | 4 | `downlink_unmapped` | An access point's group addressed stream frames with no listener mapping (§3.2) |
| 60 | 4 | `downlink_dropped` | An access point's stream frames dropped for back-pressure on the wireless port |
| 64 | 4 | `uplink_restored` | An access point's station stream frames given back their advertised destination (§3.3) |
| 68 | 2 | `station_count` | The station entries that follow, on an access point; 0 on a station |
| 70 | 8 × n | `stations` | Per associated station: its MAC (6 bytes), its RSSI in dBm, signed (1 byte, `0x7F` when not known), and flags (1 byte: bit 0 FTM_INITIATOR, bit 1 FTM_KNOWN, whether the access point knows the station's FTM capability); as many as fit the response, the rest left out |

With `bssid` and `ap_port_identity` a controller places a station under its access point; GET_AS_PATH names the bridge's clock but not its BSSID. Counters wrap. The response's `status` is SUCCESS, NO_SUCH_DESCRIPTOR for an `avb_interface_index` the entity does not have, NOT_SUPPORTED for an interface that is not wireless, or NOT_IMPLEMENTED from an entity without AVB Wireless support.

An entity shall send the response unsolicited, with `u` set, to each controller registered for unsolicited notifications whenever `flags`, `time_mode`, `band`, `channel`, `channel_width`, `phy_generation`, `as_capable_reason`, the granted burst, `listeners_unserved`, `bssid`, `ap_port_identity` or `station_count` changes. The measurements and counters change too often for that; controllers poll them, once every 5 s at most.

### 5.2 SET_WIRELESS_CONFIG

The command and the response carry, after `avb_interface_index`:

| Offset | Length | Field | Value |
|--------|--------|-------|-------|
| 4 | 1 | `config_flags` | Bit 0 CLASS_A_ALLOWED: an access point admits Class A toward its wireless port, the §1 bench-test opt-in. Other bits zero. |
| 5 | 3 | reserved | 0 |

The response carries the configuration the interface has after the command. Its `status` is SUCCESS; NO_SUCH_DESCRIPTOR; BAD_ARGUMENTS for a reserved bit, or CLASS_A_ALLOWED on a station; ENTITY_ACQUIRED or ENTITY_LOCKED when another controller holds the entity; NOT_SUPPORTED for an interface that is not wireless; or NOT_IMPLEMENTED. CLASS_A_ALLOWED does not persist across restarts, so the default comes back. A change is reported with an unsolicited GET_WIRELESS_STATUS response.

---

## 6. Conformance

| Requirement | Reference |
|-------------|-----------|
| Station recovers time in Mode A or Mode B, never through FTM `t1` mapped by the beacon's TSF | §2.1 |
| AP sends individually addressed Announce and gPTP-capable Signaling to each station | §2.2 |
| Mode A: AP publishes the beacon time element, both values sampled together, refreshed without blocking wired gPTP, TLV's GM fields kept | §2.3.1 |
| Mode A: FTM for link delay only, invalid measurements never samples | §2.3.2 |
| Mode A: station reports asCapable FALSE for its Wi-Fi port | §2.3 |
| Mode A: station deduplicates elements and tracks the upper envelope of the offset | §2.3.3 |
| Mode B: §12.7 element in FTM or TM frames, their counter mapped to gPTP time in hardware, FTM's `t1`/`t4` compared modulo 2^48 | §2.4 |
| Mode B over FTM: §12.6 burst parameters, or interim ones with asCapable FALSE and the grant reported | §2.4 |
| AP runs an FTM responder, sets its capability, keeps a fixed channel width | §2.5 |
| Station stays in active mode, treats a new BSSID or a TSF going backward as a new time source, presents no media until locked | §2.6 |
| Stream frames individually addressed on the WLAN | §3.1 |
| Bridge translates downlink streams to listener MACs from SRP state | §3.2 |
| Bridge never escalates to group addressing on the wireless port | §3.2 |
| Wireless talker advertises MAAP DA, transmits to the U/L-flipped BSSID carrier | §3.3 |
| Listener accepts own-MAC or advertised DA, matches by stream_id | §3.4 |
| Class B only by default over the wireless hop | §4 |
| Stations, and APs running an ATDECC entity, answer GET_WIRELESS_STATUS, unsolicited on change | §5.1 |
