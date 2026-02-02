# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Home Assistant configuration repository. Home Assistant uses YAML-based configuration files to define automations, integrations, sensors, templates, and more. The configuration follows a split-configuration pattern where different types of entities are organized into separate files and directories.

## Configuration Architecture

### Main Configuration Structure

The `configuration.yaml` file serves as the entry point and uses `!include` and `!include_dir_*` directives to load configuration from organized directories:

- **Automations**: Two types are used with a prototyping workflow
  - `automation ui: !include automations.yaml` - **Prototyping area** for automations being developed via the Home Assistant UI. These may deviate from git as they are works-in-progress.
  - `automation manual: !include_dir_named /config/automation` - **Production automations** stored as individual YAML files (one automation per file). Once an automation has been prototyped and tested via the UI, it should be moved here.
- **Templates**: `!include_dir_list /config/template` - Template entities loaded from individual YAML files
- **Sensors**: `!include_dir_list /config/sensor` - Sensor definitions loaded from individual YAML files
- **REST Commands**: `!include_dir_named /config/rest_command` - RESTful command definitions for external API calls
- **Scenes, Scripts, Groups**: Each has a dedicated YAML file (`scenes.yaml`, `scripts.yaml`, `groups.yaml`)

### Key Directories

- `automation/` - Production automations stored as individual YAML files (one per automation)
- `custom_components/` - Custom integrations installed via HACS or manually (e.g., hacs, frigate, anker_solix, spook, mass, icloud3)
- `esphome/` - ESPHome device configurations for ESP32/ESP8266 devices (Bluetooth proxies, sensors, etc.)
  - `esphome/archive/` - Old/deprecated device configurations
  - `esphome/secrets.yaml` - ESPHome-specific secrets (WiFi credentials, API keys)
- `appdaemon/` - AppDaemon apps (currently only watchman for monitoring missing entities)
- `blueprints/` - Automation and script blueprints from the community
- `rest_command/` - RESTful API command definitions (e.g., Megatron Redfish commands, PVOutput uploads)
- `sensor/` - Individual sensor YAML files (e.g., Solis solar inverter)
- `template/` - Template sensor/binary sensor definitions
- `themes/` - UI theme definitions
- `zigbee2mqtt/` - Zigbee2MQTT configuration and backups
- `diyhue/` - DIY Hue emulation bridge configuration

### Secrets Management

