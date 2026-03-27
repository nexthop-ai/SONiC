# HLD Name #

## Table of Content 

- [1. Revision](#1-revision)
- [2. Scope](#2-scope)
- [3. Definitions/Abbreviations](#3-definitionsabbreviations)
- [4. Overview](#4-overview)
- [5. Requirements](#5-requirements)
- [6. Architecture Design](#6-architecture-design)
- [7. High-Level Design](#7-high-level-design)
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

The following examples illustrate how the `SfpBase` and composite SFP classes can be structured to support an optional device selector while keeping composite-specific logic inside the platform implementation. These are illustrative only; exact method signatures may be refined during implementation.

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

    @abc.abstractmethod
    def set_power(self, enable: bool, device: typing.Optional[str] = None) -> None:
        """Enable or disable module power.

        Composite implementations may apply this to one or more internal
        devices depending on the selector and platform policy.
        """
        ...

    @abc.abstractmethod
    def set_lpmode(self, enable: bool, device: typing.Optional[str] = None) -> None:
        """Enable or disable low power mode."""
        ...

    @abc.abstractmethod
    def reset(self, device: typing.Optional[str] = None) -> None:
        """Reset the transceiver or a specific internal device."""
        ...
```

```python
class VendorSfp(SfpBase):
    """Traditional (non-composite) SFP implementation."""

    def read_eeprom(self, page, offset, size, device=None):
        if device is not None:
            raise ValueError("Device selector is not valid for non-composite SFPs")
        # Existing single-device EEPROM access logic
        return self._read_eeprom_raw(page, offset, size)

    def set_power(self, enable, device=None):
        if device is not None:
            raise ValueError("Device selector is not valid for non-composite SFPs")
        self._set_power_impl(enable)

    def set_lpmode(self, enable, device=None):
        if device is not None:
            raise ValueError("Device selector is not valid for non-composite SFPs")
        self._set_lpmode_impl(enable)

    def reset(self, device=None):
        if device is not None:
            raise ValueError("Device selector is not valid for non-composite SFPs")
        self._reset_impl()
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

    # Example: operations that require a specific target device
    def read_eeprom(self, page, offset, size, device=None):
        if device is None:
            raise ValueError(
                "Composite CPO SFP requires a device selector for read_eeprom; "
                "run 'sfputil show devices -p <port>' to see valid devices."
            )
        target = self.get_internal_device(device)
        return target.read_eeprom(page, offset, size)

    # Example: operations that can reasonably default to "all devices"
    def set_power(self, enable, device=None):
        targets = [self.get_internal_device(device)] if device else self.get_internal_devices()
        for t in targets:
            t.set_power(enable)

    def set_lpmode(self, enable, device=None):
        targets = [self.get_internal_device(device)] if device else self.get_internal_devices()
        for t in targets:
            t.set_lpmode(enable)

    def reset(self, device=None):
        targets = [self.get_internal_device(device)] if device else self.get_internal_devices()
        for t in targets:
            t.reset()
```

In this model, `sfputil` does not need to know whether a port is composite or how many internal devices exist. It simply passes the optional device selector through to the `SfpBase` methods and reports any exceptions (for example, missing device selector for `read_eeprom` on a composite port) as user-facing CLI errors.

### 8. SAI API 

There are no changes to SAI API in this HLD.

### 9. Configuration and management 
This section should have sub-sections for all types of configuration and management related design. Example sub-sections for "CLI" and "Config DB" are given below. Sub-sections related to data models (YANG, REST, gNMI, etc.,) should be added as required.
If there is breaking change which may impact existing platforms, please call out in the design and get platform vendors reviewed. 

#### 9.1. Manifest (if the feature is an Application Extension)

Paste a preliminary manifest in a JSON format.

#### 9.2. CLI/YANG model Enhancements 

This sub-section covers the addition/deletion/modification of CLI changes and YANG model changes needed for the feature in detail. If there is no change in CLI for HLD feature, it should be explicitly mentioned in this section. Note that the CLI changes should ensure downward compatibility with the previous/existing CLI. i.e. Users should be able to save and restore the CLI from previous release even after the new CLI is implemented. 
This should also explain the CLICK and/or KLISH related configuration/show in detail.
https://github.com/sonic-net/sonic-utilities/blob/master/doc/Command-Reference.md needs be updated with the corresponding CLI change.

#### 9.3. Config DB Enhancements  

This sub-section covers the addition/deletion/modification of config DB changes needed for the feature. If there is no change in configuration for HLD feature, it should be explicitly mentioned in this section. This section should also ensure the downward compatibility for the change. 
		
### 10. Warmboot and Fastboot Design Impact  
Mention whether this feature/enhancement has got any requirements/dependencies/impact w.r.t. warmboot and fastboot. Ensure that existing warmboot/fastboot feature is not affected due to this design and explain the same.

### Warmboot and Fastboot Performance Impact
This sub-section must cover the impact of the functionality on warmboot and fastboot performance, that is control plane and data plane downtime.
As part of the analysis cover the flowing:

- Does this feature add any stalls/sleeps/IO operations to the boot critical chain? Does it change when this feature is disabled/unused? 
- Does this feature add any additional CPU heavy processing (e.g. rendering Jinja templates) in the boot path (process, library or utility used during boot up)? Does it change when this feature is disabled/unused?
- In case this feature updates a third party dependency does it cause any impact on boot time performance?
- Can the feature (service or docker) be delayed?
- What are the possible optimizations and what is the expected boot time degradation if, by the nature of the feature, additional CPU/IO costs can't be avoided?

### 11. Memory Consumption
This sub-section covers the memory consumption analysis for the new feature: no memory consumption is expected when the feature is disabled via compilation and no growing memory consumption while feature is disabled by configuration. 
### 12. Restrictions/Limitations  

### 13. Testing Requirements/Design  
Explain what kind of unit testing, system testing, regression testing, warmboot/fastboot testing, etc.,
Ensure that the existing warmboot/fastboot requirements are met. For example, if the current warmboot feature expects maximum of 1 second or zero second data disruption, the same should be met even after the new feature/enhancement is implemented. Explain the same here.
Example sub-sections for unit test cases and system test cases are given below. 

#### 13.1. Unit Test cases  

#### 13.2. System Test cases

### 14. Open/Action items - if any 

	
NOTE: All the sections and sub-sections given above are mandatory in the design document. Users can add additional sections/sub-sections if required.