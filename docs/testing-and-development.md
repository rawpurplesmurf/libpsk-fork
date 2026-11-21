# Testing & Development Guide

This document describes upstream provenance, the automated CTest suite, standalone DSP diagnostic harnesses, markdown linting rules, and the contribution workflow for `libpsk-fork`.

---

## Repository & Fork Relationships

| Repository | Role |
| :--- | :--- |
| **`libpsk-fork`** (this repo) | Maintained, security-hardened fork (`libpsk.a`) with `PSK_BUILD_APPS=OFF`, loopback-only `psk_tcpmodem`, `c_tests/` CI layout, and full technical documentation. |
| [`jfrancis42/libpsk`](https://github.com/jfrancis42/libpsk) | Original upstream zero-dependency C99 PSK31/63/125 library by Jeff Francis (N0GQ). |
| [`jf8call-fork`](../../jf8call-fork) | Primary downstream consumer (`src/psk_modem.h`, `src/psk_modem.cpp`), forked from [`jfrancis42/jf8call`](https://github.com/jfrancis42/jf8call). |
| [`olivia-modem-fork`](../../olivia-modem-fork) | Sibling Olivia MFSK modem library fork, forked from [`jfrancis42/olivia-modem`](https://github.com/jfrancis42/olivia-modem). |
| [`gfsk8-modem-clean-fork`](../../gfsk8-modem-clean-fork) | Sibling JS8/GFSK8 modem library fork, forked from [`jfrancis42/gfsk8-modem-clean`](https://github.com/jfrancis42/gfsk8-modem-clean). |

---

## Automated CTest Suite (`c_tests/`)

All automated tests live under [`c_tests/`](../c_tests/) (renamed from `tests/` so Python `pytest` auto-discovery in CI does not exit with status 5 on C-only repositories).

Run the entire suite after building:

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel
ctest --test-dir build --output-on-failure
```

For verbose per-test output (peak/RMS levels, decoded strings, and per-SNR tables):

```bash
ctest --test-dir build -V
```

### Test Case Breakdown

| CTest Name | Source File | What It Verifies |
| :--- | :--- | :--- |
| `varicode` | [`c_tests/test_varicode.c`](../c_tests/test_varicode.c) | Encodes all 128 ASCII values (`0..127`), asserts `n >= 3`, leading bit `1`, trailing `00` separator, zero internal `00` bit pairs, exact round-trip via `psk_varicode_decode()`, known bit vectors for `' '` (`100`) and `'e'` (`1100`), and `0` return on invalid codeword `{0, 0, 1, 0}`. |
| `tx` | [`c_tests/test_tx.c`](../c_tests/test_tx.c) | Generates 3 seconds of BPSK31 PCM at `8000 Hz` / `1000 Hz` carrier for `"TEST DE W0PSK"`; verifies `psk_tx_write()` count, peak amplitude `<= 1.001f` (no clipping), RMS `> 0.01f` (carrier present), and that `psk_tx_idle()` transitions to true once drained. |
| `loopback` | [`c_tests/test_loopback.c`](../c_tests/test_loopback.c) | Ideal noise-free TX → RX loopback (5 s at `8000 Hz` / `1000 Hz`, fed in 256-sample blocks) for **`BPSK31`**, **`BPSK63`**, and **`BPSK125`**; asserts `"HELLO PSK"` is decoded in full for each mode. |
| `snr` | [`c_tests/test_snr.c`](../c_tests/test_snr.c) | Generates 6 s of BPSK31 (`"TEST"` repeated 3×), injects deterministic Box-Muller AWGN (LCG seed `0xCAFEBABE`) at `+20`, `+10`, `+5`, `0`, `-5`, and `-10 dB` SNR, and asserts `"TEST"` is decoded at least once at every level `>= -10 dB`. |
| `modes` | [`c_tests/test_modes.c`](../c_tests/test_modes.c) | Lifecycle smoke test across all 6 modes (`BPSK31`, `QPSK31`, `BPSK63`, `BPSK125`, `QPSK63`, `QPSK125`) at `8000 Hz` / `1500 Hz` (`create`, `write`, `read`, `feed`, `reset`, `destroy`), plus invalid parameter rejection (`sample_rate == 0`, `carrier_hz == 0.0f`). |

---

## Standalone DSP Diagnostic Harnesses

Two additional diagnostic programs live in `c_tests/` and are intentionally **not** wired into CTest. They instantiate the internal `src/fir.h` and `src/timing.h` filters directly (with a fixed ideal NCO and `4×baud` LPF) and print per-symbol `I_filt`, `Q_filt`, `phase`, `dp`, bit decisions, and Varicode decode events to `stdout`:

- [`c_tests/test_debug.c`](../c_tests/test_debug.c) — BPSK31 (`31.25 baud`, `sps = 256`) symbol-by-symbol trace of `"HELLO PSK"`.
- [`c_tests/test_debug2.c`](../c_tests/test_debug2.c) — BPSK63 (`62.5 baud`, `sps = 128`) and BPSK125 (`125.0 baud`, `sps = 64`) symbol-by-symbol traces.

Build and run them manually against `build/libpsk.a`:

```bash
cc -std=c99 -O2 -Iinclude -Isrc c_tests/test_debug.c  build/libpsk.a -lm -o /tmp/test_debug  && /tmp/test_debug
cc -std=c99 -O2 -Iinclude -Isrc c_tests/test_debug2.c build/libpsk.a -lm -o /tmp/test_debug2 && /tmp/test_debug2
```

These harnesses are useful when tuning filter bandwidths, diagnosing timing-loop slip, or verifying new modulation modes without Costas loop interaction.

---

## Development & Contribution Guidelines

### Technical Guardrails

1. **Zero External Dependencies:** Keep `src/` strictly C99 + `<math.h>`. Never introduce POSIX headers, threads, or third-party libraries into `libpsk.a`.
2. **Real-Time & Embedded Compatibility:**
   - Avoid heap allocation outside `psk_tx_create()` and `psk_rx_create()`.
   - Use single-precision `float` (`*f` math functions) in inner loops so ESP32-S3 hardware FPU execution stays fast.
   - Keep per-context RAM under `4 KB`.
3. **Compiler Warning Cleanliness:** All library code compiles with `-Wall -Wextra` under C99.
4. **Security Hardening Defaults:**
   - Keep `PSK_BUILD_APPS` and `PSK_BUILD_TOOLS` defaulted to `OFF` in `CMakeLists.txt`.
   - Keep `apps/linux_tcpmodem/main.c` bound strictly to `INADDR_LOOPBACK` (`127.0.0.1`).
5. **Documentation Hygiene:** Markdown files are linted according to [`.markdownlint-cli2.jsonc`](../.markdownlint-cli2.jsonc). Use ATX headings with sequential heading levels, fenced code blocks, consistent GFM tables, and a single trailing newline.

### Branch & Pull Request Workflow

Per [`AGENTS.md`](../AGENTS.md):

1. Never commit or push directly to `main`. Create an isolated `feat/*`, `fix/*`, or `docs/*` branch targeting `main`.
2. Build with `-Wall -Wextra` and verify all 5 CTest cases pass (`ctest --test-dir build --output-on-failure`).
3. Open a Pull Request targeting `main`, ensure automated code review (`APPROVED`) and CI pipeline (`success`) pass, and merge via `rebase-merge`.
