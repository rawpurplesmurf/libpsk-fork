# libpsk (`libpsk-fork`) — PSK31 / PSK63 / PSK125 Modem Library

**`libpsk-fork`** is a maintained, security-hardened fork of Jeff Francis (N0GQ)'s [`jfrancis42/libpsk`](https://github.com/jfrancis42/libpsk) — a zero-dependency, C99 static modem library (`libpsk.a`) for **PSK31, PSK63, and PSK125** weak-signal digital radio modes. Designed for embedding in Linux amateur radio stations (including [`jf8call-fork`](../jf8call-fork)), microcontrollers such as the **ESP32-S3**, or any host language with C FFI that can supply 32-bit floating-point PCM audio buffers.

---

## About This Fork

This repository is forked from [`jfrancis42/libpsk`](https://github.com/jfrancis42/libpsk) to serve as a clean, hardened, CI-verified companion modem library alongside [`jf8call-fork`](../jf8call-fork) (our fork of [`jfrancis42/jf8call`](https://github.com/jfrancis42/jf8call)), [`olivia-modem-fork`](../olivia-modem-fork), and [`gfsk8-modem-clean-fork`](../gfsk8-modem-clean-fork). Key cleanup and hardening changes in this fork include:

- **Security Hardening:** Disabled standalone `psk_tcpmodem` (`PSK_BUILD_APPS=OFF`) and optional CLI tools (`PSK_BUILD_TOOLS=OFF`) by default so standard builds emit only `libpsk.a` and tests, and restricted `psk_tcpmodem` socket binding from `INADDR_ANY` (`0.0.0.0`) to `INADDR_LOOPBACK` (`127.0.0.1`).
- **Build & CI Cleanup:** Simplified `CMakeLists.txt` conditionals, renamed `tests/` to `c_tests/` so Python `pytest` CI auto-discovery does not exit with status 5 while keeping all 5 CTest suites intact, and added [`CHANGELOG.md`](CHANGELOG.md), [`.markdownlint-cli2.jsonc`](.markdownlint-cli2.jsonc), and [`AGENTS.md`](AGENTS.md).
- **Code-Verified Technical Documentation:** Replaced outdated upstream README sections (removing references to nonexistent `cmake/toolchain-*.cmake` files, correcting `status_cb` cadence, `test_tx.c` assertions, and `esp32_modem` AT command parity, and transparently documenting current QPSK TX/RX limitations) with a comprehensive documentation suite in [`docs/`](docs/api-reference.md).
- **Zero-Dependency C99 Parity:** Preserves wire-level BPSK31/63/125 interoperability with upstream `jfrancis42/libpsk` while keeping dependencies strictly limited to C99 and `libm`.

---

## Highlights

- **Zero External Dependencies:** Pure C99 plus standard `libm` (`-lm`). No POSIX calls in the core library (`src/`), no third-party DSP frameworks.
- **Deterministic Real-Time Footprint:** Single allocation at context creation (`psk_tx_create` / `psk_rx_create`), zero heap allocations during streaming, `float` throughout (no `double` in hot paths), and `< 4 KB` RAM per context (~360 B TX, ~1.5 KB RX).
- **Complete BPSK31 / BPSK63 / BPSK125 Signal Chain:**
  - **TX:** G3PLX Varicode encoding, differential phase modulation, 32-symbol preamble/postamble framing, and raised half-cosine amplitude shaping at phase reversals.
  - **RX:** Bandpass pre-filter, complex NCO downconversion, 1st-order baseband I/Q lowpass filters, post-LPF 2nd-order Costas carrier recovery loop, Gardner symbol timing recovery, differential phase decoding, and Varicode accumulator.
- **Hardened Fork Defaults (`libpsk-fork`):**
  - Builds only `libpsk.a` and the 5 CTest suites (`c_tests/`) by default (`PSK_BUILD_APPS=OFF`, `PSK_BUILD_TOOLS=OFF`).
  - Optional standalone Hayes TCP modem (`psk_tcpmodem`) binds strictly to `127.0.0.1` (`INADDR_LOOPBACK`).
- **Ready for C, C++, Embedded & FFI:** Clean `extern "C"` header ([`include/psk.h`](include/psk.h)), static linking into CMake projects like [`jf8call-fork`](../jf8call-fork), plus reference Hayes AT modem apps for Linux TCP and ESP32-S3 UART/I2S.

---

## Supported Modes

| Mode | Enum Value | Symbol Rate | Approx. BW | FEC | Bits / Symbol | Current Status |
| :--- | :---: | :---: | :---: | :---: | :---: | :--- |
| **BPSK31** | `PSK_MODE_BPSK31` (`0`) | 31.25 baud | ~31 Hz | None | 1 | End-to-end verified (loopback + AWGN down to −10 dB) |
| **QPSK31** | `PSK_MODE_QPSK31` (`1`) | 31.25 baud | ~62 Hz | K=5, rate-1/2 conv. | 2 | API & codec scaffolding (see note below) |
| **BPSK63** | `PSK_MODE_BPSK63` (`2`) | 62.5 baud | ~63 Hz | None | 1 | End-to-end verified (loopback) |
| **BPSK125** | `PSK_MODE_BPSK125` (`3`) | 125.0 baud | ~125 Hz | None | 1 | End-to-end verified (loopback) |
| **QPSK63** | `PSK_MODE_QPSK63` (`4`) | 62.5 baud | ~126 Hz | K=5, rate-1/2 conv. | 2 | API & codec scaffolding (see note below) |
| **QPSK125** | `PSK_MODE_QPSK125` (`5`) | 125.0 baud | ~250 Hz | K=5, rate-1/2 conv. | 2 | API & codec scaffolding (see note below) |

> **Audio Format:** Mono **float32 PCM** (typically **8000 Hz** or **12000 Hz**, normalized to `[-1.0, +1.0]`). Carrier frequency is fixed per context (`0 < carrier_hz < sample_rate / 2`, typically `1000–2000 Hz`).
>
> **Note on QPSK Modes:** The K=5 convolutional encoder, depth-8 block interleaver, 4-quadrant Costas error detector, and 16-state Viterbi decoder are implemented, and all 6 modes pass lifecycle smoke tests (`c_tests/test_modes.c`). However, `src/tx.c` currently simplifies QPSK carrier shaping to BPSK-style polarity transitions (marked `TODO: full QPSK I/Q modulation`), and `src/rx.c` does not yet de-interleave dibits prior to Viterbi decoding. See [DSP Architecture](docs/dsp-architecture.md#qpsk-fec-and-implementation-status) for details.

---

## Quick Start

### Build & Run Tests

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel
ctest --test-dir build --output-on-failure
```

By default this builds `build/libpsk.a` and the five CTest binaries from `c_tests/`. To also build the optional standalone Linux Hayes TCP modem (`psk_tcpmodem`), pass `-DPSK_BUILD_APPS=ON`:

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DPSK_BUILD_APPS=ON
cmake --build build --parallel
```

### Minimal C Loopback Example

Include [`include/psk.h`](include/psk.h) and link against `libpsk.a` and `-lm`:

```c
#include "psk.h"
#include <stdio.h>

static void on_char(char c, void *ud)
{
    (void)ud;
    putchar(c);
    fflush(stdout);
}

int main(void)
{
    const unsigned sample_rate = 8000;
    const float    carrier_hz  = 1500.0f;

    psk_tx_t *tx = psk_tx_create(sample_rate, carrier_hz, PSK_MODE_BPSK31);
    psk_rx_t *rx = psk_rx_create(sample_rate, carrier_hz, PSK_MODE_BPSK31);
    if (!tx || !rx) return 1;

    psk_rx_set_char_callback(rx, on_char, NULL);
    psk_tx_write(tx, "HELLO WORLD\n", 12);

    float buf[256];
    while (!psk_tx_idle(tx)) {
        psk_tx_read(tx, buf, 256);
        psk_rx_feed(rx, buf, 256);
    }

    psk_tx_destroy(tx);
    psk_rx_destroy(rx);
    return 0;
}
```

---

## Repository Structure

```text
libpsk-fork/
├── include/
│   └── psk.h                    # Public C99 API header
├── src/
│   ├── tx.c                     # PSK transmitter, shaping & state machine
│   ├── rx.c                     # PSK receiver, Costas loop, SNR & Varicode accumulator
│   ├── varicode.c               # G3PLX 128-entry ASCII Varicode encoder/decoder
│   ├── fir.h / fir.c            # Biquad BPF and cascaded Butterworth LPF
│   ├── timing.h / timing.c      # Gardner symbol timing error detector & loop
│   ├── viterbi.h / viterbi.c    # Rate-1/2 K=5 convolutional encoder & 16-state Viterbi decoder
│   └── costas.h / costas.c      # Standalone 2nd-order Costas loop helper
├── apps/
│   ├── linux_tcpmodem/main.c    # Optional Linux loopback Hayes AT TCP modem (PSK_BUILD_APPS=ON)
│   └── esp32_modem/main.c       # ESP32-S3 UART/I2S Hayes AT modem template
├── c_tests/
│   ├── test_varicode.c          # CTest: 128-ASCII Varicode round-trip & prefix-free checks
│   ├── test_tx.c                # CTest: TX amplitude bounds, RMS, and idle drain
│   ├── test_loopback.c          # CTest: BPSK31/63/125 ideal TX→RX decode verification
│   ├── test_snr.c               # CTest: BPSK31 AWGN tolerance (+20 dB down to −10 dB)
│   ├── test_modes.c             # CTest: Lifecycle smoke test across all 6 modes & bad args
│   ├── test_debug.c             # Standalone BPSK31 per-symbol diagnostic harness
│   └── test_debug2.c            # Standalone BPSK63/125 per-symbol diagnostic harness
├── docs/                        # Detailed technical documentation suite
├── CMakeLists.txt               # CMake C99 build and CTest configuration
├── CHANGELOG.md                 # Fork release and hardening notes
└── AGENTS.md                    # Fork branch & CI workflow instructions
```

---

## Documentation

| Document | Description |
| :--- | :--- |
| **[Public API Reference](docs/api-reference.md)** | Complete C99 API specification for `include/psk.h`: TX, RX, callbacks, Varicode utilities, memory/threading model, and C/C++/FFI usage patterns. |
| **[DSP Architecture & Internals](docs/dsp-architecture.md)** | Deep dive into the TX/RX signal chains, raised-cosine shaping, biquad/Butterworth filters, Costas loop, Gardner timing recovery, Varicode, Viterbi FEC, and QPSK status. |
| **[Reference Apps & Hayes AT Guide](docs/apps-and-hayes-modem.md)** | Operating `psk_tcpmodem` (Linux loopback TCP) and porting `apps/esp32_modem` (ESP32-S3 UART/I2S), full AT command reference, and `+++` guard-time escape behavior. |
| **[Building & Integration Guide](docs/integration-and-building.md)** | CMake options, static linking into C/C++ projects, `jf8call-fork` integration (`PSK_MODEM_DIR`), cross-compilation, ESP-IDF setup, and FFI guidance. |
| **[Testing & Development Guide](docs/testing-and-development.md)** | CTest suite details (`c_tests/`), standalone DSP debug harnesses (`test_debug.c` / `test_debug2.c`), markdown linting, and branch/PR workflow. |
| **[Changelog](CHANGELOG.md)** | Notable changes and security hardening in this fork. |

---

## Integration with `jf8call-fork`

`libpsk-fork` is linked as an optional static library by [`jf8call-fork`](../jf8call-fork) (our fork of [`jfrancis42/jf8call`](https://github.com/jfrancis42/jf8call)), which auto-discovers `../libpsk-fork` via `CMakeLists.txt` or accepts `-DPSK_MODEM_DIR=<path>`. Inside `jf8call-fork`, `PskModem` implements the `IModem` interface alongside [`gfsk8-modem-clean-fork`](../gfsk8-modem-clean-fork) (JS8/GFSK8), [`olivia-modem-fork`](../olivia-modem-fork) (Olivia MFSK), and optional Codec2—wrapping `libpsk` with a `std::mutex`, `int16_t` ↔ `float32` scaling, and stream sync/EOM framing. See the [Building & Integration Guide](docs/integration-and-building.md) for full details.

---

## Credits & License

- **Original Upstream Library:** Created by **Jeff Francis (N0GQ)** / Ordo Artificum LLC in [`jfrancis42/libpsk`](https://github.com/jfrancis42/libpsk). This repository (`libpsk-fork`) is forked from that upstream project with security hardening, build/CI cleanup, and updated technical documentation.
- **Project Distribution Notice:** Distributed under **GPL-3.0**, Copyright (c) 2025 Jeff Francis N0GQ / Ordo Artificum LLC. Any project that statically links `libpsk` should comply with GPL-3.0 or a compatible license. *(Note: individual source headers under `include/`, `src/`, and `apps/` carry `MIT License. Copyright (c) 2025.` notices from the upstream release; verify licensing requirements with the copyright holder before relicensing or redistributing under non-GPL terms.)*
- **Algorithms & Specifications:**
  - PSK31 modulation and Varicode specification by **Peter Martinez (G3PLX)** (public domain, 1998).
  - Biquad bilinear transform formulas from the **Audio EQ Cookbook** by **Robert Bristow-Johnson** (public domain).
  - Timing recovery based on the **Gardner (1986)** non-data-aided timing error detector.
