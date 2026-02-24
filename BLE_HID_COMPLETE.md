# BLE HID Implementation Complete

## Summary

Successfully implemented complete Bluetooth Low Energy (BLE) HID keyboard support for Crosspoint e-reader using NimBLE 1.4.3 on ESP32-C3.

**Branch:** `bluetooth-1.1-fresh`  
**Base:** Crosspoint 1.1.0 tag  
**Library:** NimBLE-Arduino 1.4.3  
**Binary Size:** 4.67MB (71.2% flash)  
**RAM Usage:** 109KB (33.4%)

## Features Implemented

### 1. BLE Device Scanning
- **Scan Duration:** Configurable (default: 10 seconds)
- **Active Scanning:** Enabled for better device discovery
- **HID Detection:** Automatically identifies devices advertising HID service (UUID 0x1812)
- **Device Information:** Displays name, address, RSSI, and HID capability
- **Duplicate Filtering:** Prevents duplicate devices in list

### 2. Device Connection
- **HID Service Discovery:** Finds HID service (0x1812) on connected device
- **Characteristic Discovery:** Locates HID Report characteristic (0x2A4D)
- **Notification Subscription:** Subscribes to HID report notifications
- **Connection Tracking:** Maintains list of connected devices
- **Error Handling:** Comprehensive error messages for connection failures

### 3. HID Input Handling
- **Report Parsing:** Standard HID keyboard report format
  - Byte 0: Modifier keys (Ctrl, Shift, Alt, etc.)
  - Byte 1: Reserved
  - Bytes 2-7: Up to 6 simultaneous key codes
- **Input Callback:** Registered callback for HID input events
- **Keycode Format:** Combined format (modifier << 8 | keycode)

### 4. User Interface
- **Main Menu:**
  - Enable/Disable Bluetooth toggle
  - Scan for Devices option
  - Connection status display
  - Connected device count
  - Error/status messages
  
- **Device List View:**
  - Shows all discovered devices
  - Highlights HID-capable devices with [HID] tag
  - Shows connected devices with [*] indicator
  - Displays RSSI signal strength
  - Shows device address
  - Navigation: Up/Down to select, OK to connect, Back to return

### 5. State Management
- **Cleanup on Disable:** Properly disconnects all devices and deinitializes BLE stack
- **Scan Cancellation:** Can cancel active scan by pressing Back
- **Connection Management:** Tracks multiple connected devices
- **Memory Management:** Proper cleanup of NimBLE client objects

## Implementation Details

### File Structure
```
lib/hal/BluetoothHIDManager.h         # BLE manager interface
lib/hal/BluetoothHIDManager.cpp       # BLE manager implementation
src/activities/settings/BluetoothSettingsActivity.h   # UI header
src/activities/settings/BluetoothSettingsActivity.cpp # UI implementation
```

### Key Components

#### BluetoothHIDManager
- **Singleton Pattern:** Single instance for BLE management
- **Scan Callbacks:** `ScanCallbacks` class extends `NimBLEAdvertisedDeviceCallbacks`
- **Device Discovery:** `onScanResult()` processes discovered devices
- **HID Notification:** Static callback `onHIDNotify()` processes input reports
- **Report Parser:** `parseHIDReport()` extracts modifier and keycode

#### BluetoothSettingsActivity
- **View Modes:** `MAIN_MENU` and `DEVICE_LIST`
- **Input Handling:** Separate handlers for each view mode
- **Rendering:** Separate render methods for each view
- **State Tracking:** Manages scan timing and device selection

### NimBLE 1.4.3 API Compatibility

**Callback Class:**
- Use `NimBLEAdvertisedDeviceCallbacks` (not `NimBLEScanCallbacks`)
- Only implement `onResult()` method (no `onScanEnd()` in 1.4.3)

**Scan API:**
- `pScan->start(seconds)` is blocking in 1.4.3
- Scan completes after specified duration
- Callbacks fire during scan for each discovered device

