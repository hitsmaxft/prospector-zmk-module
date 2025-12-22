# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **Prospector ZMK Module** - a custom ZMK (ZMK Keyboard Firmware) module that provides advanced status monitoring capabilities for keyboards. The module has two main modes:

1. **Status Advertisement Mode**: Keyboards broadcast their status via BLE
2. **Scanner Mode**: Standalone display devices receive and show status from multiple keyboards

## Build Commands

The module uses the standard ZMK/Zephyr build system. Building is typically done from the main ZMK config repository that includes this module, not from this module directory directly.

From the main ZMK config (usually `~/projects/keyboards/zmk-config/`):
```bash
# Build scanner device (most common)
just build projector_dongle

# With error suppression for cleaner output
just build projector_dongle >/dev/null

# Build with west directly
west build -s zmk/app -b seeeduino_xiao_ble -- -DSHIELD=prospector_scanner

# Build dongle/adapter variant
west build -s zmk/app -b seeeduino_xiao_ble -- -DSHIELD=prospector_adapter
```

The `just build {artifact-name}` command references artifacts from `build.yaml` in the main config repository.

## Architecture and Key Components

### Module Structure
```
prospector-zmk-module/
├── src/                              # Core module source
│   ├── status_advertisement.c        # BLE status broadcasting for keyboards
│   └── status_scanner.c             # BLE advertisement reception for scanners
├── boards/shields/prospector_scanner/ # Scanner device implementation
│   ├── src/                         # Scanner-specific widgets and display logic
│   │   ├── scanner_display.c        # Main display controller and LVGL setup
│   │   ├── scanner_main.c           # Message queue system for thread-safe operations
│   │   ├── *_widget.c               # YADS-style status widgets
│   │   ├── touch_handler.c          # CST816S touch panel support
│   │   └── fonts/                   # NerdFont symbols for modifiers
│   ├── prospector_scanner.overlay   # Device tree hardware configuration
│   └── prospector_scanner.conf      # Default Kconfig settings
├── drivers/display/                 # Custom ST7789V display driver
├── modules/lvgl/                    # LVGL graphics library integration
└── include/zmk/                     # Public API headers
```

### Status Advertisement Protocol
The module implements a custom 26-byte BLE advertisement protocol:
- Bytes 0-3: Header (Manufacturer ID + Service UUID)
- Bytes 4-25: Status data (battery, layer, modifiers, connection info, etc.)
- Supports split keyboards with peripheral battery reporting
- Activity-based broadcasting intervals (fast during typing, slow when idle)

### Scanner Display System
- **YADS-inspired UI**: Professional status widgets with color coding
- **Multi-keyboard support**: Track up to 5 keyboards simultaneously
- **Touch interface**: Swipe navigation between settings screens (optional)
- **Auto-brightness**: APDS9960 ambient light sensor support (optional)
- **Battery operation**: Supports battery-powered operation with power management

## Hardware Configuration

### Supported Hardware
- **Primary**: Seeeduino XIAO BLE (nRF52840)
- **Display**: ST7789V 240x280 round LCD (1.69")
- **Touch**: CST816S capacitive touch controller (optional)
- **Sensor**: APDS9960 ambient light sensor (optional)

### Key Device Tree Configuration
The `.overlay` files define hardware connections:
- Display: SPI connection with CS/DC/RST pins
- Touch panel: I2C connection with interrupt pin
- Ambient light sensor: I2C connection (polling mode, no interrupt needed)

For 7789 TFT module compatibility: CS, DC, RST should all be active_high.

## Configuration System

The module uses Zephyr Kconfig extensively. Key configuration areas:

### Advertisement (Keyboard Side)
```kconfig
CONFIG_ZMK_STATUS_ADVERTISEMENT=y
CONFIG_ZMK_STATUS_ADV_KEYBOARD_NAME="MyKeyboard"
CONFIG_ZMK_STATUS_ADV_ACTIVITY_BASED=y  # Power optimization
```

### Scanner Mode
```kconfig
CONFIG_PROSPECTOR_MODE_SCANNER=y
CONFIG_PROSPECTOR_TOUCH_ENABLED=y        # Enable touch interface
CONFIG_PROSPECTOR_USE_AMBIENT_LIGHT_SENSOR=y  # Enable ALS (only if hardware present)
CONFIG_PROSPECTOR_BATTERY_SUPPORT=y      # Enable battery operation
```

### Safety Features
- Auto-detection of missing hardware (ALS, battery, touch)
- Graceful fallbacks when optional components are missing
- Thread-safe message queue system for LVGL operations

## Development Notes

### Threading Architecture
- Main loop: Timer-driven UI updates and message processing
- Touch handler: Interrupt-driven gesture detection with event posting
- Scanner: BLE advertisement reception with thread-safe data updates
- LVGL: All graphics operations must happen on main thread via message queue

### Widget System
Widgets follow YADS design patterns:
- Color-coded status indicators
- NerdFont symbols for professional appearance
- Responsive layout adapting to available data
- Touch-friendly sizing and positioning

### Power Management
- Activity-based advertisement intervals (5Hz active, ~0.03Hz idle)
- Display dimming based on keyboard activity
- Ambient light sensor for auto-brightness
- Battery status monitoring and display

### Common Development Tasks
- Adding new status widgets: Follow existing widget patterns in `boards/shields/prospector_scanner/src/`
- Extending BLE protocol: Modify both `status_advertisement.c` and `status_scanner.c`
- Display customization: Modify LVGL code in `scanner_display.c` and widget files
- Hardware support: Update device tree overlays and Kconfig options

## Dependencies and Integration

This module integrates with:
- **ZMK Core**: Uses ZMK's BLE, battery, and event systems
- **Zephyr RTOS**: Built on Zephyr's device model and Kconfig
- **LVGL**: Graphics library for display rendering
- **Custom Drivers**: ST7789V display driver included in module

The module is designed to be included in ZMK configs via `west.yml` manifest and activated through shield selection.