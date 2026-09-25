# BMW-i3-SBOX
Reverse engineering of the BMW i3 EV HV contactor box (SBOX) for use in EV conversion projects.

Part number worked here: **61278648904** (supersedes 61278635849 and 61277631110, 2014–2021 i3 / i3s).

This is **not** the PHEV SBox used in the 330e / 530e. Do not send PHEV `0x100` / `0x300` contactor frames at this unit. On the i3 private bus, `0x100` is a voltmeter, not a coil command.

<img width="4096" height="2304" alt="sboxonbench" src="https://github.com/user-attachments/assets/29b625ea-692f-41de-bc9c-d615abcd308e" />

## Hardware

Pack-internal box: Panasonic main +/− contactors, precharge relay and resistor, PEC ~350 A / 450 V fuse, current and voltage sensing, small MCU that talks to the SME on a private Local-CAN.

Intended conversion use with ZombieVerter:

- Drive the three 12 V coils from the VCU (box “bare”).
- Read pack voltage, output voltage and current from the SBOX Local-CAN.
- SME is optional. It is not required for the measurement frames.

## White 12-pin connector (SBOX ↔ SME / VCU)

The SBOX connects to the SME (BMS) via a white 12 pin connector.

| Pin | Colour | Function |
| --- | --- | --- |
| 1 | yellow | Positive contactor +12 V supply |
| 2 | black | Positive contactor 12 V negative. Connect to pos contactor pin on ZombieVerter |
| 3 | grey | Precharge relay +12 V supply |
| 4 | green | Precharge relay 12 V negative. Connect to precharge pin on ZombieVerter |
| 5 | blue | Negative contactor +12 V supply |
| 6 | brown | Negative contactor 12 V negative. Connect to neg contactor pin on ZombieVerter |
| 7 | orange | +12 V supply to SBOX electronics |
| 8 | violet | Ground |
| 9 | pink | CAN Low. Connect to ZombieVerter shunt CAN. **SBOX has 120 R termination** |
| 10 | yellow | CAN High. Connect to ZombieVerter shunt CAN. **SBOX has 120 R termination** |
| 11 | — | No connection |
| 12 | — | No connection |

SBOX Local-CAN is **500 kbit/s**.

## Local-CAN — who talks

Bench logs with the SME present and with the SBOX powered alone:

| ID | Source | Period | DLC |
| --- | --- | --- | --- |
| `0x100` | SBOX | ~2.1 ms | 8 |
| `0x110` | SBOX | ~2.1 ms | 8 |
| `0x120` | SBOX | ~2.1 ms | 8 |
| `0x130` | SBOX | ~2.1 ms | 8 |
| `0x135` | SBOX | ~1.03 s | 8 |
| `0x050` | SBOX | ~20.6 ms | 6 |
| `0x140` | SBOX | ~20.6 ms | 8 |
| `0x170` | SBOX | ~103 ms | 8 |
| `0x210` | SBOX | ~1.03 s | 8 |
| `0x150` | SME only | ~20 ms | 4 |
| `0x160` | SME only | ~1 s | 4 |

The SBOX emits the measurement IDs with only +12 V on pin 7 / 8. No SME keepalive is required to read HV data.

SME-only frames seen so far:

- `0x150` : constant `FD F4 40 A7`
- `0x160` : constant `00 00 00 0E`

## Common 2 ms frame layout

`0x100`, `0x110`, `0x120` and `0x130` share the same packing:

| Bytes | Meaning |
| --- | --- |
| B0–B3 | Little-endian analog value. Voltages fit in 16 bits; current uses signed 16 bits with sign-extended high bytes |
| B4 | Alive counter in the high nibble, `0x00, 0x10, … 0xF0`, then wrap |
| B5 | Flags. `0x00` = valid analog. On `0x130`, `0x80` marks a sentinel frame — discard the analog |
| B6 | Changes with the counter (CRC input, not the measurement) |
| B7 | CRC. Poly not identified yet |

About every tenth `0x130` has `B5 = 0x80` and a garbage payload. Same cadence as `0x050` / `0x140`. Ignore those frames.

