---
title: "Proposal — ESP-NOW Mesh Gateway"
type: proposal
tags:
  - project/esp-c6
  - proposal
  - firmware
  - protocol
  - exploration
created: 2026-04-14
status: exploring
---

# Proposal — ESP-NOW Mesh Gateway

Extend the [[ESP32-C6 XIAO Firmware (Rishab PCB)|XIAO ESP32-C6 firmware]] so out-of-BLE-range pads can still deliver their session data to the coach's phone by hopping through neighboring pads.

> [!warning] Status: exploring
> This is a design proposal. Nothing is built yet. Feasibility depends on a Phase 0 range test (see §7). If the radio can't clear 30 m through players' bodies, the whole idea gets shelved.

---

## Why

The XIAO ESP32-C6 has BLE (~20–40 m through bodies, less on a pitch) plus Wi-Fi 6 at 2.4 GHz. On a 105×68 m football pitch a single coach's phone cannot reach all 22 pads with BLE alone. Rather than forcing the coach to walk around after each match, we can have pads forward data through each other to reach whichever pad is currently in BLE range of the phone.

See [[Hardware Overview]] and [[BLE Protocol]] for current single-pad flow.

---

## Feasibility analysis (from web search, 2026-04-14)

### ESP-NOW range reality check

| Source | Claim |
|--------|-------|
| Espressif LR claim (external antenna) | up to 500 m line of sight |
| ESP32-C6 PCB antenna, indoor | ~20 m reliable |
| ESP32-C6 PCB antenna, outdoor painlessMesh | ~140 ft (~42 m) |
| LR mode on C6 specifically | Multiple users report **no improvement** vs normal mode at same TX power |
| Espressif own blog | C6 PCB antenna "not optimized for long-distance" |

**Realistic pitch estimate: 30–60 m pad-to-pad through bodies.** 2.4 GHz is heavily absorbed by human tissue (the pads are strapped to shins — every direct line has ≥2 bodies of water in the way). Signals propagate mostly via reflections. Expect to need 2–3 hops on a full pitch.

### BLE + ESP-NOW coexistence

The ESP32-C6 has **one RF path** shared between Wi-Fi/ESP-NOW, BLE, and 802.15.4 — all three time-division multiplex. Simultaneous operation is supported but prioritized, and heavy traffic on one starves the other. **Conclusion:** don't run BLE high-throughput and ESP-NOW mesh flush at the same time. Flush post-session only.

### Mesh performance

From painlessMesh / ESP-MESH benchmarks:
- 2-node delay: 2.49 ms
- Throughput: 461 msgs/sec for 10-byte payloads, **28 msgs/sec for 4400-byte payloads**
- Topology tends to degenerate into a star, not true mesh

A compressed Cresento chunk is ~1–2 KB. Even at worst-case throughput with 4 hops, a 100 KB session flushes in ~4 seconds. **Plenty fast enough.**

### RSSI triangulation for heat maps — dead end

Lab accuracy of BLE RSSI trilateration reaches 8–10 cm. Real-world moving-device accuracy in a middle-size room is 1.82 m (90th percentile). On an outdoor pitch with moving bodies absorbing signal, expect ±5–10 m — not good enough for a heat map requiring ~1 m resolution. **Industry (Kinexon, Catapult, Hudl WIMU, PlayerData) uses GPS outdoors and UWB with fixed anchors indoors. Nobody uses RSSI for pitch tracking. If they did, it'd be the cheapest option — they don't because it doesn't work.**

**If heat maps become important:** add cheap GPS (u-blox MAX-M10S ~$8) on next PCB rev, or UWB (DW3000) with fixed anchors for indoor only.

---

## Architecture

```
       [Pad 4]                      [Pad 5]
          │                            │
       ESP-NOW                      ESP-NOW
          │                            │
          ▼                            ▼
       [Pad 2] ──── ESP-NOW ──── [Pad 3]
                      │
                   ESP-NOW
                      │
                      ▼
               [Pad 1 = GATEWAY]
                      │
                     BLE
                      │
                      ▼
              [Coach's phone]
```

