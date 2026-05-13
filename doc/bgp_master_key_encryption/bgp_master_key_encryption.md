# BGP Peering Password Encryption at Rest — High-Level Design

## 1. Revision

| Revision | Date       | Author    | Change Description         |
|----------|------------|-----------|----------------------------|
| 0.1      | 2026-05-13 | Fred Xia  | Initial draft              |

---

## 2. Scope

This document describes the design for encrypting BGP peering passwords at rest in SONiC CONFIG_DB using a symmetric master key. The feature targets the `BGP_NEIGHBOR` and `BGP_PEER_GROUP` tables and ensures that plaintext passwords are not stored in Redis or in saved JSON config files such as `config_db.json`. The encryption is transparent to the operator via standard SONiC management tools (`config` CLI, `sonic-cfggen`).

---

## 3. Definitions / Abbreviations

| Term            | Definition                                                           |
|-----------------|----------------------------------------------------------------------|
| AES-GCM         | Advanced Encryption Standard — Galois/Counter Mode (authenticated symmetric cipher) |
| AAD             | Additional Authenticated Data (used in AES-GCM to bind ciphertext to context) |
| BGP             | Border Gateway Protocol                                              |
| CONFIG_DB       | SONiC's Redis-based configuration database                           |
| FRR             | Free Range Routing — the routing daemon suite used by SONiC          |
| HLD             | High-Level Design                                                    |
| MD              | Message Digest (one-way hash; NOT used in this feature)              |
| YANG            | Yet Another Next Generation — data modelling language (RFC 7950)     |
| Level 0         | No encryption; plaintext storage                                     |
| Level 8         | AES-GCM encryption (aligned with Cisco/Arista password type 8)      |

---

## 4. Overview

Community SONiC stores BGP peering passwords in plaintext in Redis CONFIG_DB and in saved JSON configuration files (e.g., `config_db.json`). Any local account on the SONiC host can read these files or dump Redis data. SONiC Redis does not provide fine-grained ACLs to restrict read access.

Proprietary vendors — Cisco, Arista, and Dell Enterprise SONiC — already support encryption of BGP passwords at rest. This feature brings similar functionality to SONiC.

