# Public API Reference (`include/psk.h`)

This document specifies the public C99 interface of `libpsk` defined in [`include/psk.h`](../include/psk.h). All symbols are wrapped in `extern "C"` for direct inclusion from C++ and foreign function interfaces (Python `ctypes`, Rust FFI, etc.).

---

## Header & Linking

```c
#include "psk.h"
```

- **Library:** `libpsk.a` (CMake target `psk`)
- **System Dependencies:** Standard C99 library and math library (`-lm`)
- **Audio Format:** Mono 32-bit IEEE 754 `float` PCM, nominal full-scale range `[-1.0f, +1.0f]`.

---

## Enumerations & Opaque Types

### `psk_mode_t`

Selects symbol rate and modulation family for a TX or RX context:

```c
typedef enum {
    PSK_MODE_BPSK31  = 0,  /* 31.25 baud, BPSK */
    PSK_MODE_QPSK31  = 1,  /* 31.25 baud, QPSK + K=5 conv. FEC */
    PSK_MODE_BPSK63  = 2,  /* 62.5  baud, BPSK */
    PSK_MODE_BPSK125 = 3,  /* 125.0 baud, BPSK */
    PSK_MODE_QPSK63  = 4,  /* 62.5  baud, QPSK + K=5 conv. FEC */
    PSK_MODE_QPSK125 = 5,  /* 125.0 baud, QPSK + K=5 conv. FEC */
} psk_mode_t;
```

### Opaque Context Handles

```c
typedef struct psk_tx_s psk_tx_t;
typedef struct psk_rx_s psk_rx_t;
```

Callers interact with contexts strictly through pointers returned by `psk_tx_create()` and `psk_rx_create()`. Each handle is completely self-contained, allowing multiple independent TX/RX instances at different carrier frequencies or modes within the same process.

### Receiver Callback Types

```c
typedef void (*psk_rx_char_cb)(char c, void *userdata);
typedef void (*psk_rx_status_cb)(float snr_db, float freq_offset_hz, void *userdata);
```

- `psk_rx_char_cb`: Invoked synchronously inside `psk_rx_feed()` for each non-NUL decoded ASCII character.
- `psk_rx_status_cb`: Invoked periodically inside `psk_rx_feed()` with the current SNR estimate (in dB) and Costas loop carrier frequency offset (in Hz).

---

## Transmitter API

### `psk_tx_create`

```c
psk_tx_t *psk_tx_create(unsigned sample_rate, float carrier_hz, psk_mode_t mode);
```

Allocates and initializes a new PSK transmitter context in the `IDLE` state.

| Parameter | Type | Constraints |
| :--- | :--- | :--- |
| `sample_rate` | `unsigned` | Must be `> 0` (typically `8000` or `12000` Hz; `sample_rate / baud` is rounded to the nearest integer `sps`). |
| `carrier_hz` | `float` | Must satisfy `0.0f < carrier_hz < sample_rate / 2.0f` (typically `1000.0f–2000.0f`). |
| `mode` | `psk_mode_t` | Must be in `0..5` (`PSK_MODE_BPSK31` .. `PSK_MODE_QPSK125`). |

- **Returns:** Pointer to a newly allocated `psk_tx_t`, or `NULL` on invalid parameters or allocation failure.

### `psk_tx_destroy`

```c
void psk_tx_destroy(psk_tx_t *tx);
```

Releases all memory associated with `tx`. Safe to call with `NULL`.

### `psk_tx_write`

```c
size_t psk_tx_write(psk_tx_t *tx, const char *text, size_t len);
```

Appends up to `len` ASCII bytes into the transmitter's internal circular text buffer.

- **Buffer Capacity:** Internal buffer size is 256 bytes (`255` usable bytes at once). If fewer than `len` slots are free, `psk_tx_write()` accepts as many bytes as fit and returns the count accepted.
- **State Transition:** If the transmitter is currently idle (`TX_STATE_IDLE`) and at least one character is queued, it transitions immediately to `TX_STATE_PREAMBLE` (32 symbols of continuous phase reversals) before modulating the queued text.
- **Character Range:** Standard 7-bit ASCII (`0x00–0x7F`). Bytes outside `0–127` produce 0 Varicode bits when dequeued.
- **Returns:** Number of characters accepted (`0` if `tx == NULL`, `text == NULL`, or the ring buffer is full).

### `psk_tx_read`

```c
size_t psk_tx_read(psk_tx_t *tx, float *out, size_t frames);
```

Synthesizes `frames` mono `float32` PCM samples into `out`.

- **Amplitude Bounds:** Peak output magnitude is `<= 1.0f` (unit-amplitude cosine carrier with raised half-cosine envelope shaping at phase transitions).
- **Idle Output:** When in `TX_STATE_IDLE`, writes `0.0f` (digital silence) for every sample.
- **Transmission Lifecycle:**
  1. `PREAMBLE` — 32 symbols of alternating phase reversals (1.024 s in BPSK31 at 8 kHz).
  2. `DATA` — Varicode bit stream for all queued characters, each terminated by `00`.
  3. `POSTAMBLE` — 32 symbols of trailing phase reversals once the text and bit queues drain.
  4. `IDLE` — returns to `0.0f` output; `psk_tx_idle(tx)` becomes non-zero.