### Locked-in design choices

| Decision | Choice | Why |
|----------|--------|-----|
| Transport | ESP-NOW broadcast + unicast | No association, coexists with BLE, ~60 m realistic |
| Routing | Flood-with-dedup, TTL=4 | Simple, self-healing, pitches are small |
| Gateway role | Dynamic — any pad with BLE to phone | No single point of failure |
| Flush timing | Post-session only | Avoids BLE/ESP-NOW contention during record |
| Chunk format | Reuse existing `ChunkHeader` | Zero changes to compression codec or app decoder |
| Reliability | Hop-by-hop ACK + end-to-end retry | ESP-NOW unicast gives this nearly for free |

---

## Protocol design

### ESP-NOW packet format (250 byte max payload)

```c
struct __attribute__((packed)) MeshPacket {
  uint8_t  magic;         // 0xCE (Cresento)
  uint8_t  type;           // BEACON | ROUTE_INFO | DATA | ACK | FLUSH_REQ | FLUSH_DONE
  uint8_t  ttl;            // Hops remaining (start at 4)
  uint8_t  flags;
  uint8_t  src_mac[6];     // Original source pad (stable ID)
  uint8_t  dst_mac[6];     // Final destination (FF:FF... = broadcast)
  uint32_t seq;            // Source-assigned sequence number
  uint16_t payload_len;    // up to 228 bytes
  uint8_t  payload[228];   // Variable — one of the structs below
};
```

### Packet types

**BEACON** (periodic, every 5 s, broadcast)
```c
struct BeaconPayload {
  uint8_t  role;            // 0=NODE, 1=GATEWAY
  int8_t   phone_rssi;      // Gateway's BLE RSSI to phone
  uint8_t  hops_to_gateway; // 0 if gateway, 0xFF if unknown
  uint32_t pending_bytes;
  uint8_t  battery_pct;
};
```

**DATA** (unicast, carries a compressed session chunk)
```c
struct DataPayload {
  uint32_t session_id;
  uint32_t chunk_seq;
  uint16_t total_chunks;
  uint16_t chunk_index;
  // Followed by raw ChunkHeader + payload (existing format)
};
```

**FLUSH_REQ** — unicast from gateway → pad ("send me your data")
**FLUSH_DONE** — unicast from pad → gateway ("I'm finished, rows=N")
**ACK** — hop-by-hop unicast confirm

### Dedup + flood routing

Every pad keeps a small LRU table (RAM only):

```c
struct SeenPacket {
  uint8_t  src_mac[6];
  uint32_t seq;
  uint32_t last_seen_ms;
};
SeenPacket seenTable[64];
```

On every received packet:
1. If `(src_mac, seq)` already in table → **drop**
2. Else insert, then:
   - If `dst_mac` matches our MAC → **consume locally**
   - Else if `ttl > 0` → decrement, re-broadcast
   - Else → drop

### Gateway election

A pad becomes `GATEWAY` when **both** are true:
- It has an active BLE connection to the coach's phone
- The phone has sent a new BLE command: `MESH_START`

When BLE disconnects or `MESH_STOP` arrives, it drops gateway role. Multiple concurrent gateways allowed — nodes route to the one with lowest `hops_to_gateway` (tiebreak: best RSSI).

---

## Session state machine (extended)

```
         ┌──────┐
         │ idle │◄─────────────────────────┐
         └──┬───┘                          │
 START_LOG  │                              │
            ▼                              │
         ┌────────┐                        │
         │logging │  (existing)            │
         └──┬─────┘                        │
  STOP_LOG  │                              │
            ▼                              │
         ┌─────────┐                       │
         │flushing │ ← BLE flush (existing)│
         └──┬──────┘                       │
   DONE or  │ connection lost              │
   no BLE   ▼                              │
         ┌───────────┐                     │
         │mesh_ready │ ← NEW               │
         └──┬────────┘                     │
    gateway │ offers route                 │
            ▼                              │
         ┌─────────────┐                   │
         │mesh_flushing│ → all chunks acked│
         └─────────────┘───────────────────┘
```

