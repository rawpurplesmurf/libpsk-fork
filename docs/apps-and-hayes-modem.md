# Reference Applications & Hayes AT Modem Guide

`libpsk` includes two reference applications that wrap the core library in a classic **Hayes AT command** interface with a `+++` guard-time escape sequence:

1. **Linux Loopback TCP Modem (`apps/linux_tcpmodem/main.c` → `psk_tcpmodem`)** — Optional standalone daemon (disabled by default in CMake).
2. **ESP32-S3 Embedded Modem (`apps/esp32_modem/main.c`)** — Bare-metal/FreeRTOS template bridging UART and I2S audio via four board-specific HAL hooks.

---

## Linux TCP Modem (`psk_tcpmodem`)

### Security Posture & Opt-In Build

In `libpsk-fork`, `PSK_BUILD_APPS` defaults to `OFF` so standard library builds produce only `libpsk.a` and tests. When explicitly enabled, `psk_tcpmodem` binds strictly to **`127.0.0.1` (`INADDR_LOOPBACK`)** so it is never exposed to external network interfaces.

Build it with:

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DPSK_BUILD_APPS=ON
cmake --build build --target psk_tcpmodem --parallel
```

### Command-Line Usage

```text
psk_tcpmodem [--port PORT] [--carrier HZ] [--mode MODE]
             [--rx-file FILE] [--tx-file FILE] [--help]
```

| Option | Default | Description |
| :--- | :--- | :--- |
| `--port PORT` | `7300` | Local TCP listen port on `127.0.0.1`. |
| `--carrier HZ` | `1500` | Initial audio carrier frequency in Hz. |
| `--mode MODE` | `BPSK31` | Initial PSK mode: `BPSK31`, `QPSK31`, `BPSK63`, `BPSK125`, `QPSK63`, `QPSK125`, or numeric `0`–`5`. |
| `--rx-file FILE` | `stdin` | File/FIFO to read raw 8000 Hz mono float32 PCM RX audio from. |
| `--tx-file FILE` | `stdout` | File/FIFO to write raw 8000 Hz mono float32 PCM TX audio to (created/truncated with mode `0644`). |
| `--help` | — | Print usage summary to `stderr` and exit. |

### Architecture & Audio I/O

- **Single-threaded `poll()` loop:** Multiplexes the TCP listening/client socket (with `TCP_NODELAY`) and RX audio file descriptor with a 50 ms poll timeout, processing `AUDIO_BLK = 256` samples (~32 ms at 8000 Hz) per iteration.
- **Single Client:** Accepts one TCP client at a time; disconnects return the modem to `MODEM_IDLE` and resume accepting connections.
- **Audio Format:** Raw 32-bit IEEE float PCM, mono, 8000 Hz, host endianness. Diagnostic logs are written to `stderr` so `stdout` stays clean for PCM output.

#### Piping Through a Sound Card (SoX)

```bash
rec -q -r 8000 -e float -b 32 -c 1 -t raw - | \
    ./build/psk_tcpmodem --port 7300 --carrier 1500 --mode BPSK31 | \
    play -q -r 8000 -e float -b 32 -c 1 -t raw -
```

#### File / FIFO Loopback Testing

```bash
mkfifo /tmp/psk_pipe.f32
./build/psk_tcpmodem --rx-file /tmp/psk_pipe.f32 --tx-file /tmp/psk_pipe.f32
```

### Connecting to `psk_tcpmodem`

```bash
nc 127.0.0.1 7300
```

Upon connection, the modem enters **COMMAND mode** and prints its greeting banner:

```text
PSK31 MODEM READY

AT+MODE=BPSK31 to change mode
ATA to connect (data mode)
```

---

## ESP32-S3 UART / I2S Modem (`apps/esp32_modem/main.c`)

`apps/esp32_modem/main.c` provides an embedded Hayes AT modem designed for an **ESP32-S3** (or any microcontroller with UART + I2S/ADC/DAC) running at `8000 Hz` in `AUDIO_BLK = 64` sample ticks (8 ms per tick).

### Default Parameters

| Macro | Default | Meaning |
| :--- | :---: | :--- |
| `SAMPLE_RATE` | `8000u` | Audio sample rate in Hz |
| `CARRIER_HZ` | `1500.0f` | Default carrier frequency in Hz |
| `DEFAULT_PSK_MODE` | `PSK_MODE_BPSK31` | Startup mode |
| `AUDIO_BLK` | `64` | Samples processed per `modem_tick()` (8 ms at 8 kHz) |
| `ESCAPE_GUARD` | `8000` | 1-second guard time in samples for `+++` |

### HAL Functions to Implement

Replace the four static stub functions at the top of `apps/esp32_modem/main.c` with your board's ESP-IDF drivers:

```c
/* Read up to max_samples float32 PCM samples from I2S ADC (non-blocking).
 * Returns number of samples read (0 if none available). */
static int hal_audio_read(float *buf, int max_samples);

/* Write samples float32 PCM samples to I2S DAC (convert float → int16).
 * Returns number of samples written. */
static int hal_audio_write(const float *buf, int samples);

/* Read up to max_bytes from UART (non-blocking).
 * Returns number of bytes read (0 if none available). */
static int hal_uart_read(char *buf, int max_bytes);

