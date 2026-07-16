# AVB Lite Specification

A profile for deploying AVB-style deterministic live media streaming over standard managed Ethernet switches: no 802.1AS, no MSRP, and no credit-based shaper required.

Version: 1.0-draft

---

## 1. Scope

AVB Lite reuses the AVB endpoint stack, including AVTP streams, stream IDs, and presentation time, but replaces the network-side requirements of IEEE 802.1BA so that the system runs on any gigabit switch with VLAN/802.1p QoS support.

Targets: live audio and video installations of up to approximately 50 endpoints and 200 streams, on a single L2 domain.

---

## 2. Automatic AVB Lite Fallback

An AVB endpoint that also supports AVB Lite must automatically fall back to AVB Lite when it detects that it is not connected through an AVB-capable bridge.

### 2.1 Endpoint Declaration TLV

To enable peer endpoints to detect each other across a non-AVB bridge that floods the bridge-group multicast (`01:80:C2:00:00:0E`), every AVB Lite-capable endpoint shall append an Organization Extension TLV (IEEE 1588-2008 §13.4 / §14, TLV type `0x0003`) to every `Pdelay_Req`, `Pdelay_Resp`, and `Pdelay_Resp_Follow_Up` it transmits, with the following content:

| Field | Width | Value |
|-------|-------|-------|
| `tlvType` | 2 bytes | `0x0003` (ORGANIZATION_EXTENSION, IEEE 1588-2008 §14.3.2) |
| `lengthField` | 2 bytes | `0x0007` (length of `organizationId` + `organizationSubType` + `dataField` = 3 + 3 + 1) |
| `organizationId` | 3 bytes | `8C:1F:64` (24-bit MA-L prefix of the MA-S OUI `0x8C1F6436C`) |
| `organizationSubType` | 3 bytes | `36:C0:01` (top 12 bits = MA-S suffix `0x36C`, bottom 12 bits = AVB Lite TLV sub-id `0x001` for Endpoint Declaration) |
| `dataField` | 1 byte | `0x01` = "I am an endpoint, not a bridge" |

A compliant 802.1AS bridge terminates the Pdelay protocol at its boundary clock and re-originates `Pdelay_Req`, `Pdelay_Resp`, and `Pdelay_Resp_Follow_Up` from its own port without copying endpoint-originated TLVs, so the Endpoint Declaration TLV does not survive transit through an AVB-aware bridge. Per IEEE 1588-2008 §13.4, recipients that do not recognize a TLV must ignore it; this preserves backward compatibility with non-AVB-Lite-aware bridges and endpoints.

### 2.2 Fallback Conditions

An endpoint shall arm fallback detection on every Ethernet link-up event. Three conditions are evaluated in parallel and continuously for the lifetime of the link. The first to fire triggers fallback to AVB Lite:

1. **Endpoint TLV detected.** A `Pdelay_Req`, `Pdelay_Resp`, or `Pdelay_Resp_Follow_Up` arrives carrying the Endpoint Declaration TLV defined in §2.1. This indicates that the sender of the message is also an endpoint (no AVB-aware bridge sits between the two devices).
2. **Nine consecutive `Pdelay_Req` attempts unanswered.** Mirrors the 802.1AS-2020 §11.5.3 `allowedLostResponses` default of 9: the same counter that flips `asCapable` to false at the gPTP layer also triggers fallback at the profile layer. The counter is reset to zero on every received `Pdelay_Resp`. This indicates either an isolated link or a network with no compliant peer. (Earlier revisions of this profile used 3, the 802.1AS-2011 value; field experience showed a transient ~3 s ingress loss on a healthy AVB link could spring the fallback latch, which §2.2.1 then makes permanent until a link-flap.)
3. **Two or more distinct responders to a single `Pdelay_Req`.** For any single Pdelay cycle, if responses arrive from two or more distinct `sourcePortIdentity` values, fallback fires. A spec-compliant AVB bridge presents exactly one boundary-clock peer per port, so cardinality > 1 to a single request indicates a non-compliant L2 substrate (e.g. a switch flooding the bridge-group multicast `01:80:C2:00:00:0E`).

Conditions 2 and 3 are evaluated identically whether the link just came up or has been operating for hours: a peer or bridge that drops out, or a topology change that introduces a flooding bridge, will trigger fallback the same way it would have at startup. This matches 802.1AS continuous-evaluation semantics rather than a startup-only window.

