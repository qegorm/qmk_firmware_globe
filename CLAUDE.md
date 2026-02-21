# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a fork of QMK firmware focused on the **NuPhy Air75 V2** keyboard (`keyboards/nuphy/air75_v2/ansi/`), with custom sleep management, RF wireless driver, and side LED power handling. The branch `air75v2-sleep` contains all keyboard-specific work on top of upstream QMK.

## Build Commands

```bash
# Build firmware for the target keyboard
make nuphy/air75_v2/ansi:via

# Build and flash
make nuphy/air75_v2/ansi:via:flash

# Clean
make nuphy/air75_v2/ansi:via:clean

# Parallel build (faster)
make -j8 nuphy/air75_v2/ansi:via
```

General QMK make syntax: `make <keyboard>:<keymap>[:<target>]`

```bash
# Run unit tests
make test

# List available unit tests
make list-tests

# Run a specific test
make test:test_name

# Python linting/formatting (requires Docker)
make pytest
make format-core
```

## Repository Architecture

QMK is organized as:
- **`quantum/`** — Core QMK features: keycodes, layers, RGB matrix, encoders, OLED, etc.
- **`tmk_core/`** — Low-level keyboard matrix scanning and USB HID
- **`platforms/`** — MCU-specific code: `avr/`, `chibios/` (STM32), `test/`
- **`drivers/`** — External hardware drivers (LED ICs, displays, sensors)
- **`keyboards/`** — Per-keyboard definitions organized by vendor/model
- **`builddefs/`** — Build system: `build_keyboard.mk`, `common_features.mk`, `common_rules.mk`
- **`tests/`** — Unit tests for core QMK functionality
- **`lib/`** — Git submodules (ChibiOS, LUFA, etc.)

## Air75 V2 Keyboard Architecture

All custom code lives in `keyboards/nuphy/air75_v2/ansi/`. Key files and their roles:

| File | Role |
|------|------|
| `ansi.h` | Custom keycodes (`RF_DFU`, `LNK_USB`, `SLEEP_MODE`, `KB_SLP`, etc.), shared structs (`DEV_INFO_STRUCT`, `kb_config_t`), and timing constants |
| `ansi.c` | QMK hooks: `keyboard_post_init_kb`, `housekeeping_task_kb`, `process_record_kb` |
| `user_kb.c/h` | High-level keyboard logic, global state variables, and most function declarations |
| `sleep.c` | Sleep state machine — `sleep_handle()` (called every 50ms) and `deep_sleep_handle()` |
| `mcu_pwr.c/h` | MCU-level power functions: `enter_deep_sleep()`, `exit_deep_sleep()`, `enter_light_sleep()`, LED power rail control |
| `rf.c` / `rf_driver.c` | 2.4GHz RF wireless driver via UART (SD1, TX: B6, RX: B7) to the NRF module |
| `rf_queue.c/h` | Command queue for RF UART communication |
| `side.c` / `side_driver.c` | Side LED animations and driver |
| `side_table.h` | LED color lookup tables |
| `rules.mk` | Adds all custom `.c` files to `SRC` and enables `UART_DRIVER_REQUIRED` |

### Hardware

- **MCU:** STM32F072 (ChibiOS platform)
- **Bootloader:** stm32-dfu
- **Connectivity:** USB, 2.4GHz RF (NRF module via UART), Bluetooth (3 channels)
- **Link modes:** `LINK_RF_24=0`, `LINK_BT_1=1`, `LINK_BT_2=2`, `LINK_BT_3=3`, `LINK_USB=4`

### Sleep System

Three sleep modes stored in `kb_config.sleep_mode`:
- `SLEEP_MODE_OFF (0)` — no sleep
- `SLEEP_MODE_LIGHT (1)` — light sleep (peripherals off, MCU active)
- `SLEEP_MODE_DEEP (2)` — WFI (wait-for-interrupt), MCU halted

Sleep is **blocked** in USB mode and when charging wirelessly (RF charge flag set). The idle timeout is `SLEEP_TIME_DELAY = 100 * 360` (36 seconds of `no_act_time` increments). `sleep_handle()` is called from `housekeeping_task_kb()` every 50ms.

### Key State Globals (defined in `user_kb.c`, declared extern in `user_kb.h`)

- `dev_info` — current link mode, RF state, battery, charge status
- `kb_config` — persisted keyboard config (sleep mode, side LED settings, RF timeout)
- `no_act_time` — idle counter (increments every 100ms)
- `f_goto_sleep` / `f_wakeup_prepare` — sleep state flags
- `rf_linking_time` — timeout counter for RF linking phase
- `sleep_time_delay` — configurable sleep timeout (defaults to `SLEEP_TIME_DELAY`)

### VIA Keymap

The keymap is in `keymaps/via/`. The VIA JSON definition is `keymaps/via/air75_v2_via_v3.json`. Dynamic keymap supports 8 layers.