/* Write len bytes to UART. */
static void hal_uart_write(const char *buf, int len);
```

### Lifecycle & FreeRTOS Loop

1. Call `modem_init_esp32()` once at startup. It initializes `g_tx` and `g_rx`, sets `MODEM_COMMAND` mode, and sends `PSK31 ESP32 MODEM READY\r\n` over UART.
2. Call `modem_tick()` periodically from a FreeRTOS task every `8 ms` (`AUDIO_BLK / SAMPLE_RATE`):

```c
void app_main(void)
{
    modem_init_esp32();
    while (1) {
        modem_tick();
        vTaskDelay(pdMS_TO_TICKS(8));
    }
}
```

### Integrating with ESP-IDF

`libpsk` requires no edits to `src/` or `include/` for ESP-IDF:

1. Create a component directory (e.g., `components/libpsk/`) containing or symlinking `include/` and `src/`.
2. Add a `components/libpsk/CMakeLists.txt`:

```cmake
idf_component_register(
    SRCS
        "src/varicode.c"
        "src/fir.c"
        "src/costas.c"
        "src/timing.c"
        "src/viterbi.c"
        "src/tx.c"
        "src/rx.c"
    INCLUDE_DIRS "include"
    PRIV_INCLUDE_DIRS "src"
)
```

3. Place your HAL-wired copy of `apps/esp32_modem/main.c` in `main/`, then build and flash:

```bash
idf.py set-target esp32s3
idf.py build
idf.py flash monitor
```

---

## Hayes AT Command Reference

Commands are **case-insensitive** and terminated by `\r` or `\n`. In `psk_tcpmodem`, backspace (`0x08`) edits the command line.

| Command | `psk_tcpmodem` | `esp32_modem` | Response | Description |
| :--- | :---: | :---: | :--- | :--- |
| `AT` | Yes | Yes | `OK` | Attention / liveness check. |
| `ATZ` | Yes | Yes | `OK` | Reset mode to `BPSK31` and carrier to `1500 Hz` (recreates TX/RX contexts). |
| `ATA` | Yes | Yes | `CONNECT` | Enter **DATA mode** (streams client bytes to TX and decoded RX chars to client). |
| `ATH` | Yes | Yes | `NO CARRIER` | Hang up — transitions to `MODEM_IDLE` (see note below). |
| `ATH0` | Yes | No | `NO CARRIER` | Alias for `ATH` (`psk_tcpmodem` only). |
| `AT+MODE=<mode>` | Yes | Yes | `OK` or `ERROR` | Set PSK mode (`BPSK31`, `QPSK31`, `BPSK63`, `BPSK125`, `QPSK63`, `QPSK125`, or `0`–`5`). Recreates TX/RX contexts and discards buffered TX data. |
| `AT+FREQ=<hz>` | Yes | Yes | `OK` or `ERROR` | Set carrier frequency in Hz (valid range `100.0–3500.0`). Recreates TX/RX contexts and discards buffered TX data. |
| `AT+MODE?` | Yes | No | `+MODE: <name>` then `OK` | Query active PSK mode (`psk_tcpmodem` only). |
| `AT+FREQ?` | Yes | No | `+FREQ: <hz>` then `OK` | Query active carrier frequency formatted to 1 decimal place (`psk_tcpmodem` only). |
| `ATI` | Yes | Yes | Version banner then `OK` | Returns `PSK31 Hayes Modem v1.0` (Linux) or `PSK31 ESP32 Modem v1.0` (ESP32). |

> **Implementation Note on `ATH`:** Issuing `ATH` (or `ATH0`) sets the internal state to `MODEM_IDLE` rather than `MODEM_COMMAND`. Because the input loop processes characters only in `MODEM_DATA` or `MODEM_COMMAND`, reconnecting the TCP session (on Linux) or resetting the device (on ESP32) is required to re-enter `MODEM_COMMAND` after `ATH`. Use `+++` when you want to return to `MODEM_COMMAND` while staying connected.

---

## `+++` Guard-Time Escape Sequence

While in **DATA mode**, you can return to **COMMAND mode** without disconnecting by sending `+++` surrounded by silence:

1. **Guard-Before:** Send no characters for at least **1 second** (`8000` audio samples).
2. **Escape Burst:** Send exactly three `+` characters (`+++`).
3. **Guard-After:** Send no characters for at least **1 second** (`8000` audio samples).

When the post-guard timer expires, the modem switches to `MODEM_COMMAND` and responds with:

```text
OK
```

- If a non-`+` character arrives before 3 `+` characters are collected (or without the guard-before silence), any held `+` characters are flushed verbatim to `psk_tx_write()` as ordinary text.
- Guard timing is driven by the audio block cadence (`esc_audio_tick()` in `psk_tcpmodem`, `modem_tick()` in `esp32_modem`), so RX audio input must be flowing for the timers to advance in `psk_tcpmodem`.

---

## Interactive Session Walkthrough (`psk_tcpmodem`)

```text
< PSK31 MODEM READY
<
< AT+MODE=BPSK31 to change mode
< ATA to connect (data mode)
<
> AT
< OK
> AT+MODE?
< +MODE: BPSK31
< OK
> AT+FREQ=1500
< OK
> ATA
< CONNECT
> CQ CQ DE W1AW K
  (transmitted as BPSK31 audio; decoded RX text appears here in real time)
  ... wait 1 second ...
> +++
  ... wait 1 second ...
< OK
> AT+MODE=BPSK63
< OK
> ATA
< CONNECT
```
