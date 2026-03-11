# Partial BFD Offload: Software Handshake with Hardware Keepalive

## Status: Design Proposal (Draft)

## Problem Statement

The BFD-Syncd HLD (BFD_Syncd_HLD.md) assumes hardware handles the full BFD session lifecycle — 3-way handshake, steady-state keepalive, and failure detection. However, some ASICs cannot perform the initial BFD handshake:

- The ASIC needs the **remote discriminator** before it can match incoming BFD packets — this is only learned during the handshake.
- **Timer negotiation** happens during session establishment — hardware needs the final negotiated values, not the configured values.
- Some ASICs only support a "canned" BFD state machine: TX at a configured rate, check RX at a configured rate, match a known discriminator. No Init→Up transition logic.

Partial offload solves this by having software (FRR bfdd) handle the initial handshake and then handing the established session to hardware for steady-state keepalive.

## Approach: Hybrid Mode in bfdd

### Overview

A new bfdd mode (`--dplane-after-up`) where bfdd performs the software BFD handshake first, learns the remote discriminator and negotiated timers, then offloads the established session to hardware via the existing Distributed BFD protocol.

### Session Lifecycle

```
Phase 1: Software Handshake
  1. Routing daemon (BGPd / OSPFd / staticd) registers BFD peer with bfdd
  2. bfdd runs software BFD — sends/receives BFD control packets via CPU
  3. Software 3-way handshake: Down → Init → Up
  4. bfdd learns:
     - Remote discriminator (from peer's BFD control packets)
     - Negotiated TX interval (max of local desired, remote required)
     - Negotiated RX interval (max of local required, remote desired)

Phase 2: Handover to Hardware
  5. bfdd sends DP_ADD_SESSION to bfd-syncd with extra fields:
     - remote_discriminator: learned from peer
     - negotiated_tx_interval: final negotiated value (microseconds)
     - negotiated_rx_interval: final negotiated value (microseconds)
     - initial_state: UP
  6. bfd-syncd writes APPL_DB entry with these values
  7. BFDOrch creates SAI BFD session in UP state with pre-populated remote discriminator
  8. Hardware starts TX/RX

Phase 3: Software TX Overlap and Cutover
  9. bfdd continues software TX until bfd-syncd confirms hardware is active
     (BFD_STATE_CHANGE UP received)
  10. bfdd stops software TX for this session
  11. Hardware is now sole owner of BFD packet processing

Phase 4: Steady State
  12. Hardware handles all BFD TX/RX
  13. State changes reported back to bfdd via bfd-syncd (BFD_STATE_CHANGE)
  14. Standard path as described in BFD_Syncd_HLD.md §3.3.2
```

### Takeover Window

The critical moment is the handover from software to hardware. If hardware hasn't started TX before software stops, the remote peer's detection timer could expire, causing a false BFD DOWN.

The solution is an **overlap window** where both software and hardware are transmitting simultaneously:

```
Timeline:

bfdd software TX:     |========================|----stop----|
                                               ^            ^
                                          HW session     BFD_STATE_CHANGE UP
                                          created        received from bfd-syncd

hardware TX:                              |===============================>

overlap window:                           |=============|
                                          (duplicate BFD packets — harmless,
                                           RFC 5880 handles gracefully)
```

**Overlap safety:** During the overlap, the remote peer receives BFD packets from both software and hardware. Both carry the same local discriminator and session parameters. The remote peer processes whichever arrives first and ignores duplicates — this is standard BFD behavior per RFC 5880. The overlap duration is bounded by bfd-syncd processing time (APPL_DB write → BFDOrch → SAI → first hardware TX → STATE_DB UP → bfd-syncd notification → bfdd stops software TX). Typical: 50–200ms.

**bfdd must NOT stop software TX until it receives BFD_STATE_CHANGE UP from bfd-syncd.** This is the invariant that prevents the takeover gap.

### Hardware DOWN Handling

