# AVB Lite Specification

A profile for deploying AVB-style deterministic audio streaming over standard managed Ethernet switches: no 802.1AS, no MSRP, and no credit-based shaper required.

Version: 1.0-draft

---

## 1. Scope

AVB Lite reuses the AVB endpoint stack, including AVTP streams, stream IDs, and presentation time, but replaces the network-side requirements of IEEE 802.1BA so that the system runs on any gigabit switch with DiffServ QoS and IGMP snooping.

Targets: pro audio installations of up to approximately 50 endpoints and 200 streams, on a single L2 domain or routed via PTP-aware boundaries.

---

## 2. Automatic AVB Lite Fallback

An AVB endpoint that also supports AVB Lite may automatically fall back to AVB Lite when connected to a non-AVB-capable switch or network segment.

Fallback detection may use any combination of the following:

- Absence of 802.1AS/gPTP neighbor synchronization.
- MSRP/SRP registration or reservation failure.
- LLDP information indicating that the adjacent bridge does not advertise AVB/TSN capabilities.
- Controller policy explicitly selecting AVB Lite mode for the port, device, or network.

When falling back, the device must clearly advertise its active operating mode to the controller and must not use MSRP reservations for AVB Lite streams. AVB Lite streams must use the timing, QoS, admission-control, and conformance requirements defined by this profile.

A device should prefer standard IEEE 802.1BA operation when a valid AVB domain is detected, unless configured otherwise by the operator or controller.

---

## 3. Bridging Between AVB and AVB Lite Domains

A system may bridge traffic between a standard AVB domain and an AVB Lite domain using an explicit gateway or boundary device.

The gateway is responsible for translating domain assumptions, including:

- Clock-domain adaptation between 802.1AS/gPTP and AVB Lite PTPv2 timing.
- Admission-control translation between MSRP/SRP and the AVB Lite controller bandwidth ledger.
- QoS mapping between AVB SR classes and AVB Lite DSCP / 802.1p traffic classes.
- Stream lifecycle coordination so that connection state is consistent on both sides.
- Presentation-time offset adjustment for the combined AVB and AVB Lite path latency.

Bridging must be explicit. Endpoints must not assume that AVB and AVB Lite domains are directly interoperable without a gateway that provides these translation functions.

---

## 4. Changes Relative to 802.1BA-2011

