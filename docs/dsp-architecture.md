# DSP Architecture & Internals

This document explains the internal signal processing algorithms, module boundaries, filter parameters, and known limitations in `libpsk`.

---

## Module Map

| Source Files | Responsibility |
| :--- | :--- |
| [`include/psk.h`](../include/psk.h) | Public C99 API declarations (`psk_mode_t`, TX, RX, Varicode). |
| [`src/tx.c`](../src/tx.c) | Text/bit ring buffers, TX state machine, differential encoder, raised half-cosine amplitude shaper, carrier NCO. |
| [`src/rx.c`](../src/rx.c) | Full receiver chain: power/SNR estimators, BPF, NCO mixer, I/Q LPF, inlined post-LPF Costas loop, symbol decision, Varicode accumulator. |
| [`src/varicode.c`](../src/varicode.c) | 128-entry G3PLX ASCII Varicode lookup table and bit encoder/decoder. |
| [`src/fir.h`](../src/fir.h), [`src/fir.c`](../src/fir.c) | Direct Form II Transposed biquad IIR filter primitives (RBJ constant-peak-gain BPF, 2nd-order Butterworth LPF, and cascaded N-th order Butterworth LPF). |
| [`src/timing.h`](../src/timing.h), [`src/timing.c`](../src/timing.c) | Non-data-aided Gardner Timing Error Detector (TED) and 1st-order symbol clock recovery loop. |
| [`src/viterbi.h`](../src/viterbi.h), [`src/viterbi.c`](../src/viterbi.c) | Rate-1/2, K=5 convolutional encoder (`G1=0x17`, `G2=0x19`) and 16-state soft-decision Viterbi decoder (traceback depth 32). |
| [`src/costas.h`](../src/costas.h), [`src/costas.c`](../src/costas.c) | Standalone 2nd-order Costas loop helper (compiled into `libpsk.a`; `rx.c` uses an inlined post-LPF variant instead — see below). |

---

## Mode Parameters

Defined identically in `MODE_TABLE[6]` (`src/tx.c`) and `RX_MODE[6]` (`src/rx.c`):

| Mode | Baud Rate | `sps` at 8000 Hz | `sps` at 12000 Hz | BPF BW (`8×baud`) | Baseband LPF `fc` (`1.5×baud`) | Costas `loop_bw` (`0.05×baud`) |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| `BPSK31` / `QPSK31` | 31.25 | 256 | 384 | 250 Hz | 46.875 Hz | 1.5625 Hz |
| `BPSK63` / `QPSK63` | 62.50 | 128 | 192 | 500 Hz | 93.75 Hz | 3.125 Hz |
| `BPSK125` / `QPSK125` | 125.00 | 64 | 96 | 1000 Hz | 187.5 Hz | 6.25 Hz |

Samples per symbol is computed as `sps = (int)(sample_rate / baud + 0.5f)`. Note that `8000 Hz` and `12000 Hz` divide cleanly by `31.25`, `62.5`, and `125.0` with zero rounding error.

---

## Transmitter Signal Chain (`src/tx.c`)

```text
ASCII text (256-byte ring buffer)
   │
   ▼
psk_varicode_encode()  ──► [codeword bits][0][0]
   │
   ├── (BPSK) ──► 32-bit staging FIFO
   └── (QPSK) ──► psk_conv_encode (K=5, rate 1/2) ──► 16-bit block interleaver ──► 32-bit FIFO
   │
   ▼
TX State Machine (IDLE → PREAMBLE[32] → DATA → POSTAMBLE[32] → IDLE)
   │
   ▼
Differential Phase Mapping (BPSK: bit 0 → π flip, bit 1 → hold)
   │
   ▼
Raised Half-Cosine Envelope Shaping (amp(k) = cur_amp_sign · cos(π·k / sps))
   │
   ▼
Carrier NCO Multiplication: out[n] = amp · cos(carrier_phase)
```

### State Machine & Framing

