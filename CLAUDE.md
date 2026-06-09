# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Zephyr RTOS example application structured as a **West T2 workspace application** — the repo itself is both the application and a Zephyr module that adds out-of-tree drivers, libraries, boards, DTS bindings, a custom west extension command, and a custom runner. It serves as a reference implementation for Zephyr-based projects.

## Build and Flash

```sh
# Build for a target board (e.g., custom_plank or nucleo_f302r8)
west build -b $BOARD app

# Build with debug configuration
west build -b $BOARD app -- -DEXTRA_CONF_FILE=debug.conf

# Flash to device
west flash

# Interactive Kconfig configuration
west build -b $BOARD app -t menuconfig
```

## Testing

```sh
# Run all integration tests via Twister
west twister -T tests --integration

# Run all app build tests via Twister
west twister -T app -v --inline-logs --integration

# Run a single test suite (e.g., custom lib)
west twister -T tests/lib/custom --integration
```

Tests use the **Ztest** framework (`zephyr/ztest.h`). The CI runs both `west twister -T app` and `west twister -T tests` across Linux, macOS, and Windows.

## Documentation

```sh
cd doc
pip install -r requirements.txt
doxygen          # API docs → _build_doxygen/
make html        # Sphinx docs → _build_sphinx/
```

## Repository Structure

```
app/                  # The Zephyr application (main.c, prj.conf, Kconfig, VERSION)
  boards/             # Board-specific overlays (e.g., nucleo_f302r8.overlay)
boards/vendor/        # Out-of-tree board definition: custom_plank (ARM/Nordic-based)
drivers/
  blink/              # Custom driver class: blink (GPIO LED with timer-based blinking)
  sensor/             # Out-of-tree sensor driver: example_sensor (GPIO proximity)
dts/bindings/         # Custom DTS bindings for blink and example_sensor
include/app/          # Public headers for drivers (blink.h) and lib (custom.h)
lib/custom/           # Out-of-tree library: custom_get_value()
scripts/              # west extension (example_west_command.py) and custom runner
tests/lib/custom/     # Ztest unit tests for the custom library
zephyr/module.yml     # Declares this repo as a Zephyr module (cmake, kconfig, board_root, dts_root, runners)
west.yml              # Manifest: pins zephyr + imports cmsis_6, hal_nordic, hal_stm32
CMakeLists.txt        # Module-level CMake: adds include dirs, drivers/, lib/ subdirectories
```

## Architecture

This repo acts as a **Zephyr module** (declared in `zephyr/module.yml`), which means the Zephyr build system automatically picks up:
- Additional boards from `boards/`
- Additional DTS files and bindings from `dts/`
- Out-of-tree drivers and libraries via the root `CMakeLists.txt`
- A custom runner from `scripts/example_runner.py`

The application (`app/`) uses `find_package(Zephyr)` and is built separately from the module — `west build -b $BOARD app` points to the `app/` subdirectory, not the root.

### Custom Driver Class (blink)

`include/app/drivers/blink.h` defines a custom driver class with `__subsystem` and `__syscall` to support Zephyr's system call mechanism. The implementation in `drivers/blink/gpio_led.c` uses a `k_timer` to toggle a GPIO pin. Enabled via `CONFIG_BLINK` / `CONFIG_BLINK_GPIO_LED` Kconfig options; activated when a `blink-gpio-led` compatible node is present in devicetree.

### Custom Sensor Driver

`drivers/sensor/example_sensor/example_sensor.c` implements the standard Zephyr `sensor` driver API (`sample_fetch` / `channel_get`) over a GPIO input, exposing `SENSOR_CHAN_PROX`. Enabled automatically when `DT_HAS_ZEPHYR_EXAMPLE_SENSOR_ENABLED` (i.e., a `zephyr,example-sensor` compatible node exists in DTS).

### Application Logic

`app/src/main.c` reads proximity sensor state every 100 ms and decrements the blink LED period by 100 ms (down to 0) on each low-to-high proximity transition, cycling back to 1000 ms at 0.

## West Workspace Setup

This repo must be initialized as a west workspace (not cloned directly):

```sh
west init -m https://github.com/zephyrproject-rtos/example-application --mr main my-workspace
cd my-workspace
west update
```

The manifest (`west.yml`) imports only the necessary HAL modules (`cmsis_6`, `hal_nordic`, `hal_stm32`) from Zephyr's module list.

## Custom West Extension

`scripts/example_west_command.py` registers `west example-west-command` (declared in `scripts/west-commands.yml`, referenced from `west.yml` under `self.west-commands`).
