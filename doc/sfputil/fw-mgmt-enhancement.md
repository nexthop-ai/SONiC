# Transceiver Firmware Management Enhancement

## Table of Content

- [1. Revision](#1-revision)
- [2. Scope](#2-scope)
- [3. Definitions/Abbreviations](#3-definitionsabbreviations)
- [4. Overview](#4-overview)
- [5. Requirements](#5-requirements)
- [6. Architecture Design](#6-architecture-design)
- [7. High-Level Design](#7-high-level-design)
- [8. SAI API](#8-sai-api)
- [9. Configuration and Management](#9-configuration-and-management)
- [12. Restrictions/Limitations](#12-restrictionslimitations)
- [13. Testing Requirements/Design](#13-testing-requirementsdesign)
- [14. Open/Action Items](#14-openaction-items)
- [Appendix A: References](#appendix-a-references)
- [Appendix B: Example Output](#appendix-b-example-output)
- [Appendix C: Command Reference Quick Guide](#appendix-c-command-reference-quick-guide)

### 1. Revision

| Rev | Date       | Author       | Change Description |
|-----|------------|--------------|-------------------|
| 0.1 | 2026-03-18 | SONiC Team   | Initial version   |

### 2. Scope

This document describes the enhancement to the `sfputil` command-line utility to support firmware management operations for optical transceivers in SONiC. The enhancement adds filtering capabilities to both firmware version display and firmware upgrade commands, enabling operators to efficiently manage firmware across multiple transceivers based on interface lists or vendor part numbers.

This design is a built-in SONiC feature that extends the existing `sonic-utilities` package, specifically the `sfputil` CLI tool.

### 3. Definitions/Abbreviations

| Term    | Meaning                                           |
|---------|---------------------------------------------------|
| SFP     | Small Form-factor Pluggable transceiver          |
| QSFP    | Quad Small Form-factor Pluggable transceiver     |
| CMIS    | Common Management Interface Specification         |
| CDB     | Command Data Block (CMIS firmware update mechanism)|
| PN      | Part Number                                       |
| HLD     | High-Level Design                                 |
| CLI     | Command Line Interface                            |

### 4. Overview

Transceiver firmware management is a critical operational task in modern data center networks. As networks scale, managing firmware updates for hundreds or thousands of transceivers becomes increasingly challenging. This enhancement addresses this operational challenge by introducing advanced filtering and firmware upgrade capabilities to the `sfputil` utility.

The enhancement provides two key capabilities:
1. **Filtered firmware version display**: View firmware versions for specific subsets of transceivers
2. **Multi-transceiver firmware upgrade**: Upgrade firmware for multiple transceivers simultaneously based on interface lists or vendor part numbers

This feature integrates seamlessly with the existing SONiC platform architecture and leverages the established Platform API for transceiver management.

### 5. Requirements

This section lists all the requirements for the transceiver firmware management enhancement.

#### 5.1. Functional Requirements

1. **Interface List Filtering**: Support filtering operations by comma-separated list of interface names
2. **Vendor Part Number Filtering**: Support filtering operations by vendor part number(s)
3. **Tabular Display**: Provide compact tabular output format for firmware version information
4. **Concurrent Upgrade**: Support concurrent firmware upgrade operations across multiple transceivers
5. **Progress Tracking**: Display upgrade progress for multi-port operations
6. **Error Handling**: Gracefully handle errors and provide clear feedback for failed operations
7. **Backward Compatibility**: Maintain compatibility with existing single-port operations

#### 5.2. Non-Functional Requirements

1. **Performance**: Leverage parallel processing for multi-port operations to minimize total upgrade time
2. **Usability**: Provide clear, intuitive command-line interface consistent with existing SONiC CLI patterns
3. **Reliability**: Include retry mechanisms for failed upgrade operations
4. **Observability**: Provide detailed status information before and after operations
5. **Scalability**: Support operations on hundreds of transceivers simultaneously

### 6. Architecture Design

This section covers the changes required in the SONiC architecture. The enhancement builds upon the existing `sfputil` architecture and does not introduce new system components or modify the core SONiC architecture. It extends the existing CLI framework with additional filtering and parallel operation capabilities.

#### 6.1. System Architecture Overview

The feature is a built-in SONiC enhancement that modifies the `sonic-utilities` repository. No changes are required to the SONiC Application Extension infrastructure.

#### 6.2. Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     sfputil CLI                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  show fwversion [--interfaces] [--vendor-pn]           │ │
│  │  firmware upgrade [--interfaces] [--vendor-pn]         │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Filtering & Selection Logic                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  - Parse interface lists                               │ │
│  │  - Match vendor part numbers                           │ │
│  │  - Validate port presence                              │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│           Firmware Upgrade Logic                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  ThreadPoolExecutor ()                                 │ │
│  │  - Concurrent firmware downloads                       │ │
│  │  - Progress tracking                                   │ │
│  │  - Status aggregation                                  │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Platform API Layer                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  - get_transceiver_info()                              │ │
│  │  - get_module_fw_info()                                │ │
│  │  - cdb_start_firmware_download()                       │ │
│  │  - cdb_firmware_download_complete()                    │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

#### 6.3. Repositories Modified

- **sonic-utilities**: Main repository containing the `sfputil` CLI tool
  - Modified files: `sfputil/main.py` (CLI command definitions and implementation)

#### 6.4. Module Dependencies

- **Platform API**: Existing dependency for transceiver hardware access
- **Click**: CLI framework (existing dependency)
- **concurrent.futures**: Python standard library for parallel processing
- **enlighten**: Progress bar library for visual feedback

### 7. High-Level Design

This section covers the high-level design of the feature enhancement.

#### 7.1. Feature Type

This is a built-in SONiC feature that extends the existing `sfputil` CLI utility.

#### 7.2. Modules and Sub-modules Modified

- **sfputil CLI module** (`sonic-utilities/sfputil/main.py`):
  - Enhanced `show fwversion` command with filtering options
  - Enhanced `firmware upgrade` command with multi-port support
  - Added filtering and selection logic
  - Added parallel processing engine for concurrent operations

#### 7.3. CLI Command Enhancements

##### 7.3.1. Firmware Version Display Enhancement

**Command**: `sfputil show fwversion`

**New Options**:
- `--tabulate`: Display output in compact tabular format
- `--interfaces <INTERFACE_LIST>`: Filter by comma-separated interface names
- `--vendor-pn <PART_NUMBER_LIST>`: Filter by comma-separated vendor part numbers

**Usage Examples**:
```bash
# Display firmware version for all transceivers in tabular format
sudo sfputil show fwversion --tabulate

# Display firmware version for specific interfaces
sudo sfputil show fwversion --interfaces Ethernet0,Ethernet4,Ethernet8

# Display firmware version for all transceivers with specific part number
sudo sfputil show fwversion --vendor-pn ALPHA123456 --tabulate

# Combine filters
sudo sfputil show fwversion --interfaces Ethernet0,Ethernet4 --tabulate
```

##### 7.3.2. Firmware Upgrade Enhancement

**Command**: `sfputil firmware upgrade`

**New Options**:
- `--interfaces <INTERFACE_LIST> <FILEPATH>`: Upgrade firmware for comma-separated interface list
- `--vendor-pn <PART_NUMBER_LIST> <FILEPATH>`: Upgrade firmware for all ports matching vendor part number

**Usage Examples**:
```bash
# Upgrade firmware for a single port (existing functionality)
sudo sfputil firmware upgrade Ethernet0 /path/to/firmware.bin

# Upgrade firmware for specific interfaces
sudo sfputil firmware upgrade --interfaces Ethernet0,Ethernet4,Ethernet8 /path/to/firmware.bin

# Upgrade firmware for all transceivers with specific part number
sudo sfputil firmware upgrade --vendor-pn ALPHA123456 /path/to/firmware_v1.2.3.bin

# Multiple vendor part numbers with different firmware files
sudo sfputil firmware upgrade \
  --vendor-pn ALPHA123456 /path/to/alpha_fw.bin \
  --vendor-pn GAMMA67890 /path/to/gamma_fw.bin
```

**Upgrade Process Flow**:
```
1. Parse and validate input parameters
2. Identify target ports based on filters
3. Retrieve current firmware information
4. Display pre-upgrade status (tabular format)
5. Execute parallel firmware upgrade operations:
   a. Download firmware to transceiver
   b. Run/activate new firmware
   c. Verify firmware switch completion
   d. Commit firmware
6. Retry failed operations (up to 2 attempts total)
7. Display post-upgrade status (tabular format)
8. Report final results
```

#### 7.4. DB and Schema Changes

**No database schema changes are required.** The implementation uses existing STATE_DB tables:

- **STATE_DB:TRANSCEIVER_INFO**: Existing table for transceiver information (vendor, model, serial number)
- **STATE_DB:TRANSCEIVER_FIRMWARE_INFO**: Existing table for firmware version tracking

#### 7.5. Implementation Details

##### 7.5.1. Port Selection and Filtering

The implementation uses a multi-stage filtering approach:

1. **Get Present Ports**: Query all ports with transceivers present
2. **Apply Interface Filter**: If `--interfaces` specified, intersect with provided list
3. **Apply Vendor PN Filter**: If `--vendor-pn` specified, match against transceiver info
4. **Validate**: Ensure all specified ports are present and accessible

##### 7.5.2. Parallel Processing

Multi-port operations leverage Python's `concurrent.futures.ThreadPoolExecutor` for parallel processing:

- **Concurrency Model**: One thread per transceiver operation
- **Progress Tracking**: Per-port progress counters using `enlighten` library
- **Error Isolation**: Failures in individual ports don't affect others

**Benefits**:
- Significant time savings for multi-port operations
- Efficient resource utilization
- Real-time progress visibility

##### 7.5.3. Retry Mechanism

The firmware upgrade process includes automatic retry logic:

1. **First Attempt**:
   - Reset delay: 0 seconds
   - Run delay: 5 seconds
2. **Retry Attempt** (if first attempt fails):
   - Reset delay: 5 seconds
   - Run delay: 5 seconds

Only ports that failed in the first attempt are retried.

##### 7.5.4. Progress Display

For multi-port operations, the implementation provides:

1. **Pre-upgrade Status**: Tabular display of current firmware versions
2. **Progress Indicators**: Real-time status updates during upgrade
3. **Post-upgrade Status**: Tabular display of new firmware versions
4. **Failure Summary**: Detailed error information for failed operations

**Status Tracking**:
- Downloading Firmware
- Activating Firmware
- Committing Firmware
- Upgraded (success)
- Failed (with reason)

#### 7.6. Data Structures

##### 7.6.1. Port to Firmware Mapping
```python
port_to_firmware_map = {
    'Ethernet0': '/path/to/firmware.bin',
    'Ethernet4': '/path/to/firmware.bin',
    'Ethernet8': '/path/to/different_firmware.bin'
}
```

##### 7.6.2. Port Status Tracking
```python
port_status = {
    'Ethernet0': 'Downloading Firmware',
    'Ethernet4': 'Activating Firmware',
    'Ethernet8': 'Upgraded'
}
```

##### 7.6.3. Failure Information
```python
ports_failed_status_info = {
    'Ethernet0': (download_ok, run_ok, commit_ok, error_reason),
    # Example: (True, False, False, "status=2")
}
```

#### 7.7. Error Handling and Serviceability

The implementation includes comprehensive error handling:

1. **Input Validation**:
   - Verify port names are valid
   - Ensure transceivers are present
   - Validate firmware file paths

2. **Operation Errors**:
   - CDB command failures
   - Firmware download errors
   - Firmware activation failures
   - Commit operation failures

3. **User Feedback**:
   - Clear error messages
   - Specific failure reasons
   - Actionable guidance

#### 7.8. Platform-Specific Considerations

This feature is platform-agnostic and works with any platform that implements the standard SONiC Platform API for transceiver management. No platform-specific changes are required.

**Platform Requirements:**
- Platform must implement the Platform API transceiver methods listed in Section 8
- Transceivers must support CMIS specification for firmware upgrade operations

#### 7.9. Scalability and Performance

- **Scalability**: Supports concurrent operations on multiple transceivers
- **Memory**: Low additional memory footprint (Less than 15 MB during firmware upgrade)

#### 7.10. Warmboot and Fastboot Requirements

No warmboot or fastboot dependencies.

#### 7.11. Docker Dependencies

No new Docker containers or dependencies. The feature runs within the existing host environment where `sfputil` is executed.

#### 7.12. Build Dependencies

No new build dependencies. Uses existing Python standard library and SONiC dependencies.

### 8. SAI API

No additional SAI support is required for this enhancement.

### 9. Configuration and Management

This section covers the configuration and management interfaces for the feature.

#### 9.1. Manifest

Not applicable

#### 9.2. CLI/YANG Model Enhancements

##### 9.2.1. CLI Enhancements

The feature extends the existing `sfputil` CLI with new options. All changes maintain backward compatibility with existing CLI usage.

**Show Firmware Version Command:**

```
# sfputil show fwversion --help
Usage: sfputil show fwversion [OPTIONS] <port_name>

  Show firmware version of the transceiver(s) (all ports if no port specified)

Options:
  --tabulate                      Display firmware version in tabular format
  --interfaces <INTERFACE_LIST>   Comma-separated list of interfaces
  --vendor-pn <PART_NUMBER_LIST>  Comma-separated list of vendor part numbers
  --help                          Show this message and exit.
```

**Firmware Upgrade Command:**

```
# sfputil firmware upgrade --help
Usage: sfputil firmware upgrade [OPTIONS] [PORT_NAME] [FILEPATH]

  Upgrade firmware on the transceiver

Options:
  --interfaces <INTERFACE_LIST> <FILEPATH>
                                  Upgrade firmware for comma-separated
                                  interface list with specified firmware file.
                                  Example: --interfaces Ethernet0,Ethernet4
                                  /path/to/firmware.bin
  --vendor-pn <PART_NUMBER_LIST> <FILEPATH>
                                  Upgrade firmware for all ports with
                                  specified vendor part number using specified
                                  firmware file
  --help                          Show this message and exit.
```

**Backward Compatibility:**
- Existing single-port commands continue to work without modification
- New options are additive and do not break existing scripts or workflows
- Command output format remains consistent with existing patterns

**CLI Framework:**
- Implementation uses Click framework (existing SONiC standard)
- No KLISH changes required (feature is CLICK-based only)

**Documentation Update:**
- The Command Reference (https://github.com/sonic-net/sonic-utilities/blob/master/doc/Command-Reference.md) will be updated with the new CLI options

##### 9.2.2. YANG Model Changes

No YANG model changes are required.

#### 9.3. Config DB Enhancements

No Config DB changes are required.

### 10. Warmboot and Fastboot Design Impact

This enhancement has **no impact** on warmboot or fastboot operations.

### 11. Memory Consumption

Low additional memory footprint (Less than 15 MB during firmware upgrade).

### 12. Restrictions/Limitations

#### 12.1. Current Limitations

1. **Firmware File Management**:
   - Firmware files must be accessible on the local filesystem
   - No automatic firmware file download or repository management

2. **Vendor Part Number Matching**:
   - Exact string match required for vendor PN filtering
   - No wildcard or regex pattern matching

3. **Interface Validation**:
   - All specified interfaces must have transceivers present
   - Operation fails if any specified port is not present

#### 12.2. Operational Considerations

1. **Error Recovery**:
   - Automatic retry mechanism (2 attempts total)
   - Manual intervention may be required for persistent failures

### 13. Testing Requirements/Design

This section explains the testing strategy for the feature, including unit testing, system testing, and regression testing to ensure existing warmboot/fastboot requirements are met.

#### 13.1. Unit Test Cases

##### 13.1.1. Test Cases for Firmware Version Display

| Test Case ID | Description | Expected Result |
|--------------|-------------|-----------------|
| FW-SHOW-01 | Display firmware version for single port | Firmware details displayed |
| FW-SHOW-02 | Display firmware version with --tabulate | Tabular format output |
| FW-SHOW-03 | Filter by single interface | Only specified interface shown |
| FW-SHOW-04 | Filter by multiple interfaces | Only specified interfaces shown |
| FW-SHOW-05 | Filter by vendor PN | Only matching transceivers shown |
| FW-SHOW-06 | Filter by non-existent vendor PN | "No matching ports" message |
| FW-SHOW-07 | Combine --interfaces and --tabulate | Filtered tabular output |
| FW-SHOW-08 | Invalid interface name | Error message displayed |
| FW-SHOW-09 | Interface without transceiver | Appropriate error handling |

##### 13.1.2. Test Cases for Firmware Upgrade

| Test Case ID | Description | Expected Result |
|--------------|-------------|-----------------|
| FW-UPG-01 | Upgrade single port | Successful upgrade |
| FW-UPG-02 | Upgrade with --interfaces option | All specified ports upgraded |
| FW-UPG-03 | Upgrade with --vendor-pn option | All matching ports upgraded |
| FW-UPG-04 | Multiple --vendor-pn options | All matching ports upgraded with correct firmware |
| FW-UPG-05 | Invalid firmware file path | Error message displayed |
| FW-UPG-06 | Non-existent interface | Error message displayed |
| FW-UPG-07 | Interface without transceiver | Error message displayed |
| FW-UPG-08 | Firmware download failure | Retry mechanism triggered |
| FW-UPG-09 | Firmware activation failure | Error reported, retry attempted |
| FW-UPG-10 | Mixed success/failure scenario | Successful ports upgraded, failures reported |
| FW-UPG-11 | No matching ports for vendor PN | Appropriate message displayed |

#### 13.2. System Test Cases

##### 13.2.1. Functional Testing

**Test Scenario 1: Multi-Port Upgrade by Interface List**
```bash
# Setup: Prepare test environment with multiple transceivers
# Execute: Upgrade firmware for specific interfaces
sudo sfputil firmware upgrade --interfaces Ethernet0,Ethernet4,Ethernet8 /tmp/test_fw.bin

# Verify:
# 1. Pre-upgrade status displayed
# 2. Progress indicators shown
# 3. Post-upgrade status displayed
# 4. All specified ports upgraded successfully
# 5. Firmware versions updated in STATE_DB
```

**Test Scenario 2: Multi-Port Upgrade by Vendor PN**
```bash
# Setup: Environment with mixed transceiver vendors
# Execute: Upgrade all transceivers from specific vendor
sudo sfputil firmware upgrade --vendor-pn ALPHA123456 /tmp/alpha_fw.bin

# Verify:
# 1. Only matching transceivers selected
# 2. Parallel upgrade execution
# 3. All matching ports upgraded
# 4. Non-matching ports unaffected
```

**Test Scenario 3: Filtered Firmware Version Display**
```bash
# Execute: Display firmware version for specific vendor
sudo sfputil show fwversion --vendor-pn ALPHA123456 --tabulate

# Verify:
# 1. Only matching transceivers displayed
# 2. Tabular format used
# 3. All firmware fields populated correctly
```

**Test Scenario 4: Error Handling and Retry**
```bash
# Setup: Simulate firmware upgrade failure
# Execute: Upgrade with expected failure
sudo sfputil firmware upgrade --interfaces Ethernet0 /tmp/test_fw.bin

# Verify:
# 1. Initial attempt fails
# 2. Retry mechanism triggered
# 3. Failure details reported
# 4. System remains stable
```

##### 13.2.2. Performance Testing

**Performance Metrics**

| Metric | Measurement Method |
|--------|-------------------|
| Single port upgrade time | Time from start to completion |
| Multi-port upgrade time | Time from start to completion |

##### 13.2.3. Warmboot/Fastboot Testing

Not applicable

### 14. Open/Action Items

Not applicable

## Appendix A: References

1. CMIS Specification: http://www.qsfp-dd.com/wp-content/uploads/2021/05/CMIS5p0.pdf
2. SONiC Platform API Documentation
3. Python concurrent.futures Documentation: https://docs.python.org/3/library/concurrent.futures.html
4. Click CLI Framework Documentation: https://click.palletsprojects.com/
5. SONiC Command Reference: https://github.com/sonic-net/sonic-utilities/blob/master/doc/Command-Reference.md

## Appendix B: Example Output

### B.1. Firmware Version Display 

#### B.1.1 Standard format
```
# sfputil show fwversion Ethernet128
Interface: Ethernet128
Vendor Name: EFGH Systems 
Vendor PN: Alphaxxxxxxxx001
Vendor SN: A11CLA5
Image A Version: 255.2.0
Image B Version: 255.2.0
Factory Image Version: 37.2.3
Running Image: B
Committed Image: B
Active Firmware: 255.2.0
Inactive Firmware: 255.2.0
```

#### B.1.2 Tabular format
```
# sfputil show fwversion Ethernet128 --tabulate
Interface    Vendor Name    Vendor PN         Vendor SN    Image A    Image B    Active    Running    Committed
-----------  -------------  ----------------  -----------  ---------  ---------  --------  ---------  -----------
Ethernet128  EFGH Systems   Alphaxxxxxxxx001  A11CLA5      255.2.0    255.2.0    255.2.0   B          B
```

#### B.1.3 Filter by vendor PN
```
# sfputil show fwversion --vendor-pn Alphaxxxxxxxx001 --tabulate
Interface    Vendor Name    Vendor PN         Vendor SN    Image A    Image B    Active    Running    Committed
-----------  -------------  ----------------  -----------  ---------  ---------  --------  ---------  -----------
Ethernet128  EFGH Systems   Alphaxxxxxxxx001  A11CLA5      255.2.0    255.2.0    255.2.0   B          B
Ethernet232  EFGH Systems   Alphaxxxxxxxx001  Z12CR69      255.2.0    255.2.0    255.2.0   B          B
Ethernet256  EFGH Systems   Alphaxxxxxxxx001  BR8C184      255.2.0    255.2.0    255.2.0   B          B
```

#### B.1.4 Filter by multiple vendor PNs

```
# sfputil show fwversion --vendor-pn Alphaxxxxxxxx001,Thetazzzzzzzz003 --tabulate
Interface    Vendor Name    Vendor PN         Vendor SN    Image A    Image B    Active    Running    Committed
-----------  -------------  ----------------  -----------  ---------  ---------  --------  ---------  -----------
Ethernet0    ABCD Corp      Thetazzzzzzzz003  UR3T010005   3.3.0      3.3.0      3.3.0     A          A
Ethernet8    ABCD Corp      Thetazzzzzzzz003  UR3T030047   3.3.0      3.3.0      3.3.0     A          A
Ethernet16   ABCD Corp      Thetazzzzzzzz003  UR19000007   3.3.0      3.3.0      3.3.0     A          A
Ethernet24   ABCD Corp      Thetazzzzzzzz003  UR3T030033   3.3.0      3.3.0      3.3.0     B          B
Ethernet56   ABCD Corp      Thetazzzzzzzz003  UR3T030010   3.3.0      3.3.0      3.3.0     A          A
Ethernet72   ABCD Corp      Thetazzzzzzzz003  UR3T010016   3.3.0      3.3.0      3.3.0     B          B
Ethernet88   ABCD Corp      Thetazzzzzzzz003  UR3T010004   3.3.0      3.3.0      3.3.0     B          B
Ethernet128  EFGH Systems   Alphaxxxxxxxx001  A11CLA5      2.3.0      2.3.0      2.3.0     B          B
Ethernet136  ABCD Corp      Thetazzzzzzzz003  UR3T030039   3.3.0      3.3.0      3.3.0     A          A
Ethernet176  ABCD Corp      Thetazzzzzzzz003  UR3T030018   3.3.0      3.2.0      3.2.0     B          B
Ethernet184  ABCD Corp      Thetazzzzzzzz003  UR3T030032   3.3.0      3.2.0      3.2.0     B          B
Ethernet232  EFGH Systems   Alphaxxxxxxxx001  Z12CR69      255.2.0    255.2.0    255.2.0   B          B
Ethernet256  EFGH Systems   Alphaxxxxxxxx001  BR8C184      255.2.0    255.2.0    255.2.0   B          B
```

### B.2. Firmware Upgrade

### B.2.1 Single Port Firmware Upgrade

```
# sfputil firmware upgrade Ethernet128 EFGH_Systems_Alphaxxxxxxxx001_V2_3.bin
Upgrading image for 1 transceiver(s)

CDB: Firmware status before upgrade:
Interface    Vendor Name    Vendor PN         Vendor SN    Image A    Image B    Active    Running    Committed
-----------  -------------  ----------------  -----------  ---------  ---------  --------  ---------  -----------
Ethernet128  EFGH Systems   Alphaxxxxxxxx001  A11CLA5      255.2.0    255.2.0    255.2.0   B          B

CDB: Starting firmware upgrade: 16:23:36
CDB: Starting firmware download
Ethernet128: Downloading   100%|################################################################| 550532.00/550596.00 B [00:00]
CDB: firmware download complete
Running firmware: Non-hitless Reset to Inactive Image
FW images switch successful : ImageA is running

CDB: Finished firmware upgrade: 16:24:45. Time taken: 69 seconds

Succeeded: 1, Failed: 0

CDB: Firmware status after upgrade:
Interface    Vendor Name    Vendor PN         Vendor SN    Image A    Image B    Active    Running    Committed
-----------  -------------  ----------------  -----------  ---------  ---------  --------  ---------  -----------
Ethernet128  EFGH Systems   Alphaxxxxxxxx001  A11CLA5      2.3.0      255.2.0    2.3.0     A          A

```

### B.2.2 Multi-Port Firmware Upgrade

### B.2.2.1 Multi-Port Firmware Upgrade using --interfaces option

```
# sfputil firmware upgrade --interfaces Ethernet248,Ethernet320,Ethernet384,Ethernet400 EFGH_Systems_Gammayyyyyyyy002-V2_5.bin
Upgrading image for 4 transceiver(s)

CDB: Firmware status before upgrade:
Interface    Vendor Name    Vendor PN         Vendor SN    Image A    Image B    Active    Running    Committed
-----------  -------------  ----------------  -----------  ---------  ---------  --------  ---------  -----------
Ethernet248  EFGH Systems   Gammayyyyyyyy002  DC3CY1E      255.5.0    255.5.0    255.5.0   B          B
Ethernet320  EFGH Systems   Gammayyyyyyyy002  DC3A3MK      255.5.0    255.5.0    255.5.0   B          B
Ethernet384  EFGH Systems   Gammayyyyyyyy002  UDNEJCA      255.5.0    255.5.0    255.5.0   B          B
Ethernet400  EFGH Systems   Gammayyyyyyyy002  UDLCZVZ      255.5.0    255.5.0    255.5.0   B          B

CDB: Starting firmware upgrade: 16:43:25

Progress: Not Started(0), Downloading FW(0), Activating FW(0), Committing FW(0), Upgraded(4), Failed(0)
CDB: Finished firmware upgrade: 16:45:33. Time taken: 127 seconds

Succeeded: 4, Failed: 0

CDB: Firmware status after upgrade:
Interface    Vendor Name    Vendor PN         Vendor SN    Image A    Image B    Active    Running    Committed
-----------  -------------  ----------------  -----------  ---------  ---------  --------  ---------  -----------
Ethernet248  EFGH Systems   Gammayyyyyyyy002  DC3CY1E      2.5.0      255.5.0    2.5.0     A          A
Ethernet320  EFGH Systems   Gammayyyyyyyy002  DC3A3MK      2.5.0      255.5.0    2.5.0     A          A
Ethernet384  EFGH Systems   Gammayyyyyyyy002  UDNEJCA      2.5.0      255.5.0    2.5.0     A          A
Ethernet400  EFGH Systems   Gammayyyyyyyy002  UDLCZVZ      2.5.0      255.5.0    2.5.0     A          A
```

### B.2.2.1 Multi-Port Firmware Upgrade using --vendor-pn option

```
# sfputil firmware upgrade --vendor-pn Gammayyyyyyyy002 EFGH_Systems_Gammayyyyyyyy002-VFF_5.bin --vendor-pn Alphaxxxxxxxx001 EFGH_Systems_Alphaxxxxxxxx001_V2_3.bin
Upgrading image for 14 transceiver(s)

CDB: Firmware status before upgrade:
Interface    Vendor Name    Vendor PN         Vendor SN    Image A    Image B    Active    Running    Committed
-----------  -------------  ----------------  -----------  ---------  ---------  --------  ---------  -----------
Ethernet128  EFGH Systems   Alphaxxxxxxxx001  A11CLA5      255.2.0    255.2.0    255.2.0   B          B
Ethernet232  EFGH Systems   Alphaxxxxxxxx001  Z12CR69      255.2.0    255.2.0    255.2.0   B          B
Ethernet248  EFGH Systems   Gammayyyyyyyy002  DC3CY1E      2.5.0      255.5.0    2.5.0     A          A
Ethernet256  EFGH Systems   Alphaxxxxxxxx001  BR8C184      255.2.0    255.2.0    255.2.0   B          B
Ethernet320  EFGH Systems   Gammayyyyyyyy002  DC3A3MK      2.5.0      255.5.0    2.5.0     A          A
Ethernet336  EFGH Systems   Alphaxxxxxxxx001  JK4DQPD      255.2.0    255.2.0    255.2.0   B          B
Ethernet352  EFGH Systems   Alphaxxxxxxxx001  A11CLAP      255.2.0    255.2.0    255.2.0   B          B
Ethernet384  EFGH Systems   Gammayyyyyyyy002  UDNEJCA      2.5.0      255.5.0    2.5.0     A          A
Ethernet400  EFGH Systems   Gammayyyyyyyy002  UDLCZVZ      2.5.0      255.5.0    2.5.0     A          A
Ethernet416  EFGH Systems   Alphaxxxxxxxx001  Z12CR5Y      255.2.0    255.2.0    255.2.0   B          B
Ethernet480  EFGH Systems   Alphaxxxxxxxx001  HNABCQT      255.2.0    255.2.0    255.2.0   B          B
Ethernet488  EFGH Systems   Alphaxxxxxxxx001  HN9CYDK      255.2.0    255.2.0    255.2.0   B          B
Ethernet496  EFGH Systems   Alphaxxxxxxxx001  HN9CYAT      255.2.0    255.2.0    255.2.0   B          B
Ethernet504  EFGH Systems   Alphaxxxxxxxx001  HN7DQ6X      255.2.0    255.2.0    255.2.0   B          B

CDB: Starting firmware upgrade: 17:03:15

Progress: Not Started(0), Downloading FW(0), Activating FW(0), Committing FW(0), Upgraded(14), Failed(0)
CDB: Finished firmware upgrade: 17:05:11. Time taken: 116 seconds

Succeeded: 14, Failed: 0

CDB: Firmware status after upgrade:
Interface    Vendor Name    Vendor PN         Vendor SN    Image A    Image B    Active    Running    Committed
-----------  -------------  ----------------  -----------  ---------  ---------  --------  ---------  -----------
Ethernet128  EFGH Systems   Alphaxxxxxxxx001  A11CLA5      2.3.0      255.2.0    2.3.0     A          A
Ethernet232  EFGH Systems   Alphaxxxxxxxx001  Z12CR69      2.3.0      255.2.0    2.3.0     A          A
Ethernet248  EFGH Systems   Gammayyyyyyyy002  DC3CY1E      2.5.0      255.5.0    255.5.0   B          B
Ethernet256  EFGH Systems   Alphaxxxxxxxx001  BR8C184      2.3.0      255.2.0    2.3.0     A          A
Ethernet320  EFGH Systems   Gammayyyyyyyy002  DC3A3MK      2.5.0      255.5.0    255.5.0   B          B
Ethernet336  EFGH Systems   Alphaxxxxxxxx001  JK4DQPD      2.3.0      255.2.0    2.3.0     A          A
Ethernet352  EFGH Systems   Alphaxxxxxxxx001  A11CLAP      2.3.0      255.2.0    2.3.0     A          A
Ethernet384  EFGH Systems   Gammayyyyyyyy002  UDNEJCA      2.5.0      255.5.0    255.5.0   B          B
Ethernet400  EFGH Systems   Gammayyyyyyyy002  UDLCZVZ      2.5.0      255.5.0    255.5.0   B          B
Ethernet416  EFGH Systems   Alphaxxxxxxxx001  Z12CR5Y      2.3.0      255.2.0    2.3.0     A          A
Ethernet480  EFGH Systems   Alphaxxxxxxxx001  HNABCQT      2.3.0      255.2.0    2.3.0     A          A
Ethernet488  EFGH Systems   Alphaxxxxxxxx001  HN9CYDK      2.3.0      255.2.0    2.3.0     A          A
Ethernet496  EFGH Systems   Alphaxxxxxxxx001  HN9CYAT      2.3.0      255.2.0    2.3.0     A          A
Ethernet504  EFGH Systems   Alphaxxxxxxxx001  HN7DQ6X      2.3.0      255.2.0    2.3.0     A          A
```

## Appendix C: Command Reference Quick Guide

```bash
# Show firmware version for all ports (tabular)
sudo sfputil show fwversion --tabulate

# Show firmware version for specific interfaces
sudo sfputil show fwversion --interfaces Ethernet0,Ethernet4

# Show firmware version for specific vendor PN
sudo sfputil show fwversion --vendor-pn Alphaxxxxxxxx001

# Upgrade single port
sudo sfputil firmware upgrade Ethernet0 /path/to/firmware.bin

# Upgrade multiple interfaces
sudo sfputil firmware upgrade --interfaces Ethernet0,Ethernet4 /path/to/firmware.bin

# Upgrade by vendor PN
sudo sfputil firmware upgrade --vendor-pn Alphaxxxxxxxx001 /path/to/alpha_fw.bin

# Multiple vendor PNs with different firmware
sudo sfputil firmware upgrade \
  --vendor-pn Alphaxxxxxxxx001 /path/to/alpha_fw.bin \
  --vendor-pn Gammayyyyyyyy002 /path/to/gamma_fw.bin
```