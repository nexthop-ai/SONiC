# SFPUtil Support for Composite SFPs #

## Table of Content 

- [1. Revision](#1-revision)
- [2. Scope](#2-scope)
- [3. Definitions/Abbreviations](#3-definitionsabbreviations)
- [4. Overview](#4-overview)
- [5. Requirements](#5-requirements)
- [6. Architecture Design](#6-architecture-design)
- [7. High-Level Design](#7-high-level-design)
  - [7.1 Overall approach](#71-overall-approach)
  - [7.2 Device enumeration and naming](#72-device-enumeration-and-naming)
  - [7.3 Device selector semantics (`-d/--device`)](#73-device-selector-semantics-ddevice)
  - [7.4 Command behavior on CPO-backed ports](#74-command-behavior-on-cpo-backed-ports)
  - [7.5 Example base and composite implementations](#75-example-base-and-composite-implementations)
  - [7.6 Rationale for CompositeSfpBase and CpoSfpOptoeBase](#76-rationale-for-compositesfpbase-and-cposfpoptoebase)
- [8. SAI API](#8-sai-api)
- [9. Configuration and management](#9-configuration-and-management)
  - [9.1 Manifest (if the feature is an Application Extension)](#91-manifest-if-the-feature-is-an-application-extension)
  - [9.2 CLI/YANG model Enhancements](#92-cliyang-model-enhancements)
  - [9.3 Config DB Enhancements](#93-config-db-enhancements)
- [10. Warmboot and Fastboot Design Impact](#10-warmboot-and-fastboot-design-impact)
- [11. Memory Consumption](#11-memory-consumption)
- [12. Restrictions/Limitations](#12-restrictionslimitations)
- [13. Testing Requirements/Design](#13-testing-requirementsdesign)
  - [13.1 Unit Test cases](#131-unit-test-cases)
  - [13.2 System Test cases](#132-system-test-cases)
- [14. Open/Action items - if any](#14-openaction-items---if-any)

### 1. Revision  

### 2. Scope  

Historically, `sfputil` has assumed a 1:1 relationship between a logical port and a single SFP-like device that exposes a flat EEPROM, power control and low-power mode. With co-packaged optics (CPO), ports can instead be backed by composite SFPs that internally aggregate multiple I2C-configurable devices (typically an optical engine and an external laser source), each of which has its own EEPROM contents and control bits.

The CPO port-mapping HLD (/doc/platform/port_mapping_for_cpo.md) introduces the composite SFP abstraction and the `optical_devices.json` data model that describe how these internal devices are wired to interfaces. This document focuses on the CLI and control-plane aspects: extending `sfputil` so that operators can reason about and manipulate composite SFPs in a predictable way, using familiar commands, regardless of whether a port is backed by a single device or a composite CPO stack.

As a result, `sfputil` needs to support the following:

- Provide a consistent user experience for core xcvr operations (`read-eeprom`, `write-eeprom`, `power`, `lpmode`, `reset`, `show`, and some debug commands) on both single-device and composite SFPs.
- Allow users to explicitly select an underlying device (for example, optical engine vs. external laser source) when operating on a CPO-backed DUT, while still behaving naturally for traditional non-composite ports.
- Handle platforms where CPO hardware is managed either via separate devices or via a joint-mode that presents a unified EEPROM view, including cases where an explicit device selector is required to disambiguate accesses.
- Surface clear, actionable error messages when commands are ambiguous (for example, attempting to read EEPROM on a composite SFP without specifying a device) or when a requested operation is not applicable to a given internal device.
- Integrate with existing platform APIs for composite SFPs to locate the appropriate internal device(s) for a logical port and route operations accordingly, without requiring broader architectural changes. Part of this is already proposed in the CPO port-mapping HLD.

This HLD does not redefine the composite SFP or `optical_devices.json` models themselves; those are covered by the CPO port-mapping HLD. The scope here is limited to changes in `sfputil` and closely-related CLI behavior necessary to make CPO-backed ports manageable in day-to-day operations.

### 3. Definitions/Abbreviations 

| Term           | Definition |
|----------------|------------|
| CPO            | Co-packaged Optics; hardware where a single logical interface is driven by multiple I2C-configurable devices (for example, an optical engine and an external laser source). |
| Composite SFP  | A logical SFP/transceiver object that internally aggregates multiple SFP-like devices (for example, OE and ELSFP) but is presented to higher layers as a single port. |
| SFP            | Small Form-factor Pluggable transceiver; in this document used generically to refer to pluggable optical modules managed by `sfputil`. |
| OE             | Optical Engine; a CPO device. |
| ELSFP          | External Laser Small Form-factor Pluggable; a CPO device that provides laser sources powering one or more optical engines. |
| EEPROM         | Electrically Erasable Programmable Read-Only Memory used by transceivers to store identification, configuration, and diagnostic information. |
| I2C            | Inter-Integrated Circuit; the serial bus used to access transceiver EEPROMs and control registers. |
| MCU            | Microcontroller Unit; in this context, a device that may proxy or aggregate EEPROM and control-plane access for multiple underlying CPO devices (joint mode). |
| DUT            | Device Under Test; a SONiC switch or system on which `sfputil` is executed. |
| `sfputil`      | SONiC command-line utility for managing SFP and transceiver devices (EEPROM access, power, low-power mode, reset, and related debug/show commands). |

### 4. Overview 

This HLD describes how `sfputil` will be extended to understand composite SFPs introduced by co-packaged optics (CPO) hardware, and how those extensions fit into the existing SONiC platform API and CPO port-mapping model.

From an user's perspective, the goal is that common transceiver workflows continue to "just work" when a port is backed by multiple internal devices (for example, an optical engine and an external laser source), while still preserving full control over each device when needed. Note that this does not mean that the same commands used for non-composite SFPs will always work for composite SFPs.

- `sfputil` must be able to discover and display the set of internal devices associated with a given logical port, including their types, banks and identifying information.
- Core operations such as reading/writing EEPROM, toggling power, entering/exiting low-power mode and resetting a transceiver must work uniformly on:
  - traditional single-device ports, and
  - composite CPO ports where multiple devices share responsibility for a single interface.
- For composite ports, `sfputil` must provide a way to explicitly target a particular internal device (for example, OE vs. ELS) when issuing these operations, while remaining backward compatible for non-composite ports.
- Platforms that implement CPO in different ways (separate devices vs. joint-mode MCU) must be able to plug into the same `sfputil` behavior using the composite SFP platform APIs and `optical_devices.json` topology information defined in the CPO port-mapping HLD.

At a high level, the design in this document introduces:

- A mechanism to enumerate and name the internal devices backing each logical port, exposed via `sfputil`.
- A small set of CLI extensions (for example, an optional device selector argument) that are applied consistently across the relevant `sfputil` subcommands.
- Clear error-reporting semantics for ambiguous or unsupported operations on composite ports.

These changes are intentionally scoped to `sfputil` and its interaction with existing platform APIs. No changes to SAI, the underlying composite SFP abstraction, or the `optical_devices.json` schema are proposed here; those are treated as prerequisites and building blocks for the behavior defined in this HLD.

### 5. Requirements

The requirements in this section describe what behavior `sfputil` must provide once composite SFPs and CPO hardware are present, and how that behavior must interact with existing, non-composite platforms.

#### 5.1 Functional requirements

- **CPO support for all existing `sfputil` commands**  
  `sfputil` **shall** support operating on CPO-backed (composite) ports for all currently supported top-level commands, unless explicitly called out as unsupported in this HLD or the CPO notes document. This includes:
  - `sfputil debug ...`
  - `sfputil firmware ...`
  - `sfputil lpmode on` / `sfputil lpmode off`
  - `sfputil power enable` / `sfputil power disable`
  - `sfputil read-eeprom`
  - `sfputil reset`
  - `sfputil show ...`
  - `sfputil version`
  - `sfputil write-eeprom`

- **New command to enumerate devices per port**  
  A new command **shall** be added to list all detected internal devices associated with a logical port, including at minimum the device name, type (for example, optical engine vs. external laser source). This command is the primary discovery mechanism for understanding how a CPO-backed port is composed.

- **Device selection for composite ports**  
  For composite SFPs, commands that act on an EEPROM or device-specific control bits **shall** accept an optional device selector argument (for example, `-d/--device <device-name>`), allowing operators to target a specific internal device. For non-composite ports this argument must remain optional and, if omitted, the command must behave as it does today.

- **Behavior when the device selector is omitted**  
  For composite ports, the behavior when the device selector is omitted **shall** be well-defined on a per-command basis:
  - For operations where a single unambiguous target does not exist (for example, `read-eeprom` on a composite SFP without an MCU that aliases the memory map), the command **shall** fail with a clear error instructing the user to specify a device.
  - For operations that are naturally broadcast or symmetric across internal devices (for example, powering both devices on/off when supported by the platform), the design **may** define a default behavior that applies the operation to all applicable internal devices when the device selector is omitted.

- **Support for both joint and separate CPO modes**  
  The design **shall** support platforms where CPO hardware is:
  - managed via separate devices (independent EEPROMs and control bits for OE and ELS), and
  - managed via a joint-mode MCU that exposes a unified EEPROM view and internal memory mapping.  
  In both cases, the same `sfputil` CLI and device-selector semantics must apply; platform differences are handled behind the platform API and composite SFP abstraction.

- **Interaction with composite SFP platform APIs**  
  `sfputil` **shall** use the composite SFP interfaces defined in the CPO port-mapping HLD (for example, `get_internal_devices()` / `get_internal_device()`) to discover and operate on the internal devices backing a logical port. Direct EEPROM access to composite SFP wrapper objects (which do not implement `read_eeprom` / `write_eeprom`) must not be required.

- **Techsupport and diagnostics**  
  For composite ports, the existing techsupport flows that rely on `sfputil` (for example, EEPROM hexdumps) **shall** be extended so that they can collect per-device data (OE and ELS) where supported, without regressing existing behavior for non-composite ports.

#### 5.2 Backward compatibility and error handling

- Existing `sfputil` commands and options **shall** continue to work unchanged on platforms that do not use composite SFPs or CPO hardware.
- When a user attempts to use CPO- or device-specific options on a non-composite port, `sfputil` **shall** return a error indicating that the option is not applicable to that port.
- When a command on a composite port cannot determine a unique or valid target device (for example, missing or invalid `--device`), `sfputil` **shall**:
  - fail the command, and
  - emit an error message that explains what additional information is required (for example, "EthernetX is a composite SFP; please specify a device using `-d`" or similar wording to be detailed later in this document).

#### 5.3 Non-functional requirements

- The changes in this HLD **shall not** require any changes to SAI APIs, the composite SFP abstraction, or the `optical_devices.json` schema; they must build solely on the abstractions provided by the CPO port-mapping HLD and the platform API.
- The additions **shall not** introduce significant performance regressions for `sfputil` operations on either composite or non-composite platforms (for example, no additional blocking I/O or expensive lookups in the hot path beyond what is necessary to resolve and enumerate internal devices).
- Warmboot and fastboot behavior **shall not** be impacted by these changes; any initialization needed for composite SFP awareness must be compatible with existing boot constraints.

### 6. Architecture Design 

This HLD proposes no architectural change to SONiC.

### 7. High-Level Design 

This section describes how `sfputil` is extended to support composite SFPs and CPO hardware. All changes are confined to the `sonic-utilities` repository and reuse the composite SFP and platform APIs defined in the CPO port-mapping HLD.

#### 7.1 Overall approach

- The platform layer continues to return a per-port `SfpBase` object (for example, via `platform.get_sfp(port_index)`). Both traditional and composite SFPs are implemented as concrete subclasses of `SfpBase`; **any composite class** (such as `CpoSfpOptoeBase`) **MUST also implement `CompositeSfpBase`** so that its internal devices can be introspected where needed, but from `sfputil`'s perspective everything is accessed through the `SfpBase` interface.
- `sfputil` invokes high-level methods on `SfpBase` (for example, `read_eeprom`, `write_eeprom`, power/low-power/reset control, firmware operations), always forwarding any optional device selector from the CLI (for example, `-d/--device`) as a method parameter when present.
- The concrete `SfpBase` implementation is responsible for abstracting away whether the underlying hardware is composite or not:
  - Non-composite implementations may ignore the device selector argument.
  - Composite implementations internally route the call to one or more underlying devices as appropriate.
  - For operations that logically require a specific target device (for example, `read_eeprom` on a CPO-backed port without a MCU map), the implementation must validate that a target device was supplied; if not, it raises a well-defined exception.
- `sfputil` command handlers follow almost the same flow for all commands:
  1. Resolve the logical port (`-p/--port`) to a platform `SfpBase` object.
  2. Parse any optional device selector (`-d/--device`) from the CLI and pass it to the relevant `SfpBase` method.
  3. Catch any exceptions raised by the `SfpBase` implementation (for example, "device required but not specified", "invalid device name for this port") and translate them into clear, user-facing error messages that reference `sfputil show devices` where appropriate.

This keeps composite-specific logic encapsulated inside the composite-SFP implementation, while `sfputil` justremains a CLI wrapper that enforces consistent argument parsing and error reporting across platforms.

#### 7.2 Device enumeration and naming

- A new `sfputil show devices` command is introduced as the primary way to discover the internal devices associated with a logical port.
  - For composite SFPs, it lists one row per internal device, including at least: device name, device type (for example, "Optical Engine", "External Laser Source"), and any platform-provided bank or identifier information.
  - For non-composite SFPs, it may return a single row representing the underlying device, or a short message indicating that the port is not composite.
- Device names exposed by `sfputil` are taken from the platform implementation (for example, `OE1`, `ELS1`) so that they align with the identifiers used in `optical_devices.json` and the composite SFP implementation.
- All other `sfputil` commands that accept a device selector use these same names.

#### 7.3 Device selector semantics (`-d/--device`)

- Commands that act on EEPROM contents or device-specific control bits (`read-eeprom`, `write-eeprom`, `power`, `lpmode`, `reset`, `firmware`, selected `debug` subcommands) accept a new optional argument `-d/--device <device-name>`. `sfputil` does not itself interpret the device name beyond basic parsing; it passes the value through to the underlying `SfpBase` implementation.
- Non-composite `SfpBase` implementations may treat any non-empty device selector as invalid (for example, by raising an exception that `sfputil` surfaces as a CLI error instructing the user to omit `-d`).
- Composite `SfpBase` implementations use the selector to choose which internal device(s) to operate on. For methods that logically require a specific device (for example, `read_eeprom` on hardware without a unified EEPROM map), the implementation must:
  - raise an exception if the selector is missing or does not resolve to a valid internal device for that port; and
  - include in the exception message enough context for `sfputil` to emit a helpful error (for example, "EthernetX is a composite SFP; device must be specified").
- For control-plane operations that are naturally symmetric across devices (`power`, `lpmode`, `reset`), composite implementations may define a reasonable default when the selector is omitted (for example, apply the operation to all internal devices) while still allowing the caller to target a single device when `-d` is present.
- `sfputil` is responsible for catching these exceptions and converting them into consistent, user-facing error messages, optionally suggesting that the user run `sfputil show devices -p <port>` to discover valid device names.

#### 7.4 Command behavior on CPO-backed ports

At a high level, the behavior of each existing `sfputil` command on CPO-backed ports is as follows (detailed CLI examples and options will be covered in Section 9):

- **read-eeprom / write-eeprom**
  - Reuse the current CLI (`-p/--port`, `-n/--page`, `-o/--offset`, `-s/--size`, formatting options) and add `-d/--device`.
  - On composite SFPs, `sfputil` selects the target internal device based on `-d` or the platform's default-device policy (as described in 7.3) and issues the EEPROM access through that device.
  - `sfputil` does not attempt to interpret the device's CMIS or CPO memory layout; it relies entirely on the platform and underlying Xcvr API to implement the correct mapping of page/offset to physical EEPROM locations, including bank selection.

- **power enable/disable**
  - For non-composite ports, behavior is unchanged.
  - For composite CPO ports:
    - With `-d/--device`, only the selected device's power state is changed (for example, OE-only or ELS-only).
    - Without `-d`, `sfputil` attempts to enable/disable power on all internal devices associated with the port. If the platform only exposes power control for a subset of devices, `sfputil` logs or reports which devices were successfully updated.

- **lpmode on/off**
  - Mirrors the semantics of the power command:
    - `-d/--device` targets a specific internal device.
    - Omitting `-d` attempts to apply the low-power-mode change to all internal devices associated with the port, subject to platform support.

- **reset**
  - For composite ports, reset operations may affect either a single device or all devices:
    - With `-d/--device`, only the specified internal device is reset.
    - Without `-d`, `sfputil` issues reset requests to all internal devices that support reset, so that the logical port is fully reset.

- **firmware**
  - The firmware subcommands (for example, download/upgrade) are extended to accept `-p/--port` and optional `-d/--device` so that firmware can be upgraded independently on OE and ELS devices where supported.
  - Where joint-mode or platform policies require coordinated upgrades (for example, OE and ELS firmware must be updated together), platform logic behind the composite SFP is responsible for enforcing the correct sequence; from the CLI perspective, the same `sfputil firmware` interface is used.

- **debug and show**
  - Existing `sfputil show` output is extended to display whether a port is composite and, where appropriate, high-level status aggregated across all internal devices.
  - Debug subcommands that expose raw EEPROM contents or other device-specific diagnostics are treated like EEPROM operations: they use `-d/--device` to select an internal device and follow the same rules when the selector is omitted.
  - Commands used by `show techsupport` to collect EEPROM hexdumps are updated to iterate over internal devices on composite ports so that both OE and ELS data can be captured when desired.

- **version**
  - The `sfputil version` command remains unchanged; it is not device- or port-specific and therefore has no composite-specific behavior.

These behaviors ensure that all existing `sfputil` commands can operate on CPO-backed ports while providing explicit, predictable control over each internal device when required.

#### 7.5 Example base and composite implementations

The following examples illustrate how the `SfpBase`, `SfpOptoeBase` (non-composite), and composite SFP classes can be structured to support an optional device selector while keeping composite-specific logic inside the platform implementation.

```python
class SfpBase(abc.ABC):
    """Base interface for all SFP-like devices (composite or non-composite)."""

    @abc.abstractmethod
    def read_eeprom(
        self,
        page: int,
        offset: int,
        size: int,
        device: typing.Optional[str] = None,
    ) -> bytes:
        """Read bytes from the transceiver EEPROM.

        For composite devices, a device selector may be required to
        disambiguate which internal device to read from.
        """
        ...
```

```python
class SfpOptoeBase(SfpBase):
    """Existing base class for optoe-backed, non-composite SFPs."""

    def read_eeprom(
        self,
        page: int,
        offset: int,
        size: int,
        device: std | None = None,
    ) -> bytes:
        if device is not None:
            raise ValueError("Device selector is not valid for non-composite SFPs")
        # Existing single-device EEPROM access logic 
        return self._read_eeprom_raw(page, offset, size)


class VendorSfp(SfpOptoeBase):
    """Concrete non-composite SFP for a specific platform/vendor."""

    def _read_eeprom_raw(self, page: int, offset: int, size: int) -> bytes:
        ...  # Platform-specific implementation
```

```python
class CpoSfpOptoeBase(SfpOptoeBase, CompositeSfpBase):
    """Composite SFP representing a CPO port with OE and ELS devices."""

    def __init__(self, oe: SfpOptoeBase, els: SfpOptoeBase) -> None:
        super().__init__()
        self._oe = oe
        self._els = els

    def get_internal_devices(self) -> typing.List[SfpOptoeBase]:
        return [self._oe, self._els]

    def get_number_of_internal_devices(self) -> int:
        return 2

    def get_internal_device(self, name: str) -> SfpOptoeBase:
        if "OE" in name:
            return self._oe
        if "ELS" in name:
            return self._els
        raise ValueError(f"No internal SFP found for {name}")

    def read_eeprom(
        self,
        page: int,
        offset: int,
        size: int,
        device: typing.Optional[str] = None,
    ) -> bytes:
        """Read bytes from the EEPROM of a specific internal device.

        This operation requires a device selector because OE and ELS
        expose independent EEPROMs.
        """
        if device is None:
            raise ValueError(
                "Composite CPO SFP requires a device selector for read_eeprom; "
                "run 'sfputil show devices -p <port>' to see valid devices."
            )
        target = self.get_internal_device(device)
        return target.read_eeprom(page, offset, size)
```

In this model, `sfputil` does not need to know whether a port is composite or how many internal devices exist. It simply passes the optional device selector (for example, `-d/--device`) through to the `SfpBase` methods and reports any exceptions (for example, missing device selector for a composite-only operation) as user-facing CLI errors.

#### 7.6 Rationale for CompositeSfpBase and CpoSfpOptoeBase

The design introduces an explicit `CompositeSfpBase` interface and a CPO-specific base class `CpoSfpOptoeBase` in addition to the existing `SfpOptoeBase` for several reasons:

- **Separation of concerns between per-device logic and composition**  
  `SfpOptoeBase` is fundamentally a helper for talking to a *single* optoe-backed SFP (via `XcvrApi`). It does not, and should not, encode semantics about multiple devices being grouped into a single logical port. By adding `CompositeSfpBase(SfpBase)` we give composite implementations a clear, contract for:
  - Enumerating their internal devices (`get_internal_devices()` / `get_internal_device(name)`), and
  - Exposing a composite view of those devices where needed.

- **A generic way for tooling to discover composite SFPs**  
  With `CompositeSfpBase`, any consumer (including `sfputil`) can determine whether a port is composite and, if so, query its internals. This is essential for features such as:
  - `sfputil show devices -p <port>` (device enumeration), and
  - Commands that need to iterate over OE/ELS devices.  
  Without a dedicated composite base, callers would have to hard-code knowledge of vendor-specific subclasses or rely on ad-hoc methods on `SfpOptoeBase`, making the abstraction not portable.

- **Preserving the existing SfpOptoeBase contract for non-composite SFPs**  
  If composite behavior were added directly to `SfpOptoeBase`, every non-composite implementation would inherit composite-related APIs that do not make sense for it (for example, `get_internal_devices()` returning an empty list, or raising by convention). Introducing `CompositeSfpBase` keeps `SfpOptoeBase` focused on per-device behavior and makes composite capabilities explicit only where they are needed.

- **CpoSfpOptoeBase as the glue for CPO logical ports**  
  `CpoSfpOptoeBase(SfpOptoeBase, CompositeSfpBase)` exists to bridge the gap between the per-device world and the composite world for CPO ports:
  - As an `SfpOptoeBase`, it continues to look like a normal optoe-backed SFP to existing SONiC components that are unaware of CPO.
  - As a `CompositeSfpBase`, it can hold internal `SfpOptoeBase` instances for the OE and ELS devices and provide the necessary introspection.  
  This allows `sfputil` and other tools to interact with CPO ports purely through the `SfpBase`/`CompositeSfpBase` abstractions while all hardware-specific details (how OE and ELS are wired, whether an MCU provides a joint view, etc.) remain hidden inside the composite implementation.

- **Centralizing CPO-specific semantics**  
  By putting CPO composition logic into `CpoSfpOptoeBase` (and its vendor subclasses), we avoid duplicating the “OE + ELS” wiring and device-selection rules across multiple call sites. In particular, rules like:
  - "`read_eeprom` on a CPO port requires a device selector", and
  - "`power`/`lpmode` may default to all devices when `device` is omitted"  
  are enforced once in the composite SFP implementation. `sfputil` simply forwards the optional `-d/--device` argument and surfaces any exceptions as user-facing CLI errors.

Taken together, `CompositeSfpBase` and `CpoSfpOptoeBase` let SONiC support CPO-backed, composite SFPs in a way that is both backward compatible (non-composite platforms continue to use `SfpOptoeBase` as before) and extensible future composite transceiver patterns can reuse the same abstraction without modifying `sfputil` or `SfpOptoeBase`.

### 8. SAI API 

There are no changes to SAI API in this HLD.

### 9. Configuration and management 

This section describes the user-visible configuration and management changes associated with extending `sfputil` for CPO and composite SFPs. As this HLD is limited to CLI behavior in `sonic-utilities`, the primary impact is on the `sfputil` command set and its documentation.

#### 9.1. Manifest (if the feature is an Application Extension)

This feature is not implemented as a SONiC Application Extension. No manifest changes are required.

#### 9.2. CLI/YANG model Enhancements 

This HLD introduces CLI extensions to `sfputil` but does not change any YANG models.

##### 9.2.1 New `sfputil show devices` command

A new command is added to allow users to list the internal devices associated with a logical port:

```text
$ sfputil show devices -p <port>
Device Name    Type                   Bank  Part Number  Vendor
-----------    ---------------------  ----  -----------  ----------------
OE1            Optical Engine         0     ...          ...
ELS1           External Laser Source  0     ...          ...
```

- For **composite** SFPs (for example, CPO ports with OE and ELS devices), this command lists one row per internal device, including at least the device name and type. Platforms may optionally populate additional columns (for example, bank, part number, vendor) if that information is readily available.
- For **non-composite** SFPs, the command may either return a single row describing the underlying device or print a short message indicating that the port is not composite.

The device names shown by this command (for example, `OE1`, `ELS1`) are the same identifiers accepted by the `-d/--device` option described below.

##### 9.2.2 Optional `-d/--device` selector for existing commands

All `sfputil` commands that operate on a specific transceiver device are extended with an optional device selector argument:

```text
-d DEVICE, --device DEVICE
    Name of the internal device backing the specified port (for example, OE1, ELS1).
```

This option is accepted by at least the following commands:

- `sfputil read-eeprom`
- `sfputil write-eeprom`
- `sfputil power enable` / `sfputil power disable`
- `sfputil lpmode on` / `sfputil lpmode off`
- `sfputil reset`
- `sfputil firmware ...` subcommands
- Selected `sfputil debug ...` subcommands that access EEPROM or device-local control bits

Example CLI invocations:

```text
# Read 2 bytes from page 0, offset 0 on the optical engine for Ethernet224
$ sfputil read-eeprom -p Ethernet224 -n 0 -o 0 -s 2 -d OE1

# Power-cycle only the external laser source on Ethernet0
$ sfputil power disable Ethernet0 -d ELS1
$ sfputil power enable  Ethernet0 -d ELS1

# Apply low-power mode to all devices associated with Ethernet8
$ sfputil lpmode on Ethernet8

# Upgrade firmware on the OE backing Ethernet32
$ sfputil firmware download -p Ethernet32 -d OE1 --image /tmp/oe-fw.bin
```

Behavior is as follows:

- On **non-composite** ports, specifying `-d/--device` is treated as invalid; the underlying `SfpBase` implementation raises an error which `sfputil` surfaces as a CLI error instructing the user to omit the device selector.
- On **composite** ports:
  - For operations that **require** a specific target device (for example, `read-eeprom` on hardware where OE and ELS EEPROMs are independent and not remapped by an MCU), omitting `-d/--device` causes the `SfpBase` implementation to raise an exception. `sfputil` converts this into an error message indicating that the port is composite and that a device must be specified, and suggests running `sfputil show devices -p <port>`.
  - For operations that are naturally symmetric across devices (for example, `power`, `lpmode`, `reset`), omitting `-d/--device` is interpreted by composite implementations as "apply to all internal devices". Supplying `-d/--device` limits the operation to the named device.

These CLI changes are designed to be backward compatible: existing commands and options continue to behave as before on non-composite platforms, and additional behavior is only enabled when composite SFPs are present and a device selector is used.

SONiC's `Command-Reference.md` in `sonic-utilities` will be updated to document the new `sfputil show devices` command and the `-d/--device` option for the affected commands.

##### 9.2.3 No YANG model changes

This HLD does not introduce any new management models or augment existing YANG definitions. All CPO-related YANG and Config DB schema changes (such as `OPTICAL_DEVICE` and `associated_devices`) are covered by the CPO port-mapping HLD and reused here.

#### 9.3 Config DB Enhancements  

This HLD introduces **no new Config DB schema** or keys.

- The mapping between interfaces and optical devices (including OE and ELS) is modeled via `optical_devices.json` and the corresponding CONFIG_DB enhancements described in the CPO port-mapping HLD (for example, `associated_devices` and `OPTICAL_DEVICE`).
- `sfputil` reads this information indirectly via the platform APIs and composite SFP abstractions; it does not write to or extend CONFIG_DB.

As a result, there are no backward-compatibility concerns for configuration files specific to this HLD.
		
### 10. Warmboot and Fastboot Design Impact  

There is no warmboot or fastboot design impact specific to this HLD.

- All changes are confined to the `sfputil` CLI and its use of existing platform APIs. No new daemons, services or long-running processes are introduced.
- `sfputil` continues to be invoked on demand by users or higher-level tools (for example, `show` or `show techsupport`) and is not part of the boot-critical control plane path.
- Any additional work performed by `sfputil` on composite SFPs (for example, resolving internal devices or iterating over them) occurs only when commands are explicitly executed and therefore does not affect warmboot or fastboot sequencing.

#### Warmboot and Fastboot Performance Impact

Given that `sfputil` is not in the warmboot/fastboot critical path and no new boot-time initialization is added as part of this HLD, there is no expected change in control-plane or data-plane downtime during warmboot or fastboot.

In particular:

- No new blocking I/O, network calls heavy processing is added to boot-time components.
- No additional dependencies are introduced for services involved in warmboot/fastboot.
- When the feature is unused (for example, on non-composite platforms), there is effectively zero impact.

### 11. Memory Consumption

There should be no noteworthy change in memory consumption from the changes in this HLD.

- The additional logic in `sfputil` consists of argument parsing, simple branching, and invoking existing platform APIs.
- No long-lived data structures are introduced; any temporary objects (for example, lists of devices returned by composite SFP APIs) are created and freed within the lifetime of a single CLI invocation.
- When composite SFP support is not used (for example, on platforms without CPO hardware), the memory footprint is effectively identical to the existing `sfputil` utility.

### 12. Restrictions/Limitations  

### 13. Testing Requirements/Design  
Explain what kind of unit testing, system testing, regression testing, warmboot/fastboot testing, etc.,
Ensure that the existing warmboot/fastboot requirements are met. For example, if the current warmboot feature expects maximum of 1 second or zero second data disruption, the same should be met even after the new feature/enhancement is implemented. Explain the same here.
Example sub-sections for unit test cases and system test cases are given below. 

#### 13.1. Unit Test cases  

#### 13.2. System Test cases

### 14. Open/Action items - if any 

	
NOTE: All the sections and sub-sections given above are mandatory in the design document. Users can add additional sections/sub-sections if required.