1. **`TX_STATE_IDLE`**: Outputs `0.0f`. Transitions to `TX_STATE_PREAMBLE` as soon as `psk_tx_write()` places at least one byte in the text buffer.
2. **`TX_STATE_PREAMBLE`**: Transmits `PREAMBLE_SYMS = 32` consecutive `0`-bits (continuous phase inversions every symbol). This produces the classic PSK31 two-tone idling spectrum (`carrier ± baud/2`) that lets a remote receiver acquire carrier phase and symbol timing before the first character arrives. At the final preamble symbol boundary, `tx.c` immediately pops the first data bit to avoid inserting a spurious `1`-bit at the preamble/data boundary.
3. **`TX_STATE_DATA`**: Refills the 32-bit staging queue from the text ring buffer whenever `< 18` bits are buffered (`TX_BIT_BUF - 14`), modulating one bit (BPSK) or dibit (QPSK) per symbol. When both buffers empty, transitions to `TX_STATE_POSTAMBLE`.
4. **`TX_STATE_POSTAMBLE`**: Sends `32` trailing phase-reversal symbols so remote demodulators and Viterbi/filter pipelines can flush the final character cleanly, then returns to `TX_STATE_IDLE`.

### Raised Half-Cosine Amplitude Shaping

Abrupt 180° phase flips generate wide key-click sidebands. To confine BPSK31 to ~31 Hz bandwidth, `src/tx.c` shapes every phase-reversing symbol across its `sps` samples (`k = 0 .. sps - 1`):

- **No phase change (`bit == 1`):**
  `amp(k) = cur_amp_sign` (constant `+1.0f` or `-1.0f`)
- **Phase reversal (`bit == 0`):**
  `amp(k) = cur_amp_sign * cosf(π * k / sps)`

Because `cos(0) = +1`, `cos(π/2) = 0`, and `cos(π) = -1`:

- The envelope starts at `cur_amp_sign` (continuous with the previous symbol),
- Passes smoothly through zero at the exact symbol midpoint (`k = sps / 2`), and
- Reaches `-cur_amp_sign` at the end of the symbol, where `cur_amp_sign` is negated for the next symbol.

The final sample is `amp * cosf(carrier_phase)`, keeping peak amplitude strictly `<= 1.0f`.

---

## Receiver Signal Chain (`src/rx.c`)

```text
PCM float32 x[n]
   │──────────────────────────────► Wideband power EMA (noise_power, τ ≈ 0.1 s)
   ▼
2nd-Order Biquad BPF (center = carrier_hz, BW = 8×baud)
   │──────────────────────────────► Passband power EMA (sig_power, τ ≈ 0.1 s)
   ▼
Complex NCO Downconverter:
   I_raw =  bp · cos(nco_phase)
   Q_raw = -bp · sin(nco_phase)
   │
   ▼
1st-Order Baseband I/Q LPF (fc = 1.5×baud) ──► I_filt, Q_filt
   │                                                │
   │  ◄──── Costas Loop PI Filter (Kp, Ki) ◄────────┘
   │        (error computed POST-LPF, clamp ±0.03 rad/sample)
   ▼
Gardner Timing Recovery (psk_timing_step) ──► Symbol Instants (I_sym, Q_sym)
   │
   ▼
Phase Detector & Differential Decoder:
   phase = atan2f(Q_sym, I_sym)
   dp    = wrap_pi(phase - prev_phase)
   ├── (BPSK) bit = (|dp| < π/2) ? 1 : 0
   └── (QPSK) dibit = round(dp / (π/2)) mod 4 ──► Viterbi (16-state, depth 32)
   │
   ▼
Varicode Accumulator (vacc_t with 1-bit pending_zero lookahead)
   │
   ▼
char_cb(c, userdata)   &   periodic status_cb(snr_db, freq_offset_hz, userdata)
```

### 1. Biquad & Cascaded Filters (`src/fir.c`)

All IIR sections use **Direct Form II Transposed** arithmetic (`psk_bq_step`) for numerical stability in single-precision `float`:

- **Pre-filter BPF (`psk_bq_init_bpf`):** Constant-peak-gain bandpass from the Robert Bristow-Johnson Audio EQ Cookbook with `alpha = sin(w0) * bw / (2 * f0)` (`Q = f0 / bw`) and `-3 dB` bandwidth `8 × baud`. At BPSK31 this gives a 250 Hz pre-filter window, rejecting adjacent stations without inducing severe ringing or ISI.
- **Baseband I/Q LPF (`psk_bqc_init_lpf`):** Supports up to 8th-order (`4` cascaded biquads) Butterworth lowpass, with odd orders terminating in a bilinear-transformed 1st-order section (`wc = tan(π * fc / fs)`). `rx.c` instantiates a **1st-order** LPF (`n = 1`) at `fc = 1.5 × baud` on both I and Q arms to reject out-of-band noise and `2 × carrier_hz` mixer images while maintaining near-zero group delay and minimal inter-symbol interference.