**Connection API:**
- `NimBLEDevice::createClient()` creates client
- `pClient->connect(address)` initiates connection
- `pClient->getService(uuid)` discovers services
- `pService->getCharacteristic(uuid)` gets characteristic
- `pChar->subscribe(notify, callback)` subscribes to notifications

## Usage Instructions

### 1. Enable Bluetooth
1. Navigate to **Settings → System → Bluetooth**
2. Select **Enable Bluetooth**
3. Wait for confirmation message "Bluetooth enabled"

### 2. Scan for Devices
1. Select **Scan for Devices**
2. Wait 10 seconds for scan to complete
3. View shows discovered devices with HID indicators

### 3. Connect to Device
1. In device list, use Up/Down to select device
2. Press OK to connect
3. Wait for "Connected to [device name]" message
4. Device shows [*] indicator when connected

### 4. Disconnect/Disable
1. Press Back to return to main menu
2. Select **Disable Bluetooth** to disconnect all devices

## Next Steps

### Input Mapping (Not Yet Implemented)
The HID input callback is registered but not yet connected to the navigation system. To complete integration:

1. **Map HID Keycodes to MappedInputManager:**
   ```cpp
   btMgr->setInputCallback([](uint16_t keycode) {
     uint8_t modifier = (keycode >> 8) & 0xFF;
     uint8_t key = keycode & 0xFF;
     
     // Map to MappedInputManager buttons
     // Example: Arrow keys, Enter, Escape, etc.
   });
   ```

2. **Add Keycode Reference:**
   - Standard HID keyboard scan codes (see `docs/ble-keycode-reference.md`)
   - Game Brick T-mode specific mappings

3. **Test Input Events:**
   - Verify button presses register in navigation
   - Test page turning, menu navigation, etc.
   - Handle key press and release events

### State Persistence (Stubs Present)
Currently `saveState()` and `loadState()` are stubbed. To implement:

1. **Save Paired Devices:**
   - Store bonded device addresses to file
   - Save connection preferences
   
2. **Auto-Reconnect:**
   - On BLE enable, attempt reconnect to known devices
   - Handle failed reconnections gracefully

## Testing Checklist

- [x] Bluetooth enables without crashing
- [x] Scan discovers BLE devices
- [x] HID service detection works
- [x] Device connection successful
- [x] HID reports received via notifications
- [x] UI shows device list correctly
- [x] Navigation between views works
- [x] Proper cleanup on disable
- [ ] Test with actual Game Brick controller
- [ ] Verify input mapping
- [ ] Test auto-reconnect (after state persistence)
- [ ] Test multiple device connections
- [ ] Test connection stability over time

## Known Issues

1. **Blocking Scan:** The scan blocks the main loop for 10 seconds. This may cause UI freezes during scan. Consider implementing async scan if this becomes problematic.

2. **No Input Mapping:** HID reports are parsed but not yet mapped to navigation events. This requires integration with `MappedInputManager`.

3. **No State Persistence:** Paired devices are not saved between reboots.

4. **Single Report Characteristic:** Currently only subscribes to one HID report characteristic. Some devices may have multiple.

## Build Information

```
Firmware Size: 4,667,674 bytes (71.2% of 6,553,600 bytes)
RAM Usage: 109,364 bytes (33.4% of 327,680 bytes)
Build Time: ~15 seconds
Upload Time: ~30 seconds
Platform: espressif32 @ 6.12.0
Framework: Arduino
Toolchain: riscv32-esp @ 8.4.0
```

## References

- **NimBLE Documentation:** [NimBLE-Arduino GitHub](https://github.com/h2zero/NimBLE-Arduino)
- **HID Specification:** [USB HID Usage Tables](https://usb.org/sites/default/files/hut1_4.pdf)
- **Crosspoint:** [Crosspoint GitHub](https://github.com/jonmadison/crosspoint)
- **Game Brick Controller:** T-mode Bluetooth keyboard functionality

## Commit History

1. **Initial BLE Foundation** - Added BLE enable/disable with NimBLE 1.4.3
2. **Complete BLE HID** - Scan, connect, HID discovery, input parsing, full UI

---

**Status:** ✅ Core implementation complete, ready for input mapping and testing with hardware