The NSA [Cyber Security Information Sheet](https://media.defense.gov/2022/Feb/17/2002940795/-1/-1/1/CSI_CISCO_PASSWORD_TYPES_BEST_PRACTICES_20220217.PDF) recommends Type 8 (AES-GCM) as the preferred password encryption method, matching the Level 8 designation in this design.

**Design philosophy:**

- Encryption applies only to CONFIG_DB (Redis and JSON). FRR routing daemons continue to receive and use cleartext passwords. This avoids introducing a dependency on FRR for decryption and prevents complication of the FRR project.
- The encryption mechanism intercepts existing `ConfigDBConnector` write methods, making it transparent to all management tools that use the standard Python ConfigDB API (i.e., `config` CLI and `sonic-cfggen`).
- The design is built for master key distribution from a centralized server. A network controller seeds the master key to each device via standard management interfaces. Local generation of random master keys is intentionally avoided: a locally generated key that is not recorded externally becomes unrecoverable if local storage is corrupted, making all encrypted passwords permanently inaccessible.
- No additional daemons are introduced; the feature is implemented entirely as libraries.

---

## 5. Requirements

### 5.1 Functional Requirements

1. Encryption can be enabled or disabled. When enabled, BGP neighbor and peer group passwords (`BGP_NEIGHBOR.auth_password`, `BGP_PEER_GROUP.auth_password`) must be encrypted in CONFIG_DB. When disabled, passwords are stored as cleartext in CONFIG_DB.
2. Encryption and decryption must be transparent when using `config` CLI commands and `sonic-cfggen`.
3. `config reload`, `config apply`, `sonic-cfggen -w` must honor the configured encryption level without operator intervention.
4. A management CLI tool (`bgp-master-key`) must be available for:
   - Setting/rotating the master key
   - Activating and deactivating encryption
   - Manual encryption/decryption of passwords or config files (for debugging and recovery)
   - Listing historical master keys
5. Encryption must be symmetric (AES-GCM, reversible) — not a one-way hash — because FRR requires the plaintext password for BGP MD5 session establishment.
6. The master key storage file must be accessible only by root (`chmod 600`).
7. Historical master keys (up to 8) must be retained to support decryption of entries encrypted with previous keys.
8. When no master key is configured, passwords are stored as plaintext (backward compatible).

### 5.2 Non-Requirements / Exclusions

- **FRR password encryption**: FRR daemons (`bgpd`, `frrcfgd`, `bgpcfgd`) continue to consume plaintext passwords. Encrypting passwords within FRR is out of scope.
- **Other protocol passwords**: TACACS, RADIUS, LDAP passwords are out of scope for this release. The encryption library is designed to support them in future via separate feature names.
- **Auto key rotation / local key generation**: No automatic rotation or device-local key generation. The master key must be pushed by the operator or a centralized controller.
- **Native C/C++ tools**: `redis-cli` and `sonic-db-cli` bypass the Python ConfigDB connector and do not go through the encryption hook. This is a known limitation (see [Restrictions/Limitations](#restrictionslimitations)).
- **ASIC or dataplane impact**: This feature is purely a management-plane change with no ASIC or dataplane interaction.

---

## 6. Architecture Design

The feature integrates into the existing SONiC architecture at the CONFIG_DB write path, without modifying FRR or any southbound component.

```
Operator / Controller
        │
        │  config CLI / sonic-cfggen / direct API
        ▼
┌────────────────────────────────────────────────────────────────────┐
│              ConfigDBConnector (Python)                            │
│   sonic-py-swsssdk  OR  sonic-swss-common (SWIG binding)           │
│                                                                    │
│  set_entry()  ──► ConfigDBEncryptor.set_entry()  ──► _set_entry()  │
│  mod_entry()  ──► ConfigDBEncryptor.mod_entry()  ──► _mod_entry()  │
│  mod_config() ──► ConfigDBEncryptor.mod_config() ──► _mod_entry()  │
└──────────────────────────────┬─────────────────────────────────────┘
                               │  writes encrypted password
                               ▼
                         Redis CONFIG_DB
                        ┌──────────────────────────────────────┐
                        │ BGP_NEIGHBOR                         │
                        │ auth_password = <base64 ciphertext>  │
                        │ auth_encryption_level = level_8      │
                        └──────┬───────────────────────────────┘
                               │  bgpcfgd / frrcfgd reads CONFIG_DB
                               ▼
                           FRR Container
                        (bgpd receives PLAINTEXT password
                         after decryption by bgpcfgd/frrcfgd)
```

Key integration points:

| Component             | Role                                                     |
|-----------------------|----------------------------------------------------------|
| `sonic-py-swsssdk`    | Python-native `ConfigDBConnector`; hooks injected into `set_entry`, `mod_entry`, `mod_config` |
| `sonic-swss-common`   | C++ `ConfigDBConnector` with SWIG Python bindings; same hooks added via `%pythoncode` |
| `sonic-utilities`     | Hosts the encryption library, `ConfigDBEncryptor`, and `bgp-master-key` CLI |
| `sonic-yang-models`   | Extended with `password_encryption_level` type and new fields in BGP tables |
| `bgpcfgd` / `frrcfgd` | Reads CONFIG_DB and generates FRR config; must decrypt passwords before writing to FRR |

---

## 7. High-Level Design

### 7.1 Yang Model Changes

The YANG models define both the schema for CONFIG_DB and the data types used throughout the feature. The following changes are made across three YANG files.

#### `sonic-types` — new typedef

```yang
typedef password_encryption_level {
    type enumeration {
        enum level_0 {
            value 0;
            description "Level 0. Clear text";
        }
        enum level_8 {
            value 8;
            description "Level 8. AES-GCM encryption";
        }
    }
    description "Password encryption levels";
}
```

#### `sonic-bgp-common` — new leaf in `sonic-bgp-cmn` grouping

```yang
leaf auth_encryption_level {
    type stypes:password_encryption_level;
    description "Authentication password encryption level";
}
```

This grouping is included in `BGP_NEIGHBOR_LIST` (via `sonic-bgp-neighbor.yang`) and `BGP_PEER_GROUP_LIST`.

#### `sonic-bgp-device-global` — new leaf in STATE container

```yang
import sonic-types {
    prefix stypes;
}
...
leaf peer_password_encryption_level {
    type stypes:password_encryption_level;
    default level_0;
    description "BGP peer password encryption level";
}
```

#### Resulting CONFIG_DB Schema

**`BGP_NEIGHBOR` table** (and `BGP_PEER_GROUP` via the shared `sonic-bgp-cmn` grouping):

| Field | Type | Description |
|-------|------|-------------|
| `auth_password` | string | BGP neighbor authentication password (existing); stores AES-GCM ciphertext when `auth_encryption_level = level_8` |
| `auth_encryption_level` | `password_encryption_level` enum | New. `level_0` = plaintext, `level_8` = AES-GCM encrypted. Absent when no password is configured |

**`BGP_DEVICE_GLOBAL|STATE`** table:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `peer_password_encryption_level` | `password_encryption_level` enum | `level_0` | Current encryption level for all BGP peer passwords on this device |

### 7.2 Software Components

#### 1. Encryption Library — `utilities_common/master_key_encryption.py`

This is the foundational layer. It provides:

- **`aesgcm_encrypt_blob_b64(master_key, value, aad, nonce=None) → str`**: Encrypts `value` using AES-GCM. The 256-bit key is derived by zero-right-padding the master key string to 32 bytes. A 12-byte random nonce is generated unless one is supplied (for testing). The feature name is used as the 16-byte AAD, binding the ciphertext to its context (prevents cross-feature misuse). Returns `base64(nonce[12] || ciphertext || auth_tag[16] || aad[16])`.

- **`aesgcm_decrypt_blob_b64(master_key, combo_b64_str) → str | None`**: Decodes the blob and decrypts. Returns `None` on any error — AES-GCM authentication failure, wrong key, or truncated blob — making it a fail-early operation.

- **`MasterKeyManager`**: Manages the JSON-format master key file at `/etc/bgp_master_key`. Operations:
  - `update_master_key(name, key)`: Prepends a new key entry, pruning to 8 entries maximum.
  - `encrypt_string(name, value) / decrypt_string(name, value)`: Encrypts/decrypts using the current (index 0) master key.
  - `set_enabled(enabled) / enabled()`: Reads/writes the `encryption_enabled` flag in the key file.
  - File locking via `fcntl.LOCK_EX` on a lock file in `/tmp` prevents concurrent access.
  - File permissions are enforced: only `S_IRUSR | S_IWUSR` (mode `0600`) is accepted.
  - Atomic save via temp file + `shutil.copy` + verify-before-rename.

Master key file format (JSON):
```json
{
  "master_keys": {
    "bgp_peer": [
      {
        "name": "bgp_peer",
        "master_key": "<plaintext key>",
        "algorithm": "aes-gcm",
        "timestamp": "2026-02-18T23:40:00+00:00"
      }
    ]
  },
  "encryption_enabled": true
}
```

#### 2. Config DB Encryptor — `utilities_common/config_db_encryption.py`

`ConfigDBEncryptor` is the hook object loaded by `ConfigDBConnector` at runtime. It intercepts write operations and applies encryption or decryption according to the current encryption level stored in CONFIG_DB (`BGP_DEVICE_GLOBAL|STATE.peer_password_encryption_level`).

**Key methods:**

- **`entry_need_encryption(db_connector, table, key, data) → bool`**: Returns `True` if the current encryption level is `level_8`, the table is `BGP_NEIGHBOR`, and `auth_password` is present in `data`.

- **`set_entry(db_connector, table, key, data)`**: Called when encryption is active. Checks if the password is already encrypted (by attempting decryption — if decryption succeeds, the value is already ciphertext). Otherwise encrypts the plaintext and sets `auth_encryption_level = level_8`.

- **`mod_entry(db_connector, table, key, data)`**: Same as `set_entry` logic for `mod_entry` path.

- **`mod_config(db_connector, data)`**: Handles bulk config updates (e.g., `config reload`, `sonic-cfggen -w`). Two cases:
  1. If the bulk update does not modify `BGP_DEVICE_GLOBAL`, only passwords in the incoming `data` are encrypted to match the current level.
  2. If `BGP_DEVICE_GLOBAL` is changing (activation/deactivation), the encryptor reconciles: it reads the full `BGP_NEIGHBOR` table from CONFIG_DB and re-encrypts or decrypts all entries to the target level, merging the result back into `data`.

#### 3. BGP Master Key Library and CLI — `BgpPasswordCryptor` and `bgp_master_key`

Built on top of the encryption library, this package manages BGP-specific master key operations and provides an operator-facing CLI tool (`bgp-master-key`).

**`BgpPasswordCryptor`** is a singleton class that manages the interaction between the master key file and CONFIG_DB. It uses `sonic-yang` for JSON schema validation when processing config files. Its public methods are:

| Method | Description |
|--------|-------------|
| `init() → bool` | Initialize the instance: load the YANG model and, if `--config-input` is given, read the JSON config file instead of connecting to Redis |
| `encryption_enabled() → bool` | Return `True` if `BGP_DEVICE_GLOBAL\|STATE.peer_password_encryption_level` is `level_8` |
| `set_encryption_enabled(enabled: bool) → bool` | Enable or disable encryption. Writes the new level to CONFIG_DB and re-encrypts or decrypts all `BGP_NEIGHBOR` passwords accordingly |
| `decrypt_string(value: str) → str \| None` | Decrypt a single base64-encoded ciphertext using the current master key; returns `None` on failure |

**CLI commands (`bgp-master-key`):**

| Command | Description |
|---------|-------------|
| `bgp-master-key set <key>` | Store a new master key (prepends to history, no activation) |
| `bgp-master-key activate` | Enable encryption; requires a master key to already be set |
| `bgp-master-key deactivate` | Disable encryption; decrypts all BGP passwords in CONFIG_DB |
| `bgp-master-key encrypt --str <plaintext>` | Encrypt a single password string (debugging) |
| `bgp-master-key decrypt --str <ciphertext>` | Decrypt a single password string (debugging/recovery) |
| `bgp-master-key encrypt --file <json> [-o <out>]` | Encrypt all passwords in a BGP config JSON file |
| `bgp-master-key decrypt --file <json> [-o <out>]` | Decrypt all passwords in a BGP config JSON file |
| `bgp-master-key status` | Show current encryption status |
| `bgp-master-key list` | Show all stored master key entries with timestamps |

Common options: `--key-file` (override default path), `--config-input` (JSON file instead of live Redis), `--socket-path` (non-default Redis socket), `--force` (clear stale lock file).

#### 4. ConfigDBConnector Hooks

The encryption hooks are injected into both ConfigDB implementations that SONiC uses:

**`sonic-py-swsssdk` (`src/swsssdk/configdb.py`)**

Used by Python tools that import directly from the `swsssdk` package. The hook is loaded lazily — only when `set_entry`, `mod_entry`, or `mod_config` are first called:

```python
@staticmethod
def load_encryptor():
    if ConfigDBConnector.db_encryptor is None:
        try:
            mod = importlib.import_module("utilities_common.config_db_encryption")
            ConfigDBConnector.db_encryptor = mod.ConfigDBEncryptor()
        except Exception:
            ConfigDBConnector.db_encryptor = ""  # sentinel: load attempted, not available
    return ConfigDBConnector.db_encryptor

def set_entry(self, table, key, data):
    encryptor = ConfigDBConnector.load_encryptor()
    if encryptor and encryptor.entry_need_encryption(self, table, key, data):
        encryptor.set_entry(self, table, key, data)
    else:
        self._set_entry(table, key, data)

def mod_entry(self, table, key, data):
    encryptor = ConfigDBConnector.load_encryptor()
    if encryptor and encryptor.entry_need_encryption(self, table, key, data):
        encryptor.mod_entry(self, table, key, data)
    else:
        self._mod_entry(table, key, data)

def mod_config(self, data):
    encryptor = ConfigDBConnector.load_encryptor()
    if encryptor:
        encryptor.mod_config(self, data)
    for table_name in data:
        ...
        self._mod_entry(table_name, key, table_data[key])
```

**`sonic-swss-common` (`common/configdb.h` SWIG pythoncode block)**

Used by all tools that link against `libswsscommon` (e.g., the main `config` CLI). The pattern is identical. A `DbEncryptorNotLoaded` placeholder class implements `is_loaded() → False` so callers can distinguish between "encryptor loaded" and "encryptor unavailable":

```python
@staticmethod
def load_encryptor():
    if ConfigDBConnector.db_encryptor is None:
        try:
            import importlib
            mod = importlib.import_module("utilities_common.config_db_encryption")
            ConfigDBConnector.db_encryptor = mod.ConfigDBEncryptor()
        except Exception:
            ConfigDBConnector.db_encryptor = DbEncryptorNotLoaded()
    return ConfigDBConnector.db_encryptor

def set_entry(self, table, key, data):
    encryptor = ConfigDBConnector.load_encryptor()
    if encryptor.is_loaded() and encryptor.entry_need_encryption(self, table, key, data):
        encryptor.set_entry(self, table, key, data)
    else:
        self._set_entry(table, key, data)

def mod_config(self, data):
    encryptor = ConfigDBConnector.load_encryptor()
    if encryptor:
        encryptor.mod_config(self, data)
    ...
```

If `utilities_common.config_db_encryption` is not installed (e.g., on a minimal image), the hook silently falls back to unencrypted behavior, preserving backward compatibility.

### 7.3 Repositories Modified

| Repository | Changes |
|------------|---------|
| `sonic-utilities` | New: `utilities_common/master_key_encryption.py`, `utilities_common/config_db_encryption.py`, `bgp_master_key/` package; new dep: `dataclasses-json`, `cryptography` |
| `sonic-yang-models` | YANG model changes to `sonic-types`, `sonic-bgp-common`, `sonic-bgp-device-global` |
| `sonic-py-swsssdk` | Hook injection in `src/swsssdk/configdb.py` |
| `sonic-swss-common` | Hook injection in `common/configdb.h` (SWIG pythoncode) |

### 7.4 Interaction with FRR Container

The FRR configuration generators (`bgpcfgd` and `frrcfgd`) read `BGP_NEIGHBOR.auth_password` from CONFIG_DB. When `auth_encryption_level = level_8`, they must decrypt the password before passing it to FRR.

The `BgpPasswordCryptor` class provides a `decrypt_string()` method for this purpose. The master key file `/etc/bgp_master_key` is bind-mounted into the FRR container so that the decryption key is available there.

> **Design rationale**: FRR (`bgpd`) itself does not perform any encryption or decryption. It receives cleartext passwords for MD5 session authentication. Keeping FRR unchanged avoids dependency on the FRR project and simplifies the design. The net result is that passwords are protected at rest in CONFIG_DB and in saved JSON files; they are only decrypted in-memory at the point where the FRR config template is rendered.

### 7.5 Typical Operational Workflow

**Initial setup (encryption not yet active):**

1. The switch loads `config_db.json` with plaintext passwords. `BGP_DEVICE_GLOBAL|STATE.peer_password_encryption_level` is absent or `level_0`.
2. Passwords in CONFIG_DB are stored as plaintext. FRR is configured normally.

**Adding a new BGP peer (encryption active):**

```bash
config bgp neighbor add 10.0.0.5 ...
config bgp neighbor auth-password 10.0.0.5 mypassword
```

The `config` CLI calls `ConfigDBConnector.set_entry()`. The hook detects that `peer_password_encryption_level = level_8` and that the entry contains `auth_password`. It encrypts `mypassword` before writing. The plaintext never touches Redis.

**Applying a config patch (encryption active):**

```bash
config apply ./patch.json
```

`sonic-cfggen` calls `ConfigDBConnector.mod_config()`. The hook is invoked and encrypts all `auth_password` fields in the patch before they are committed to CONFIG_DB.

**Master Key Management:**

```bash
# Enable encryption: store a master key then activate
bgp-master-key set <32-char-key>
bgp-master-key activate

# After activation, BGP_DEVICE_GLOBAL|STATE.peer_password_encryption_level becomes level_8
# and every BGP_NEIGHBOR/BGP_PEER_GROUP auth_password is replaced with AES-GCM ciphertext.

# Rotate the master key (old key is retained in history)
bgp-master-key set <new-32-char-key>
bgp-master-key activate

# Disable encryption — decrypts all passwords back to plaintext in CONFIG_DB
bgp-master-key deactivate

# Decrypt a single password for debugging or recovery
bgp-master-key decrypt --str <base64-ciphertext>

# Decrypt an entire config file (e.g. for backup/restore)
bgp-master-key decrypt --file config_db.json -o config_db_plaintext.json
```

---

## 8. SAI API

No SAI API changes are required. This feature is entirely confined to the management plane.

---

## 9. Configuration and Management

### 9.1 CLI

The `bgp-master-key` command (Python module `bgp_master_key`) is the primary management interface. It requires root privileges because it accesses `/etc/bgp_master_key`.

```
bgp-master-key [--key-file FILE] [--config-input FILE] [--socket-path PATH]
               [--verbose] [--no-format] [--force] <command>

Commands:
  set <key>              Set/rotate the master key
  activate               Activate encryption (encrypt all existing passwords)
  deactivate             Deactivate encryption (decrypt all existing passwords)
  encrypt --str <pw>     Encrypt a single password string
  encrypt --file <f>     Encrypt passwords in a BGP config JSON file
  decrypt --str <ct>     Decrypt a single ciphertext string
  decrypt --file <f>     Decrypt passwords in a BGP config JSON file
  status                 Show encryption status
  list                   List all stored master key entries
```

### 9.2 YANG Models

See [YANG Model Changes](#yang-model-changes) above.

### 9.3 Config DB Changes

See [Database Changes](#database-changes) above.

### 9.4 Master Key File

- Path: `/etc/bgp_master_key`
- Permissions: `0600` (root read/write only)
- Format: JSON
- Bind-mounted into the FRR container

Example file after two key rotations:

```json
{
  "master_keys": {
    "bgp_peer": [
      {
        "name": "bgp_peer",
        "master_key": "newkey_abcdefghabcdefghabcdefgh",
        "algorithm": "aes-gcm",
        "timestamp": "2026-04-01T10:00:00+00:00"
      },
      {
        "name": "bgp_peer",
        "master_key": "oldkey_12345678123456781234567",
        "algorithm": "aes-gcm",
        "timestamp": "2026-01-15T08:30:00+00:00"
      }
    ]
  },
  "encryption_enabled": true
}
```

The `master_keys` dictionary is keyed by feature name (e.g., `"bgp_peer"`). Each feature maintains its own ordered list of historical keys, newest first, up to a maximum of 8. This design extends naturally to other SONiC components that need to encrypt data fields — each component uses a separate master key file (e.g., `/etc/tacacs_master_key`, `/etc/radius_master_key`), so that a corrupted or lost key file for one component does not affect the operation or recoverability of any other component.

---

## 10. Warmboot and Fastboot Design Impact

**Warmboot**: No impact. The encryption state and master key persist across reboots because they are stored in the master key file on the filesystem and in CONFIG_DB. After warmboot, `bgpcfgd`/`frrcfgd` will decrypt passwords from CONFIG_DB before programming FRR, exactly as they do in normal operation. No BGP peer flap is expected because the decrypted passwords are unchanged.

**Fastboot**: No impact. The same reasoning applies. The encryption hooks are passive — they only act on write operations. Fastboot's read-path from CONFIG_DB is unaffected.

**Config reload**: `config reload` invokes `sonic-cfggen -j config_db.json -w`, which calls `ConfigDBConnector.mod_config()`. The hook is present and will encrypt passwords in the incoming JSON if encryption is active, maintaining the encrypted state in CONFIG_DB.

---

## 11. Memory Consumption

The encryption library (`master_key_encryption.py`) and the `ConfigDBEncryptor` are loaded once per process into the Python interpreter's memory space. The `MasterKeyManager` singleton maintains minimal in-memory state (the filename and a dictionary of pending updates). The master key file itself is small (< 4 KB even with 8 historical keys). The overall memory overhead is negligible (estimated < 1 MB across all management plane processes).

When encryption is disabled (`level_0`), the lazy loader detects that `entry_need_encryption` returns `False` on every call, and the encryption code paths are not exercised. Memory impact is the same whether enabled or disabled, since the module is loaded on first write regardless.

---

## 12. Restrictions / Limitations

1. **Native C/C++ tools bypass encryption.** Tools that write directly to Redis without using the Python `ConfigDBConnector` — such as `redis-cli` and `sonic-db-cli` — will not trigger the encryption hook. If an operator uses these tools to write a BGP neighbor entry with a plaintext password, it will be stored as plaintext even when encryption is active.

   *Mitigation*: In normal operations, network administrators use `config` commands or `sonic-cfggen` to configure the switch. Direct Redis access is not a standard workflow. The `bgp-master-key encrypt --file` command provides a path to encrypt any manually created config files before loading them.

2. **FRR vtysh hides passwords in show commands with `*`.** Since FRR receives cleartext passwords for MD5 session setup, `vtysh` would ordinarily display them in `show running-config`. A separate FRR patch mitigates this by default-masking BGP neighbor passwords in vtysh output (`neighbor <addr> password ******`).

3. **`BGP_NEIGHBOR` and `BGP_PEER_GROUP` tables are supported.** Both tables share the `sonic-bgp-cmn` YANG grouping and have `auth_encryption_level` in their schema. The encryptor handles `auth_password` entries in both tables.

4. **Master key stored in plaintext as file `/etc/bgp_master_key`.** The master key file stores the master key as plaintext JSON. It relies on filesystem permissions (`0600`, root-only) for protection. Hardware Security Module (HSM) or TPM-based key protection is not implemented.

5. **Key recovery via central vault.** If the master key file is lost (e.g., disk failure), the central key vault and distribution system is responsible for re-seeding the master key to the switch. The operator then re-activates encryption to re-encrypt the passwords. Because the master key originates from and is retained by the central vault, it is fully recoverable — unlike an approach based on locally generated random master keys, where loss of local storage would permanently destroy the key and render all encrypted passwords inaccessible.

---

## 13. Testing Requirements / Design

### 13.1 Unit Tests

#### `tests/master_key_test.py` (sonic-utilities)

Tests the low-level encryption library and `MasterKeyManager`:

- **`test_aes_gcm_encryption`**: Verifies AES-GCM output against ground-truth test vectors from external sources (independent of the implementation).
- **`test_ase_gcm_decyption`**: Round-trip decrypt verification against the same test vectors.
- **`test_master_key_manager`**: Tests `update_master_key`, file format, permission enforcement, and key history rotation.
- **Key rotation**: Verifies that setting a new key prepends to history and prunes at 8 entries.
- **File locking**: Verifies that concurrent access is serialized.

#### `tests/bgp_master_key_test.py` (sonic-utilities)

Tests the `bgp-master-key` CLI end-to-end using subprocess invocations with `--config-input` (file mode, no live Redis):

- **`test_config_master_key`**: `set` command creates and updates master key entries.
- **`test_new_master_key_file`**: Newly created key file has correct `0600` permissions.
- **`test_encrypt_decrypt_string`**: `encrypt --str` / `decrypt --str` round-trip; verifies ciphertext length > plaintext + overhead.
- **`test_encrypt_decrypt_config_file`**: `encrypt --file` / `decrypt --file` round-trip on a BGP config JSON containing neighbors with and without passwords.
- **`test_activate_deactivate`**: `activate` encrypts all BGP_NEIGHBOR passwords; `deactivate` decrypts them back.

#### `tests/test_db_encryptor.py` (sonic-utilities)

Integration tests for `ConfigDBEncryptor` hooks, using a mock `ConfigDBConnector`:

- **`test_mod_config_hook`**: Verifies that `mod_config()` encrypts `auth_password` in-place when encryption is enabled.
- **`test_set_entry_hook`**: Verifies that `set_entry()` encrypts the password and adds `auth_encryption_level`.
- **`test_set_entry_already_encrypted`**: Verifies idempotency — an already-encrypted password is not double-encrypted.
- **Disabled encryption**: Verifies that when `peer_password_encryption_level = level_0`, entries pass through unmodified.

#### `tests/test_db_encryptor.py` (sonic-swss-common)

Tests the hook mechanism in the SWIG `ConfigDBConnector` using a mock `ConfigDBEncryptor`:

- Verifies that `set_entry`, `mod_entry`, and `mod_config` on `ConfigDBConnector` invoke the mock encryptor's methods.
- Verifies that `_set_entry` and `_mod_entry` bypass the hook (private methods for direct writes).

### 13.2 System Test (Manual / E2E)

1. Deploy SONiC image on a test switch or VS (virtual switch).
2. Load a config with BGP neighbors that have `auth_password` set.
3. Run `bgp-master-key set <key>` followed by `bgp-master-key activate`.
4. Verify via `redis-cli hget BGP_NEIGHBOR:<neighbor> auth_password` that the value is base64-encoded ciphertext.
5. Verify BGP sessions remain established (no flap) after activation.
6. Run `bgp-master-key decrypt --str <ciphertext>` to recover the original password.
7. Run `config reload` and verify passwords remain encrypted after reload.
8. Run `bgp-master-key deactivate` and verify passwords revert to plaintext in CONFIG_DB.
9. Verify BGP sessions remain established after deactivation.
10. Test `config bgp neighbor auth-password` when encryption is active — verify the newly added neighbor's password is stored encrypted.

### 13.3 sonic-mgmt Test

System integration testing in the `sonic-mgmt` repository is under development.

---

## 14. Open / Action Items

| # | Item | Status |
|---|------|--------|
| 1 | Extend master key framework to TACACS, RADIUS, LDAP password encryption using the same `MasterKeyManager` library with per-component key files | Future |
| 2 | Upstream contribution to SONiC community — coordinate with sonic-net maintainers | In Progress |
| 3 | `bgpcfgd` / `frrcfgd` integration — ensure password decryption before writing to FRR templates | In Progress |
