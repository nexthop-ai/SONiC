# AS range on BGP dynamic-neighbor listen ranges

| | |
|---|---|
| Ticket | [NOS-18750](https://nexthopai.atlassian.net/browse/NOS-18750) — Support an AS range on BGP listen range for dynamic neighbour establishment |
| FRR `bgpd` | nexthop-ai/private-sonic-frr [#298](https://github.com/nexthop-ai/private-sonic-frr/pull/298) (FRR 10.5.4 base) |
| YANG + frrcfgd | nexthop-ai/private-sonic-buildimage [#12270](https://github.com/nexthop-ai/private-sonic-buildimage/pull/12270) |
| nhcli | nexthop-ai/sonic-nhcli [#921](https://github.com/nexthop-ai/sonic-nhcli/pull/921) |
| sonic-mgmt test | nexthop-ai/private-sonic-mgmt [#4593](https://github.com/nexthop-ai/private-sonic-mgmt/pull/4593) |

## Table of contents

1. [Motivation](#1-motivation)
   - [Other vendors offer this](#other-vendors-offer-this)
2. [Summary of behavior](#2-summary-of-behavior)
   - [Control plane only](#control-plane-only)
3. [FRR CLI](#3-frr-cli)
   - [3.1 Syntax](#31-syntax)
   - [3.2 Syntax checks](#32-syntax-checks)
   - [3.3 Scope and number of statements](#33-scope-and-number-of-statements)
   - [3.4 Interaction with peer-group remote-as](#34-interaction-with-peer-group-remote-as)
   - [3.5 Show commands](#35-show-commands)
4. [FRR implementation](#4-frr-implementation)
   - [4.1 Data structures](#41-data-structures)
   - [4.2 Parsing](#42-parsing)
   - [4.3 Admission check](#43-admission-check)
   - [4.4 Configuration changes take effect immediately](#44-configuration-changes-take-effect-immediately)
   - [4.5 Files changed](#45-files-changed)
   - [4.6 Compatibility](#46-compatibility)
5. [CONFIG_DB and YANG](#5-config_db-and-yang)
6. [frrcfgd](#6-frrcfgd)
   - [6.1 Live configuration](#61-live-configuration)
   - [6.2 Validation](#62-validation)
   - [6.3 Boot render](#63-boot-render)
   - [6.4 Tests](#64-tests)
7. [nhcli](#7-nhcli)
   - [7.1 Command](#71-command)
   - [7.2 Checks](#72-checks)
   - [7.3 Implementation](#73-implementation)
   - [7.4 Tests](#74-tests)
8. [FRR topotest](#8-frr-topotest)
   - [8.1 Topology](#81-topology)
   - [8.2 Test cases](#82-test-cases)
   - [8.3 Not covered by the topotest](#83-not-covered-by-the-topotest)
9. [sonic-mgmt test](#9-sonic-mgmt-test)
10. [FAQ](#10-faq)

## 1. Motivation

`bgp listen range <prefix> peer-group <PG>` lets bgpd accept BGP sessions from any
address in a prefix, without configuring each neighbor. It is the usual way to bring up
a fabric tier whose members are not known in advance, such as leaves dialing in to a
spine or servers peering with a ToR.

Today the only control over *which AS* such a neighbor may use is the peer-group's
`remote-as`:

- `remote-as <N>`: every dynamic neighbor must be in AS N. This does not fit a fabric
  where each leaf or rack has its own AS.
- `remote-as external` / `remote-as auto`: any eBGP AS is accepted. Any device that
  can reach the prefix can join with any AS, including a mis-cabled or misconfigured
  device, or one from another pod that happens to use an overlapping address plan.

Operators want something in between: accept dynamic neighbors from the prefix, but only
if their AS is in an expected set, for example the leaf AS block 65000-65100.

### Other vendors offer this

| Vendor | Mechanism |
|---|---|
| Arista EOS | `peer-filter` with `match as-range A-B result accept`, attached as `bgp listen range <prefix> peer-group <PG> peer-filter <name>` |
| Cisco NX-OS | `neighbor <prefix> remote-as route-map <RM>`, where the route-map uses `match as-number <list/range>` |
| Cisco IOS / IOS-XE | `bgp listen range <prefix> peer-group <PG>` together with `neighbor <PG> alternate-as <AS> ...` (a short list of accepted ASes) |

Customers migrating from these platforms expect the same control. FRR 10.5.x (including
10.5.1 and 10.5.4) has no equivalent; it only has the peer-group `remote-as` above.

There is no RFC for this feature; it is a local acceptance policy. The behavior on
rejection reuses RFC 4271's OPEN Message Error / Bad Peer AS (section 6.2).

## 2. Summary of behavior

- A listen range may carry an optional list of AS numbers and AS ranges.
- When a dynamic neighbor from that prefix sends its OPEN, bgpd checks the neighbor's
  AS against the list. If the AS is not in any range, bgpd sends a NOTIFICATION
  **OPEN Message Error (2) / Bad Peer AS (2)** and drops the connection. No session is
  formed and no routes are exchanged.
- A listen range without the list behaves exactly as today.
- Changing the list takes effect **immediately**; no `clear bgp` is needed (section 4.4).
- In SONiC the list is the `as_range` field of `BGP_GLOBALS_LISTEN_PREFIX`:
  - nhcli configures it;
  - frrcfgd applies it to bgpd;
  - the boot render keeps it across a bgp restart (sections 5-7).

### Control plane only

The feature is purely a control-plane admission check on the BGP OPEN message:

- It changes no route attributes, best-path selection, update-group formation, or
  route advertisement.
- It adds nothing to zebra, the RIB, the FIB, fpmsyncd/SAI, or the ASIC. **The data
  plane is not affected at all.**
- Routes appear or disappear only as an indirect, ordinary consequence of a session
  coming up or going down, exactly as with any BGP neighbor.

## 3. FRR CLI

### 3.1 Syntax

```
router bgp <ASN> [vrf <VRF>]
 bgp listen range <A.B.C.D/M|X:X::X:X/M> peer-group PGNAME [as-range ASRANGE]
```

`ASRANGE` is one token: a comma-separated list, **without spaces**, of items. Each item
is one of:

- `A`: a single AS, the same as `A-A`.
- `A-B`: every AS from A to B, both inclusive.

**There is no space anywhere in the comma-separated range list**: not after a comma, and
not around the `-` of a range. `65000-65100,65105` is valid; `65000-65100, 65105` and
`65000 - 65100` are not. A space ends the `ASRANGE` token, so the CLI refuses the rest
of the line.

Example:

```
router bgp 65000
 neighbor LEAFS peer-group
 neighbor LEAFS remote-as external
 bgp listen range 10.0.0.0/24 peer-group LEAFS as-range 65000-65100,65105,63001
```

The `no` form is unchanged in meaning:

```
 no bgp listen range <prefix> peer-group PGNAME [as-range ASRANGE]
```

It removes the whole listen range and its dynamic neighbors, as today. The optional
`as-range ASRANGE` is accepted, so a line copied from `show running-config` can be
negated, but its value is ignored.

### 3.2 Syntax checks

| Input | Result |
|---|---|
| AS outside 1..4294967295, e.g. `0`, `4294967296` | `% Malformed AS number: <input>` |
| Leading zero, e.g. `065000` | `% Malformed AS number` |
| Empty item, e.g. `65000,,1`, `65000-`, trailing comma | `% Malformed AS number` |
| Trailing junk, e.g. `65000x` | `% Malformed AS range` |
| Downward range, e.g. `65100-65000` | `% AS range start is greater than its end` |
| More than 256 items | `% Too many AS ranges` |
| asdot notation, e.g. `1.10` (= 65546) | Accepted, the same as FRR's other AS inputs; stored and displayed in plain notation |

Any failure rejects the whole command and leaves the existing configuration unchanged.

**Overlap.** bgpd *accepts* overlapping, duplicate, and adjacent items, sorts them, and
merges them. For example, `65050,65000-65100,65101-65110` is stored and shown as
`65000-65110`. This follows the FRR convention of normalizing rather than refusing a
harmless input.

The SONiC layers above FRR are deliberately **stricter**. The YANG model states the
rule, and nhcli and frrcfgd enforce it: they refuse overlaps, so CONFIG_DB and the
bgpd running config never differ. An overlap there is almost always a typo. See
sections 6.2 and 7.2.

### 3.3 Scope and number of statements

**VRF.** Supported. `bgp listen range` is per BGP instance and so is `as-range`. Each
`router bgp <ASN> vrf <VRF>` has its own listen ranges, each with its own list. No
VRF-specific code was needed. The topotest covers only the default VRF.

**Peer group.** Yes. A listen range always belongs to a peer group; this is existing FRR
behavior. `as-range` is an attribute of the *listen range*, i.e. of the (prefix,
peer-group) pair, not of the peer group as a whole.

**IPv4 and IPv6.** Both are supported. The list is per listen-range prefix, so an IPv4
range and an IPv6 range can have different lists.

**How many statements.**

- One statement per prefix. A prefix can be attached to only one peer group (existing
  rule: `% Same listen range is attached to peer-group X`), and listen ranges may not
  overlap (`% Listen range overlaps with existing listen range`).
- A peer group may have several listen ranges (different, non-overlapping prefixes),
  and each carries its own independent `as-range`.
- Entering the command again for the same prefix and peer group **replaces** the list;
  it does not append to it.
- Entering it without `as-range` **removes** the restriction and keeps the listen
  range.

Two prefixes in the same peer group, each with its own AS ranges:

```
router bgp 65000
 neighbor LEAFS peer-group
 neighbor LEAFS remote-as external
 bgp listen range 10.0.0.0/24 peer-group LEAFS as-range 65000-65100
 bgp listen range 10.1.0.0/24 peer-group LEAFS as-range 65050-65200,65400
```

Here a neighbor from 10.0.0.0/24 is accepted only with an AS in 65000-65100, and a
neighbor from 10.1.0.0/24 only with 65050-65200 or 65400. Both sets of neighbors share
the LEAFS peer-group configuration.

- **The two AS lists may overlap.** 65050-65100 is in both, which is allowed. Each list
  is checked only against neighbors from its own prefix, so AS 65075 is accepted from
  either prefix.
- **A listed AS applies only to its own prefix.** A neighbor from 10.1.0.0/24 in AS 65020
  is rejected, although 65020 is listed on 10.0.0.0/24.
- **The prefixes themselves must not overlap.** Adding
  `bgp listen range 10.0.0.0/16 peer-group LEAFS ...` would be refused with
  `% Listen range overlaps with existing listen range`. So a neighbor's address always
  matches exactly one listen range, and exactly one AS list applies to it.

Replacing and removing the list of one prefix:

```
 bgp listen range 10.0.0.0/24 peer-group LEAFS as-range 65000-65100
 bgp listen range 10.0.0.0/24 peer-group LEAFS as-range 65000-65050,65200   ! replaces
 bgp listen range 10.0.0.0/24 peer-group LEAFS                              ! removes restriction
```

### 3.4 Interaction with peer-group remote-as

The `as-range` check runs first, then the existing `remote-as` check. A neighbor must
pass both, so the accepted set is the **intersection**:

| peer-group remote-as | Effect with as-range |
|---|---|
| `external` | Any eBGP AS in the list (recommended use) |
| `auto` | Any AS in the list; iBGP if it equals the local AS |
| `<N>` | Only N, and only if N is in the list; the list adds nothing useful |
| `internal` | Only the local AS, and only if it is in the list |

The documentation recommends `remote-as external` or `remote-as auto`.

### 3.5 Show commands

**`show running-config`.** Emits the normalized list:

```
 bgp listen range 10.0.0.0/24 peer-group LEAFS as-range 63001,65000-65100,65105
```

**`show bgp peer-group <PG>`.** The text output lists the list after each range:

```
  1 IPv4 listen range(s)
    10.0.0.0/24 as-range 63001,65000-65100,65105
```

**`show bgp peer-group <PG> json`.** Adds `asRanges`, keyed by prefix, under
`dynamicRanges.<AFI>`. It is present only when some range of that AFI has a list, so
existing consumers of `ranges` are unaffected:

```json
"dynamicRanges": {
  "IPv4": {
    "count": 1,
    "ranges": ["10.0.0.0/24"],
    "asRanges": {"10.0.0.0/24": ["63001", "65000-65100", "65105"]}
  }
}
```

**`debug bgp neighbor-events`.** Logs each rejection:

```
<host> bad OPEN, remote AS is 65530, not accepted by listen range 10.0.0.0/24
```

and each disconnect caused by a configuration change:

```
Deleting dynamic neighbor <host> AS <as> group <PG>, AS not accepted by listen range <prefix>
```

## 4. FRR implementation

### 4.1 Data structures

In `bgpd/bgpd.h`: before this change, `peer_group->listen_range[AFI]` was a list of
`struct prefix *`. It is now a list of:

```c
struct bgp_as_range {
	as_t min;
	as_t max;            /* inclusive */
};

struct peer_group_listen_range {
	struct prefix prefix;
	struct bgp_as_range *as_range;   /* sorted, merged; NULL = no restriction */
	uint16_t as_range_count;
};

#define BGP_LISTEN_RANGE_AS_RANGE_MAX 256
```

Memory comes from a new MTYPE, `BGP_LISTEN_RANGE`, so `show memory` accounts for it.
The cost is at most 256 × 8 bytes per listen range.

All existing users of the list now take `&lr->prefix`:

- the listen-range lookups and the overlap check;
- MD5 password set and unset on ranges;
- `show bgp neighbor` ("Belongs to the subnet range group");
- config write;
- peer-group delete.

### 4.2 Parsing

`bgp_as_range_parse()` in `bgpd/bgpd.c`:

- Each AS is parsed with FRR's `asn_str2asn_parse()`, the same parser as every other AS
  input, so plain and asdot notation, the 0 rule, the leading-zero rule, and overflow
  are handled consistently.
- The items are collected, sorted with `qsort`, and overlapping or adjacent items are
  merged in one pass.
- The result is a compact, sorted, disjoint array, so the stored form is canonical and
  config write is deterministic.

### 4.3 Admission check

In `bgp_open_receive()` (`bgpd/bgp_packet.c`), for a dynamic neighbor:

1. Find its listen range with `peer_group_lookup_dynamic_neighbor_range()`.
2. Test the OPEN's AS with `peer_group_listen_range_as_match()`.
3. If the AS does not match, send NOTIFICATION 2/2 (Bad Peer AS, with the AS as data,
   as for the existing remote-as mismatch) and return `BGP_Stop`. The dynamic peer
   object is then deleted, as for any failed dynamic neighbor.

Abbreviated code (new lines marked `+`; debug logging and unrelated checks elided):

```c
static int bgp_open_receive(struct peer_connection *connection,
			    struct peer *peer, bgp_size_t size)
{
	...
	/* OPEN parsed; capabilities processed, so remote_as is the 4-byte AS */
	...
+	/* Check the AS of a dynamic neighbor against its listen range. */
+	if (peer_dynamic_neighbor(peer) && peer->group) {
+		struct peer_group_listen_range *lr = NULL;
+		struct prefix prefix;
+
+		if (sockunion2hostprefix(&connection->su, &prefix))
+			lr = peer_group_lookup_dynamic_neighbor_range(peer->group,
+								      &prefix);
+		if (lr && !peer_group_listen_range_as_match(lr, remote_as)) {
+			bgp_notify_send_with_data(connection,
+						  BGP_NOTIFY_OPEN_ERR,
+						  BGP_NOTIFY_OPEN_BAD_PEER_AS,
+						  notify_data_remote_as, 2);
+			return BGP_Stop;
+		}
+	}

	/* Check neighbor as number. */          /* existing remote-as check */
	if (peer->as_type == AS_UNSPECIFIED) {
	...
}
```

The matcher it calls, in `bgpd/bgpd.c`:

```c
bool peer_group_listen_range_as_match(const struct peer_group_listen_range *lr,
				      as_t as)
{
	if (!lr->as_range_count)        /* no as-range: accept any AS */
		return true;

	for (i = 0; i < lr->as_range_count; i++)
		if (as >= lr->as_range[i].min && as <= lr->as_range[i].max)
			return true;

	return false;
}
```

The check sits immediately before the existing "Check neighbor as number" step. That
step already runs after 4-byte AS capability processing, so `remote_as` is the real
4-byte AS and not AS_TRANS (23456).

The match is a linear scan over at most 256 ranges, once per received OPEN. This is
negligible, so no binary search was added.

### 4.4 Configuration changes take effect immediately

Adding or replacing a list (`peer_group_listen_range_add()`) calls
`peer_group_listen_range_as_reconcile()`. For every dynamic neighbor of that range, it
decides as follows.

| Neighbor state | AS still accepted? | Action |
|---|---|---|
| OpenConfirm or Established | yes | **Nothing.** The session is not reset; no route churn. |
| OpenConfirm or Established | no | NOTIFICATION Cease / Peer De-configured, then the dynamic peer is deleted. |
| Before OpenConfirm (no OPEN received yet) | — | Nothing now; checked when its OPEN arrives (section 4.3). |

The cases are therefore:

- **AS added to the list.** A neighbor rejected earlier comes up on its own next connect
  retry. Dynamic neighbors are passive on our side, so the remote end re-dials.
- **AS removed from the list.** Matching sessions go down immediately.
- **Other sessions.** Unaffected.
- **List removed.** Every AS the peer-group `remote-as` allows is accepted from then on;
  no session is reset.

**No `clear bgp` is required** in any case.

### 4.5 Files changed

| File | Change |
|---|---|
| `bgpd/bgpd.h` | New structs, constant, prototypes; listen-range lookup signatures |
| `bgpd/bgpd.c` | Parse, format, match, reconcile; add/del/delete/lookup on the new struct; MD5 call sites |
| `bgpd/bgp_packet.c` | Admission check in `bgp_open_receive()` |
| `bgpd/bgp_vty.c` | CLI (set/replace/no), config write, `show bgp peer-group` text and JSON, `show bgp neighbor` |
| `bgpd/bgp_memory.[ch]` | `MTYPE_BGP_LISTEN_RANGE` |
| `doc/user/bgp.rst` | User documentation for `as-range` |
| `tests/topotests/bgp_listen_range_as_range/` | New topotest |

### 4.6 Compatibility

- **Existing configuration.** A config without `as-range` parses, behaves, and is
  written exactly as before.
- **Northbound.** `bgp listen range` is a classic vty command in bgpd, not a northbound
  (mgmtd) YANG node, so there is no FRR YANG change.
- **Downgrade.** An older bgpd will reject a config line that carries `as-range`, so
  remove it before downgrading.
- **Upstream.** The change is local to the Nexthop FRR fork and is self-contained. It
  is a candidate for upstreaming to FRRouting.

## 5. CONFIG_DB and YANG

The list is an optional field on the existing listen-prefix table:

```
BGP_GLOBALS_LISTEN_PREFIX|<vrf>|<prefix>
    peer_group = LEAFS
    as_range   = 65000-65100,65105,63001      <- new, optional
```

YANG (`src/sonic-yang-models/yang-models/`):

- **`sonic-bgp-common.yang`.** New typedef `bgp_as_range_list`. It is a string whose
  pattern accepts exactly the list syntax of section 3.1:
  - plain ASNs 1..4294967295 with no leading zero;
  - each one optionally followed by `-ASN`;
  - items joined by `,`, with no spaces.
- **asdot is not accepted in CONFIG_DB**, even though bgpd takes it. CONFIG_DB keeps one
  spelling, the one bgpd writes back, so what frrcfgd pushes and what bgpd shows stay
  the same.
- **Rules a pattern cannot state.** A pattern cannot compare numbers, so these are
  written in the typedef description and enforced by nhcli and frrcfgd:
  - every range has A <= B;
  - no AS is listed twice (items must not overlap);
  - at most 256 items.
- **`sonic-bgp-peergroup.yang`.** New `leaf as_range { type bgpcmn:bgp_as_range_list; }`
  on `BGP_GLOBALS_LISTEN_PREFIX_LIST`.
- **Revisions and tests.** Both modules get a 2026-10-29 revision. Positive and negative
  cases are in `tests/yang_model_tests/tests{,_config}/bgp.json`, and
  `sample_config_db.json` gains an `as_range`.

**Only frrcfgd consumes the field.** In traditional (bgpcfgd) mode `as_range` is ignored,
so the sonic-mgmt test skips that mode.

## 6. frrcfgd

`src/sonic-frr-mgmt-framework/frrcfgd/frrcfgd.py`, used in `frr_mgmt_framework` config
mode.

### 6.1 Live configuration

The listen-prefix key map now hands both fields to a handler:

```python
listen_prefix_key_map = [(('peer_group', '++as_range'), 'bgp listen range', hdl_listen_range)]
```

`hdl_listen_range()` turns each CONFIG_DB change into exactly one vtysh command:

| CONFIG_DB change | Command sent to bgpd |
|---|---|
| Row added, or `as_range` added/changed | `bgp listen range P peer-group PG as-range X` |
| Row added without `as_range` | `bgp listen range P peer-group PG` |
| Only `as_range` deleted | `bgp listen range P peer-group PG` (re-entered without it) |
| Row deleted, or `peer_group` changed | `no bgp listen range P peer-group PG` |

A change or removal of `as_range` is applied by **re-entering** the line, never by
`no` + add. bgpd replaces the list in place (section 4.4), so sessions whose AS is still
accepted are not reset. A `no` first would delete every dynamic neighbor of the range.

On a bgp container start, the same handler replays the table: `BGPKeyMapList.run_command`
with OP_ADD.

### 6.2 Validation

`as_range_list_valid()` checks each value before it is sent:

- every item matches `[1-9][0-9]*(-[1-9][0-9]*)?`;
- every AS is at most 4294967295, and A <= B;
- at most 256 items (`AS_RANGE_LIST_MAX`, matching bgpd);
- after sorting, no item overlaps the next.

This repeats the nhcli checks (section 7.2) because CONFIG_DB can be written by other
paths: `sonic-db-cli`, `config load`, GCU or gNMI.

An invalid value is logged and **nothing is sent**:

```
ERR bgp#frrcfgd: invalid as_range 65100-65000 for listen range 10.0.0.0/24: each range must be A-B with A <= B, no AS listed twice, at most 256 items
```

bgpd keeps its previous state for that prefix:

- for a new row, no listen range is created;
- for a change, the old list stays in force.

Either way the result fails closed: an invalid list never opens the range to any AS.

### 6.3 Boot render

`templates/bgpd/bgpd.conf.db.j2` renders bgpd.conf at container start. It now appends
the list:

```jinja
 bgp listen range {{lpfx[1]}} peer-group {{lpfx_val['peer_group']}}{% if 'as_range' in lpfx_val %} as-range {{lpfx_val['as_range']}}{% endif %}

```

The template is rendered with `trim_blocks`, which eats the newline after a closing
`{% endif %}`, so the line is followed by an explicit blank line. Without the boot
render, a bgp restart would reopen the listen range to every AS while CONFIG_DB still
holds the list.

### 6.4 Tests

`src/sonic-frr-mgmt-framework/tests/test_listen_range_as_range.py` covers:

- the boot render and the live handler produce the same line;
- change in place, add, removal of `as_range` only, and row delete;
- invalid values (each sends nothing);
- the `as_range_list_valid()` cases;
- the replay path through `BGPKeyMapList.run_command`.

The full frrcfgd suite passes (653 tests).

## 7. nhcli

### 7.1 Command

nhcli builds its config commands from the YANG models at runtime, so the new leaf
appears without any per-command code:

```
set routing bgp globals-listen-prefix default 10.0.0.0/24 peer-group LEAFS as-range 65000-65100,65105,63001
delete routing bgp globals-listen-prefix default 10.0.0.0/24 as-range
commit
```

The YANG pattern (section 5) already refuses a malformed list at `set`. That covers
spaces, empty items, `0`, values above 4294967295, leading zeros, and asdot.

### 7.2 Checks

As tight as the model allows. A pattern cannot compare numbers, so three mistakes would
pass YANG. frrcfgd would then refuse them only in its log, after `commit` had reported
success. nhcli refuses them at the prompt:

| Value | nhcli message |
|---|---|
| `65000,65100-65050` | `AS range 65100-65050 runs downward; write it as 65050-65100` |
| `65000-65100,65050` | `AS ranges 65000-65100 and 65050 overlap; list each AS once` |
| `65105,65105` | `AS ranges 65105 and 65105 overlap; list each AS once` |
| 257 items | `lists 257 AS ranges; a listen range takes at most 256` |

Each message is prefixed with the path in CLI words. Adjacent items such as
`65000-65004,65005-65010` share no AS and are accepted. Order is free.

A malformed value gets only the YANG pattern's message. The model check skips it rather
than report it twice.

### 7.3 Implementation

`crates/nhcli-config-mode/`:

- **`model_values.rs` (new).** Rules for leaves whose value the model constrains beyond
  what YANG can express, keyed by (CONFIG_DB table, field):
  - `check_model_values(config)` returns a `ModelRefusal { table, key, field, value, reason }`
    for each value refused.
  - The first rule is `("BGP_GLOBALS_LISTEN_PREFIX", "as_range") -> check_as_range_list`.
  - `AS_RANGE_LIST_MAX = 256` matches bgpd.
- **`validator.rs`.** `SchemaValidator` adds each refusal as a `ConstraintViolation`. It
  is a model violation, not a device refusal: the same value is refused on every device,
  off-box included.
- **`libyang_validator.rs`.** After the device-value checks, model refusals are hard
  errors. The path is shown in CLI vocabulary via `configdb_to_display`.

The check sits in the validators rather than the `set` dispatcher, so it covers every
path that writes a candidate: `set`, `load`, `nhcli -f`, and rollback.

Changelog fragment: `debian/changelog.d/NOS-18750-listen-range-as-range.md`
(`bump: minor`).

### 7.4 Tests

- **Unit tests in `model_values.rs`:**
  - accepted lists (single AS, ranges, full 1-4294967295, adjacent, unordered);
  - downward ranges;
  - overlap and duplicates;
  - the 256 limit;
  - malformed values left to YANG;
  - only the one table and field is checked.
- **Validator test.** A downward range is refused as a model violation.
- **Full `yang` build test run.** 1256 passed, 0 failed.

## 8. FRR topotest

`tests/topotests/bgp_listen_range_as_range` in private-sonic-frr.

### 8.1 Topology

```
    r2 (AS 65005)   r3 (AS 65020)   r4 (AS 65105)
          \               |               /
           +------------- s1 ------------+          192.168.255.0/24
                          |
                    r1 (AS 65000)
     neighbor LEAFS remote-as external
     bgp listen range 192.168.255.0/24 peer-group LEAFS as-range 65105,65001-65010
```

- r1 is the listener; r2, r3 and r4 dial r1 as static neighbors.
- Timers are 1/3 s with a connect retry of 1 s, so rejected peers retry quickly.
- The configured list is deliberately unsorted, so the test also covers normalization.

### 8.2 Test cases

| Test | What it checks |
|---|---|
| `test_bgp_listen_range_as_range_initial` | r2 (65005, inside 65001-65010) and r4 (65105, single-AS item) reach Established. r3 (65020, not listed) never establishes, and its last error is `0202` (OPEN Message Error / Bad Peer AS). |
| `test_bgp_listen_range_as_range_show` | Running-config shows the sorted list `as-range 65001-65010,65105`. `show bgp peer-group LEAFS json` has `asRanges: {"192.168.255.0/24": ["65001-65010","65105"]}`. |
| `test_bgp_listen_range_as_range_change` | Replaces the list with `65001-65004,65020,65105` without clearing anything. r3 (added) establishes; r2 (removed) is disconnected and then rejected with `0202`; r4 (kept) stays Established and its `connectionsDropped` counter does not change (no reset). |
| `test_bgp_listen_range_as_range_remove` | Re-enters the range without `as-range`. All of r2, r3 and r4 establish, and `as-range` is gone from running-config. |
| `test_bgp_listen_range_as_range_invalid` | `65100-65000`, `65000,,1`, `0`, `4294967296` and `65000x` are each rejected with a `%` error, and running-config is unchanged. |

Result: 5/5 pass in the FRR topotest container, both on the original base and after
rebasing onto `87be2ddc81`.

To run it:

```
cd tests/topotests
TOPOTEST_NOLOAD=1 sudo -E pytest -s bgp_listen_range_as_range/
```

### 8.3 Not covered by the topotest

Each gap is covered elsewhere or is low risk:

- **VRF instance.** Same code path, no VRF-specific logic.
- **IPv6 listen range.** Same code path, keyed by AFI.
- **asdot input.** Comes from the shared FRR ASN parser.
- **More than 256 items.** Unit-level parser limit, also enforced above FRR by nhcli
  and frrcfgd.
- **Reconcile of a neighbor caught in OpenConfirm.** Timing-dependent.

## 9. sonic-mgmt test

`tests/bgp/test_bgp_listen_range_as_range.py` (topology t0) covers the SONiC path end to
end: CONFIG_DB → frrcfgd → bgpd, including a bgp container restart.

Setup:

- Three exabgp speakers on the PTF dial the DUT's VLAN address from 192.0.2.0/29, a
  TEST-NET-1 range that no existing listen range covers:
  - in_range: AS 65505
  - out_of_range: AS 65530
  - single_as: AS 65520
- Peer group ASRANGE_TEST: `peer_type external`, ebgp-multihop.
- Initial `as_range`: `65500-65509,65520`.

| Test | What it checks |
|---|---|
| `test_as_range_admits_only_listed_as` | 65505 and 65520 establish; 65530 is refused |
| `test_as_range_change_is_applied_in_place` | Changing to `65500-65504,65520,65530` drops 65505 and admits 65530, while 65520's session is not reset |
| `test_as_range_removal_admits_any_as` | Deleting `as_range` admits all three |
| `test_as_range_survives_bgp_restart` | After `systemctl restart bgp`, as-range is back in bgpd (boot render) and still enforced |

The test runs only in `frr_mgmt_framework` mode. A new `frr_frrcfgd_only` marker skips
the traditional variant before switching modes.

Result on gold208 (t0-8, NH-4010) with build
`SONiC.fredxia.nos-18750-build.Nexthop.282559-e34fa660c0`: 4 passed, and the 4
traditional variants were skipped.

## 10. FAQ

**Do I need to clear BGP sessions after changing `as-range`?**
No. The change is applied immediately (section 4.4), and frrcfgd re-enters the line
rather than deleting it (section 6.1). Sessions whose AS is no longer listed are torn
down, sessions still listed are untouched, and newly listed ASes are admitted on the
peer's next connection attempt.

**Does it affect forwarding?**
No. It is a control-plane check on the OPEN message only (section 2).

**Is overlap rejected?**
Not by bgpd, which merges overlaps. nhcli (at `set`) and frrcfgd (before applying)
refuse overlap, downward ranges and more than 256 items, as the YANG model states
(sections 6.2 and 7.2).

**What if an invalid list reaches CONFIG_DB without going through nhcli?**
frrcfgd logs it and does not apply it. bgpd keeps its previous state for that prefix,
so the range never opens to any AS by mistake (section 6.2).

**Does it work in a VRF?**
Yes; it is per BGP instance (section 3.3), and the CONFIG_DB key carries the VRF.

**Is it only for peer groups?**
Yes. Listen ranges are always bound to a peer group, and `as-range` is per (prefix,
peer-group) listen range.

**Can a peer group have more than one `as-range`?**
Yes, one per listen-range prefix. A given prefix has exactly one statement, and
re-entering it replaces the list.

**What does a rejected peer see?**
A NOTIFICATION with OPEN Message Error / Bad Peer AS, carrying its own AS. It retries
per its connect timer and is rejected again until its AS is listed.

**Do rejected peers count against `bgp listen limit`?**
Only for the moment between TCP accept and the OPEN check. The dynamic peer is deleted
on rejection.

**Does it work with bgpcfgd (traditional config mode)?**
No. Only frrcfgd consumes `as_range`, and bgpcfgd ignores it.
