# Building & Integration Guide

This guide covers building `libpsk.a` with CMake, configuring optional targets, linking `libpsk` into `jf8call-fork` or standalone C/C++ projects, and cross-compiling for embedded and secondary platforms.

---

## Prerequisites

- **C99 compiler:** GCC, Clang, or MinGW-w64
- **CMake:** `>= 3.16`
- **Build tool:** Make or Ninja
- **Standard Math Library:** `libm` (`-lm`, linked automatically via CMake)

On Ubuntu 24.04 LTS / Debian 12+:

```bash
sudo apt-get update
sudo apt-get install -y build-essential cmake ninja-build
```

---

## CMake Build Options

Defined in [`CMakeLists.txt`](../CMakeLists.txt):

| Option | Default | Effect |
| :--- | :---: | :--- |
| `PSK_BUILD_APPS` | `OFF` | Builds the standalone loopback TCP modem `psk_tcpmodem` (`apps/linux_tcpmodem/main.c`). |
| `PSK_BUILD_TOOLS` | `OFF` | Enables optional CLI tool targets (`tools/psk31_beacon.c`, `tools/psk31_decode.c`) if present in a downstream tree. |
| `CMAKE_BUILD_TYPE` | *(unset)* | Set to `Release` to enable `-O2` optimization (`-Wall -Wextra` are always enabled on target `psk`). |

### Standard Library + Test Build

```bash
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel
ctest --test-dir build --output-on-failure
```

This produces:

- `build/libpsk.a` — Static modem library
- `build/test_varicode`, `build/test_tx`, `build/test_loopback`, `build/test_snr`, `build/test_modes` — CTest executables

### Enabling the Standalone Linux TCP Modem

```bash
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DPSK_BUILD_APPS=ON
cmake --build build --parallel
```

---

## Integrating with `jf8call-fork`

[`jf8call-fork`](../../jf8call-fork) (our fork of [`jfrancis42/jf8call`](https://github.com/jfrancis42/jf8call)) automatically discovers and links `libpsk.a` as an optional modem (`src/psk_modem.h` / `src/psk_modem.cpp`, `kPskModemType = 3`) alongside [`gfsk8-modem-clean-fork`](../../gfsk8-modem-clean-fork) and [`olivia-modem-fork`](../../olivia-modem-fork).

### Automatic Sibling Discovery Order

When configuring `jf8call-fork`, its `CMakeLists.txt` searches for `include/psk.h` in this exact order unless `-DPSK_MODEM_DIR` is explicitly provided:

1. `-DPSK_MODEM_DIR=<path>` (CMake cache variable)
2. `$ENV{PSK_MODEM_DIR}` (environment variable)
3. `../libpsk`
4. `../libpsk-fork`
5. `$HOME/libpsk-fork`
6. `$HOME/libpsk`
7. `$HOME/psk31`

It then locates the built static archive at:

1. `${PSK_MODEM_BUILD_DIR}/libpsk.a` (if `-DPSK_MODEM_BUILD_DIR` is set)
2. `${PSK_MODEM_DIR}/build/libpsk.a`
3. `${PSK_MODEM_DIR}/libpsk.a`

### Build Sequence with `jf8call-fork`

```bash
# 1. Build libpsk static archive
cmake -S libpsk-fork -B libpsk-fork/build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build libpsk-fork/build --parallel
ctest --test-dir libpsk-fork/build --output-on-failure

# 2. Configure and build jf8call-fork (auto-discovers ../libpsk-fork/build/libpsk.a)
cmake -S jf8call-fork -B jf8call-fork/build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build jf8call-fork/build --parallel
```

To point `jf8call-fork` at a non-sibling location:

```bash
cmake -S jf8call-fork -B jf8call-fork/build -G Ninja -DCMAKE_BUILD_TYPE=Release \
    -DPSK_MODEM_DIR=/path/to/libpsk-fork \
    -DPSK_MODEM_BUILD_DIR=/path/to/libpsk-fork/build
```

### How `jf8call-fork`'s `PskModem` Wraps `libpsk`

- **Sample Rate & Carrier:** Runs at `8000 Hz`, default `1500.0f Hz` carrier, exposing all 6 submodes (`BPSK31`, `BPSK63`, `BPSK125`, `QPSK31`, `QPSK63`, `QPSK125`) as streaming (non-UTC-aligned) modes.
- **Stateless TX (`modulate`):** Creates a short-lived `psk_tx_t`, enqueues the frame payload (including stream sync/EOM markers), reads up to `80000` float samples (10 s at 8 kHz), and destroys the TX context.
- **Stateful RX (`feedAudio`):** Converts `int16_t` audio to `float32` (`sample / 32767.0f`), locks `m_rxMutex`, registers a per-call `charCb`, feeds `psk_rx_feed()`, and flushes accumulated text on `\n`, EOM marker, or 256 characters.

---

## Embedding in Other C / C++ Projects

### Option A: CMake `add_subdirectory`

```cmake
add_subdirectory(external/libpsk-fork EXCLUDE_FROM_ALL)

add_executable(my_radio_app main.c)
target_link_libraries(my_radio_app PRIVATE psk)
```

Target `psk` exports `include/` as a `PUBLIC` include directory and `m` (`-lm`) as a `PUBLIC` link library automatically.

### Option B: Prebuilt Static Library Import

```cmake
add_library(psk STATIC IMPORTED)
set_target_properties(psk PROPERTIES
    IMPORTED_LOCATION "/path/to/libpsk-fork/build/libpsk.a"
)
target_include_directories(psk INTERFACE "/path/to/libpsk-fork/include")

target_link_libraries(my_radio_app PRIVATE psk m)
```

### Option C: Direct Compiler Invocation (No CMake)

Because `libpsk` has zero dependencies, you can compile all library sources directly into a project or shared object:

```bash
# Static archive
cc -std=c99 -O2 -Wall -Wextra -Iinclude -Isrc -c \
    src/varicode.c src/fir.c src/costas.c src/timing.c \
    src/viterbi.c src/tx.c src/rx.c
ar rcs libpsk.a varicode.o fir.o costas.o timing.o viterbi.o tx.o rx.o

# Shared object (for Python ctypes / FFI)
cc -std=c99 -O2 -fPIC -shared -Iinclude -Isrc \
    src/varicode.c src/fir.c src/costas.c src/timing.c \
    src/viterbi.c src/tx.c src/rx.c -lm -o libpsk.so
```

---

## Cross-Compilation Notes

> **Note:** This repository does not bundle pre-made `cmake/toolchain-*.cmake` files; pass your system cross-compiler flags or your own CMake toolchain file directly.

### Linux ARM64 (Raspberry Pi `aarch64`)

With `gcc-aarch64-linux-gnu` installed:

```bash
cmake -S . -B build-rpi -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_SYSTEM_NAME=Linux \
    -DCMAKE_SYSTEM_PROCESSOR=aarch64 \
    -DCMAKE_C_COMPILER=aarch64-linux-gnu-gcc
cmake --build build-rpi --target psk --parallel
```

### Windows 64-bit (`mingw-w64`)

With `gcc-mingw-w64-x86-64` installed (build target `psk` only; `psk_tcpmodem` uses POSIX sockets):

```bash
cmake -S . -B build-win -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_SYSTEM_NAME=Windows \
    -DCMAKE_C_COMPILER=x86_64-w64-mingw32-gcc
cmake --build build-win --target psk --parallel
```

### ESP32-S3 (ESP-IDF)

See [Integrating with ESP-IDF](apps-and-hayes-modem.md#integrating-with-esp-idf) for the `idf_component_register` snippet and FreeRTOS loop setup.
