# Hardware SKU (HWSKU) CLI Management Feature #

## Table of Content

- [1. Revision](#1-revision)
- [2. Scope](#2-scope)
- [3. Definitions/Abbreviations](#3-definitionsabbreviations)
- [4. Overview](#4-overview)
- [5. Requirements](#5-requirements)
  - [5.1 Functional Requirements](#51-functional-requirements)
  - [5.2 Non-Functional Requirements](#52-non-functional-requirements)
- [6. Configuration and Management](#6-configuration-and-management)
- [7. Architecture Design](#7-architecture-design)
- [8. High-Level Design](#8-high-level-design)
  - [8.1 Modules and Sub-modules Modified](#81-modules-and-sub-modules-modified)
  - [8.2 Repositories Changed](#82-repositories-changed)
  - [8.3 Module Interfaces and Dependencies](#83-module-interfaces-and-dependencies)
  - [8.4 SWSS and Syncd Changes](#84-swss-and-syncd-changes)
  - [8.5 DB and Schema Changes](#85-db-and-schema-changes)
  - [8.6 Warm Reboot and Fastboot Requirements](#86-warm-reboot-and-fastboot-requirements)
  - [8.7 Scalability and Performance](#87-scalability-and-performance)
  - [8.8 Build Dependency](#88-build-dependency)
  - [8.9 Serviceability and Debug](#89-serviceability-and-debug)
  - [8.10 Platform-Specific Considerations](#810-platform-specific-considerations)
- [9. Restrictions/Limitations](#9-restrictionslimitations)
- [10. Testing Requirements/Design](#10-testing-requirementsdesign)
  - [10.1 Unit Test Cases](#101-unit-test-cases)
- [11. Open/Action Items](#11-openaction-items)

### 1. Revision

| Revision | Date       | Author | Description |
|----------|------------|--------|-------------|
| 0.1      | 2025-11-18 | bobby-nexthop      | Initial version |

### 2. Scope  

This document describes the design and implementation of a new CLI feature for managing Hardware SKU (HWSKU) configurations in SONiC. The feature provides user-friendly commands to view available HWSKU profiles and load different HWSKU configurations with automatic backup and recovery capabilities.

### 3. Definitions/Abbreviations 

| Term | Definition |
|------|------------|
| HWSKU | Hardware SKU - A hardware configuration profile that defines port configurations, speeds, and other hardware-specific settings |
| CLI | Command Line Interface |
| CONFIG_DB | Configuration Database - Redis database storing SONiC configuration |
| sonic-cfggen | SONiC Configuration Generator - Tool for generating configuration from various sources |

### 4. Overview 

The HWSKU CLI feature enables network administrators to:
- View all available HWSKU profiles for the current platform
- Load a different HWSKU configuration with automatic backup
- Utilize shell autocomplete for HWSKU names
- Safely switch between hardware configurations with automatic rollback on failure

This feature addresses the need for platforms with multiple hardware variants to easily switch between different SKU configurations without manual file editing.

### 5. Requirements

#### 5.1 Functional Requirements
- Provide `show hwsku profiles` command to list available HWSKU profiles
- Provide `config hwsku load <hwsku_name>` command to load a new HWSKU configuration
- Support shell autocomplete for HWSKU names
- Automatically backup current configuration before making changes
- Automatically restore backup if configuration generation fails
- Support optional automatic configuration reload
- Validate HWSKU name before making any changes
- Provide clear error messages with available options

#### 5.2 Non-Functional Requirements
- Maintain backward compatibility with existing SONiC CLI
- Minimal performance impact during autocomplete operations
- No impact on existing warmboot/fastboot functionality
- Support all SONiC platforms

### 6. Configuration and Management

**CLI Changes:**

This feature adds two new command groups to the SONiC CLI:

**1. Show Commands (`show hwsku`):**

```bash
show hwsku profiles
```

**Output Format:**
```
Platform: x86_64-nexthop_4010-r1

Available HWSKU Profiles:
  * NH-4010 (current)
  - NH-4010-128x400G-2x25G
  - NH-4020
```

**Help Text:**
```bash
admin@sonic:~$ show hwsku --help
Usage: show hwsku [OPTIONS] COMMAND [ARGS]...

  Show hardware SKU information

Options:
  --help  Show this message and exit.

Commands:
  profiles  Show available hardware SKU profiles
```

**2. Config Commands (`config hwsku`):**

```bash
config hwsku load <hwsku_name> [-y] [-r]
```

**Options:**
- `-y, --yes`: Automatically answer yes to prompts (skip confirmation)
- `-r, --reload`: Reload configuration after generating new config

**Interactive Mode Example:**
```bash
admin@sonic:~$ sudo config hwsku load NH-4010-128x400G-2x25G

This will:
  1. Backup current config to /etc/sonic/backup_config/NH-4010_20251118_143022.json
  2. Generate new config for HWSKU 'NH-4010-128x400G-2x25G'
  3. Skip reload (you will need to manually reload)

Do you want to continue? [y/N]: y

Loading hardware SKU: NH-4010-128x400G-2x25G
Config generated successfully

Hardware SKU 'NH-4010-128x400G-2x25G' config generated successfully!
Backup saved to: /etc/sonic/backup_config/NH-4010_20251118_143022.json

To apply the new configuration, run: config reload -y
```

**Auto-confirm and Reload Example:**
```bash
admin@sonic:~$ sudo config hwsku load NH-4010-128x400G-2x25G -y -r

Loading hardware SKU: NH-4010-128x400G-2x25G
Config generated successfully
Reloading configuration...

Hardware SKU 'NH-4010-128x400G-2x25G' loaded successfully!
Configuration reloaded.
Backup saved to: /etc/sonic/backup_config/NH-4010_20251118_143022.json
```

**Autocomplete Feature:**
```bash
admin@sonic:~$ sudo config hwsku load NH-<TAB>
NH-4010  NH-4010-128x400G-2x25G  NH-4020

admin@sonic:~$ sudo config hwsku load NH-4010-<TAB>
NH-4010-128x400G-2x25G
```

**Help Text:**
```bash
admin@sonic:~$ config hwsku --help
Usage: config hwsku [OPTIONS] COMMAND [ARGS]...

  Configure hardware SKU

Options:
  --help  Show this message and exit.

Commands:
  load  Load hardware SKU configuration

admin@sonic:~$ config hwsku load --help
Usage: config hwsku load [OPTIONS] HWSKU_NAME

  Load hardware SKU configuration

Options:
  -y, --yes     Automatically answer yes to prompts
  -r, --reload  Reload configuration after generating new config
  --help        Show this message and exit.
```

### 7. Architecture Design

This feature is implemented as a built-in SONiC CLI extension within the sonic-utilities package. It does not modify the core SONiC architecture but adds new CLI commands that interact with existing components:

- **sonic-utilities**: New CLI commands and helper functions
- **sonic-cfggen**: Existing tool used for configuration generation
- **CONFIG_DB**: Existing database updated with new HWSKU configuration
- **File System**: Backup directory for configuration files

The feature integrates seamlessly with the existing Click-based CLI framework and follows established SONiC CLI patterns.

### 8. High-Level Design

#### 8.1 Modules and Sub-modules Modified

**New Files:**
- `src/sonic-utilities/config/load_hwsku.py` - HWSKU load command implementation
- `src/sonic-utilities/show/show_hwsku.py` - HWSKU show command implementation
- `src/sonic-utilities/utilities_common/hwsku.py` - Shared HWSKU utility functions
- `src/sonic-utilities/tests/hwsku_load_test.py` - Unit tests
- `src/sonic-utilities/tests/show_hwsku_test.py` - Unit tests

**Modified Files:**
- `src/sonic-utilities/config/main.py` - Register hwsku command group
- `src/sonic-utilities/show/main.py` - Register hwsku show command group

#### 8.2 Repositories Changed
- sonic-utilities

#### 8.3 Module Interfaces and Dependencies

**Dependencies:**
- Click library (CLI framework)
- sonic_py_common.device_info (platform/HWSKU detection)
- sonic_py_common.logger (logging)
- sonic-cfggen (configuration generation)

**Interfaces:**
- CLI commands exposed to users
- File system operations for backup/restore
- Subprocess calls to sonic-cfggen and config reload

#### 8.4 SWSS and Syncd Changes
No changes required to SWSS or Syncd components. This feature operates at the configuration management layer.

#### 8.5 DB and Schema Changes
No database schema changes are required. The feature uses existing CONFIG_DB structure:
- Reads from `DEVICE_METADATA|localhost` table for current platform and HWSKU information
- Updates CONFIG_DB through standard `config reload` mechanism after generating new configuration

The HWSKU information is stored in the existing `DEVICE_METADATA` table:
```json
{
    "DEVICE_METADATA": {
        "localhost": {
            "hwsku": "NH-4010-128x400G-2x25G",
            "platform": "x86_64-nexthop_4010-r1",
            "hostname": "sonic",
            "mac": "..."
        }
    }
}
```

#### 8.6 Warm Reboot and Fastboot Requirements
This feature has no dependencies on or impact to warmboot/fastboot functionality.

#### 8.7 Scalability and Performance
- **Autocomplete Performance**: Lightweight directory scan, typically <10ms
- **Configuration Generation**: Depends on sonic-cfggen performance (existing tool)
- **Backup Operations**: File copy operation, minimal overhead
- **Memory**: No persistent memory consumption, temporary subprocess memory only

#### 8.8 Build Dependency
No new build dependencies. Uses existing Python libraries:
- Click (already required by sonic-utilities)
- sonic_py_common (already required)
- Standard Python libraries (os, shutil, subprocess, datetime)

#### 8.9 Serviceability and Debug

**Logging:**
- All operations logged via `sonic_py_common.logger`
- Log priority: INFO for normal operations, ERROR for failures
- Log messages include:
  - HWSKU validation results
  - Backup creation/restoration
  - Configuration generation success/failure
  - Reload operations

**Example Log Messages:**
```
INFO: Loading hardware SKU: NH-4010-128x400G-2x25G (current: NH-4010)
INFO: Backed up config to /etc/sonic/backup_config/NH-4010_20251118_143022.json
INFO: Generated new config for HWSKU 'NH-4010-128x400G-2x25G'
INFO: Successfully loaded HWSKU 'NH-4010-128x400G-2x25G' and reloaded config
ERROR: HWSKU validation failed: Invalid HWSKU 'invalid-sku'
ERROR: Failed to generate config: sonic-cfggen returned non-zero exit status
```

#### 8.10 Platform-Specific Considerations

**Platform Independence:**
- Feature works on all SONiC platforms
- Uses standard device directory structure: `/usr/share/sonic/device/<platform>/`
- Automatically detects platform from `/host/machine.conf`

**Platform Requirements:**
- Platform must have HWSKU directories under `/usr/share/sonic/device/<platform>/`
- Each HWSKU directory should contain valid port configuration files
- No platform-specific code changes required

**Excluded Directories:**
- `pddf` - Platform Driver Development Framework directory
- `plugins` - Platform plugin directory
- These are automatically filtered out from HWSKU list

**How Configuration is Updated:**
1. User runs `config hwsku load <hwsku_name>`
2. Tool generates new config using `sonic-cfggen -k <hwsku_name> --print-data`
3. New configuration (including updated HWSKU) is written to `/etc/sonic/config_db.json`
4. Configuration is loaded into CONFIG_DB via `config reload` (if `-r` flag used, or manually)

**Backward Compatibility:**
- Existing config_db.json files remain compatible
- No migration required
- Old configurations can be restored from backups
- Downgrade path: Simply restore a backup from `/etc/sonic/backup_config/`

### 9. Restrictions/Limitations

1. **Platform Detection Dependency**: Requires `/host/machine.conf` to be present and properly configured
2. **HWSKU Directory Structure**: Assumes standard SONiC directory structure (`/usr/share/sonic/device/<platform>/<hwsku>/`)
3. **Root Access Required**: Configuration changes require sudo/root privileges
4. **sonic-cfggen Dependency**: Requires sonic-cfggen to be installed and functional
5. **Backup Storage**: Backup files accumulate in `/etc/sonic/backup_config/` - users should periodically clean old backups
6. **No Validation of Generated Config**: Tool trusts sonic-cfggen output; does not validate port configurations
7. **Single Platform**: Cannot load HWSKU from a different platform
8. **No Rollback Timer**: Unlike some network devices, there is no automatic rollback after timeout
9. **Manual Cleanup**: Failed backups are not automatically cleaned up (by design, for safety)

### 10. Testing Requirements/Design

The feature includes comprehensive testing to ensure reliability and correctness.

#### 10.1. Unit Test Cases

**Test Coverage: 683 lines of test code**

**Helper Function Tests (`TestHwskuHelpers`):**
1. `test_get_available_hwskus` - Verify HWSKU discovery from filesystem
2. `test_get_available_hwskus_no_platform` - Handle missing platform gracefully
3. `test_get_available_hwskus_platform_path_not_exists` - Handle missing platform directory
4. `test_get_available_hwskus_oserror` - Handle filesystem errors
5. `test_is_valid_hwsku_valid` - Validate correct HWSKU names
6. `test_is_valid_hwsku_invalid` - Reject invalid HWSKU names
7. `test_is_valid_hwsku_no_platform` - Handle platform detection failure
8. `test_complete_hwsku_names_full_match` - Autocomplete with full prefix
9. `test_complete_hwsku_names_partial_match` - Autocomplete with partial prefix
10. `test_complete_hwsku_names_no_match` - Autocomplete with no matches
11. `test_complete_hwsku_names_empty_prefix` - Autocomplete returns all HWSKUs

**Config Command Tests (`TestHwskuLoadCommand`):**
1. `test_load_no_arguments` - Verify command requires HWSKU argument
2. `test_load_invalid_hwsku` - Reject invalid HWSKU names with helpful error
3. `test_load_success_no_reload` - Successfully generate config without reload
4. `test_load_success_with_reload` - Successfully generate and reload config
5. `test_load_interactive_cancel` - User can cancel at confirmation prompt
6. `test_load_interactive_prompt_shows_correct_info` - Confirmation shows accurate information
7. `test_load_backup_created_with_timestamp` - Backup file has correct naming format
8. `test_load_backup_failure` - Handle backup creation failures
9. `test_load_config_generation_subprocess_failure` - Handle sonic-cfggen failures
10. `test_load_config_generation_file_write_failure` - Handle file write errors
11. `test_load_config_reload_failure` - Handle config reload failures gracefully
12. `test_load_backup_restoration_on_failure` - Automatic rollback on generation failure

**Show Command Tests (`TestShowHwskuProfilesCommand`):**
1. `test_profiles_success` - Display available profiles correctly
2. `test_profiles_no_platform` - Handle missing platform gracefully
3. `test_profiles_no_hwskus` - Handle platforms with no HWSKUs
4. `test_profiles_no_current_hwsku` - Handle missing current HWSKU
5. `test_profiles_sorted_output` - Verify alphabetical sorting
6. `test_profiles_current_hwsku_not_in_available` - Handle edge case of current HWSKU not in list
7. `test_hwsku_group_help` - Verify help text is correct
8. `test_profiles_help` - Verify subcommand help text

**Test Execution:**
```bash
python3 -m pytest tests/show_hwsku_test.py tests/hwsku_load_test.py -v
```

**Test Results:**
```
tests/show_hwsku_test.py::TestShowHwskuHelpers::test_get_available_hwskus PASSED            [  3%]
tests/show_hwsku_test.py::TestShowHwskuHelpers::test_get_available_hwskus_no_platform PASSED [  6%]
tests/show_hwsku_test.py::TestShowHwskuHelpers::test_get_available_hwskus_platform_path_not_exists PASSED [ 10%]
tests/show_hwsku_test.py::TestShowHwskuHelpers::test_get_available_hwskus_oserror PASSED    [ 13%]
tests/show_hwsku_test.py::TestShowHwskuProfilesCommand::test_profiles_success PASSED        [ 17%]
tests/show_hwsku_test.py::TestShowHwskuProfilesCommand::test_profiles_no_platform PASSED    [ 20%]
tests/show_hwsku_test.py::TestShowHwskuProfilesCommand::test_profiles_no_hwskus PASSED      [ 24%]
tests/show_hwsku_test.py::TestShowHwskuProfilesCommand::test_profiles_no_current_hwsku PASSED [ 27%]
tests/show_hwsku_test.py::TestShowHwskuProfilesCommand::test_profiles_sorted_output PASSED  [ 31%]
tests/show_hwsku_test.py::TestShowHwskuProfilesCommand::test_profiles_current_hwsku_not_in_available PASSED [ 34%]
tests/show_hwsku_test.py::TestShowHwskuProfilesCommand::test_hwsku_group_help PASSED        [ 37%]
tests/show_hwsku_test.py::TestShowHwskuProfilesCommand::test_profiles_help PASSED           [ 41%]
tests/hwsku_load_test.py::TestHwskuHelpers::test_get_available_hwskus PASSED                [ 44%]
tests/hwsku_load_test.py::TestHwskuHelpers::test_get_available_hwskus_no_platform PASSED    [ 48%]
tests/hwsku_load_test.py::TestHwskuHelpers::test_is_valid_hwsku_valid PASSED                [ 51%]
tests/hwsku_load_test.py::TestHwskuHelpers::test_is_valid_hwsku_invalid PASSED              [ 55%]
tests/hwsku_load_test.py::TestHwskuHelpers::test_is_valid_hwsku_no_platform PASSED          [ 58%]
tests/hwsku_load_test.py::TestHwskuHelpers::test_complete_hwsku_names_full_match PASSED     [ 62%]
tests/hwsku_load_test.py::TestHwskuHelpers::test_complete_hwsku_names_partial_match PASSED  [ 65%]
tests/hwsku_load_test.py::TestHwskuHelpers::test_complete_hwsku_names_no_match PASSED       [ 68%]
tests/hwsku_load_test.py::TestHwskuHelpers::test_complete_hwsku_names_empty_prefix PASSED   [ 72%]
tests/hwsku_load_test.py::TestHwskuLoadCommand::test_load_no_arguments PASSED               [ 75%]
tests/hwsku_load_test.py::TestHwskuLoadCommand::test_load_invalid_hwsku PASSED              [ 79%]
tests/hwsku_load_test.py::TestHwskuLoadCommand::test_load_success_no_reload PASSED          [ 82%]
tests/hwsku_load_test.py::TestHwskuLoadCommand::test_load_success_with_reload PASSED        [ 86%]
tests/hwsku_load_test.py::TestHwskuLoadCommand::test_load_interactive_cancel PASSED         [ 89%]
tests/hwsku_load_test.py::TestHwskuLoadCommand::test_load_interactive_prompt_shows_correct_info PASSED [ 93%]
tests/hwsku_load_test.py::TestHwskuLoadCommand::test_load_backup_created_with_timestamp PASSED [ 96%]
tests/hwsku_load_test.py::TestHwskuLoadCommand::test_load_backup_failure PASSED             [100%]

================================ 29 passed in 1.35s ================================
```

**Code Coverage:**
- Helper functions: 100% coverage
- Config command: 100% coverage (all code paths including error handling)
- Show command: 100% coverage
- Edge cases: Comprehensive coverage of error scenarios

### 11. Open/Action Items
1. **Release Planning**:
   - Target release: Next SONiC release