### 2. Post-LPF Costas Carrier Recovery Loop (`src/rx.c` & `src/costas.c`)

The Costas loop is **inlined in `src/rx.c`** specifically so the phase error detector operates on `(I_filt, Q_filt)` **after** the baseband LPF rather than on raw mixer outputs `(I_raw, Q_raw)`. This prevents double-frequency mixer products from jittering the loop filter.

- **Phase Error Detector:**
  - BPSK: `err = sign(I_filt) * Q_filt`
  - QPSK: `err = sign(I_filt) * Q_filt - sign(Q_filt) * I_filt`
- **Critically-Damped PI Loop Filter (`ζ = 1/√2 ≈ 0.7071068`):**
  - `Bn = (0.05 * baud) / sample_rate`
  - `denom = 1 + 2 * ζ * Bn + Bn²`
  - `Kp = 4 * ζ * Bn / denom`
  - `Ki = 4 * Bn² / denom`
  - `costas_int += Ki * err`
  - `nco_freq = clamp(costas_int + Kp * err, -0.03, +0.03)` rad/sample (`±38.2 Hz` pull-in/tracking limit at 8000 Hz)

> **Note on `src/costas.c`:** The standalone `psk_costas_*` module is compiled into `libpsk.a` as an internal component, but `rx.c` does not call it because `psk_costas_step()` mixes and evaluates the error before external lowpass filtering (and accumulates `nco_freq += integrator + Kp * err` instead of assigning `nco_freq = integrator + Kp * err`).

### 3. Gardner Symbol Timing Recovery (`src/timing.c`)

Uses a non-data-aided **Gardner Timing Error Detector** (Gardner 1986) operating on the filtered baseband `(I_filt, Q_filt)` stream:

- A fractional `clock_phase` increments by `1 / sps` per sample.
- When `clock_phase` crosses `0.5`, `(mid_I, mid_Q)` captures the mid-symbol sample.
- When `clock_phase` reaches `1.0`, the symbol instant `(I_sym, Q_sym)` is emitted and the timing error is computed:
  `err = (prev_I - I_in) * mid_I + (prev_Q - Q_in) * mid_Q`
- Clock phase wraps by `-1.0f` and is nudged by `K_timing * err` where `K_timing = 0.01f / sps`, clamped symmetrically to `[-0.5f, +0.5f]` so the loop can both advance and retard symbol sampling instants.

### 4. Differential Phase Decoding & Varicode Framing

Because BPSK is differentially encoded, carrier lock with a 180° Costas ambiguity has **no effect** on decoded bits:

1. `phase = atan2f(Q_sym, I_sym)`
2. `dp = wrap_to_pi(phase - prev_phase)`
3. BPSK bit decision: `|dp| < π/2 → 1` (no phase change); otherwise `0` (phase reversal).

Bits enter `vacc_t` (`src/rx.c`), which exploits two structural properties of G3PLX Varicode:

- Every valid codeword begins with `1` and ends with `1`.
- No codeword contains two consecutive `0` bits (`00` appears exclusively as the inter-character delimiter).

`vacc_t` uses a 1-bit `pending_zero` lookahead flag:

- Leading `0` bits before the first `1` (such as preamble/postamble idle) are ignored (`active == 0`).
- A single `0` after `active == 1` sets `pending_zero = 1`.
- If the next bit is `1`, the pending `0` is committed into the shift register followed by `1`.
- If the next bit is `0`, `00` delimiter is complete: `psk_varicode_decode()` looks up the accumulated codeword (up to 14 bits), emits the character if non-NUL, and resets `vacc_t`.
- If `len > 14` (e.g., noise burst), `vacc_t` resets automatically to prevent overflow.

### 5. SNR & Power Estimator (`src/rx.c`)

Two single-pole exponential moving averages with time constant `τ = 0.1 s` (`alpha = 1 - exp(-1 / (0.1 * fs))`) track signal and noise power:

- `noise_power`: tracks `x²` (wideband input power before BPF).
- `sig_power`: tracks `bp²` (passband power after BPF).
- Estimated SNR: `10 * log10f(sig_power / (noise_power - sig_power))`, defaulting to `-99.0f` if `sig_power <= 1e-12f` or `noise_power <= sig_power`.

---

## Varicode Encoding (`src/varicode.c`)

`varicode_tbl[128]` stores the public-domain G3PLX PSK31 Varicode table as minimal `uint16_t` integers whose bit length is `floor(log2(code)) + 1` (range `1–10` bits, plus `2` separator bits = `3–12` bits per character). High-frequency English characters use the shortest codewords:

| Character | ASCII | Hex Entry | Codeword (MSB first) | Wire Bits (with `00`) |
| :--- | :---: | :---: | :--- | :--- |
| Space (`' '`) | `0x20` | `0x0001` | `1` | `100` (3 bits) |
| `'e'` | `0x65` | `0x0003` | `11` | `1100` (4 bits) |
| `'t'` | `0x74` | `0x0005` | `101` | `10100` (5 bits) |
| `'o'` | `0x6F` | `0x0007` | `111` | `11100` (5 bits) |
| `'a'` | `0x61` | `0x000B` | `1011` | `101100` (6 bits) |
| `'E'` | `0x45` | `0x0077` | `1110111` | `111011100` (9 bits) |
| `NUL` | `0x00` | `0x02AB` | `1010101011` | `101010101100` (12 bits) |

---

## QPSK FEC and Implementation Status

`src/viterbi.c` implements a rate-1/2, constraint-length `K = 5` convolutional codec:

- **Generator Polynomials:** `G1 = 0x17` (`10111₂`), `G2 = 0x19` (`11001₂`).
- **Decoder:** 16-state (`2^(K-1)`) Euclidean-distance soft-decision Viterbi decoder with circular traceback depth `PSK_VITERBI_DEPTH = 32` (`6.4 × K`) and per-step minimum-metric subtraction to avoid float overflow.

### Known Limitations in QPSK Modes

If you plan to use or finish QPSK31/QPSK63/QPSK125, note the current gaps in the implementation:

1. **Simplified QPSK TX Modulation (`src/tx.c` lines 350–357):** Although `push_char()` runs bits through `psk_conv_encode()` and a 16-bit (depth-8 dibit) block interleaver and `qpsk_diff_encode()` computes a 4-phase state (`0°/90°/180°/270°`), the sample generator currently maps `new_abs != 0.0f` onto a single-axis BPSK `phase_change` flag rather than true quadrature I/Q raised-cosine modulation (`I(k) * cos(φ) - Q(k) * sin(φ)`).
2. **Missing RX De-interleaver (`src/rx.c` lines 337–359):** `rx.c` passes differentially decoded dibits directly into `psk_viterbi_decode()` without reversing the depth-8 block interleaver applied in `tx.c`.
3. **Hard-Quantized Viterbi Inputs:** `rx.c` quantizes phase deltas to `0..3` and passes `±1.0f` into `psk_viterbi_decode()` instead of continuous soft-decision LLRs from the complex phase vector.
4. **Encoder State Ordering Check:** In `psk_conv_encode()`, `c->sr` is shifted before computing `word = (bit << 4) | c->sr`, whereas `build_trellis()` computes `word = (b << 4) | s` using the pre-shift state `s` — verify alignment when completing full QPSK interoperability.

Because of these items, **BPSK31, BPSK63, and BPSK125** are the production-ready modes today, while QPSK modes are currently covered only by API lifecycle smoke tests (`c_tests/test_modes.c`).

---

## Memory Footprint Breakdown

Approximate static/heap sizes per context (32-bit/64-bit platforms):

| Context | Key Fields | Approximate Size |
| :--- | :--- | :---: |
| `psk_tx_t` | `text[256]`, `bits[32]`, `il_buf[16]`, NCO/shaping scalars | ~360 B |
| `psk_rx_t` | `bpf` (1 biquad), `lpf_I`/`lpf_Q` (4-stage containers each), `timing`, `vacc`, `viterbi` (`tb_state[32][16]` + `tb_bit[32][16]` + `path_metric[16]` + `out_buf[32]` ≈ 1.15 KB) | ~1.5 KB |

Both contexts stay well below the `< 4 KB` embedded target (suitable for ESP32-S3 SRAM).