If none of the three conditions fires — i.e. exactly one peer is responding with valid `Pdelay_Resp` messages and that peer is not advertising itself as an endpoint — the endpoint remains in standard AVB / gPTP operation, treating that lone responder as an AVB-aware bridge. Condition 1 continues to fire on any later-arriving endpoint TLV (e.g. a peer that boots and begins beaconing per §2.3), pulling the endpoint into AVB Lite without requiring a link-flap.

The decision to fall back, once made, shall be latched until the next Ethernet link-up event to prevent flapping. Unlike 802.1AS's per-port `asCapable` flag — which can re-acquire automatically when Pdelay starts succeeding again — the AVB Lite fallback is a profile-level switch (gPTP → standard PTP) and is too invasive to bounce per packet. A link-up event is the canonical re-arming point.

#### 2.2.1 Promotion back to standard AVB is not permitted

Once an endpoint has fallen back to AVB Lite, it shall not promote itself back to standard AVB / gPTP operation while the link remains up, even if conditions on the network appear to change (e.g. an AVB-aware bridge later joins the segment, or peers stop emitting the Endpoint Declaration TLV). Spontaneous promotion is prohibited because:

- It would disrupt active CVU SRP admission, AVB Lite media streams, and the §2.3 beacon contract that other peers depend on.
- A profile switch from standard PTP back to gPTP requires re-running BMCA, re-establishing peer-delay state, and re-issuing MSRP attributes — all of which would interrupt audio/video that is already flowing over AVB Lite.
- Detection-side races could cause flapping if multiple endpoints attempt to promote concurrently.

If the operator or controller knows the topology has changed and the network is now genuinely AVB-aware, promotion shall be initiated by an external action: either an Ethernet link-flap that re-arms the §2.2 evaluator, or a controller-issued reconfiguration command that restarts the PTP daemon. Implementations may expose a vendor-specific control to trigger such a restart, but the criteria for using it are out of scope for this profile.

A future revision of this profile may define a coordinated, network-wide promotion protocol if real-world deployments demonstrate the need.

When falling back, the device must clearly advertise its active operating mode to the controller and must not use MSRP reservations for AVB Lite streams. AVB Lite streams must use the timing, QoS, admission-control, and conformance requirements defined by this profile.

A device should prefer standard IEEE 802.1BA operation when neither §2.2 condition is met, unless configured otherwise by the operator or controller.

### 2.3 Post-Fallback Endpoint Beacon

Once an AVB Lite-capable endpoint has fallen back to standard PTP per §2.2, it no longer participates in the gPTP `Pdelay_{Req,Resp,Resp_Follow_Up}` exchange and therefore stops emitting the Endpoint Declaration TLV defined in §2.1. This silence can prevent peers still operating in gPTP mode from observing the §2.2 condition 1 signal, leaving the network in a mixed state.

To avoid this, every AVB Lite-capable endpoint operating in standard PTP mode after fallback shall periodically emit an "endpoint beacon" frame consisting of a `Pdelay_Req` PDU carrying the Endpoint Declaration TLV from §2.1, sent at a beacon cadence of 1 frame every 3 seconds. The beacon cadence shall not exceed the §2.2 condition 2 evaluation window (nine Pdelay_Req attempts at the nominal 1 s interval, ~9 s) so that a peer inside its arming window is guaranteed to receive at least one beacon — with margin — within that window.

The beacon `Pdelay_Req` shall be sent to the gPTP destination MAC address `01:80:C2:00:00:0E` (bridge-group multicast), not the standard PTP destination, regardless of the transmitter's current PTP profile, so that peers still operating in gPTP mode receive it on the channel they are listening on.

The beacon `Pdelay_Req` is informational only:

- It is sent to the bridge-group multicast `01:80:C2:00:00:0E`.
- The transmitting endpoint does not expect, track, or process any `Pdelay_Resp` to a beacon frame.
- The beacon shall set `messageType` to `PDELAY_REQ` with the gPTP `SDOID_GPTP` flag, so receivers in gPTP mode parse it as a normal `Pdelay_Req`.
- The beacon's `sequenceId` shall continue from the endpoint's normal Pdelay sequence so packet captures remain coherent, but slot semantics are otherwise irrelevant.
- A receiver in gPTP mode that observes the beacon shall treat it identically to a §2.1 Endpoint Declaration TLV in any other Pdelay message: the §2.2 condition 1 signal fires and that receiver also falls back.

Endpoints in gPTP mode are not required to emit beacon frames — the §2.1 TLV in their normal `Pdelay_{Req,Resp,Resp_Follow_Up}` traffic is sufficient. The beacon exists only to maintain the §2.2 condition 1 signal after fallback has silenced the normal Pdelay exchange.