- **Returns:** `frames` on success, or `0` if `tx == NULL` or `out == NULL`.

### `psk_tx_idle`

```c
int psk_tx_idle(psk_tx_t const *tx);
```

Checks whether the transmitter has finished sending all queued text and its 32-symbol postamble.

- **Returns:** Non-zero (`1`) if `tx == NULL` or `tx` is in `TX_STATE_IDLE`; `0` while transmitting preamble, data, or postamble.

---

## Receiver API

### `psk_rx_create`

```c
psk_rx_t *psk_rx_create(unsigned sample_rate, float carrier_hz, psk_mode_t mode);
```

Allocates and initializes a new PSK receiver context tuned to `carrier_hz`.

| Parameter | Type | Constraints |
| :--- | :--- | :--- |
| `sample_rate` | `unsigned` | Must be `> 0` (typically `8000` or `12000` Hz). |
| `carrier_hz` | `float` | Must satisfy `0.0f < carrier_hz < sample_rate / 2.0f`. |
| `mode` | `psk_mode_t` | Must be in `0..5` (`PSK_MODE_BPSK31` .. `PSK_MODE_QPSK125`). |

- **Returns:** Pointer to a newly allocated `psk_rx_t`, or `NULL` on invalid parameters or allocation failure.

### `psk_rx_destroy`

```c
void psk_rx_destroy(psk_rx_t *rx);
```

Releases all memory associated with `rx`. Safe to call with `NULL`.

### `psk_rx_set_char_callback`

```c
void psk_rx_set_char_callback(psk_rx_t *rx, psk_rx_char_cb cb, void *userdata);
```

Registers (or replaces) the character decode callback. Pass `cb = NULL` to disable character delivery. Safe to call with `rx == NULL` (no-op).

- Decoded NUL (`0x00`) values are not emitted (`if (c && rx->char_cb)`).

### `psk_rx_set_status_callback`

```c
void psk_rx_set_status_callback(psk_rx_t *rx, psk_rx_status_cb cb, void *userdata);
```

Registers (or replaces) the periodic signal metrics callback. Pass `cb = NULL` to disable status reporting. Safe to call with `rx == NULL` (no-op).

#### Status Callback Cadence & Semantics

Inside `src/rx.c`, `status_count` increments once per recovered **symbol** and fires when it reaches `status_interval = (unsigned)(sample_rate / baud)`:

| Mode (at 8000 Hz) | `status_interval` (symbols) | Approx. Callback Period |
| :--- | :---: | :---: |
| BPSK31 / QPSK31 | 256 | ~8.19 s |
| BPSK63 / QPSK63 | 128 | ~2.05 s |
| BPSK125 / QPSK125 | 64 | ~0.51 s |

Callback arguments:

- `snr_db`: Computed from 0.1 s exponential averages of wideband input power (`noise_power`) and BPF passband power (`sig_power`):
  `10 * log10(sig_power / (noise_power - sig_power))`.
  Returns `-99.0f` when `sig_power <= 1e-12f` or `noise_power <= sig_power` (e.g., pure tone with no out-of-band noise or digital silence).
- `freq_offset_hz`: Instantaneous Costas loop NCO offset converted to Hz (`nco_freq * sample_rate / (2π)`), clamped to `±0.03 rad/sample` (approx. `±38.2 Hz` at 8000 Hz).

### `psk_rx_feed`

```c
void psk_rx_feed(psk_rx_t *rx, const float *buf, size_t frames);
```

Processes `frames` mono `float32` PCM samples through the demodulator.

- **Synchronous Callbacks:** Both `char_cb` and `status_cb` execute inline on the calling thread during `psk_rx_feed()`. Do not call `psk_rx_destroy(rx)` from inside a callback.
- **Chunk Size Independence:** Samples may be fed in arbitrary block sizes (1 sample, 64 samples, 256 samples, etc.) with identical DSP results.
- Safe to call with `rx == NULL` or `buf == NULL` (no-op).

### `psk_rx_reset`

```c
void psk_rx_reset(psk_rx_t *rx);
```

Clears all dynamic demodulator state while preserving configuration (`sample_rate`, `carrier_hz`, `mode`, filter coefficients) and registered callbacks:

- Resets BPF and I/Q LPF delay lines.
- Resets NCO phase, frequency offset, and Costas integrator to `0.0f`.
- Resets Gardner timing loop, differential phase reference, Viterbi trellis metrics, Varicode accumulator, power estimators, and status counter.

Useful when re-opening squelch, after a frequency jump (QSY), or between half-duplex TX/RX turnarounds.

---

## Varicode Utilities

### `psk_varicode_encode`

