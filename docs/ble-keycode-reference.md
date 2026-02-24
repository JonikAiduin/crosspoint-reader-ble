# BLE HID Keycode Reference

## Standard HID Keyboard Scan Codes

This document provides the HID usage codes for implementing input mapping from Bluetooth HID keyboards to Crosspoint navigation events.

### Modifier Keys (Byte 0)
```
Bit 0: Left Control   (0x01)
Bit 1: Left Shift     (0x02)
Bit 2: Left Alt       (0x04)
Bit 3: Left GUI       (0x08)
Bit 4: Right Control  (0x10)
Bit 5: Right Shift    (0x20)
Bit 6: Right Alt      (0x40)
Bit 7: Right GUI      (0x80)
```

### Navigation Keys (Byte 2-7)
```
Arrow Up        0x52 (82)
Arrow Down      0x51 (81)
Arrow Left      0x50 (80)
Arrow Right     0x4F (79)
Page Up         0x4B (75)
Page Down       0x4E (78)
Home            0x4A (74)
End             0x4D (77)
```

### Action Keys
```
Enter/Return    0x28 (40)
Escape          0x29 (41)
Backspace       0x2A (42)
Tab             0x2B (43)
Space           0x2C (44)
Delete          0x4C (76)
Insert          0x49 (73)
```

### Function Keys
```
F1              0x3A (58)
F2              0x3B (59)
F3              0x3C (60)
F4              0x3D (61)
F5              0x3E (62)
F6              0x3F (63)
F7              0x40 (64)
F8              0x41 (65)
F9              0x42 (66)
F10             0x43 (67)
F11             0x44 (68)
F12             0x45 (69)
```

### Letter Keys
```
A               0x04 (4)
B               0x05 (5)
...
Z               0x1D (29)
```

### Number Keys
```
1               0x1E (30)
2               0x1F (31)
...
9               0x26 (38)
0               0x27 (39)
```

## Crosspoint MappedInputManager Buttons

Based on the codebase analysis, these are the available navigation buttons:

```cpp
enum class Button {
  Up,
  Down,
  Left,
  Right,
  Confirm,
  Back,
  Menu,
  Power
};
```

## Suggested Mapping

### Basic Navigation
```cpp
HID Keycode           → MappedInput Button
--------------------------------------------
0x52 (Arrow Up)       → Button::Up
0x51 (Arrow Down)     → Button::Down
0x50 (Arrow Left)     → Button::Left
0x4F (Arrow Right)    → Button::Right
0x28 (Enter)          → Button::Confirm
0x29 (Escape)         → Button::Back
```

### Alternative Mappings
```cpp
'W' key (0x1A)        → Button::Up       (WASD games)
'S' key (0x16)        → Button::Down
'A' key (0x04)        → Button::Left
'D' key (0x07)        → Button::Right
Space (0x2C)          → Button::Confirm  (common in games)
```

### Page Turner Mappings
```cpp
0x4E (Page Down)      → Button::Down     (next page)
0x4B (Page Up)        → Button::Up       (previous page)
0x50 (Arrow Left)     → Button::Left     (previous chapter)
0x4F (Arrow Right)    → Button::Right    (next chapter)
```

### Game Brick T-Mode Expected Keys
The Game Brick controller in T-mode (keyboard mode) likely sends:
- D-pad as arrow keys (0x50-0x52)
- A button as Enter (0x28) or Space (0x2C)
- B button as Escape (0x29) or Backspace (0x2A)
- Menu/Select buttons as Tab (0x2B) or other function keys

**Note:** Actual Game Brick mapping should be verified by testing with the controller.

## Implementation Example

```cpp
// In main.cpp or initialization code
btMgr->setInputCallback([](uint16_t keycode) {
  uint8_t modifier = (keycode >> 8) & 0xFF;
  uint8_t key = keycode & 0xFF;
  
  // Ignore key release (all zeros)
  if (key == 0) return;
  
  // Map HID codes to buttons
  MappedInputManager::Button btn;
  bool hasMapping = true;
  
  switch (key) {
    case 0x52: btn = MappedInputManager::Button::Up; break;
    case 0x51: btn = MappedInputManager::Button::Down; break;
    case 0x50: btn = MappedInputManager::Button::Left; break;
    case 0x4F: btn = MappedInputManager::Button::Right; break;
    case 0x28: // Enter
    case 0x2C: // Space
      btn = MappedInputManager::Button::Confirm; break;
    case 0x29: // Escape
      btn = MappedInputManager::Button::Back; break;
    default:
      hasMapping = false;
      LOG_DBG("BT", "Unmapped HID key: 0x%02X", key);
  }
  
  if (hasMapping) {
    // Inject into MappedInputManager
    // This requires adding a method to inject external input
    mappedInput.injectButtonPress(btn);
  }
});
```

## Testing Process

1. **Enable Bluetooth** and connect Game Brick controller
2. **Log all keycodes** received from HID reports
3. **Press each button** on Game Brick and note the keycode
4. **Create mapping table** based on observed keycodes
5. **Implement mapping** in input callback
6. **Test navigation** in Crosspoint UI

## HID Report Format

Standard HID keyboard report (8 bytes):
```
Byte 0: Modifier keys bitmap
Byte 1: Reserved (usually 0x00)
Byte 2: Key 1 (0x00 if no key)
Byte 3: Key 2 (0x00 if no key)
Byte 4: Key 3 (0x00 if no key)
Byte 5: Key 4 (0x00 if no key)
Byte 6: Key 5 (0x00 if no key)
Byte 7: Key 6 (0x00 if no key)
```

**Example:** Pressing 'A' key
```
[0x00, 0x00, 0x04, 0x00, 0x00, 0x00, 0x00, 0x00]
```

**Example:** Pressing Shift+A
```
[0x02, 0x00, 0x04, 0x00, 0x00, 0x00, 0x00, 0x00]
```

**Example:** Pressing Arrow Up
```
[0x00, 0x00, 0x52, 0x00, 0x00, 0x00, 0x00, 0x00]
```

**Example:** Releasing all keys
```
[0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00]
```

## References

- [USB HID Usage Tables v1.4](https://usb.org/sites/default/files/hut1_4.pdf) - Official USB HID specification
- [HID Keyboard Scan Codes](https://www.usb.org/sites/default/files/documents/hut1_12v2.pdf) - Keyboard/Keypad page
- [Arduino BLE HID](https://github.com/T-vK/ESP32-BLE-Keyboard) - Example ESP32 BLE keyboard library

---

**Next Step:** Test with Game Brick controller to determine actual keycode mappings, then implement the input injection into MappedInputManager.
