# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ckb-next is an open-source RGB driver for Corsair keyboards and mice on Linux. It replaces Corsair's proprietary CUE software with native Linux support for RGB animations, key bindings, DPI settings, and hardware profiles.

## Build System

This project uses **CMake** with out-of-source builds required.

### Quick Build Commands

```bash
# Quick install (recommended for first-time setup)
./quickinstall

# Manual build process
cmake -H. -Bbuild -DCMAKE_BUILD_TYPE=Release
cmake --build build --target all -- -j$(nproc)
sudo cmake --build build --target install

# Uninstall
sudo cmake --build build --target uninstall
```

### Common CMake Options

- `-DWITH_GUI=ON/OFF` - Build Qt GUI (default: ON)
- `-DPREFER_QT6=ON/OFF` - Prefer Qt6 over Qt5 (default: ON)
- `-DWITH_ANIMATIONS=ON/OFF` - Build animation plugins (default: ON)
- `-DDISABLE_UPDATER=ON` - Disable firmware updater (for packaging)
- `-DSAFE_INSTALL=ON` - Execute pre-install safety tasks
- `-DSAFE_UNINSTALL=ON` - Execute pre-uninstall safety tasks
- `-DUSE_XCB=ON/OFF` - Enable XCB features on Linux (default: ON)
- `-DUSE_WAYLAND=ON/OFF` - Enable Wayland support (default: ON with Qt6)

### Debug Options

- `-DDEBUG_USB_SEND=ON` - Show USB packets sent to device
- `-DDEBUG_USB_RECV=ON` - Show USB packets received via usbrecv()
- `-DDEBUG_USB_INPUT=ON` - Show USB packets from input thread
- `-DDEBUG_MUTEX=ON` - Show thread synchronization debug info
- `-DNO_FAIR_MUTEX_QUEUEING=ON` - Required when using TSAN
- `-DFPS_COUNTER=ON` - Enable FPS counters for animations
- `-DWERROR=ON` - Treat warnings as errors (CI usage)

### Build Targets

- `all` - Build daemon, GUI, and animations
- `daemon` - Build only ckb-next-daemon
- `install` - Install binaries and system files
- `uninstall` - Remove installed files
- `macos-package` - Create macOS DMG installer (macOS only)

## Architecture

### Two-Process Architecture

ckb-next consists of two main components that communicate via FIFO pipes:

1. **ckb-next-daemon** (src/daemon/)
   - Userspace driver running as root
   - Handles USB communication with devices
   - Manages hardware profiles and lighting
   - Creates device nodes in `/dev/input/ckb*`
   - Entry point: [src/daemon/main.c](src/daemon/main.c)

2. **ckb-next** (src/gui/)
   - Qt5/Qt6 GUI application (user-space)
   - Communicates with daemon via device nodes
   - Provides animation scripting interface
   - Entry point: [src/gui/main.cpp](src/gui/main.cpp)

### Communication Protocol

The daemon and GUI communicate through character device nodes created by the daemon. Commands are sent as text-based messages defined in [src/daemon/command.c](src/daemon/command.c) and [src/daemon/command.h](src/daemon/command.h).

### Device Protocol Layers

The codebase supports two USB protocol families:

**Legacy Protocol** (older devices):
- Implementation: `src/daemon/usb_legacy.c`, `src/daemon/legacykb_proto.h`
- Direct USB HID communication
- Device-specific command sets

**BRAGI Protocol** (wireless/newer devices):
- Implementation: `src/daemon/bragi_*.c`, `src/daemon/bragi_proto.h`
- Unified protocol for wireless devices and receivers
- Property-based configuration system
- Support for dongle/child device relationships

### Device Abstraction (vtable pattern)

Devices use function pointer tables (vtables) to abstract hardware differences:
- Defined in: [src/daemon/device_vtable.c](src/daemon/device_vtable.c)
- Referenced via: `kb->vtable.function_name()`
- Types: `vtable_keyboard`, `vtable_mouse`, `vtable_bragi_dongle`, etc.
- Allows unified code paths while supporting device-specific implementations