Pads always try BLE first. If no phone in 30 s after STOP_LOG, enter `mesh_ready` and advertise pending bytes in beacon.

---

## Firmware changes

New files:

| File | Purpose | Approx LOC |
|------|---------|-----------|
| `mesh.h` / `mesh.cpp` | MeshPacket, neighbor table, routing | ~400 |
| `mesh_roles.cpp` | Gateway election, role transitions | ~150 |
| `mesh_bridge.cpp` | BLE ↔ ESP-NOW data pipe (gateway only) | ~200 |

Changes to existing `.ino`:
- Add `esp_now.h` include + init in `setup()`
- New BLE commands: `MESH_START`, `MESH_STOP`, `MESH_STATUS`
- Extend `pData` notifications with 6-byte source-MAC prefix when forwarding mesh data

---

## Mobile app changes (minimal)

1. **Start mesh session** — after connect + STOP_LOG, app sends `MESH_START`
2. **Demultiplex by source MAC** — chunks now have 6-byte source prefix; app maintains one in-memory session per pad, pipes into existing `SessionUploader`
3. **Mesh status UI** — "3 pads reachable, 2 outstanding", per-pad progress
4. **End with MESH_STOP** — once all pads DONE or user cancels

**No changes to:**
- [[Firebase Backend|Firestore schema]] (each pad still writes to `sensorData/` as if direct)
- Compression codec or decoder
- [[03 - Data Pipeline|SessionUploader]] paging logic
- BLE service/characteristic UUIDs (adding commands, not breaking format)

---

## Power profile (estimates)

| Mode | Current draw |
|------|--------------|
| Recording (existing) | ~40 mA |
| Mesh-ready idle (beacon every 5 s) | ~25 mA |
| Mesh-flushing active | ~80 mA |
| Gateway + BLE + ESP-NOW together | ~100 mA worst case |

**Gateway drains ~2× faster during flush.** Mitigation: pick best-battery pad as gateway (add `battery_pct` to beacon — already planned), or rotate gateway role every 60 s.

---

## Testing plan — critical, do in order

### Phase 0 — Range sanity (1 day, 2 pads)
- Flash ESP-NOW echo example
- Measure RSSI + packet success at 5, 10, 25, 50, 75, 100 m in open park
- Also test through a person (have someone stand between)
- **Gate:** ≥50 m reliable at ≥80% packet success line-of-sight; ≥30 m through one body
- **If fails:** PCB antenna is worse than expected → need uFL + external antenna before proceeding, OR shelve the idea

### Phase 1 — Beacon + neighbor table (1 week, 3 pads)
- Implement `MeshPacket` + BEACON type only
- Each pad logs neighbor table to serial
- **Gate:** 3 pads at corners of a triangle all see each other
- **Gate:** walking a pad out of range → drops from others' tables within 15 s

### Phase 2 — Single-hop gateway (1 week, 3 pads)
- Pad A connects to phone via BLE → becomes gateway
- Pad B sends small test session directly to A
- A forwards over BLE
- **Gate:** app receives B's session correctly, identified as source=B
- **Gate:** if A disconnects mid-flush, B retries when A reconnects

### Phase 3 — Multi-hop flooding (2 weeks, 4 pads)
- Arrange pads in a line: Phone — A — B — C — D (each 50 m apart)
- D sends session, must hop C → B → A
- **Gate:** D's session arrives intact, in order, with correct source MAC
- **Gate:** remove C mid-flush → D re-routes through different path after beacon refresh