```c
int psk_varicode_encode(char c, uint8_t *bits_out);
```

Encodes a single 7-bit ASCII character into its G3PLX Varicode bit sequence (MSB first), followed by the mandatory two-bit `00` inter-character separator.

| Parameter | Description |
| :--- | :--- |
| `c` | ASCII character (`0..127`). Values outside `0..127` (when cast to `unsigned char`) return `0`. |
| `bits_out` | Caller-allocated buffer of at least **14 bytes** (`uint8_t`), each written as `0` or `1`. |

- **Returns:** Total bit count written (`3..12` for valid ASCII, including the 2 trailing zero bits), or `0` if `bits_out == NULL` or `(unsigned char)c >= 128`.

### `psk_varicode_decode`

```c
char psk_varicode_decode(const uint8_t *bits, size_t len);
```

Decodes an accumulated Varicode bit sequence (MSB first, **without** the trailing `00` separator) back to ASCII.

| Parameter | Description |
| :--- | :--- |
| `bits` | Array of `0`/`1` bit values (only bit 0 of each byte is inspected). |
| `len` | Number of code bits (`1..14`). |

- **Returns:** Matching ASCII character (`0..127`), or `0` if `bits == NULL`, `len == 0`, `len > 14`, or the bit pattern is not in the Varicode table.
- **Note:** ASCII `NUL` (`0x00`, codeword `1010101011`) also returns `0`.

---

## Memory, Real-Time & Threading Guarantees

| Property | Behavior |
| :--- | :--- |
| **Heap Allocation** | Exactly one `calloc()` in `psk_tx_create()` and one in `psk_rx_create()`. Zero dynamic allocations during `psk_tx_write()`, `psk_tx_read()`, `psk_rx_feed()`, or `psk_rx_reset()`. |
| **Memory per Context** | `psk_tx_t`: ~360 bytes; `psk_rx_t`: ~1.5 KB (dominated by the 16-state × 32-depth Viterbi traceback tables). Well under 4 KB total. |
| **Floating Point** | Single-precision `float` (`cosf`, `sinf`, `atan2f`, `fabsf`, `expf`, `log10f`) throughout the DSP loops. |
| **Thread Safety** | Individual `psk_tx_t` and `psk_rx_t` instances are **not** internally locked. Protect concurrent access to the same handle with an external mutex (as done in [`jf8call-fork`](../../jf8call-fork)'s `PskModem`). Independent TX and RX handles can be driven concurrently on separate threads after creation. *(Minor caveat: `src/viterbi.c` lazily populates a read-only process-global trellis table on the first `psk_viterbi_reset()` call; in multi-threaded apps, construct or reset the first RX instance before spawning worker threads.)* |

---

## Usage Patterns

### Streaming TX with Flow Control

Because the TX ring buffer holds 255 bytes, stream long messages by checking the return value of `psk_tx_write()` in your audio callback loop:

```c
const char *msg = "CQ CQ CQ DE W1AW W1AW PSE K\r\n";
size_t total = strlen(msg);
size_t sent  = 0;
float  pcm[256];

while (sent < total || !psk_tx_idle(tx)) {
    if (sent < total) {
        sent += psk_tx_write(tx, msg + sent, total - sent);
    }
    psk_tx_read(tx, pcm, 256);
    /* send pcm[0..255] to audio output device */
}
```

### Bridging `int16_t` Audio (C++)

```cpp
#include <vector>
#include <cstdint>
extern "C" {
#include "psk.h"
}

void feedInt16(psk_rx_t *rx, const int16_t *samples, size_t count)
{
    std::vector<float> fbuf(count);
    for (size_t i = 0; i < count; ++i)
        fbuf[i] = static_cast<float>(samples[i]) / 32767.0f;
    psk_rx_feed(rx, fbuf.data(), fbuf.size());
}
```

### Python `ctypes` Example (Shared Build)

If you compile `libpsk` into a shared object (`libpsk.so`), the C99 ABI maps directly to `ctypes`:

```python
import ctypes

lib = ctypes.CDLL("./libpsk.so")
CHAR_CB = ctypes.CFUNCTYPE(None, ctypes.c_char, ctypes.c_void_p)

lib.psk_tx_create.argtypes = [ctypes.c_uint, ctypes.c_float, ctypes.c_int]
lib.psk_tx_create.restype  = ctypes.c_void_p
lib.psk_tx_write.argtypes  = [ctypes.c_void_p, ctypes.c_char_p, ctypes.c_size_t]
lib.psk_tx_write.restype   = ctypes.c_size_t
lib.psk_tx_read.argtypes   = [ctypes.c_void_p, ctypes.POINTER(ctypes.c_float), ctypes.c_size_t]
lib.psk_tx_read.restype    = ctypes.c_size_t
lib.psk_tx_idle.argtypes   = [ctypes.c_void_p]
lib.psk_tx_idle.restype    = ctypes.c_int
lib.psk_tx_destroy.argtypes = [ctypes.c_void_p]
```