| 802.1BA Component | Status in AVB Lite | Replacement |
|-------------------|--------------------|-------------|
| 802.1AS (gPTP) | Removed | IEEE 1588v2 default profile with hardware timestamping at endpoints |
| 802.1Qat (MSRP) | Removed | Centralized admission control in controller software |
| 802.1Qav (FQTSS / credit-based shaper) | Removed | Strict-priority queueing via 802.1p / DSCP |
| 802.1Q VLAN / SR class A & B | Retained, redefined | Class A → DSCP EF (46); Class B → DSCP AF41 (34) |
| AVTP (1722) stream format | Retained unchanged | — |
| AVDECC (1722.1) discovery/control | Retained unchanged | — |
| Stream Reservation Class latency targets | Relaxed | See [§7 Forwarding & QoS](#7-forwarding--qos) |
| Bandwidth limit: 75% of link | Retained as engineering rule | Enforced by controller, not switches |

---

## 5. Timing — PTP Profile

- **Protocol:** IEEE 1588-2008 (PTPv2), default profile, end-to-end delay mechanism.
- **Transport:** UDP/IPv4 multicast (`224.0.1.129` / `224.0.1.130`). Layer-2 PTP is optional but not required.
- **Timestamping:** Hardware timestamping is mandatory at all endpoints, at NIC or PHY level. Switches are not required to be boundary clocks or transparent clocks.
- **Sync interval:** 125 ms, log -3, for Sync; 1 s for Announce.
- **Servo:** PI controller with outlier rejection. A Kalman filter is recommended on networks greater than 3 hops.
- **QoS marking for PTP:** DSCP EF (46), 802.1p priority 6.
- **Best Master Clock Algorithm:** Standard BMCA, with configurable static `priority1` / `priority2` values to allow operators to pin the grandmaster.

### Expected sync performance

| Topology | Expected offset, 1σ | Worst case |
|----------|----------------------|------------|
| Single switch, lightly loaded | < 200 ns | < 1 µs |
| 3 hops, prioritized PTP | 500 ns – 2 µs | 5 µs |
| 5 hops or congested | 2 – 10 µs | 50 µs |

---

## 6. Admission Control — MSRP Replacement

A logically centralized AVB Lite Controller, implemented in software and optionally co-located with one of the endpoints, performs the role that MSRP plays in classic AVB.

Required behaviors:

1. **Topology discovery:** Use LLDP queries from each endpoint, plus optional SNMP polling of switches for link speed.
2. **Bandwidth ledger:** The controller maintains a per-link committed bandwidth tally. New stream advertisements that would push any link beyond 75% of its rate are refused.
3. **Path computation:** Use simple shortest-path computation on the discovered graph for admission accounting only. Actual forwarding is done by switches via IGMP snooping.
4. **Stream lifecycle:** Use an `advertise → admit → ready → start → stop` state machine, replacing MSRP's Talker/Listener attribute exchange. The state machine is encoded in JSON over TCP, port `17220` suggested.
5. **Stale reservation cleanup:** Each endpoint sends a 30 s heartbeat. Missing heartbeats release reserved bandwidth.

Endpoints must rate-limit their own egress to the advertised stream rate. There is no in-network shaper to catch a misbehaving talker.

---

## 7. Forwarding & QoS

### Switch requirements, off-the-shelf

- 802.1p / DSCP-based strict-priority queueing, minimum 4 queues.
- IGMP snooping v2 or v3.
- EEE (802.3az) disabled on all ports carrying audio.
- Flow control (802.3x) disabled.
- Optional but recommended:
  - Per-port storm control.
  - Jumbo frames disabled on the audio VLAN.

### Traffic classes

| Class | DSCP | 802.1p | Queue | Use |
|-------|------|--------|-------|-----|
| PTP | EF (46) | 6 | Highest | Sync / Delay-Req / Delay-Resp |
| Audio Class A | EF (46) | 5 | Highest, shared with PTP | ≤ 2 ms latency streams |
| Audio Class B | AF41 (34) | 4 | High | ≤ 10 ms latency streams |
| Control: AVDECC, controller | CS3 (24) | 3 | Medium | Discovery, enumeration |
| Best effort | 0 | 0 | Default | Everything else |

### Expected end-to-end latency

Talker sample to listener sample. Assumes gigabit links, ≤ 75% utilization on audio queues, and hardware-timestamped endpoints.

| Hops | Class A target | Class B target |
|------|----------------|----------------|
| 1 | 1.0 ms | 5 ms |
| 3 | 1.5 ms | 6 ms |
| 5 | 2.0 ms | 8 ms |
| 7 | 2.5 ms | 10 ms |

Configured presentation-time offset should be set to the table value for the network's worst-case path.

---

## 8. Stream Format

Unchanged from AVB:

- AVTP (IEEE 1722) AAF or CRF payload.
- 48 kHz / 96 kHz, 16/24/32-bit PCM.
- Up to 8 channels per stream; higher counts use stream bundling.
- Presentation time in the AVTP header references the PTP-disciplined clock.

---

## 9. Failure Modes & Mitigations

| Failure | Behavior | Mitigation |
|---------|----------|------------|
| GM clock loss | BMCA elects new GM; brief offset transient | Servo holdover; 200 ms typical recovery |
| Switch congestion in audio queue | Increased jitter, possible drops | Controller refuses admission past 75%; operator alarm |
| Endpoint misbehaving, over-rate | No in-network protection | Source-side rate limiter mandatory; controller monitors via SNMP/telemetry where available |
| EEE accidentally enabled | Sync degrades, audible artifacts | Controller probes for EEE via LLDP-MED; warns operator |
| PTP packets deprioritized by misconfigured switch | Sync collapses | Continuous offset monitoring; alarm at > 50 µs sustained |

---

## 10. Conformance

A device is AVB Lite conformant if it:

1. Implements PTPv2 default profile with hardware timestamping.
2. Marks all egress traffic with the DSCP / 802.1p values in [§7 Forwarding & QoS](#7-forwarding--qos).
3. Implements source rate limiting per advertised stream.
4. Speaks the controller protocol in [§6 Admission Control — MSRP Replacement](#6-admission-control--msrp-replacement) for stream advertisement and admission.
5. Holds presentation-time accuracy within ±2× the [§5 sync target](#expected-sync-performance) under nominal load.

A network is AVB Lite conformant if every switch in the audio path meets the [§7 switch requirements](#switch-requirements-off-the-shelf) and is configured per the QoS table.

---

## 11. Open Questions

- Whether to define a redundancy mode, using parallel paths à la Milan / IEEE 802.1CB. This is currently out of scope.
- Whether to allow L3-routed audio across PTP boundary clocks. Section 3 does not preclude it, but the latency table assumes a single L2 domain.
- DSCP remarking by intermediate switches is a real-world hazard and may need a trust-boundary appendix.