---

## 3. Bridging Between AVB and AVB Lite Domains

A system may bridge traffic between a standard AVB domain and an AVB Lite domain using an explicit gateway or boundary device.

The gateway is responsible for translating domain assumptions, including:

- Clock-domain adaptation between 802.1AS/gPTP and the AVB Lite PTP profile.
- Admission-control translation between MSRP/SRP and the AVB Lite controller bandwidth ledger.
- QoS mapping between AVB SR classes and AVB Lite 802.1p traffic classes.
- VLAN ID translation where the AVB and AVB Lite domains use different VLAN IDs or where one side is untagged.
- Stream lifecycle coordination so that connection state is consistent on both sides.
- Presentation-time offset adjustment for the combined AVB and AVB Lite path latency.

Bridging must be explicit. Endpoints must not assume that AVB and AVB Lite domains are directly interoperable without a gateway that provides these translation functions. VLAN identity is local to each domain and must not be assumed to be the same across the gateway.

---

## 4. Changes Relative to 802.1BA-2011

| 802.1BA Component | Status in AVB Lite | Replacement |
|-------------------|--------------------|-------------|
| 802.1AS (gPTP) | Removed | AVB Lite PTP profile over Layer-2 Ethernet, with hardware timestamping at endpoints |
| 802.1Qat (MSRP) | Removed | Centralized admission control in controller software |
| 802.1Qav (FQTSS / credit-based shaper) | Removed | Strict-priority queueing via 802.1p |
| 802.1Q VLAN / SR class A & B | Retained, redefined | Media streams use configurable VLAN ID, default 2, with Class A and Class B → 802.1p priority 5 |
| AVTP (1722) stream format | Retained unchanged | — |
| AVDECC (1722.1) discovery/control | Retained unchanged | — |
| Stream Reservation Class latency targets | Relaxed | See [§7 Forwarding & QoS](#7-forwarding--qos) |
| Bandwidth limit: 75% of link | Retained as engineering rule | Enforced by controller, not switches |

---

## 5. Timing — AVB Lite PTP Profile

AVB Lite defines a media-oriented IEEE 1588 PTP profile that uses Layer-2 Ethernet transport and borrows the media-alignment goals of SMPTE ST 2059-2 where applicable. It is not a claim of strict SMPTE ST 2059-2 conformance.

Required profile behavior:

- **Protocol:** IEEE 1588 PTP. All devices in an AVB Lite timing domain must use this AVB Lite PTP profile.
- **Transport:** PTP must use Layer-2 Ethernet transport within an AVB Lite timing domain. UDP/IP PTP transport must not be used unless explicitly specified by a future profile extension.
- **Timescale:** Devices must use a common PTP timescale for media presentation. Grandmasters should provide valid UTC/TAI offset and time-property information when available.
- **Media alignment:** Audio sample clocks, video frame timing, and AVTP presentation times must be derived from the PTP-disciplined clock so independently transported streams can be phase-aligned at receivers.
- **Timestamping:** Hardware timestamping is mandatory at all endpoints, at NIC or PHY level. Switches are not required to be boundary clocks or transparent clocks.
- **Sync interval:** 125 ms, log -3, for Sync; 1 s for Announce.
- **Delay mechanism:** End-to-end delay measurement must be supported. Peer-to-peer delay measurement is outside the base profile.
- **Servo:** PI controller with outlier rejection. A Kalman filter is recommended on networks greater than 3 hops.
- **QoS marking for PTP:** VLAN ID 0 priority-tagged frames with 802.1p priority 7.
- **Best Master Clock Algorithm:** Standard BMCA, with configurable static `priority1` / `priority2` values to allow operators to pin the grandmaster.
- **Domain:** The PTP domain number must be configurable. All devices participating in the same AVB Lite media system must use the same PTP domain.

### Expected sync performance

| Topology | Expected offset, 1σ | Worst case |
|----------|----------------------|------------|
| Single switch, lightly loaded | < 200 ns | < 1 µs |
| 3 hops, prioritized PTP | 500 ns – 2 µs | 5 µs |
| 5 hops or congested | 2 – 10 µs | 50 µs |

---

## 6. Admission Control — CVU SRP

AVB Lite replaces bridge-processed MSRP with endpoint-processed MSRP-compatible signaling that is forwarded by non-AVB switches. The mechanism is named CVU SRP (Common Vendor Unique Stream Reservation Protocol).

All talkers and listeners operating in AVB Lite mode must support and send these messages regardless of any other ATDECC support.

Two CVU SRP messages are defined:

| CVU SRP message | Purpose |
|-----------------|---------|
| `cvu_srp_talker` | Carries MSRP Talker declarations, including Talker Advertise and Talker Failed. |
| `cvu_srp_listener` | Carries MSRP Listener declarations, including Listener Ready, Listener Ready Failed, and Listener Asking Failed. |

The payload of each CVU SRP message must contain the MSRP message content starting at the MSRP Attribute Type field and continuing through the End Mark of the MSRP message. The payload is therefore the MSRP attribute list and end mark, without requiring the MRP/MSRP link-local transport that non-AVB switches may block.

### Vendor-unique protocol identifier

CVU SRP messages are carried as IEEE 1722.1 AECP `VENDOR_UNIQUE_COMMAND` / `VENDOR_UNIQUE_RESPONSE` PDUs. The 6-byte (48-bit) `protocol_id` field is structured around an IEEE MA-S (36-bit) OUI:

| Bits  | Field | Value |
|-------|-------|-------|
| 47–12 | MA-S OUI | `0x8C1F6436C` |
| 11–0  | Sub-protocol identifier | `0x002` |

Encoded as 6 bytes on the wire: `8C 1F 64 36 C0 02`.

Endpoints must use this exact `protocol_id` for AVB Lite CVU SRP messages and must ignore vendor-unique messages with any other `protocol_id`.

Required behavior:

1. **Talker declaration:** A talker advertises available streams using `cvu_srp_talker` sent to the Ethernet broadcast destination address `ff:ff:ff:ff:ff:ff`, VLAN-tagged in the media VLAN (see [Stream transport addressing](#stream-transport-addressing), item 5).
2. **Listener declaration:** A listener sends `cvu_srp_listener` unicast to the source MAC address of the received talker declaration, VLAN-tagged in the media VLAN.
3. **Leave behavior:** Listener leave and talker withdrawal use the corresponding MSRP declaration semantics carried in the appropriate CVU SRP message.
4. **Payload compatibility:** MSRP attribute contents, declaration types, intervals, and timeout behavior should mirror MSRP where applicable.
5. **Admission rule:** A talker must not start or continue a stream if doing so would exceed 75% committed egress utilization on its transmitting interface.
6. **Refresh and timeout:** Talkers and listeners must refresh declarations using MSRP-like timing. Stale declarations must be aged out using MSRP-like timeout behavior.

Endpoints must rate-limit their own egress to the advertised stream rate. There is no in-network shaper to catch a misbehaving talker.

### Stream transport addressing

AVB Lite media streams are **unicast by default**. A talker must not transmit
a stream to a multicast destination address except through the escalation
path in item 3 below. On non-AVB switches, multicast-addressed streams are
flooded to every port of the broadcast domain (there is no MSRP pruning, and
IGMP snooping does not apply to non-IP L2 multicast), degrading every
attached device and any Wi-Fi segment. Unicast confines each stream to the
switch's learned path for that listener.

Required behavior:

1. **Unicast required by default:** A talker must transmit each admitted
   stream as unicast to the listener's MAC address, learned from the source
   MAC of that listener's `cvu_srp_listener` declaration. The AVTP
   `stream_id` is unchanged and remains the stream's identity; the
   destination MAC is transport addressing only.
2. **Multiple listeners:** A talker supporting more than one listener per
   stream duplicates the stream frames per listener, each unicast, up to a
   talker-defined fan-out limit (minimum 1, recommended 2). Bandwidth
   admission (§6.5) counts each copy against the 75% egress budget.
3. **Multicast escalation:** Beyond its unicast fan-out limit, a talker may
   escalate the stream to a multicast destination address (MAAP-allocated).
   The active destination MAC is always the one carried in the talker's
   `cvu_srp_talker` declaration; listeners must follow destination-address
   changes there and re-filter without tearing down the stream. Deployments
   that escalate to multicast must isolate the AVB Lite segment or confine
   the media VLAN at a managed-switch boundary.
4. **Listener behavior:** Listeners accept stream frames whose destination is
   either their own MAC or the advertised multicast address, matching streams
   by `stream_id`.
5. **Address learning:** Endpoints transmit their CVU SRP declarations
   VLAN-tagged in the media VLAN. Switches with independent (per-VLAN) MAC
   learning otherwise never observe a listener's source address inside the
   stream VLAN and flood the talker's unicast stream frames to every port as
   unknown-unicast, silently defeating unicast transport. The periodic CVU
   refresh doubles as the filtering-database keep-alive, well inside typical
   switch ageing times.

MAAP address acquisition is required only when multicast escalation is used.

---

## 7. Forwarding & QoS

AVB Lite uses the following VLAN and priority model:

- **PTP:** VLAN ID 0 priority-tagged frames with 802.1p priority 7.
- **CVU SRP messages:** VLAN-tagged in the media VLAN with 802.1p priority 5 (they double as the filtering-database keep-alive, see [Stream transport addressing](#stream-transport-addressing), item 5).
- **Other ATDECC messages:** untagged frames with no 802.1p priority requirement.
- **Media streams:** VLAN-tagged frames using VLAN ID 2 by default, or another configured VLAN ID, with 802.1p priority 5.

The media stream VLAN ID must be configurable. Untagged media-stream operation may be supported for simple networks, but 802.1p priority marking requires VLAN-tagged frames.

### Switch requirements, off-the-shelf

- 802.1p-based strict-priority queueing, minimum 4 queues.
- EEE (802.3az) disabled on all ports carrying media streams.
- Flow control (802.3x) disabled.
- Optional but recommended:
  - Per-port storm control.
  - Jumbo frames disabled on the media VLAN.
  - IGMP snooping (general multicast hygiene only — it does not contain
    AVTP L2 multicast; escalated streams need segment isolation or a
    managed-switch VLAN boundary, see
    [Stream transport addressing](#stream-transport-addressing)).

### Traffic classes

| Class | 802.1p | Queue | Use |
|-------|--------|-------|-----|
| PTP | 7 | Highest | Sync / Delay-Req / Delay-Resp |
| Media Class A | 5 | High | ≤ 2 ms latency streams |
| Media Class B | 5 | High | ≤ 10 ms latency streams |
| ATDECC / CVU SRP | N/A | Default | Discovery, enumeration, admission signaling |
| Best effort | 0 | Default | Everything else |

### Expected end-to-end latency

Talker media timestamp to listener presentation. Assumes gigabit links, ≤ 75% utilization on media queues, and hardware-timestamped endpoints.

| Hops | Class A target | Class B target |
|------|----------------|----------------|
| 1 | 1.0 ms | 5 ms |
| 3 | 1.5 ms | 6 ms |
| 5 | 2.0 ms | 8 ms |
| 7 | 2.5 ms | 10 ms |

Configured presentation-time offset should be set to the table value for the network's worst-case path.

---

## 8. Stream Format

AVB Lite does not define a media payload format. Stream formats are defined by the applicable audio, video, clock-reference, or application profile.

General stream requirements:

- Media streams use AVTP (IEEE 1722).
- Stream IDs and presentation time are retained from AVB operation.
- Presentation time in the AVTP header references the PTP-disciplined clock defined by this profile.
- Stream bandwidth and packet rate must be advertised to the AVB Lite Controller for admission control.

---

## 9. Failure Modes & Mitigations

| Failure | Behavior | Mitigation |
|---------|----------|------------|
| GM clock loss | BMCA elects new GM; brief offset transient | Servo holdover; 200 ms typical recovery |
| Switch congestion in media queue | Increased jitter, possible drops | Controller refuses admission past 75%; operator alarm |
| Endpoint misbehaving, over-rate | No in-network protection | Source-side rate limiter mandatory; controller monitors via SNMP/telemetry where available |
| EEE accidentally enabled | Sync degrades, audible artifacts | Controller probes for EEE via LLDP-MED; warns operator |
| PTP packets deprioritized by misconfigured switch | Sync collapses | Continuous offset monitoring; alarm at > 50 µs sustained |

---

## 10. Conformance

A device is AVB Lite conformant if it:

1. Implements the AVB Lite PTP profile over Layer-2 Ethernet, with hardware timestamping.
2. Marks PTP and media stream egress traffic with the VLAN and 802.1p values in [§7 Forwarding & QoS](#7-forwarding--qos).
3. Implements source rate limiting per advertised stream.
4. Supports the `cvu_srp_talker` and `cvu_srp_listener` messages defined in [§6 Admission Control — CVU SRP](#6-admission-control--cvu-srp) for stream advertisement and admission.
5. Holds presentation-time accuracy within ±2× the [§5 sync target](#expected-sync-performance) under nominal load.

A network is AVB Lite conformant if every switch in the media path meets the [§7 switch requirements](#switch-requirements-off-the-shelf) and is configured per the QoS table.