If the hardware session goes DOWN — whether due to genuine peer failure, ASIC reload, SAI error, or SWSS restart — bfd-syncd reports `BFD_STATE_CHANGE DOWN` to bfdd, which notifies upper layers (BGP/OSPF/staticd) via the standard path. No software fallback is attempted.

**Rationale:** Hardware BFD DOWN means the forwarding path is impaired. If the ASIC is restarting or in error state, the data plane cannot forward traffic either — reporting DOWN is the correct signal. Silently switching to software BFD would mask the failure from upper layers and operators. Warm restart scenarios are handled separately by the existing reconciliation path (BFD_Syncd_HLD.md §7), not by bfdd software fallback.

### DP_ADD_SESSION Message Extension

Three optional fields are added to the `DP_ADD_SESSION` message in the Distributed BFD protocol (`bfdd/bfddp_packet.h`):

| Field | Type | Description |
|-------|------|-------------|
| `remote_discriminator` | uint32 | Peer's local discriminator, learned during software handshake. 0 means not pre-negotiated (full offload mode). |
| `negotiated_tx_interval` | uint32 | Final negotiated TX interval in microseconds. 0 means use configured value (full offload mode). |
| `negotiated_rx_interval` | uint32 | Final negotiated RX interval in microseconds. 0 means use configured value (full offload mode). |

These fields are **backward compatible** — existing data plane implementations that do not understand them will ignore the zero/absent values and perform full handshake in hardware (current behavior). Data plane implementations that support partial offload use these values to create the session directly in UP state.

### bfd-syncd Changes

Minimal changes to bfd-syncd:

1. Pass `remote_discriminator` to APPL_DB if present and non-zero
2. Pass `negotiated_tx_interval` / `negotiated_rx_interval` to APPL_DB if present and non-zero
3. No state machine changes — bfd-syncd remains a stateless bridge

New APPL_DB fields:

```
BFD_SESSION_TABLE:{{vrf}}:{{ifname}}:{{ipaddr}}
    ...existing fields from BFD_Syncd_HLD.md §4.1...
    "remote_discriminator" : {{uint32}}        ; Optional. Pre-negotiated remote discriminator.
    "negotiated_tx_interval" : {{interval}}    ; Optional. Final negotiated TX (microseconds).
    "negotiated_rx_interval" : {{interval}}    ; Optional. Final negotiated RX (microseconds).
```

### BFDOrch / SAI Changes

BFDOrch must support creating a session with a pre-populated remote discriminator:

| SAI Attribute | Full Offload (current) | Partial Offload |
|---------------|----------------------|-----------------|
| `SAI_BFD_SESSION_ATTR_REMOTE_DISCRIMINATOR` | 0 (learned by ASIC) | Pre-populated from APPL_DB |
| `SAI_BFD_SESSION_ATTR_BFD_ENCAPSULATION_TYPE` | Unchanged | Unchanged |
| `SAI_BFD_SESSION_ATTR_MIN_TX` | Configured value | Negotiated value |
| `SAI_BFD_SESSION_ATTR_MIN_RX` | Configured value | Negotiated value |

**SAI prerequisite:** The ASIC must support `sai_create_bfd_session` with a non-zero `SAI_BFD_SESSION_ATTR_REMOTE_DISCRIMINATOR` and be able to start sending BFD packets immediately in UP state without performing its own handshake. This is ASIC-dependent — verify with your vendor before implementing.

### bfdd CLI Flag

```bash
# Full offload (current behavior — ASIC handles handshake)
bfdd --dplaneaddr unix:/var/run/frr/bfdd_dplane.sock

# Partial offload (software handshake, then hardware keepalive)
bfdd --dplaneaddr unix:/var/run/frr/bfdd_dplane.sock --dplane-after-up
```

### supervisord Template Update

