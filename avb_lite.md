# AVB Lite Profile

A profile that gives AVB endpoints a working, if degraded, path over Ethernet switches that are not AVB bridges, unmanaged consumer switches included, using Layer 2 mechanisms alone.

Version: 1.0-draft

---

## 1. Scope

AVB Lite gives AVB endpoints a working, if degraded, path where the network has no AVB bridges. It keeps AVB's endpoint stack, with AVTP streams, stream IDs, presentation time, ATDECC and MSRP's declarations, and comes as close to AVB's behavior and performance as Layer 2 mechanisms allow over switches that take no part in AVB. AVB remains the way to its guaranteed performance; AVB Lite is the path an AVB endpoint takes when it finds no AVB bridge (§2).

Where the switches do none of a bridge's work, the endpoints do what they can of it, and the rest is lost:

- **Time:** an IEEE 1588 profile end to end, with hardware timestamps at the endpoints only. No switch measures its own delay as an 802.1AS bridge does, so synchronization is looser and depends on load (§5).
- **Reservation:** MSRP's declarations exchanged between endpoints, each of which admits streams against its own link (§6). Links between switches are not checked.
- **Shaping:** pacing at the talker and, where the switch offers it, 802.1p priority (§7). Without the bridges' credit-based shapers, latency stays bounded only while priority holds or best-effort traffic stays light.

AVB Lite runs on two kinds of network (§7):

- **Baseline:** any Ethernet switch, unmanaged consumer switches included, as it comes. Streams are admitted, confined and synchronized as on an enhanced network; latency stays within the §7 targets while best-effort traffic on media ports stays light.
- **Enhanced:** managed switches set up as §7 describes, which keep the latency targets under load.

Endpoints and switches may link at any speed of 100 Mb/s or more, at different speeds in one network (§5 corrects for the difference). Only Layer 2 mechanisms are used: no IP, IGMP or switch management protocol is required.

An AVB Lite system is a single L2 domain. Its size is bounded by bandwidth and by PTP's message load, not by a count of endpoints or streams:

- **Unicast streams** use only the links on their path. Each endpoint admits what crosses its own link (§6). An eight-channel 48 kHz AAF stream takes about 17 Mb/s on the wire, so a 100 Mb/s endpoint sends or receives about four, a 1 Gb/s endpoint about 45. Links between switches carry every stream that crosses them and no endpoint checks them; the operator keeps their media within 75%.
- **Escalated multicast streams** (§6) are flooded to every port in the domain by non-AVB switches, so together they must fit within 75% of the slowest port in the domain, endpoints that do not listen to them included: about four eight-channel streams where a 100 Mb/s device is present. No admission rule checks this, so escalation is off unless an operator allows it where the media VLAN is confined (§6).
- **PTP** grows with the number of endpoints at the grandmaster, which timestamps and answers one `Delay_Req` per follower per second. With unicast delay requests (§5) every other endpoint receives only the grandmaster's 17 Sync, Follow_Up and Announce messages a second (two-step); where followers send them multicast, every endpoint also receives each follower's `Delay_Req` and its `Delay_Resp`, two more messages a second per follower. A grandmaster-capable endpoint should state how many followers it can serve, and the smallest such figure among the endpoints that may become grandmaster bounds the domain.

---

## 2. Automatic AVB Lite Fallback

An AVB endpoint that also supports AVB Lite must automatically fall back to AVB Lite when it detects that it is not connected through an AVB-capable bridge.

### 2.1 Endpoint Declaration TLV

To enable peer endpoints to detect each other across a non-AVB bridge that floods the bridge-group multicast (`01:80:C2:00:00:0E`), every AVB Lite-capable endpoint shall append an organization extension TLV (IEEE 1588-2019, 14.3) to every `Pdelay_Req`, `Pdelay_Resp`, and `Pdelay_Resp_Follow_Up` it transmits, with the following content:

| Field | Width | Value |
|-------|-------|-------|
| `tlvType` | 2 bytes | `0x8000` (ORGANIZATION_EXTENSION_DO_NOT_PROPAGATE, IEEE 1588-2019, 14.3.2.1) |
| `lengthField` | 2 bytes | `0x0008` (length of `organizationId` + `organizationSubType` + `dataField` + `pad` = 3 + 3 + 1 + 1) |
| `organizationId` | 3 bytes | `8C:1F:64` (24-bit MA-L prefix of the MA-S OUI `0x8C1F6436C`) |
| `organizationSubType` | 3 bytes | `36:C0:01` (top 12 bits = MA-S suffix `0x36C`, bottom 12 bits = AVB Lite TLV sub-id `0x001` for Endpoint Declaration) |
| `dataField` | 1 byte | `0x01` = "I am an endpoint, not a bridge" |
| `pad` | 1 byte | `0x00`, ignored by receivers |

IEEE 1588 requires every TLV's `lengthField` to be even, and implementations enforce it: linuxptp's ptp4l, for one, discards a whole message carrying a TLV with an odd `lengthField`. The `pad` octet keeps the TLV even. IEEE 1588-2019 keeps the ORGANIZATION_EXTENSION type `0x0003` only for extensions that predate it, and requires new ones to use `0x4000` or `0x8000` (14.3.2.1), as IEEE 802.1AS-2020 does for its own new TLVs. The TLV is of the do-not-propagate type: it describes the link it is sent on.

Earlier revisions of this profile gave `tlvType` as `0x0003`, and `lengthField` as `0x0007` with no `pad`. Receivers shall recognize the TLV with a `tlvType` of `0x8000` or `0x0003` and a `lengthField` of 7 or more, reading `dataField` at the same offset, so that endpoints of every revision recognize each other.

The TLV belongs in gPTP-mode Pdelay messages as well as in the §2.3 beacon. Two AVB Lite endpoints joined by a switch that floods `01:80:C2:00:00:0E` each see exactly one Pdelay responder, the other endpoint, so neither condition 2 nor condition 3 of §2.2 fires; the TLV is then the only sign that no AVB bridge is between them.

