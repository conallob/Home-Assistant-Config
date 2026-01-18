# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Home Assistant configuration repository. Home Assistant uses YAML-based configuration files to define automations, integrations, sensors, templates, and more. The configuration follows a split-configuration pattern where different types of entities are organized into separate files and directories.

## Configuration Architecture

### Main Configuration Structure

The `configuration.yaml` file serves as the entry point and uses `!include` and `!include_dir_*` directives to load configuration from organized directories:

- **Automations**: Two types are used
  - `automation ui: !include automations.yaml` - GUI-maintained automations in a single file
  - `automation manual: !include_dir_named /config/automation` - Manually coded automations in separate files (currently empty directory)
- **Templates**: `!include_dir_list /config/template` - Template entities loaded from individual YAML files
- **Sensors**: `!include_dir_list /config/sensor` - Sensor definitions loaded from individual YAML files
- **REST Commands**: `!include_dir_named /config/rest_command` - RESTful command definitions for external API calls
- **Scenes, Scripts, Groups**: Each has a dedicated YAML file (`scenes.yaml`, `scripts.yaml`, `groups.yaml`)

### Key Directories

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

Home Assistant doesn't provide a standalone CLI validator. To validate configuration changes:

1. Use the Home Assistant UI: Developer Tools → YAML → "Check Configuration"
2. Restart Home Assistant to apply changes
3. Monitor `home-assistant.log` files for errors

### Working with ESPHome

ESPHome devices have their own configuration workflow:

```bash
# Validate an ESPHome configuration
esphome config esphome/device-name.yaml

# Compile firmware
esphome compile esphome/device-name.yaml

# Upload firmware over-the-air
esphome upload esphome/device-name.yaml

# View device logs
esphome logs esphome/device-name.yaml
```

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
