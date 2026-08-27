# AVB Wireless Specification

A profile for extending AVB and AVB Lite media streaming across an IEEE 802.11 hop: a wireless access point acting as a bridge to a wired AVB or AVB Lite domain, and Wi-Fi station endpoints acting as talkers and listeners.

Version: 1.0-draft

---

## 1. Scope

This profile covers a single 802.11 infrastructure hop: one access point that is also an AVB (or AVB Lite) bridge on its wired port, and the stations associated with it. It defines two interim mechanisms that make AVB streaming workable on commodity embedded Wi-Fi platforms today, plus the stream addressing rules both sides must follow:

1. **Beacon-carried time synchronization** (§2), for platforms whose Wi-Fi stacks do not allow attaching the custom 802.1AS information element to Timing Measurement or Fine Timing Measurement action frames.
2. **Unicast stream translation** (§3), because 802.11 delivers group addressed frames roughly an order of magnitude slower than individually addressed frames, in both directions.

Both mechanisms are interim: when a platform exposes the standard carriers (802.1AS-2020 §12 timing over TM/FTM, 802.11aa GCR/DMS for group delivery), implementations should migrate to them. The wire formats defined here are chosen so that migration changes the carrier, not the payload.

Only SR Class B streams are admitted over the wireless hop. IEEE 802.1BA excludes Class A over 802.11 media, whose per-hop latency contribution cannot meet the 2 ms budget; implementations may offer a bench-test opt-in for Class A but must not admit it by default.

---

## 2. Time Synchronization, Beacon Carrier

IEEE 802.1AS-2020 §12 defines gPTP transport over 802.11 using the §12.7 VendorSpecific information element carried inside TM or FTM action frames, and defines no new frames of its own. Most embedded Wi-Fi APIs expose no way to attach a custom IE to TM/FTM action frames; typically the only writable IE slots are the Beacon and Probe Response Vendor IEs. This section defines the interim carrier for such platforms.

### 2.1 The §12.7 IE in the Beacon

The access point shall publish the 802.1AS §12.7 VendorSpecific element as a Beacon (and Probe Response) Vendor IE, byte-format identical to the specification: the full Follow_Up equivalent, PTP common header, preciseOriginTimestamp, and the FollowUpInformation TLV, 76 bytes total. A standards-aware parser sees an unmodified §12.7 IE; only the carrier frame type deviates from §12.1.2.

The AP refreshes the IE from its own gPTP state (the wired-side best timing clock, correction field, and rate ratio) so that a station consuming only beacons receives everything a Follow_Up would have carried.

### 2.2 Companion TSF Mapping IE

A beacon-only carrier is bounded by beacon scheduling jitter, which limits phase accuracy to tens of milliseconds. To recover precision, the AP shall publish a second Vendor IE in the same beacon, the TSF mapping IE:

| Field | Width | Value |
|-------|-------|-------|
| OUI | 3 bytes | `8C:1F:64` (MA-L prefix of the AVB Lite MA-S OUI) |
| subtype | 1 byte | `0x01`, TSF mapping |
| payload | 8 bytes | AP TSF in microseconds, unsigned little-endian |

The TSF value shall be captured as close as possible to beacon transmission (on split host/radio architectures, patched in by the radio-side processor immediately before the beacon is queued). Together the two IEs give the station a running pair, `(gPTP_marker_ns, ap_tsf_marker_us)`, mapping the AP's TSF timebase to grandmaster time.

### 2.3 Station Behavior

A station shall recover grandmaster time as follows:

1. Parse both IEs from received beacons and maintain the latest `(gPTP_marker, ap_tsf_marker)` pair.
2. Where FTM is available, initiate FTM bursts against the AP (which shall run an FTM responder). The responder timestamp `t1` is in AP TSF picoseconds; convert to GM time via `t1_gPTP_ns = gPTP_marker + (t1_us - ap_tsf_marker) * 1000`, pair it with the local receive timestamp, and feed the pair to the local servo. FTM round-trip measurements also provide the propagation delay.
3. Where FTM is unavailable, or FTM measurements are invalid (short-range links commonly produce zero or negative RTT that responders discard), the station shall fall back to beacon-pair servoing with a fixed propagation-delay assumption. Sub-microsecond phase should not be expected in this mode.

A station needs only beacon reception and, optionally, FTM initiation: no custom frame or IE transmission capability is required of the station side.

### 2.4 Migration

When the platform exposes IE injection on TM/FTM action frames, the §12.7 IE moves to its standard carrier unchanged and the TSF mapping IE is dropped. Parsers should treat the mapping IE as optional from the start.

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
3. Multiple wireless listeners on one stream receive per-listener unicast copies, up to a bridge-defined fan-out limit. Beyond the limit the bridge shall not escalate to group addressing on the wireless port (that would degrade every listener below a single stream's rate); additional listeners are simply not served, and the condition should be made observable.
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

The bridge applies SRP admission on the wireless port against a nominal link-rate cap (75% per SR class), and shall declare Talker Failed with `insufficient_bandwidth_for_traffic_class` for Class A toward the wireless port unless a non-default opt-in is set. Implementations should log the admission basis and every rejection, so a relayed upstream failure can be told apart from a local decision.

Note that the deliverable unicast rate bounds the practical stream count: with roughly 8000 individually addressed packets per second available on the air, two Class B streams (or their fan-out equivalent) is a realistic ceiling per BSS, and the fan-out limit of §3.2 should be set accordingly.

---

## 5. Conformance

| Requirement | Reference |
|-------------|-----------|
| AP publishes byte-exact §12.7 IE in Beacon/Probe Response | §2.1 |
| AP publishes TSF mapping IE, TSF captured at transmit | §2.2 |
| AP runs an FTM responder | §2.3 |
| STA recovers GM time from beacon pair, FTM when available | §2.3 |
| Stream frames individually addressed on the WLAN | §3.1 |
| Bridge translates downlink streams to listener MACs from SRP state | §3.2 |
| Bridge never escalates to group addressing on the wireless port | §3.2 |
| Wireless talker advertises MAAP DA, transmits to the U/L-flipped BSSID carrier | §3.3 |
| Listener accepts own-MAC or advertised DA, matches by stream_id | §3.4 |
| Class B only by default over the wireless hop | §4 |