- `secrets.yaml` - Contains sensitive credentials and API keys referenced throughout the configuration using the `!secret` directive
- Never commit `secrets.yaml` to version control (it's in `.gitignore`)
- When referencing secrets in YAML files, use: `password: !secret secret_key_name`

## Common Development Tasks

### Validating Configuration

There are multiple ways to validate Home Assistant configuration:

1. **CI/CD Pipeline**: A GitHub Actions workflow automatically validates configuration on pull requests
2. **Home Assistant UI**: Developer Tools → YAML → "Check Configuration"
3. **MCP Integration**: The live running Home Assistant system is accessible via MCP for real-time validation and interaction
4. Monitor `home-assistant.log` files for errors

### Automatic Deployment

Changes merged to the `main` branch are automatically deployed:
- A webhook triggers `git pull` on the live system when commits are merged to `main`
- Valid configuration changes take effect within seconds
- The system automatically reloads configuration where possible

### Contribution Guidelines

**Use a Pull Request for:**
- Non-trivial refactors (restructuring files, changing configuration patterns)
- New features (new automations, sensors, integrations, etc.)
- Changes that affect multiple files or components
- Any changes you want reviewed before deployment

**Direct merge to main is acceptable for:**
- Small fixes to existing configurations
- Typo corrections
- Minor adjustments to thresholds, timings, or values
- Documentation updates
- Adding or editing annotations (comments, descriptions, aliases)
- Must pass CI/CD configuration validation checks before merging

### Working with ESPHome

ESPHome devices have their own configuration workflow:

**Validation** can be done locally or via GitHub Actions:
```bash
# Validate an ESPHome configuration locally
esphome config esphome/device-name.yaml
```

**Compiling and installing firmware** should be done via the ESPHome Add-on in Home Assistant, not locally. The add-on handles OTA updates to devices on the network:
1. Open the ESPHome Add-on in Home Assistant
2. Select the device configuration
3. Click "Install" to compile and upload OTA

ESPHome secrets are stored separately in `esphome/secrets.yaml`.

### Custom Components

Custom components are typically installed and managed via HACS (Home Assistant Community Store). The `custom_components/` directory contains third-party integrations that extend Home Assistant's capabilities. Each component has its own structure with Python code, translations, and service definitions.

When modifying custom components, be aware that HACS updates will overwrite local changes unless the integration is removed from HACS management.

## Important Patterns and Conventions

### REST Commands for Redfish API

The configuration includes REST commands for managing server hardware via Redfish API (Dell server "Megatron"):
- Power control commands in `rest_command/megatron_redfish_*.yaml`
- Uses BMC credentials stored in `secrets.yaml`
- Fan control is currently disabled (`.yaml.disabled` extension)

### Energy Monitoring and Solar Integration

The system tracks electricity usage with tariff-based utility meters:
- Three tariffs: day, peak, night
- Integration with Solis solar inverter via custom component
- PVOutput.org integration for solar generation data upload
- Powercalc integration for virtual power sensors with tariff support

### Automation Workflow

Automations follow a two-stage workflow:

1. **Prototyping Stage** (`automations.yaml`)
   - New automations are created and tested via the Home Assistant UI
   - The UI writes to `automations.yaml` which may diverge from the git repository
   - This allows rapid iteration without git commits for every tweak
   - Automations here are considered works-in-progress

2. **Production Stage** (`automation/` directory)
   - Once an automation is stable and tested, move it to `automation/` as its own file
   - Use descriptive filenames with lowercase and underscores: `my_automation_name.yaml`
   - Each file contains a single automation definition
   - These files are version-controlled and deployed via git

**Moving an automation from prototype to production:**
1. Copy the automation from `automations.yaml`
2. Create a new file in `automation/` (e.g., `automation/my_new_automation.yaml`)
3. Paste the automation content (remove the leading `- ` since it's no longer a list item)
4. Delete the automation from `automations.yaml` via the Home Assistant UI
5. Commit and push the new file

### Automation Patterns

Automations use a mix of:
- Device-based triggers (specific device IDs)
- State-based triggers (entity state changes)
- Time-based triggers (time patterns, specific times)

When creating automations:
- Add a descriptive `alias` field
- Include a `description` field for complex automations
- Use `mode: single` to prevent concurrent executions (most common)

### Template Sensors

Template sensors in the `template/` directory calculate derived values:
- Net power/energy calculations (solar generation - consumption)
- Binary sensors for custom logic (e.g., school day detection)
- Tariff-related calculations for electricity pricing

### Include Directory File Structure

When using `!include_dir_list` (for `sensor/` and `template/` directories), each YAML file must contain exactly **one** sensor/entity definition. Do not put multiple sensors in the same file - this will cause validation errors like "expected a dictionary, got list".

**Correct** - One sensor per file as a dictionary (no leading dash):
```yaml
# sensor/my_sensor.yaml
platform: history_stats
name: "My Sensor"
entity_id: binary_sensor.something
state: "on"
type: time
start: "{{ now().replace(month=1, day=1, hour=0, minute=0, second=0, microsecond=0) }}"
end: "{{ now() }}"
```

**Incorrect** - Using list syntax (with dash) will fail:
```yaml
# sensor/my_sensor.yaml - THIS WILL FAIL
- platform: history_stats
  name: "My Sensor"
  ...
```

If you need multiple related sensors, create separate files (e.g., `conall_wfh_annual_days.yaml` and `ciara_wfh_annual_days.yaml`).

## Configuration Best Practices

1. **Organize by Function**: Keep related entities in their respective directories (`sensor/`, `template/`, `rest_command/`)
2. **Use Includes**: Split large configurations using `!include_dir_*` directives for maintainability
3. **Secrets Management**: Always use `!secret` for credentials, API keys, and IP addresses that should remain private
4. **Descriptive Naming**: Use clear, descriptive names for entities (e.g., `binary_sensor.schoolday` instead of `binary_sensor.bs1`)
5. **Archive Old Configs**: Move deprecated configurations to `archive/` subdirectories rather than deleting them

## Integration-Specific Notes

### ESPHome Bluetooth Proxies

Multiple ESP32-based Bluetooth proxy devices extend Bluetooth range throughout the home. Configuration pattern:
- Uses ESP-IDF framework for ESP32-C6 boards
- IPv6 enabled for modern network compatibility
- Secrets referenced from `esphome/secrets.yaml`
- OTA updates enabled with password protection

### Zigbee2MQTT

Zigbee devices are managed via Zigbee2MQTT (separate from Home Assistant core):
- Configuration in `zigbee2mqtt/configuration.yaml`
- Backup configurations are automatically created (`configuration_backup_v*.yaml`)
- Database stored in `zigbee.db` (not version controlled)

### Go2RTC

External video streaming service integration at `http://ratchet.int.taku.ie:1984` for camera feeds.

## File Naming Conventions

- Use lowercase with underscores: `my_sensor_name.yaml`
- Disabled configurations use `.disabled` extension: `automation.yaml.disabled`
- Archived configurations go in `archive/` subdirectories
- Backup files use `.bak` extension: `fitbit.bak`