### Threading Model

The daemon uses a multi-threaded architecture with queued mutexes to ensure fair locking:

- **Main thread**: Device discovery and command processing
- **Input threads**: Per-device USB input handling
- **Macro threads**: Per-device macro execution

Critical sections use `queued_mutex_t` (fair FIFO ordering) unless `NO_FAIR_MUTEX_QUEUEING` is set:
- `devmutex[DEV_MAX]` - USB device handle and profile access
- `inputmutex[DEV_MAX]` - Key input and output FIFO access
- `macromutex[DEV_MAX]` - Macro key synchronization
- `interruptmutex[DEV_MAX]` - URB interrupt data transfer

### Animation System

Animations are **separate executables** (not shared libraries):
- Location: `src/animations/*/main.c`
- Built types: gradient, heat, invaders, life, mviz, pinwheel, pipe, rain, random, ripple, snake, wave
- Communication: Stdin (commands) / Stdout (RGB frame data)
- Each animation is a standalone process spawned by the GUI

## Device Support

### Device List

Supported devices are defined in [src/daemon/usb.c](src/daemon/usb.c) via the `models[]` array. Each entry specifies:
- Vendor ID (typically `V_CORSAIR = 0x1b1c`)
- Product ID (e.g., `P_K70_RGB = 0x1b13`)

### Adding New Device Support

1. Add product ID constant to appropriate header
2. Add device entry to `models[]` array in `usb.c`
3. Add DPI limits to `mouse_dpi_list[]` if it's a mouse
4. Map device to appropriate vtable in `device_vtable.c`
5. Test with actual hardware

### Firmware Database

Firmware versions are tracked in `FIRMWARE` (PGP-signed) and `FIRMWARE.unsigned`:
- Contains URLs to official Corsair firmware
- SHA256 checksums for validation
- Minimum ckb-next version requirements
- **Never edit `FIRMWARE` directly** - edit `FIRMWARE.unsigned` and run `./scripts/sign_firmware.sh`

## Key Source Files

### Daemon Core
- `src/daemon/main.c` - Daemon entry point, signal handling, shutdown
- `src/daemon/usb.c` - USB device discovery and initialization
- `src/daemon/device.c` - Device lifecycle management
- `src/daemon/devnode.c` - Device node creation and management
- `src/daemon/command.c` - Command parsing and execution
- `src/daemon/input.c` - Input event handling
- `src/daemon/led.c` - RGB LED control
- `src/daemon/notify.c` - Event notification system
- `src/daemon/profile.c` - Hardware profile management
- `src/daemon/firmware.c` - Firmware update functionality

### Protocol Implementations
- `src/daemon/usb_legacy.c` - Legacy USB protocol
- `src/daemon/bragi_*.c` - BRAGI wireless protocol
- `src/daemon/device_bragi.c` - BRAGI device initialization
- `src/daemon/device_keyboard.c` - Keyboard-specific logic
- `src/daemon/device_mouse.c` - Mouse-specific logic

### GUI Core
- `src/gui/main.cpp` - GUI entry point, command line parsing
- `src/gui/mainwindow.cpp` - Main application window
- `src/gui/kbmanager.cpp` - Device manager interface
- `src/gui/animscript.cpp` - Animation script execution
- `src/gui/kblight.cpp` - Lighting control widget
- `src/gui/rebindwidget.cpp` - Key rebinding interface

### Shared Libraries
- `src/libs/ckb-next/` - Shared utility code between daemon and GUI
- `src/libs/kissfft/` - FFT library for music visualizer

## Development Workflow

### Code Style

This is a C/C++ codebase:
- C11 for daemon code
- C++11/14 for GUI (Qt5/Qt6)
- Follow existing code style in each file
- Use `_Atomic` types for thread-shared state in C code

