# HLD: BGP ADDPATH Yang Model Changes

Rev v0.1

## Table of Contents

1. [Revisions](#revisions)
2. [Scope](#scope)
3. [Definitions and Abbreviations](#definitions-and-abbreviations)
4. [Overview](#overview)
5. [Background](#background)
6. [Requirements](#requirements)
7. [Design](#design)
   - [Yang Model Changes](#yang-model-changes)
   - [FRR Configuration Support](#frr-configuration-support)
   - [Default Behavior Change for ADDPATH RX](#default-behavior-change-for-addpath-rx)
8. [Configuration and Management](#configuration-and-management)
   - [CONFIG_DB Schema](#configdb-schema)
9. [Backward Compatibility](#backward-compatibility)
10. [Testing](#testing)
11. [Limitations and Pending Issues](#limitations-and-pending-issues)
12. [References](#references)

---

## Revisions

| Rev  | Date       | Author    | Description            |
|------|------------|-----------|------------------------|
| v0.1 | 2026-05-27 | Fred Xia  | Initial draft          |

---

## Scope

This document describes Yang model changes to `sonic-bgp-common.yang` to expose all FRR BGP
ADDPATH configuration options in SONiC. It covers:

- New Yang leaves and a choice node for ADDPATH TX path-selection methods
- New Yang leaves for ADDPATH RX control
- Deprecation of the legacy `tx_add_paths` leaf
- A change in the default value of `addpath_rx_disable` to `true` (disabled by default)
- Supporting template and frrcfgd changes that translate CONFIG_DB entries to FRR CLI commands

---

## Definitions and Abbreviations

| Term      | Definition                                                                                  |
|-----------|---------------------------------------------------------------------------------------------|
| ADDPATH   | BGP ADD-PATH extension (RFC 7911) — allows advertising multiple paths per prefix            |
| AFI/SAFI  | Address Family Identifier / Subsequent Address Family Identifier                            |
| FRR       | FRRouting — open-source routing suite used as the BGP implementation in SONiC               |
| NLRI      | Network Layer Reachability Information                                                      |
| Path ID   | 4-octet identifier prepended to NLRI to distinguish multiple paths for the same prefix      |
| RX        | Receive direction — paths received from a BGP peer                                          |
| TX        | Transmit direction — paths advertised to a BGP peer                                         |

---

## Overview

BGP ADDPATH (RFC 7911) enables a BGP speaker to advertise multiple paths for the same address
prefix to a peer. Each path is identified by a locally assigned 4-octet Path Identifier prepended
to the NLRI encoding. The capability is negotiated using BGP Capability Code 69, with separate
send and receive directions per AFI/SAFI.

This feature is widely implemented by major vendors and is fully supported by FRR. However, the
SONiC Yang model historically exposed only one attribute — `tx_add_paths` — with two enumerated
values (`tx_all_paths`, `tx_best_path_per_as`). This left the full set of FRR ADDPATH knobs
unreachable through the SONiC configuration interface.

Extending the Yang model to cover all ADDPATH options serves two purposes. First, it provides a
more complete and accurate picture of the BGP router configuration at the SONiC level, enabling
operators to manage and audit the full ADDPATH policy through SONiC tooling. Second, and more
importantly, it allows ADDPATH configuration to be persisted in CONFIG_DB — the source of truth
for SONiC configuration. Any configuration applied directly via `vtysh` is transient: the FRR
container is ephemeral, and its running state is discarded on restart or reboot. Only
configuration reflected in CONFIG_DB survives across restarts and is reliably replayed when FRR
is brought back up.

This HLD describes the Yang model extensions required to support all FRR ADDPATH configuration
options, along with corresponding changes in FRR template rendering and frrcfgd to wire them
through to the running FRR configuration.

---

## Background

### RFC 7911 — Advertisement of Multiple Paths in BGP

RFC 7911 defines the ADD-PATH Capability (code 69) which allows a BGP speaker to advertise
multiple paths for the same prefix without new paths implicitly replacing previous ones. Key
points:

- Each path is tagged with a 4-octet Path Identifier assigned locally by the advertising speaker.
- The capability is negotiated per `<AFI, SAFI>` with independent send/receive directions.
- A speaker SHOULD include the best route when advertising multiple paths, unless that path was
  received from the same peer.
- RFC 7911 explicitly calls out a security concern: multiple paths for a large number of prefixes
  may deplete memory or cause network-wide instability (a potential denial-of-service vector).

### ADDPATH Paths Limit (draft-abraitis-idr-addpath-paths-limit)

This IETF draft extends ADDPATH negotiation to allow a receiver to advertise a limit on the
number of additional paths it is willing to accept per prefix. The limit is signaled as part
of the ADD-PATH Capability during BGP OPEN. The sending peer is expected to honor the limit.

FRR supports the receiving side of this draft via `addpath-rx-paths-limit`. Enforcement on the
receiving side (dropping paths from a non-compliant peer that exceeds the limit) is not yet
implemented in FRR.

### FRR ADDPATH Configuration Options

FRR exposes the following per-neighbor, per-address-family ADDPATH knobs:

```
addpath-tx-all-paths               Advertise all paths to the peer
addpath-tx-best-selected <N>       Advertise N best-selected paths to the peer (N: 1–6)
addpath-tx-bestpath-per-AS         Advertise the best path from each neighboring AS

disable-addpath-rx                 Do not accept additional paths from the peer
addpath-rx-paths-limit <N>         Signal a limit of N paths willing to receive (N: 1–65535)
```

All TX options are mutually exclusive. In FRR, ADDPATH RX is enabled by default.

### Prior SONiC Yang Model

The `sonic-bgp-cmn-af` grouping in `sonic-bgp-common.yang` contained a single deprecated leaf:

```yang
leaf tx_add_paths {
    type bgp_tx_add_paths_type;  // enum: tx_all_paths | tx_best_path_per_as
}
```

This covered only two of the five FRR knobs and provided no RX-direction control.

---

## Requirements

| # | Requirement                                                                                        |
|---|----------------------------------------------------------------------------------------------------|
| 1 | Expose all three mutually exclusive FRR TX ADDPATH modes in the Yang model                         |
| 2 | Expose the FRR `disable-addpath-rx` knob in the Yang model                                         |
| 3 | Expose the FRR `addpath-rx-paths-limit` knob in the Yang model                                     |
| 4 | Enforce mutual exclusion of TX options at the Yang schema level                                    |
| 5 | Deprecate the legacy `tx_add_paths` leaf without removing it (backward compatibility)              |
| 6 | Default ADDPATH RX to disabled to protect against memory exhaustion from misbehaving peers         |
| 7 | Translate new Yang attributes to FRR CLI commands via existing template and frrcfgd mechanisms     |

---

## Design

### Yang Model Changes

The changes are in the `sonic-bgp-cmn-af` grouping of `sonic-bgp-common.yang`. This grouping is
shared by neighbor and peer-group address-family tables.

#### New `addpath_tx` Choice Node

A YANG `choice` node enforces mutual exclusion of the three TX path-selection methods at the
schema level. Only one of the three `case` branches can be active at any time; setting a leaf in
one case implicitly removes any previously set leaf in another case.

```yang
choice addpath_tx {
    description
        "Mutually exclusive add-path TX path-selection methods.
         Configuring one implicitly de-configures the other.";
    case tx_all_paths {
        leaf addpath_tx_all_paths {
            type empty;
            description "Advertise all paths to this neighbor using BGP add-path";
        }
    }
    case best_selected {
        leaf addpath_tx_best_selected {
            type uint8 {
                range "1..6";
            }
            description "Advertise N best paths to this neighbor using BGP add-path";
        }
    }
    case bestpath_per_as {
        leaf addpath_tx_bestpath_per_as {
            type empty;
            description "Advertise the bestpath per each neighboring AS using BGP add-path";
        }
    }
}
```

The `addpath_tx_all_paths` and `addpath_tx_bestpath_per_as` leaves use type `empty` (presence
semantics — the leaf either exists or does not). The `addpath_tx_best_selected` leaf carries the
count of paths (1–6, matching the FRR range).

#### New ADDPATH RX Leaves

```yang
leaf addpath_rx_disable {
    type boolean;
    default true;
    description "Disable receiving add-path advertisements from this neighbor";
}

leaf addpath_rx_paths_limit {
    type uint16 {
        range "1..65535";
    }
    description "Limit the number of add-path paths received from this neighbor";
}
```

`addpath_rx_disable` defaults to `true` (RX disabled). See
[Default Behavior Change for ADDPATH RX](#default-behavior-change-for-addpath-rx) for the
rationale.

`addpath_rx_paths_limit` is optional. When set, it signals the paths-limit value to the peer
during capability negotiation as specified by draft-abraitis-idr-addpath-paths-limit.

#### Deprecation of `tx_add_paths`

The existing `tx_add_paths` leaf and the `bgp_tx_add_paths_type` typedef are marked `status
deprecated`. They remain in the schema for backward compatibility with existing configurations
but will not be extended further.

```yang
typedef bgp_tx_add_paths_type {
    ...
    status deprecated;
    description "Deprecated. Use addpath_tx instead.";
}

leaf tx_add_paths {
    type bgp_tx_add_paths_type;
    status deprecated;
    description "Deprecated. Use addpath_tx instead.";
}
```

#### YANG Revision Entry

```yang
revision 2026-04-22 {
    description
        "Add BGP address-family level add-path configuration parameters supported by FRR to
         sonic-bgp-cmn-af grouping: addpath_tx (choice with cases tx_all_paths,
         best_selected, bestpath_per_as), addpath_rx_disable, addpath_rx_paths_limit.
         TX addpath options are mutually exclusive. Deprecate tx_add_paths leaf.";
}
```

---

### FRR Configuration Support

The FRR configuration templates and frrcfgd scripts are adjusted to support both legacy
(`tx_add_paths`) and the new unified ADDPATH configuration model. The new attributes are
translated to the corresponding FRR CLI commands in both static template mode (full config
generation at container start) and dynamic mode (incremental CONFIG_DB-driven updates via
frrcfgd).

---

### Default Behavior Change for ADDPATH RX

In stock FRR, ADDPATH RX is enabled by default. A BGP peer that also supports ADD-PATH may
therefore send an unbounded number of additional paths. RFC 7911 Section 8 explicitly identifies
this as a potential denial-of-service vector: memory exhaustion from a large volume of additional
paths for many prefixes.

To protect SONiC switches — particularly in operator environments where peering with untrusted
or misbehaving peers is possible — the default value of `addpath_rx_disable` is changed to
`true`. This means:

- **New deployments**: ADDPATH RX is disabled on all peers until an operator explicitly sets
  `addpath_rx_disable = false`.
- **Existing deployments**: Peers that previously had RX implicitly enabled (via the FRR
  default) will now have RX disabled after upgrade, since `addpath_rx_disable` defaults to
  `true` in the Yang schema. Operators who need ADDPATH RX must explicitly enable it.

---

## Configuration and Management

### CONFIG_DB Schema

The new attributes land in the `BGP_NEIGHBOR_AF` and `BGP_PEER_GROUP_AF` tables, keyed as
`<vrf>|<neighbor>|<afi_safi>`. All attributes are optional except `addpath_rx_disable` which
has a default of `true`.

| Field                      | Type          | Values / Range | Description                                       |
|----------------------------|---------------|----------------|---------------------------------------------------|
| `addpath_tx_all_paths`     | empty string  | `""`           | Advertise all paths (mutually exclusive TX)       |
| `addpath_tx_best_selected` | uint8         | 1–6            | Advertise N best paths (mutually exclusive TX)    |
| `addpath_tx_bestpath_per_as` | empty string | `""`          | Advertise best path per AS (mutually exclusive TX)|
| `addpath_rx_disable`       | boolean       | `true`/`false` | Disable ADDPATH RX (default: `true`)              |
| `addpath_rx_paths_limit`   | uint16        | 1–65535        | Paths-limit signaled during capability negotiation|

Only one of `addpath_tx_all_paths`, `addpath_tx_best_selected`, or `addpath_tx_bestpath_per_as`
may be present at a time.

---

## Backward Compatibility

The `tx_add_paths` leaf and the `bgp_tx_add_paths_type` typedef are retained as deprecated.
Existing configurations that use `tx_add_paths` continue to be accepted and rendered by the
existing template logic. No migration is required for configurations using `tx_add_paths`.

The behavioral change is in the ADDPATH RX default. Existing deployments where ADDPATH RX was
implicitly active (via FRR default) will see RX disabled after upgrade unless `addpath_rx_disable`
is explicitly set to `false` in CONFIG_DB. Operators should audit their peer configurations if
they rely on ADDPATH RX being enabled.

---

## Testing

### Unit Tests — Yang Model Constraints

A pytest suite (`test_bgp_addpath.py`) verifies the Yang model constraints using libyang:

- Setting `addpath_tx_all_paths` validates successfully.
- Setting `addpath_tx_best_selected` with a value in range 1–6 validates successfully.
- Setting `addpath_tx_best_selected` with value 0 or 7 fails validation.
- Setting `addpath_tx_bestpath_per_as` validates successfully.
- Setting two TX leaves simultaneously fails validation (choice constraint).
- Setting `addpath_rx_disable true` validates successfully.
- Setting `addpath_rx_paths_limit` with a value in range 1–65535 validates successfully.
- Setting `addpath_rx_paths_limit 0` or `65536` fails validation.

### Unit Tests — Template Rendering

The test suite in `test_config.py` verifies that CONFIG_DB entries for each new attribute produce
the correct FRR CLI output:

| CONFIG_DB input                        | Expected FRR command                                      |
|----------------------------------------|-----------------------------------------------------------|
| `addpath_tx_all_paths = ""`            | `neighbor <N> addpath-tx-all-paths`                       |
| `addpath_tx_best_selected = 3`         | `neighbor <N> addpath-tx-best-selected 3`                 |
| `addpath_tx_bestpath_per_as = ""`      | `neighbor <N> addpath-tx-bestpath-per-AS`                 |
| `addpath_rx_disable = true`            | `neighbor <N> disable-addpath-rx`                         |
| `addpath_rx_disable = false`           | `no neighbor <N> disable-addpath-rx`                      |
| `addpath_rx_paths_limit = 100`         | `neighbor <N> addpath-rx-paths-limit 100`                 |

### Integration Test

A new test `test_bgp_addpath.py` will be added to verify the ADDPATH functionality end-to-end,
covering configuration of all TX and RX attributes, capability negotiation with a peer, and
correct behavior of the RX-disabled default across restart and reboot scenarios.

---

## Limitations and Pending Issues

### FRR Does Not Enforce ADDPATH RX Paths Limit Locally

The `addpath-rx-paths-limit` value is conveyed to the peer during ADD-PATH capability
negotiation (per draft-abraitis-idr-addpath-paths-limit). However, FRR currently does not
enforce this limit locally on the receiving side — it is the peer's responsibility to honor the
advertised limit.

A peer that does not implement the paths-limit draft, or a misbehaving peer, may still send more
paths than the declared limit. In such cases, the SONiC switch will accept and process all
received paths, potentially consuming memory beyond the intended limit.

Effort is underway to patch FRR to enforce the RX paths limit locally, dropping excess paths
from non-compliant peers. Until that patch is available, the `addpath_rx_disable` default of
`true` serves as the primary protection against this scenario. Operators who enable ADDPATH RX
should be aware of this limitation and deploy `addpath_rx_paths_limit` in conjunction with peer
configuration audits.

---

## References

- [RFC 7911 — Advertisement of Multiple Paths in BGP](https://www.rfc-editor.org/rfc/rfc7911)
- [draft-abraitis-idr-addpath-paths-limit — Paths Limit for BGP ADD-PATH](https://www.ietf.org/archive/id/draft-abraitis-idr-addpath-paths-limit-04.txt)
- [SONiC HLD Guidelines](https://github.com/sonic-net/SONiC/blob/master/doc/guidelines)
- `src/sonic-yang-models/yang-models/sonic-bgp-common.yang`