### Phase 4 — Full scale (ongoing, 11+ pads)
- Real 11-a-side match setup
- Measure total flush time, packet loss, battery drain
- **Gate:** <60 s flush time for a 30-min session from farthest pad

### Phase 5 — Coexistence stress (continuous)
- During flush, have app simultaneously read battery characteristic
- Verify no BLE drops, no ESP-NOW packet loss correlated with BLE activity

---

## Rough timeline

| Week | Milestone |
|------|-----------|
| 1 | Phase 0 range test, decide antenna path |
| 2–3 | Phase 1: beacon + neighbor table |
| 4 | Phase 2: single-hop gateway |
| 5–6 | Phase 3: multi-hop flooding |
| 7 | App-side demux + UI |
| 8 | Phase 4: full-scale field test |
| 9+ | Tuning, power optimization, edge cases |

---

## Kill criteria

Be honest about when to bail:

1. **Phase 0 fails** (range <30 m through bodies) → mesh not worth the complexity; keep single-pad BLE and tell coaches to walk toward pads after match. **Fallback is cheap.**
2. **Regional TX power limits** (Wi-Fi 6) further hurt range below usable threshold.
3. **BLE/ESP-NOW contention** causes BLE drops even with careful scheduling → might need to fully stop BLE during ESP-NOW flush, hurting UX.
4. **Gateway battery drain** unacceptable even with rotation.

---

## Heat map addendum — not possible with this hardware

Triangulation via RSSI on a pitch won't work (see feasibility §). If heat maps become a priority:
- **Best option:** add GPS to next PCB rev (u-blox MAX-M10S, ~$8/unit, ±3 m outdoor, zero infrastructure — same as pros)
- **IMU dead reckoning** can give *relative* within-session movement (already in [[StatsEngine Cross-Platform|StatsEngine]]), not absolute position
- **UWB** (DW3000) gives sub-10 cm but requires fixed anchors around/in the venue — a venue install, not a pad change

---

## Open questions

- [ ] External antenna path: does the XIAO ESP32-C6 uFL pad give reliable pitch-scale range? Needs Phase 0 with both variants.
- [ ] Regulatory: is ESP-NOW LR mode legal at Cresento's target markets' TX power caps?
- [ ] Security: should mesh traffic be encrypted (ESP-NOW supports it)? Mat probably yes — otherwise adjacent Cresento deployments would interfere.
- [ ] Time sync: is there enough precision for IMU data timestamping across pads? ESP-NOW doesn't give sub-ms sync natively.

---

## Sources (feasibility research 2026-04-14)

- [Espressif ESP-NOW for Outdoor Applications](https://developer.espressif.com/blog/esp-now-for-outdoor-applications/)
- [ESP-NOW LR mode issue on ESP32-C6](https://github.com/espressif/esp-now/issues/144)
- [ESP-IDF RF Coexistence — ESP32-C6](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c6/api-guides/coexist.html)
- [Mesh performance comparison (UCLA LEMUR)](https://uclalemur.com/blog/perfomance-of-different-network-communication-protocols-used-for-mesh-networks)
- [BLE RSSI indoor positioning study (MDPI Sensors)](https://www.mdpi.com/1424-8220/21/15/5181)
- [RF absorption in human bodies at 2.4 GHz (PubMed)](https://pubmed.ncbi.nlm.nih.gov/31746011/)
- [KINEXON UWB player tracking](https://kinexon-sports.com/technology/player-tracking/)
- [Athlete tracking buyer's guide](https://simplifaster.com/articles/athlete-tracking-systems/)
- [PlayerData GPS trackers](https://www.playerdata.com/)

---

## Related

- [[ESP32-C6 XIAO Firmware (Rishab PCB)]] — the firmware this would extend
- [[BLE Protocol]] — frozen GATT contract (not broken by this proposal)
- [[Session State Machine]] — existing state machine (extended, not replaced)
- [[Hardware Overview]] — all three board variants
- [[01 - Critical Preservation Rules]] — what must not break