A compliant 802.1AS bridge terminates the Pdelay protocol at its boundary clock and re-originates `Pdelay_Req`, `Pdelay_Resp`, and `Pdelay_Resp_Follow_Up` from its own port without copying endpoint-originated TLVs, so the Endpoint Declaration TLV does not survive transit through an AVB-aware bridge. Per IEEE 1588-2019, 14.1 and 14.3.2.3, recipients that do not recognize a TLV must ignore it; this preserves backward compatibility with non-AVB-Lite-aware bridges and endpoints.

### 2.2 Fallback Conditions

An endpoint shall arm fallback detection on every Ethernet link-up event. Three conditions are evaluated in parallel and continuously for the lifetime of the link. The first to fire triggers fallback to AVB Lite:

1. **Endpoint TLV detected.** A `Pdelay_Req`, `Pdelay_Resp`, or `Pdelay_Resp_Follow_Up` arrives carrying the Endpoint Declaration TLV defined in §2.1. This indicates that the sender of the message is also an endpoint (no AVB-aware bridge sits between the two devices).
2. **Nine consecutive `Pdelay_Req` attempts unanswered.** Mirrors the 802.1AS-2020 §11.5.3 `allowedLostResponses` default of 9: the same counter that flips `asCapable` to false at the gPTP layer also triggers fallback at the profile layer. The counter is reset to zero on every received `Pdelay_Resp`. This indicates either an isolated link or a network with no compliant peer. (Earlier revisions of this profile used 3, the 802.1AS-2011 value; field experience showed a transient ~3 s ingress loss on a healthy AVB link could spring the fallback latch, which §2.2.1 then makes permanent until a link-flap.)
3. **Two or more distinct responders to a single `Pdelay_Req`.** For any single Pdelay cycle, if responses arrive from two or more distinct `sourcePortIdentity` values, fallback fires. A spec-compliant AVB bridge presents exactly one boundary-clock peer per port, so cardinality > 1 to a single request indicates a non-compliant L2 substrate (e.g. a switch flooding the bridge-group multicast `01:80:C2:00:00:0E`).

A switch that filters `01:80:C2:00:00:0E`, as IEEE 802.1Q bridges and many managed switches do, delivers no Pdelay responses at all; condition 2 then fires about 9 s after link-up, and neither the TLV nor the §2.3 beacon crosses that switch. Switches that flood the address and switches that filter it are both handled without configuration.

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

