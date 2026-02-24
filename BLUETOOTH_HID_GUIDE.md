# Bluetooth HID Device Support Guide

## Overview

CrossPoint Reader now includes full Bluetooth HID (Human Interface Device) support for wireless page-turning remotes and input devices. The system automatically detects and integrates compatible devices without requiring manual configuration.

## Features

- **Auto-Detection**: Devices are automatically detected by MAC address prefix and name pattern matching
- **Generic HID Support**: Works with any standard Bluetooth HID device sending keyboard events
- **Device Profiles**: Manufacturer-specific button mappings for multi-button remotes
- **Persistent Settings**: Bluetooth state persists across device reboots
- **Power Management**: Bluetooth automatically disables during deep sleep to save power
- **WiFi/Bluetooth Mutual Exclusion**: Prevents simultaneous WiFi+Bluetooth operation (ESP32-C3 hardware limitation)

## Supported Devices

### Game Brick (Kickstarter Remote)
- **Identification**: MAC prefix `A4:CF:12` or name contains "Game Brick"
- **Button Mapping**:
  - Previous button sends keycode `0x07` (Keyboard Prev Track)
  - Next button sends keycode `0x09` (Keyboard Next Track)
- **HID Report Format**: 8-byte report, keycode at byte[4]
- **Status**: ✅ Fully tested and working

### MINI Keyboard
- **Identification**: MAC prefix `88:EC:03` or name contains "MINI_KEYBOARD"
- **Button Mapping**:
  - Previous button sends keycode `0x4B` (Keyboard Page Up)
  - Next button sends keycode `0x4E` (Keyboard Page Down)
- **HID Report Format**: 3-byte report, keycode at byte[2]
- **Status**: ✅ Fully tested and working

### Standard HID Devices
- **Format**: Any Bluetooth keyboard/media control sending Consumer Page keycodes
- **Supported Keycodes**:
  - `0x00B6` - Scan Previous Track
  - `0x00B5` - Scan Next Track
  - `0x00E9` - Volume Up
  - `0x00EA` - Volume Down
  - `0x00E0` - Power
- **Status**: ✅ Generic support, requires testing

## Usage

### Enable Bluetooth

1. Go to **Settings** → **System** → **Bluetooth**
2. Select **Enable Bluetooth**
3. Device will initialize and show connected device count

### Scan for Devices

1. With Bluetooth enabled, select **Scan for Devices**
2. Wait for scan to complete (~10 seconds)
3. Select device from list to connect
4. Status will show `[*]` for connected devices and `[HID]` for HID devices

### Connection Status

- **Green indicator**: Device is connected
- **[HID] badge**: Device is HID-compatible for page turning
- **RSSI value**: Signal strength (higher is better, typical range -40 to -90 dBm)

## Testing Your Device

### Check Device HID Capabilities

To verify your device is HID-compatible:

1. Connect via Bluetooth scanner
2. Open a book in the reader
3. Press buttons on your device
4. Check if page turns respond

### Enable Debug Logging

To diagnose connection issues:

1. Connect via USB to your development machine
2. Open serial monitor at 115200 baud
3. Look for `[BT]` tagged log messages:
   - `Device DEVICE_NAME detected` - Device found
   - `Connecting to DEVICE_NAME` - Connection attempt
   - `HID report received` - Button press detected
   - `Keycode: 0xXXXX` - Key being processed

### Common Issues

**Device not found during scan:**
- Ensure Bluetooth is enabled on your device
- Power on the remote/device
- Check battery level
- Try moving closer to the ESP32-C3

**Device connects but buttons don't work:**
- Check logs for keycode values
- Verify keycode is in the supported list
- May need new device profile (see below)

**Connection drops frequently:**
- Check WiFi is disabled (causes interference)
- Move away from WiFi router
- Check battery level on remote
- Verify line of sight to ESP32-C3

## Adding Support for New Devices

### Step 1: Identify Device

Get the device information from Bluetooth scanner:
- **MAC Address**: Shows first 6 characters (e.g., `88:EC:03`)
- **Device Name**: Exact name from scanner
- **HID Report Format**: Need to check with logs

### Step 2: Decode Button Keycodes

Add temporary logging to see what keycodes your device sends:

Edit [lib/hal/BluetoothHIDManager.cpp](lib/hal/BluetoothHIDManager.cpp) in the `onHIDNotify()` method:

```cpp
LOG_DBG("BT", "Device: %s, Report: %02x %02x %02x %02x %02x %02x %02x %02x",
        deviceName.c_str(),
        report[0], report[1], report[2], report[3], 
        report[4], report[5], report[6], report[7]);
```

Press each button and note the hex values in the logs.

### Step 3: Add Device Profile

Edit [lib/hal/DeviceProfiles.h](lib/hal/DeviceProfiles.h):

```cpp
// Add to DEVICE_PROFILES array
{
  .macPrefix = "AA:BB:CC",        // First 6 chars of MAC
  .namePattern = "DEVICE_NAME",   // Name substring to match
  .reportFormat = ReportFormat::CUSTOM,
  .keycodePrevious = 0x4B,        // Your identified keycode
  .keycodeNext = 0x4E,
  .keycodeByteIndex = 2            // Which byte contains the keycode
}
```