## Encodings (bench)

Values from 0–30 V PSU sweeps on each HV side and 0–10 A current sweeps through the closed **negative** contactor. Contactores otherwise open. No traction load.

| ID | Signal | Decode | Evidence |
| --- | --- | --- | --- |
| `0x100` | Pack / battery-side voltage | `u16le(B0,B1) / 1000` → volts | Three ramps on battery posts, peak **31552 mV**. Stays ~0.1 V when the PSU is on the output posts |
| `0x110` | Vehicle / output-side voltage | `u16le(B0,B1) / 1000` → volts | Follows output-side PSU ramps. Does not follow pack-side ramps. **Peaks at ~17.5 V** in every 30 V output sweep so far — scale or sense-point still to confirm with a meter on the output busbars |
| `0x130` | Pack current | `i16le(B0,B1) / 1000` → amps, **only if B5 == 0x00** | 10 A from battery side peaked **+10447**. 10 A from car side peaked **−10305**. Idle / voltage-only logs stay within about ±0.2 A |
| `0x120` | Output-side companion | signed LE | Tracks output voltage on V sweeps. Not traction current. Do not map to `idc` |

Current sign from the 10 A tests:

- Current injected from the **battery** side of the closed negative pole → `0x130` **positive**
- Current injected from the **car** side → `0x130` **negative**

Working current scale is **1 count ≈ 1 mA**. A clamp meter on the same cable at the top of a ramp will confirm or nudge the factor.

## Other SBOX frames (not fully decoded)

| ID | Notes |
| --- | --- |
| `0x050` | Almost always `40 54 FF FF DF FF` |
| `0x140` | After wake, `01 00 FF FF` plus alive / CRC. Looks like “alive / measurement valid” |
| `0x135` | ~1 Hz. B0 ticks by 1 per second (session timer) |
| `0x170` | ~100 ms. `00 00 00 xx 50 81 05` plus CRC |
| `0x210` | Static `75 69 00 05 06 5F 01 03` |

## ZombieVerter scratch map

```text
udc2 = u16le(id 0x100 B0-B1) / 1000.0    // pack V
udc  = u16le(id 0x110 B0-B1) / 1000.0    // output V, confirm 30 V peak
idc  = i16le(id 0x130 B0-B1) / 1000.0    // amps; skip frame if B5 & 0x80
alive = (id 0x100 B4 >> 4) incrementing
```

Existing `bmw_sbox.cpp` (`ShuntType=2`, PHEV IDs `0x100` / `0x300`) will not speak this protocol. A new shunt class is required.

Contactors: VCU GPIO on pins 2 / 4 / 6 as above. Do not expect Local-CAN frames to close the Panasonic pair.

## Logs

Raw Savvy-style CSV captures are in [`CAN_Logs/`](CAN_Logs/).

| File | Setup |
| --- | --- |
| `sbox_and_sme_coldstart1.csv` | SME + SBOX, cold start |
| `sboxandsme_sweep0_30v_3timesbattside.csv` | SME + SBOX, 0–30 V on battery posts, three ramps |
| `sboxandsme_sweep0_30v_3timescarside.csv` | SME + SBOX, 0–30 V on output posts, three ramps |
| `sbox_onown_coldstart1.csv` | SBOX alone, cold start |
| `sboxonown_0_30v_2timescarside.csv` | SBOX alone, 0–30 V on output posts |
| `sboxonown_0_10AmpSweep_frombattside.csv` | SBOX alone, neg contactor closed, 0–10 A from battery side |
| `sboxonown_0_10AmpSweep_fromcarside.csv` | SBOX alone, neg contactor closed, 0–10 A from car side |

## Still open

- CRC polynomial on B7 (usual CRC8 polys over B0–B6 did not match; ID may be in the input).
- Why `0x110` tops out near 17.5 V on every output-side 30 V sweep.
- Exact `0x120` meaning.
- Isolation / weld-detect / heater PWM (present on some 904 units).
- Full SME wake and any contactor commands the SME would send if we kept it in the loop.