When falling back, the device must clearly advertise its active operating mode to the controller, through the status query defined in [§2.4](#24-mode-reporting), and must not use MSRP reservations for AVB Lite streams. AVB Lite streams must use the timing, QoS, admission-control, and conformance requirements defined by this profile.

A device should prefer standard IEEE 802.1BA operation when none of the §2.2 conditions is met, unless configured otherwise by the operator or controller.

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

### 2.4 Mode Reporting

Every AVB Lite-capable endpoint shall report its operating mode to controllers through the AVB Lite status query, an IEEE 1722.1 AECP `VENDOR_UNIQUE_COMMAND` / `VENDOR_UNIQUE_RESPONSE` exchange under the same MA-S OUI as CVU SRP (§6), with sub-protocol identifier `0x003`:

| Bits  | Field | Value |
|-------|-------|-------|
| 47–12 | MA-S OUI | `0x8C1F6436C` |
| 11–0  | Sub-protocol identifier | `0x003` |

Encoded as 6 bytes on the wire: `8C 1F 64 36 C0 03`.

The command and the response start with the same fields after `protocol_id`:

| Offset | Length | Field | Value |
|--------|--------|-------|-------|
| 0 | 2 | `u` (bit 15), `command_type` (bits 14–0) | `u` is 1 in an unsolicited response; `command_type` is `0x0000`, GET_LITE_STATUS (`0x0001` is SET_LITE_CONFIG, below) |
| 2 | 2 | `avb_interface_index` | The AVB_INTERFACE descriptor the status is for |

The command carries nothing more. The response continues:

| Offset | Length | Field | Value |
|--------|--------|-------|-------|
| 4 | 1 | `flags` | Bit 0 CAPABLE: the endpoint supports AVB Lite. Bit 1 ACTIVE: the interface operates in AVB Lite. Bit 2 OFFSET_VALID: `offset_from_grandmaster` holds a measurement. Bit 3 EGRESS_VALID: `committed_egress` is kept. Bit 4 ESCALATION_ALLOWED: talkers on the interface may escalate streams to multicast (§6). Other bits zero. |
| 5 | 1 | `fallback_reason` | Why the interface operates in AVB Lite: 0 none (standard AVB), 1 an Endpoint Declaration TLV was received (§2.2 condition 1), 2 nine `Pdelay_Req` went unanswered (condition 2), 3 two or more responders answered one `Pdelay_Req` (condition 3), 4 configured by the operator or a controller |
| 6 | 1 | `ptp_profile` | 0 IEEE 802.1AS gPTP, 1 the AVB Lite PTP profile (§5) |
| 7 | 1 | `ptp_domain` | The PTP domain number |
| 8 | 2 | `media_vlan_id` | The media VLAN (§7), 0 when media frames carry priority only or go untagged |
| 10 | 1 | `unicast_fanout_limit` | The most listeners of a stream the talker serves unicast; beyond it the talker escalates, where allowed, or refuses (§6). 0 for a listener-only endpoint |
| 11 | 1 | reserved | 0 |
| 12 | 4 | `link_speed` | The interface's negotiated link speed in Mb/s, following renegotiation, 0 when not known |
| 16 | 4 | `committed_egress` | Bandwidth of the admitted streams the interface sends, in kb/s, each unicast copy counted, as the §6 admission rule counts it |
| 20 | 8 | `grandmaster_identity` | The clock identity of the grandmaster the endpoint follows |
| 28 | 4 | `offset_from_grandmaster` | The servo's last offset from the grandmaster in nanoseconds, signed |

The response's `status` is SUCCESS, NO_SUCH_DESCRIPTOR for an `avb_interface_index` the endpoint does not have, or NOT_IMPLEMENTED from an endpoint without AVB Lite support. An endpoint shall answer within the AECP command timeout.

An endpoint shall send the response unsolicited, with `u` set, to each controller registered for unsolicited notifications (IEEE 1722.1-2021 REGISTER_UNSOLICITED_NOTIFICATION) whenever any field other than `offset_from_grandmaster` changes, and whenever the offset crosses the §9 alarm threshold of 50 µs in either direction. Controllers wanting to follow the offset more closely poll the query; once every 5 s is enough for the §9 alarm.

#### Configuration

The same protocol carries SET_LITE_CONFIG, `command_type` `0x0001`, with which a controller sets how an interface runs AVB Lite. The command and the response carry:

| Offset | Length | Field | Value |
|--------|--------|-------|-------|
| 0 | 2 | `u` (bit 15), `command_type` (bits 14–0) | `u` 0; `command_type` `0x0001`, SET_LITE_CONFIG |
| 2 | 2 | `avb_interface_index` | The AVB_INTERFACE descriptor to configure |
| 4 | 1 | `config_flags` | Bit 0 ESCALATION_ALLOWED: talkers on the interface may escalate streams to multicast (§6). Other bits zero. |
| 5 | 3 | reserved | 0 |

The response carries the configuration the interface has after the command. Its `status` is SUCCESS; NO_SUCH_DESCRIPTOR for an `avb_interface_index` the endpoint does not have; BAD_ARGUMENTS when a reserved bit is set; ENTITY_ACQUIRED or ENTITY_LOCKED when another controller has acquired or locked the entity, as for AEM's SET commands; or NOT_IMPLEMENTED from an endpoint without AVB Lite support. The configuration persists across restarts, and escalation is not allowed until a controller or the operator allows it. A change is reported with an unsolicited GET_LITE_STATUS response.

### 2.5 Start-up Sequences (informative)

This section shows how §2.2 and §2.3 play out after a link-up, and how an endpoint can keep start-up short. It adds no requirements.

#### What sets the pace

- **The decision.** Conditions 1 and 3 fire on what an endpoint hears: an Endpoint Declaration TLV, or a second responder to one `Pdelay_Req`. With `Pdelay_Req` sent once a second, that is within the first Pdelay cycle when peers are in gPTP mode, or within 3 s when a peer that has fallen back announces itself only through its §2.3 beacon. Condition 2 waits out nine unanswered requests, about 9 s, and is the slowest path.
- **The PTP start.** Whichever profile the endpoint ends up in, its PTP must then reach the grandmaster: in gPTP, a few Pdelay exchanges before the port is asCapable, then the grandmaster's Announce and Sync; in the AVB Lite PTP profile, the grandmaster's Announce and Sync with no peer delay at all. Either takes a few seconds, depending on the implementation; call these times *T*<sub>gPTP</sub> and *T*<sub>Lite</sub>.
- **A switch of profile.** Moving between gPTP and the AVB Lite PTP profile starts the PTP from scratch, so each switch costs that start again. An endpoint that switches profiles twice during start-up pays twice.

#### The cases

Each row starts from a link-up with the endpoint in gPTP, as §2.2 arms it.

| Network at link-up | What the endpoint hears | Decision | In sync after |
|---|---|---|---|
| An 802.1AS bridge | One responder to each `Pdelay_Req`, no TLV; the bridge also asks for peer delay itself, once a second | Stays in gPTP | *T*<sub>gPTP</sub> |
| A non-AVB switch, with an AVB Lite endpoint that has already fallen back | Its beacon | Condition 1, within 3 s | at most 3 s + *T*<sub>Lite</sub> |
| A non-AVB switch, with AVB Lite endpoints starting together | Each other's `Pdelay_Req` and `Pdelay_Resp` carrying the TLV | Condition 1, within about 1 s | about 1 s + *T*<sub>Lite</sub> |
| A non-AVB switch, with two or more gPTP devices that are not AVB Lite endpoints | Two or more responders to one `Pdelay_Req` | Condition 3, within about 1 s | about 1 s + *T*<sub>Lite</sub> |
| A non-AVB switch, with exactly one gPTP device that is not an AVB Lite endpoint, and no AVB Lite endpoint | One responder, no TLV | Stays in gPTP, that device its peer; a later beacon still brings the fallback | *T*<sub>gPTP</sub> |
| A switch that filters `01:80:C2:00:00:0E`, or nothing else on the link | No responder | Condition 2, after about 9 s | about 9 s + *T*<sub>Lite</sub> |

The third row is why an endpoint in gPTP mode carries the TLV in its own Pdelay messages (§2.1): endpoints that start together find each other within one Pdelay cycle instead of waiting for a beacon.

#### Keeping start-up short

1. **Start gPTP at link-up rather than waiting to decide.** The conditions are evaluated on gPTP's own Pdelay exchange, so behind a bridge no time is lost, and on a non-AVB switch the evidence usually arrives within the first second or two.
2. **Act on evidence at once.** A TLV or a second responder needs no further waiting; only condition 2 waits out its window.
3. **Resume rather than restart.** An endpoint re-armed while it already runs the AVB Lite PTP profile, as after a restart of its PTP (§2.2.1) or a cable moved from one switch to another, need not leave the profile to evaluate. A beacon or other TLV keeps it there. An AVB bridge's port asks for peer delay once a second, so a single gPTP peer asking without the TLV points to a bridge; two or more asking show a switch flooding `01:80:C2:00:00:0E`, which the endpoint may treat as condition 3; and none asking within about 9 s is condition 2's silence. Starting gPTP as soon as that single peer is heard, while still listening for a beacon, overlaps *T*<sub>gPTP</sub> with the rest of the beacon interval. Behind a bridge the endpoint is then in sync about 1 s + *T*<sub>gPTP</sub> after re-arming, instead of a full beacon interval followed by *T*<sub>gPTP</sub>.
4. **Correct a wrong guess at once.** If an endpoint that moved to gPTP early then hears a TLV, it returns to the AVB Lite PTP profile immediately, not after any rate limit it applies to profile switches; the §2.2 latch then holds it there.

#### Example timelines

Behind an 802.1AS bridge:

```
0 s        link-up; gPTP starts; fallback evaluation armed
within 1 s the bridge's Pdelay_Req heard and answered; the endpoint's
           own first Pdelay_Req answered by one responder, no TLV
a few s    asCapable; the bridge's Announce and Sync; grandmaster chosen
T(gPTP)    in sync; still in gPTP
```

On a non-AVB switch whose grandmaster is an AVB Lite endpoint that has fallen back:

```
0 s        link-up; gPTP starts; fallback evaluation armed
within 3 s the grandmaster's beacon carries the TLV: condition 1
           PTP restarts in the AVB Lite PTP profile; CVU SRP declarations
+T(Lite)   in sync; link-speed correction from the Grandmaster Link TLV
```

Resuming after a restart of the endpoint's PTP, its last profile the AVB Lite PTP profile, with the cable now on an 802.1AS bridge:

```
0 s        re-armed while still in the AVB Lite PTP profile
within 1 s the bridge asks for peer delay, no TLV: one gPTP peer
           gPTP starts at once, while listening for a beacon
3 s        no beacon: no AVB Lite endpoint has fallen back on this link
1 s+T(gPTP) in sync in gPTP
```

---

## 3. Bridging Between AVB and AVB Lite Domains

A system may bridge traffic between a standard AVB domain and an AVB Lite domain using an explicit gateway or boundary device.

The gateway is responsible for translating domain assumptions, including:

- Clock-domain adaptation between 802.1AS/gPTP and the AVB Lite PTP profile.
- Admission-control translation between MSRP on the AVB side and CVU SRP declarations (§6) on the AVB Lite side.
- QoS mapping between AVB SR classes and AVB Lite 802.1p traffic classes.
- VLAN ID translation where the AVB and AVB Lite domains use different VLAN IDs or where one side is untagged.
- Stream lifecycle coordination so that connection state is consistent on both sides.
- Presentation-time offset adjustment for the combined AVB and AVB Lite path latency.

Bridging must be explicit. Endpoints must not assume that AVB and AVB Lite domains are directly interoperable without a gateway that provides these translation functions. VLAN identity is local to each domain and must not be assumed to be the same across the gateway.

---

## 4. Differences From 802.1BA-2011

| 802.1BA component | Over non-AVB switches | What AVB Lite does |
|-------------------|-----------------------|--------------------|
| 802.1AS (gPTP) | Not available: switches neither terminate nor timestamp PTP | AVB Lite PTP profile end to end over Layer-2 Ethernet, with hardware timestamping at endpoints (§5) |
| 802.1Qat (MSRP) | Not available: no bridge registers declarations or checks its ports | CVU SRP (§6): the same declarations exchanged between endpoints in AECP; talkers check their egress and listeners their ingress |
| 802.1Qav (FQTSS / credit-based shaper) | Not available | Pacing at the talker (§6) and, on enhanced networks, strict-priority queueing via 802.1p (§7) |
| 802.1Q VLAN / SR class A & B | Kept, adjusted | Media streams use a configurable VLAN ID, default 2, or 0 for priority only; Class A and Class B → 802.1p priority 5 |
| AVTP (1722) stream format | Unchanged | — |
| AVDECC (1722.1) discovery/control | Unchanged | — |
| Stream Reservation Class latency targets | Relaxed | See [§7 Forwarding & QoS](#7-forwarding--qos) |
| Bandwidth limit: 75% of link | Kept | Checked by talkers and listeners on their own links (§6), not by switches |

---

## 5. Timing — AVB Lite PTP Profile

AVB Lite defines a media-oriented IEEE 1588 PTP profile that uses Layer-2 Ethernet transport and borrows the media-alignment goals of SMPTE ST 2059-2 where applicable. It is not a claim of strict SMPTE ST 2059-2 conformance.

Required profile behavior:

- **Protocol:** IEEE 1588 PTP. All devices in an AVB Lite timing domain must use this AVB Lite PTP profile.
- **Transport:** PTP must use Layer-2 Ethernet transport within an AVB Lite timing domain, to `01:1B:19:00:00:00`, with `transportSpecific` 0. UDP/IP PTP transport must not be used unless explicitly specified by a future profile extension.
- **Timescale:** Devices must use a common PTP timescale for media presentation. Grandmasters should provide valid UTC/TAI offset and time-property information when available.
- **Media alignment:** Audio sample clocks, video frame timing, and AVTP presentation times must be derived from the PTP-disciplined clock so independently transported streams can be phase-aligned at receivers.
- **Timestamping:** Hardware timestamping is mandatory at all endpoints, at NIC or PHY level. Switches are not required to be boundary clocks or transparent clocks.
- **Sync interval:** 125 ms, log -3, for Sync; 1 s for Announce. Sync must be sent on that grid on average, not merely advertised with it.
- **Delay request interval:** 1 s, log 0, as both the least and the most. A grandmaster shall advertise `logMessageInterval` 0 in every `Delay_Resp`, and followers send `Delay_Req` no faster than that on average (IEEE 1588-2019, 7.7.2.4), so the grandmaster answers one `Delay_Req` per follower a second, the load §1 counts.
- **One-step and two-step:** A clock may be either; endpoints must follow both.
- **Delay mechanism:** End-to-end delay measurement must be supported. Peer-to-peer delay measurement is outside the base profile.
- **Unicast delay requests:** A grandmaster-capable endpoint shall support mixed multicast/unicast operation (IEEE 1588-2019, 16.9), answering a unicast `Delay_Req` with a unicast `Delay_Resp`. A follower should send its `Delay_Req` unicast, to the MAC address the grandmaster's Announce came from, and send them multicast again if no `Delay_Resp` answers. Announce, Sync and Follow_Up stay multicast. Endpoints then no longer receive every other endpoint's delay exchange.
- **Servo:** PI controller with outlier rejection. A Kalman filter is recommended on networks greater than 3 hops.
- **QoS marking for PTP:** Endpoints should send PTP priority-tagged, VLAN ID 0 with 802.1p priority 7, and must accept PTP both tagged that way and untagged. A platform that cannot tag PTP sends it untagged; on an enhanced network its PTP then waits in the default queue, and its sync degrades under load. A switch may also remove the tag, so the priority can count at the first switch only ([Priority tags](#priority-tags)).
- **Best Master Clock Algorithm:** Standard BMCA. Default `priority1` values follow IEEE 802.1AS-2020, Table 8-1: 246 for an endpoint its maker intends as grandmaster, 248 for other grandmaster-capable endpoints that stay on the network, 250 for transient ones such as laptops, and 255 for an endpoint that cannot be grandmaster, which then never becomes one. Default `priority2` is 248. Operators may set `priority1` and `priority2` to pin the grandmaster.
- **Domain:** The PTP domain number must be configurable, default 0. All devices participating in the same AVB Lite media system must use the same PTP domain.

### Link-speed asymmetry

End-to-end delay measurement assumes a message takes as long in each direction. A store-and-forward switch holds each frame for its length at the speed of the link it arrived on, so where the grandmaster and an endpoint link at different speeds the two directions differ. Sync and `Delay_Req` are minimum-size frames of the same length L, 64 octets untagged or 66 priority-tagged. The holding times at the links in between then cancel, whatever the number of hops, and the difference comes from the two end links alone:

`delayAsymmetry = 4 × L × (1 / R_gm − 1 / R_local)` seconds, with L in octets and the link speeds R in bit/s, and `delayAsymmetry` as IEEE 1588-2019, 7.4.2 defines it: positive when the master-to-slave direction takes longer.

An endpoint at 1 Gb/s following a grandmaster at 100 Mb/s would otherwise settle 2.3 µs off (L = 64), whatever lies between them. With link speeds of 100 Mb/s or more the error stays below `4 × L / 100 Mb/s`, 2.6 µs.

A grandmaster announces its link speed in a Grandmaster Link TLV appended to every Announce it sends:

| Field | Width | Value |
|-------|-------|-------|
| `tlvType` | 2 bytes | `0x8000` (ORGANIZATION_EXTENSION_DO_NOT_PROPAGATE, IEEE 1588-2019, 14.3.2.1) |
| `lengthField` | 2 bytes | `0x000A` (3 + 3 + 4) |
| `organizationId` | 3 bytes | `8C:1F:64` |
| `organizationSubType` | 3 bytes | `36:C0:04` (AVB Lite sub-id `0x004`, Grandmaster Link) |
| `linkSpeed` | 4 bytes | The grandmaster's negotiated link speed in Mb/s |

The TLV is of the do-not-propagate type, as IEEE 1588-2019 requires of new organization extensions: a boundary clock that does not know it passes no stale link speed on, and its followers, whose master it then is, apply no correction for a grandmaster beyond it.

An endpoint following a grandmaster whose Announce carries the TLV shall apply the `delayAsymmetry` above, recomputing it when either link speed changes; without the TLV it applies none. Where a switch cuts frames through from the faster link to the slower one rather than storing them, at most `4 × L / R_fast` stays uncorrected, 0.26 µs at 1 Gb/s.

### Expected sync performance

For endpoints on enhanced networks, and on baseline networks under light load, with link speeds equal or corrected as above:

| Topology | Expected offset, 1σ | Worst case |
|----------|----------------------|------------|
| Single switch, lightly loaded | < 200 ns | < 1 µs |
| 3 hops, prioritized PTP | 500 ns – 2 µs | 5 µs |
| 5 hops or congested | 2 – 10 µs | 50 µs |

On a baseline network PTP shares queues with media and best-effort traffic. A frame waiting ahead of a Sync or `Delay_Req` delays it by that frame's length, 123 µs for a full-size frame at 100 Mb/s; the servo's outlier rejection removes most but not all of that, and the §9 alarm reports what remains.

---

## 6. Admission Control — CVU SRP

With no bridge to register MSRP declarations, AVB Lite endpoints exchange the same declarations directly, in messages that non-AVB switches forward. The mechanism is named CVU SRP (Common Vendor Unique Stream Reservation Protocol).

All talkers and listeners operating in AVB Lite mode must support and send these messages regardless of any other ATDECC support.

Two CVU SRP messages are defined:

| CVU SRP message | Purpose |
|-----------------|---------|
| `cvu_srp_talker` | Carries MSRP Talker declarations, including Talker Advertise and Talker Failed. |
| `cvu_srp_listener` | Carries MSRP Listener declarations, including Listener Ready, Listener Ready Failed, and Listener Asking Failed. |

Each CVU SRP message carries one MSRP message, from its Attribute Type field through its End Mark, after a one-octet kind ([Message format](#message-format)), so MSRP's declarations travel without the link-local MRP transport that non-AVB switches may block.

### Vendor-unique protocol identifier

CVU SRP messages are carried as IEEE 1722.1 AECP `VENDOR_UNIQUE_COMMAND` PDUs. The 6-byte (48-bit) `protocol_id` field is structured around an IEEE MA-S (36-bit) OUI:

| Bits  | Field | Value |
|-------|-------|-------|
| 47–12 | MA-S OUI | `0x8C1F6436C` |
| 11–0  | Sub-protocol identifier | `0x002` |

Encoded as 6 bytes on the wire: `8C 1F 64 36 C0 02`.

Endpoints must use this exact `protocol_id` for AVB Lite CVU SRP messages and must ignore vendor-unique messages with any other `protocol_id`.

### Required behavior

1. **Talker declaration:** A talker advertises available streams using `cvu_srp_talker` sent to the Ethernet broadcast destination address `ff:ff:ff:ff:ff:ff`, VLAN-tagged in the media VLAN (see [Stream transport addressing](#stream-transport-addressing), item 5). It sends one unicast only to refuse a listener (item 10).
2. **Listener declaration:** A listener sends `cvu_srp_listener` unicast to the source MAC address of the received talker declaration, VLAN-tagged in the media VLAN.
3. **Leave behavior:** Listener leave and talker withdrawal use the corresponding MSRP declaration semantics carried in the appropriate CVU SRP message.
4. **Payload compatibility:** MSRP attribute contents and declaration types follow MSRP, as [Message format](#message-format) and [Declaration content](#declaration-content) set out.
5. **Admission rule, talker:** A talker must not start or continue a stream if doing so would exceed 75% committed egress utilization on its transmitting interface, each unicast copy counted. It refuses the listener that would take it past, with failure code 1, Insufficient bandwidth (item 10).
6. **Admission rule, listener:** A listener must not declare Ready for a stream if receiving it would take the streams it is Ready for, each counted by its talker's TSpec, past 75% of its link; it declares Asking Failed instead. The switch port toward a listener is where streams from several talkers meet, and no bridge checks it. Links between switches are not visible to endpoints: the operator keeps the media they carry within 75% (§7).
7. **Refresh and timeout:** Talkers and listeners send, refresh, withdraw and age out declarations as [Declaration timing](#declaration-timing) sets out.
8. **Pacing:** A talker sends no more than `MaxIntervalFrames` frames of each copy of a stream in each class measurement interval (125 µs for Class A, 250 µs for Class B), spread evenly rather than back to back, and interleaves the copies and streams it sends. With no shaper in the network, the talker's own pacing is what keeps a stream's frames from arriving at a switch in bursts.
9. **Endpoint link:** An endpoint must not advertise Energy-Efficient Ethernet (IEEE 802.3az) on its own link, and must neither advertise nor act on PAUSE (IEEE 802.3x flow control). An endpoint controls only its own link, but that keeps EEE wake-up delays and pause stalls off every endpoint link, through an unmanaged switch as well.
10. **Refusing a listener:** A talker that cannot serve a listener tells that listener alone, as an MSRP bridge turns Talker Advertise into Talker Failed only on the port toward a listener it cannot serve. It sends the listener's MAC address a `cvu_srp_talker` declaration of Talker Failed for the stream, carrying a failure code from IEEE 802.1Q-2022 Table 46-15, while its broadcast declaration stays Talker Advertise for every other listener. For the listener it is sent to, a talker's unicast declaration takes precedence over its broadcast one, until the talker withdraws it with `Lv` or it ages out ([Declaration timing](#declaration-timing)). Once it can serve the listener, the talker withdraws the unicast declaration. The listener reports the failure code as it would an MSRP failure.

There is no in-network shaper or policer to catch a misbehaving talker.

### Message format

A CVU SRP message is an AECP `VENDOR_UNIQUE_COMMAND` whose header carries:

| Field | Value |
|-------|-------|
| `target_entity_id` | 0: the message is addressed by MAC address, not to an entity |
| `controller_entity_id` | The entity ID of the endpoint declaring |
| `sequence_id` | One more for each CVU SRP message the endpoint sends |
| `protocol_id` | `8C 1F 64 36 C0 02` |

and whose payload is:

| Offset | Length | Field | Value |
|--------|--------|-------|-------|
| 0 | 1 | `message_kind` | The MSRP attribute type of the declarations carried: 1 Talker Advertise or 2 Talker Failed (`cvu_srp_talker`), 3 Listener (`cvu_srp_listener`) |
| 1 | n | MSRP message | One MSRP message of that attribute type, from its AttributeType through its own EndMark: the MRPDU's ProtocolVersion before it and the MRPDU's final EndMark after it are left out. It may carry several streams' declarations. |

CVU SRP commands are not answered. A `target_entity_id` of 0 names no entity, so IEEE 1722.1-2021's Vendor Unique entity state machine (9.6.5) would neither process nor answer them; endpoints take them by `protocol_id` outside that state machine, and send no response. MRP has no acknowledgement either: the repetition and refresh of [Declaration timing](#declaration-timing) carry reliability. Receivers must ignore `VENDOR_UNIQUE_RESPONSE` messages with this `protocol_id`, which earlier implementations sent.

### Declaration content

Talker declarations carry MSRP's DataFrameParameters and TSpec as follows:

- **`destination_address`:** where the stream's frames go now. `00:00:00:00:00:00` while the talker sends the stream unicast, to each listener's own address; the stream's multicast address (MAAP-allocated) once it has escalated ([Stream transport addressing](#stream-transport-addressing) item 3). Listeners, controllers and other endpoints tell the mode from it. A change between the two is a new declaration: the talker sends it with MRP's New event, at once and again JoinTime later, and receivers replace the stream's registration with it. (An MSRP bridge refuses such a change for a registered stream, IEEE 802.1Q-2022 Table 46-15 failure code 16; between AVB Lite endpoints the new declaration replaces the old.)
- **`vlan_identifier`:** the media VLAN (§7).
- **`MaxFrameSize`, `MaxIntervalFrames`:** those of one copy of the stream.
- **`PriorityAndRank`:** priority 5 (§7), whatever SR class the stream belongs to; rank 1, or 0 for an emergency stream.
- **`AccumulatedLatency`:** the talker's own latency, as in MSRP. No switch adds to it, so it does not describe the path; presentation offsets come from the §7 latency targets.
- **Talker Failed:** MSRP's failure codes. The Bridge ID of the FailureInformation holds the failing endpoint's MAC address, preceded by a priority of 0.

Listener declarations carry the stream ID and MSRP's four listener states: Ignore, Asking Failed, Ready and Ready Failed.

### Declaration timing

CVU SRP keeps MRP's timers (IEEE 802.1Q-2022, Table 10-7 and 10.7.11) and what they do, carried over from a shared link to messages between endpoints:

| Timer | Value | Use in CVU SRP |
|-------|-------|----------------|
| JoinTime | 200 ms | A new or changed declaration is sent at once and again 200 ms later, as an MRP applicant makes sure two Joins have gone out. |
| LeaveTime | 600–1000 ms | A receiver removes a declaration this long after its `Lv`, unless a Join for it arrives first, as an MRP registrar does. A withdrawn declaration is sent once with `Lv`. |
| LeaveAllTime | 10 s | Each endpoint sends all its declarations again, twice, JoinTime apart, each time a timer started at a random value from LeaveAllTime to 1.5 × LeaveAllTime (10 to 15 s) expires, as an MRP participant redeclares after a LeaveAll. |
| Age-out | 16 s | A receiver removes a declaration that has not been sent again within 1.5 × LeaveAllTime + LeaveTime, the longest MRP keeps a registration whose declarant has fallen silent. |

MRP's PeriodicTime is not used. A 10 to 15 s refresh also stays well inside switches' address ageing (300 s by default), which [Stream transport addressing](#stream-transport-addressing) item 5 relies on.

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
3. **Multicast escalation:** Beyond its unicast fan-out limit, a talker
   escalates the stream to a multicast destination address (MAAP-allocated)
   only where escalation is allowed on its interface (§2.4, Configuration),
   which it is not by default. Otherwise it refuses each further listener
   with failure code 2, Insufficient Bridge resources (§6 item 10). An
   escalated stream returns to unicast once its Ready listeners fit within
   the fan-out limit again. The talker's `cvu_srp_talker` declaration always shows the mode
   ([Declaration content](#declaration-content)): `00:00:00:00:00:00` while
   the stream goes unicast, the multicast address once escalated. To
   escalate, a talker declares the multicast address and moves the stream's
   frames to it JoinTime later, by when its listeners have the declaration;
   to return to unicast, it moves the frames back to each listener's own
   address first and then declares `00:00:00:00:00:00`. Listeners follow the
   declared address and re-filter without tearing down the stream.
   Escalation should be allowed only where the media VLAN is confined, on an
   isolated segment or at a managed-switch boundary: a non-AVB switch floods
   an escalated stream to every port in the domain.
4. **Listener behavior:** Listeners accept stream frames sent to their own
   MAC address, and to the multicast address while the talker declares one,
   matching streams by `stream_id`.
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

### VLAN and priority model

- **PTP:** priority-tagged, VLAN ID 0 with 802.1p priority 7, where the endpoint can tag it (§5); otherwise untagged.
- **Media streams:** VLAN-tagged in the media VLAN with 802.1p priority 5.
- **CVU SRP messages:** tagged as media streams are, in the media VLAN with priority 5, so that switches learn listeners' addresses where the streams go ([Stream transport addressing](#stream-transport-addressing), item 5).
- **Other ATDECC messages and MAAP:** untagged, as in AVB, with no 802.1p priority requirement.

The media VLAN ID must be configurable, and every AVB Lite device in a system uses the same one:

- **2 (default):** as in AVB. Unmanaged switches pass VLAN 2 frames unchanged. Managed switches must carry VLAN 2 on media ports, or they drop the frames; in return the VLAN keeps media apart from other traffic.
- **0:** the frames carry their priority only and stay in each port's untagged VLAN. This works on any switch without VLAN configuration and keeps priority 5, but media and other traffic share a VLAN.

Untagged media streams may be supported for simple networks, but carry no priority; VLAN 0 is preferred there.

### Priority tags

A priority tag, VLAN ID 0, does not always survive a switch. A VLAN-aware switch assigns a frame tagged with VLAN ID 0 to the receiving port's untagged VLAN, and a port that sends that VLAN untagged sends the frame on without a tag: its priority counts at that switch and is lost beyond it. Some unmanaged switches do the same. A frame tagged with any other VLAN ID, the media VLAN 2 among them, keeps its tag. On the bench this profile was checked on, an unmanaged switch delivered an endpoint's PTP, sent with VLAN ID 0 and priority 7, untagged, while the same endpoint's VLAN 2 media kept their tags. So:

- Receivers must accept PTP untagged as well as priority-tagged, and, with media VLAN 0, media frames untagged as well.
- Media VLAN 2 keeps priority 5 across every switch; media VLAN 0 keeps it to the first switch only.
- PTP's priority 7 holds to the first switch only, unless the switches give PTP its priority themselves ([Enhanced networks](#enhanced-networks-managed-switch-configuration), item 1).

### Baseline networks: unmanaged switches

AVB Lite assumes only what common unmanaged switches do as they come:

- **Tagged frames:** forwarded unchanged, up to 1522 octets, with addresses learned whatever the VLAN; a priority tag may be removed ([Priority tags](#priority-tags)).
- **Multicast:** flooded to every port. This carries PTP to `01:1B:19:00:00:00`, ATDECC to `91:E0:F0:01:00:00`, and escalated streams.
- **`01:80:C2:00:00:0E`:** flooded or filtered; fallback handles either (§2.2).
- **Queueing:** anything from none to a fixed 802.1p or DSCP mapping, strict or weighted. Tags cost nothing where priority is ignored and help where it is honored.
- **EEE and flow control:** often on and fixed. Endpoints keep both off their own links (§6); links between unmanaged switches may still use them.

What a baseline network gives:

- Fallback, admission, unicast confinement and synchronization work as on an enhanced network.
- The latency targets below and the §5 sync figures hold while best-effort traffic on the ports and links carrying media stays light. Nothing gives media or PTP precedence, so a bulk transfer to a port that also carries media can queue them behind it, up to the switch's buffer: milliseconds on small switches. Listeners count the late frames, and the §9 alarm reports the offset.
- Guidance: keep bulk transfers off ports and links that carry media; where that cannot be done, use Class B presentation offsets, or an enhanced network.

### Enhanced networks: managed switch configuration

On every switch port that carries AVB Lite media, PTP or CVU SRP, links between switches included:

1. **QoS classification:** by 802.1p (trust CoS) on ingress, not by port or by DSCP alone. Where the switch can, also put PTP in the highest queue by its ethertype (`0x88F7`) or destination (`01:1B:19:00:00:00`), since a previous switch may have removed its priority tag; or, on links between switches, send each port's untagged VLAN tagged, so the priority travels on ([Priority tags](#priority-tags)).
2. **Scheduling:** strict priority, at least for the queues PTP and media use. Weighted round robin lets best-effort traffic delay media.
3. **Priority to queue:** 7 to the highest queue, 5 to the one below it, everything else lower. IEEE 802.1Q's recommended mapping (IEEE 802.1Q-2022, Table 8-5) already does this with three or more queues; with four, priorities 6 and 7 go to queue 3, 4 and 5 to queue 2, 2 and 3 to queue 1, and 0 and 1 to queue 0. With two queues PTP and media share the high one.
4. **Media VLAN 2:** create VLAN 2, and make every port with an AVB Lite device and every link between switches a tagged member of it. Leave each port's untagged VLAN (PVID) as it was: ATDECC, MAAP and priority-tagged PTP use it. With media VLAN 0 there is nothing to configure here.
5. **Accepted frame types:** all, so frames tagged with VLAN ID 0 are taken.
6. **EEE ("Green Ethernet", 802.3az):** off.
7. **Flow control (802.3x):** off.
8. **Unregistered multicast:** flooded. With IGMP snooping on, some switches drop multicast that no IGMP report asked for, which here includes PTP, ATDECC and escalated streams; turn off "filter unregistered multicast", or IGMP snooping.
9. **Spanning tree:** endpoint ports as edge ports (PortFast), so that a port forwards at link-up rather than after the listening and learning states, 30 s with classic STP, which would stop PTP, ATDECC and CVU SRP at every link-up.
10. **Storm control,** where used: broadcast and multicast thresholds above what AVB Lite sends, escalated multicast streams included, or media ports exempt.
11. **Jumbo frames:** off on media ports. A 9000-octet frame ahead of a media frame holds it 72 µs at 1 Gb/s, 720 µs at 100 Mb/s.
12. **Address ageing:** the default, 300 s, is fine; CVU SRP refreshes every 10 to 15 s.

### Traffic classes

| Class | 802.1p | Queue | Use |
|-------|--------|-------|-----|
| PTP | 7 | Highest | Sync, Follow_Up, Delay_Req, Delay_Resp, Announce |
| Media Class A | 5 | High | ≤ 2 ms latency streams |
| Media Class B | 5 | High | ≤ 10 ms latency streams |
| CVU SRP | 5 | High | Admission signaling, address learning |
| Other ATDECC, MAAP | Untagged | Default | Discovery, enumeration, control |
| Best effort | 0 or untagged | Default | Everything else |

### Expected end-to-end latency

Talker media timestamp to listener presentation, on an enhanced network or on a baseline network under light load, with talkers pacing their streams (§6) and media within 75% of every link, at any link speed of 100 Mb/s or more. At each switch a Class A frame waits at most for one full-size frame of lower priority (12,336 bit times: 123 µs at 100 Mb/s, 12 µs at 1 Gb/s) and the media of one class interval queued ahead of it (94 µs at 75%, whatever the speed). The figures are worked out for 100 Mb/s, the slowest link allowed, with the rest of each figure the endpoints' allowance; faster links meet them with room to spare.

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
- Stream bandwidth and packet rate are advertised in the talker's `cvu_srp_talker` TSpec, which admission counts (§6).

---

## 9. Failure Modes & Mitigations

| Failure | Behavior | Mitigation |
|---------|----------|------------|
| GM clock loss | BMCA elects new GM; brief offset transient | Servo holdover; 200 ms typical recovery |
| Switch congestion in media queue | Increased jitter, possible drops | Talkers and listeners refuse admission past 75% of their links (§6); operator alarm |
| Bulk traffic through a baseline switch | Media and PTP queue behind it: late frames, offset excursions | Keep bulk transfers off media ports, or use an enhanced network (§7) |
| Endpoint misbehaving, over-rate | No in-network protection | Pacing at the talker mandatory (§6); listeners count late and early frames |
| EEE enabled | Sync degrades, audible artifacts | Endpoints keep EEE off their own links (§6); managed switches configured per §7 |
| PTP packets deprioritized by misconfigured switch | Sync collapses | Continuous offset monitoring; alarm at > 50 µs sustained |
| Media VLAN missing on a managed switch | Media and CVU SRP dropped at the port; listeners never settle | Configure per §7, or use media VLAN 0 |
| Unregistered multicast filtered (IGMP snooping) | PTP and ATDECC lost: endpoints free-run, entities disappear | Flood unregistered multicast (§7) |
| More listeners than a talker's fan-out limit, escalation not allowed | The further listeners are refused with Talker Failed, failure code 2 | Allow escalation where the media VLAN is confined (§2.4, §6), or use a talker with a higher fan-out limit |
| Priority tags removed by a switch | PTP, and media in VLAN 0, lose their priority past the first switch | Media VLAN 2; on managed switches, classify PTP by ethertype (§7) |
| Mixed link speeds | Constant offset of up to 2.6 µs between endpoints | Grandmaster Link TLV correction (§5) |

---

## 10. Conformance

A device is AVB Lite conformant if it:

1. Implements the AVB Lite PTP profile over Layer-2 Ethernet with hardware timestamping, follows PTP tagged and untagged, and applies the link-speed correction where the grandmaster announces its link speed ([§5](#5-timing--avb-lite-ptp-profile)).
2. Marks media streams and CVU SRP with the VLAN and 802.1p values in [§7](#7-forwarding--qos), and PTP as well where the platform can tag it.
3. Paces each stream it sends per [§6](#6-admission-control--cvu-srp).
4. Supports the `cvu_srp_talker` and `cvu_srp_listener` messages with the format and timing of [§6](#6-admission-control--cvu-srp), admits streams against its own link (egress as a talker, ingress as a listener), and as a talker refuses a listener it cannot serve with a unicast Talker Failed.
5. Answers GET_LITE_STATUS and SET_LITE_CONFIG ([§2.4](#24-mode-reporting)), and escalates a stream to multicast only where escalation is allowed.
6. Keeps EEE and PAUSE off its own link.
7. Sends the Grandmaster Link TLV in its Announce messages, if it can be grandmaster.
8. Holds presentation-time accuracy within ±2× the [§5 sync target](#expected-sync-performance) under nominal load.

A baseline network is AVB Lite conformant if every switch in the media path behaves as [Baseline networks](#baseline-networks-unmanaged-switches) assumes. An enhanced network is AVB Lite conformant if every switch in the media path is configured per [Enhanced networks](#enhanced-networks-managed-switch-configuration).