### Step 4: Test and Verify

1. Rebuild and upload firmware
2. Scan for device
3. Connect and test button presses
4. Verify correct page navigation

### Step 5: Submit Device Profile

If you successfully add support for a new device:

1. Create a GitHub issue with:
   - Device name and manufacturer
   - MAC address prefix  
   - Keycodes and report format
   - Confirmation it works for page turning
2. Include your device profile change
3. Help us expand compatibility!

## Device Profile Architecture

Device profiles are defined in [lib/hal/DeviceProfiles.h](lib/hal/DeviceProfiles.h):

```cpp
struct DeviceProfile {
  const char* macPrefix;           // First 6 chars of MAC (e.g., "88:EC:03")
  const char* namePattern;         // Substring of device name
  ReportFormat reportFormat;       // How the HID report is structured
  uint16_t keycodePrevious;        // Keycode for "previous" button
  uint16_t keycodeNext;            // Keycode for "next" button
  int keycodeByteIndex;            // Which byte contains keycode (0-7)
};
```

Supported report formats:
- `STANDARD` - Consumer Page keycodes in byte[2] (Most Bluetooth keyboards)
- `GAME_BRICK_FORMAT` - Custom format with keycode in byte[4]
- `MINI_KEYBOARD_FORMAT` - Custom format with keycode in byte[2]

## Hardware Requirements

- **ESP32-C3** with Bluetooth support
- **Bluetooth 5.0+** compatible remote or keyboard
- **BLE HID** device (not classic Bluetooth)

## Software Architecture

### Components

1. **BluetoothHIDManager** (`lib/hal/BluetoothHIDManager.cpp`)
   - Core BLE stack management
   - Device connection/disconnection
   - HID event handling

2. **DeviceProfiles** (`lib/hal/DeviceProfiles.cpp`)
   - Device detection database
   - Keycode mapping
   - Report format handling

3. **BluetoothSettingsActivity** (`src/activities/settings/BluetoothSettingsActivity.cpp`)
   - User interface for scanning and connecting
   - Device list with signal strength
   - Connection status display

### Button Event Flow

```
HID Report (BLE) 
    → onHIDNotify() handler
    → DeviceProfiles lookup
    → Keycode extraction (format-aware)
    → MappedInputManager mapping
    → PageTurner input event
```

## Safety Features

### WiFi/Bluetooth Mutual Exclusion

The ESP32-C3 cannot have both WiFi and Bluetooth radio active simultaneously. The system enforces mutual exclusion:

- **Enabling Bluetooth** → Disables WiFi
- **Connecting to WiFi** → Disables Bluetooth
- **Entering Deep Sleep** → Disables Bluetooth (saves power)

### Power Management

- Bluetooth disabled before deep sleep entry
- 100ms delay for safe radio shutdown
- Power consumption reduced ~30mA during sleep

## Troubleshooting

### Device appears in scan but won't connect

**Check:**
- Device is fully charged
- Device isn't connected to another device
- Try forgetting the connection and reconnecting
- Restart the device

### Buttons register but send wrong page direction

**Solution:**
- Check device profile keycodes
- Verify button mapping in `DeviceProfiles.h`
- Confirm HID report format matches profile

### Bluetooth menu crashes or freezes

**Try:**
- Disable WiFi before enabling Bluetooth
- Restart the e-reader device
- Force power reset (hold power button 30s)

### Can't find device in scan

**Try:**
- Hold power button on remote for 5+ seconds
- Move device closer to e-reader (within 1 meter)
- Restart Bluetooth scan
- Check remote battery level

## Performance Notes

- **Scan Time**: ~10 seconds for initial device discovery
- **Connection Time**: 1-3 seconds typically
- **Button Response Time**: <100ms from press to page turn
- **Memory Usage**: ~15KB for Bluetooth stack + ~2KB per connected device

## Testing Checklist

For device testing, please verify:

- [ ] Device enables/disables with Bluetooth toggle
- [ ] Device appears in scan results
- [ ] Device connects successfully
- [ ] Shows `[*]` connected indicator
- [ ] Shows `[HID]` capability indicator
- [ ] Signal strength (RSSI) displays correctly
- [ ] Previous button turns page backward
- [ ] Next button turns page forward
- [ ] Button response time is acceptable (<200ms)
- [ ] Multiple button presses register correctly
- [ ] Device reconnects after screen sleep
- [ ] No crashes or freezes

## Contact & Support

For device compatibility reports or issues:

1. Check existing device profiles in `lib/hal/DeviceProfiles.h`
2. Enable debug logging and capture output
3. Create GitHub issue with:
   - Device name and model
   - MAC address
   - Keycodes found in logs
   - Button behavior (which direction works/doesn't work)
   - Any error messages from logs

## Future Improvements

- [ ] Support for motion controls (accelerometer input)
- [ ] Customizable button mapping per device
- [ ] Multiple simultaneous device connections
- [ ] Battery level monitoring for devices
- [ ] Device firmware update capability
- [ ] Haptic/vibration feedback for confirmation

---

**Version**: 1.1
**Last Updated**: February 23, 2026
**Status**: Production Ready