```jinja
{% if FEATURE.bgp.bfd_hw_offload is defined and FEATURE.bgp.bfd_hw_offload == "true" %}
{% set dplane_flag = "--dplane-after-up" if FEATURE.bgp.bfd_partial_offload is defined and FEATURE.bgp.bfd_partial_offload == "true" else "" %}
[program:bfdd]
command=/usr/lib/frr/bfdd -A 127.0.0.1 --dplaneaddr unix:/var/run/frr/bfdd_dplane.sock {{ dplane_flag }}
{% endif %}
```

### New CONFIG_DB Field

```
FEATURE|bgp
    "bfd_hw_offload": "true"
    "bfd_partial_offload": "true"    ; Optional. Default: "false".
                                     ; When true, bfdd performs software handshake
                                     ; before offloading to hardware.
```

## Edge Cases

### Timer Renegotiation

If the remote peer requests a timer change after hardware takeover (RFC 5880 §6.8.3), bfdd cannot process it — hardware is handling packets. Two options:

1. **Ignore renegotiation in hardware** — hardware continues with original timers. The remote peer adapts (RFC 5880 requires the sender to use the slower of its desired and the peer's required interval). This is the simpler approach and is acceptable for data center deployments where timers are configured symmetrically.

2. **Fall back to software for renegotiation** — hardware reports a parameter mismatch, bfd-syncd signals bfdd, bfdd resumes software BFD, renegotiates, then re-offloads. This is more correct but adds complexity.

Recommendation: Option 1 for initial implementation. Option 2 as future enhancement if needed.

### Session Flap During Handover

If the remote peer flaps during the overlap window (between hardware session creation and bfdd stopping software TX), both software and hardware detect the flap. bfdd receives the DOWN event twice — once from its own software detection and once from bfd-syncd. bfdd must deduplicate: once hardware is confirmed active (BFD_STATE_CHANGE received), bfdd should only act on bfd-syncd notifications and ignore its own software detection for that session.

### Link-Local Sessions

Partial offload for IPv6 link-local sessions follows the same flow, with the additional MAC resolution step from BFD_Syncd_HLD.md §3.4 occurring between steps 6 and 7 (after bfd-syncd receives DP_ADD_SESSION but before writing to APPL_DB). The software handshake in steps 2–4 uses the CPU and kernel networking stack, which resolves link-local MACs via NDP automatically — no special handling needed during the software phase.

## Implementation Scope

| Component | Change Required | Complexity |
|-----------|----------------|------------|
| FRR bfdd | New `--dplane-after-up` mode: delay DP_ADD_SESSION until software UP, add fields, overlap logic | Medium — core change |
| bfd-syncd | Pass through 3 new optional APPL_DB fields | Low |
| BFDOrch | Support pre-populated remote discriminator in `sai_create_bfd_session` | Low |
| SAI/ASIC | Create session with known remote discriminator, start in UP state | Vendor-dependent |
| CONFIG_DB | New `bfd_partial_offload` field in FEATURE table | Low |
| YANG | New leaf for `bfd_partial_offload` | Low |

## Open Questions

1. **SAI support:** Does `sai_create_bfd_session` with non-zero `SAI_BFD_SESSION_ATTR_REMOTE_DISCRIMINATOR` work on the target ASIC? Does the ASIC start TX immediately or still require its own Init→Up transition?

2. **FRR upstream appetite:** Would the FRR community accept `--dplane-after-up`? The Distributed BFD protocol was designed for full offload. Partial offload is a reasonable extension but may face pushback as added complexity.

3. **Overlap duration bound:** What is the worst-case latency from APPL_DB write to first hardware BFD TX? If this exceeds the remote peer's detection timeout (e.g., 3 × 100ms = 300ms), the overlap window may not be sufficient, and bfdd may need to temporarily increase its software TX rate during handover.

## Relationship to BFD_Syncd_HLD.md

This design is an **additive extension** to the BFD-Syncd HLD. When `bfd_partial_offload` is false (default), the system behaves exactly as described in the HLD — full hardware offload. When true, the hybrid mode activates. All other aspects of the HLD (link-local MAC resolution, convergence acceleration, multi-owner refcount, StaticRouteBFD elimination, warm restart) apply unchanged.