### Testing Changes

1. Build with debug options enabled
2. Run daemon manually: `sudo ./build/bin/ckb-next-daemon`
3. Run GUI in another terminal: `./build/bin/ckb-next`
4. Check logs in `/tmp/ckb-next-daemon.log` (or use `journalctl` on systemd)
5. Monitor USB traffic with debug flags if needed

### Common Development Tasks

**Run daemon in foreground (debug mode):**
```bash
sudo ./build/bin/ckb-next-daemon --foreground
```

**Test without installing:**
```bash
# Build
cmake --build build

# Daemon needs root
sudo ./build/bin/ckb-next-daemon

# GUI can run as user
./build/bin/ckb-next --background
```

**Test single animation:**
```bash
# Animations are executables that expect specific input format
./build/lib/ckb-next-animations/ckb-next-animation-name
```

**Add support for a new Corsair device:**
1. Find USB VID/PID: `lsusb | grep Corsair`
2. Add product ID to `usb.c:models[]`
3. Determine if device uses legacy or BRAGI protocol
4. Map to appropriate vtable in `device_vtable.c`

## Platform-Specific Notes

### Linux
- Requires udev rules: `linux/udev/99-ckb-next-daemon.rules`
- Service file: `linux/systemd/ckb-next-daemon.service` (systemd) or sysvinit scripts
- Installation paths: `/usr/bin/` (binaries), `/usr/libexec/ckb-next-animations/` (animations)

### macOS
- **No longer officially supported** (see issue #660)
- Historical support exists but is unmaintained
- Uses IOKit framework for USB access

## Important Implementation Details

### Device Node Structure
Each connected device gets a directory tree under `/dev/input/ckb*/`:
```
/dev/input/ckb0/          # Root controller
/dev/input/ckb1/          # First device
    cmd                   # Command pipe (write)
    notify                # Notification pipe (read)
    model
    serial
    fwversion
    profile1/
        mode1/
            ...
```

### Wireless Device Parent/Child Relationships
BRAGI wireless devices have a dongle (parent) and one or more wireless peripherals (children):
- Parent manages RF communication
- Children represent actual devices (keyboard/mouse)
- Commands route through parent to children

### Profile and Mode System
- Devices support multiple **profiles** (up to 3 hardware-stored)
- Each profile contains multiple **modes** (different lighting/binding configs)
- GUI manages software profiles that can exceed hardware limits

### Color Data Format
RGB data is sent to hardware as byte arrays:
- Format depends on device keymap
- Usually: one RGB triplet per LED
- Order matches physical key layout defined in keymap headers

## Troubleshooting

**Daemon won't start:**
- Check if another instance is running: `ps aux | grep ckb-next-daemon`
- Check PID file: `cat /run/ckb-next-daemon.pid`
- Look at logs: `journalctl -u ckb-next-daemon` or `/tmp/ckb-next-daemon.log`

**Device not detected:**
- Verify USB connection: `lsusb | grep Corsair`
- Check udev rules are installed: `ls -l /lib/udev/rules.d/99-ckb-next-daemon.rules`
- Reload udev: `sudo udevadm control --reload-rules && sudo udevadm trigger`

**Build fails with Qt errors:**
- Install Qt5 or Qt6 development packages
- Or disable GUI: `cmake -DWITH_GUI=OFF`

**Animation not working:**
- Verify animation executable exists: `ls build/lib/ckb-next-animations/`
- Check animation process spawning in GUI debug output

## Additional Resources

- Wiki: https://github.com/ckb-next/ckb-next/wiki
- Supported Hardware: https://github.com/ckb-next/ckb-next/wiki/Supported-Hardware
- Linux Installation: https://github.com/ckb-next/ckb-next/wiki/Linux-Installation
- Troubleshooting: https://github.com/ckb-next/ckb-next/wiki/Troubleshooting
- Issue Tracker: https://github.com/ckb-next/ckb-next/issues
