# Acrel AGF-AE-D — register map notes

The register table is [`Acrel_AGF-AE-D_register_map.csv`](Acrel_AGF-AE-D_register_map.csv),
87 registers covering addresses **40000–40121**. It is a SunSpec-shaped map: a two-register
header, then one fixed block.

Transcribed from Acrel's own SunSpec information model for the AGF-AE-D. Reformatted as CSV and
tidied (duplicated footer rows removed, `~` range separators repaired), but the register data is
unchanged. This is the client side of the MITM — the real meter this board polls.

## Reading the table

| column | meaning |
|---|---|
| `PLC Register` | 1-based, as most vendor docs print it |
| `Address` | 0-based, **the number you put in a config** — always `PLC Register − 1` |
| `SF` | names the register holding this value's scale factor |
| `Value` | a fixed value where the register has one, otherwise blank |

`acc32` entries occupy **two** registers, which is why the addresses step by 2 through the energy
accumulators.

## Three things that will catch you out

**1. `W_SF` is not constant.** The real-power scale factor at address 40022 switches between `0`
and `1` — coefficient 1 or 10 — depending on magnitude. You cannot cache it or assume it. The
same applies to `VA_SF` (40027) and `VAR_SF` (40032). Read the scale factor alongside the value,
every time.

**2. `VA_SF` has hysteresis, and it is wide.** It reads `1` above 32 kVA and `0` below 24 kVA,
and **between 24 and 32 kVA it holds its previous value**. So the same apparent power can carry
either scale factor depending on which direction you arrived from. `PF_SF` (40037) carries the
same note in the vendor table.

**3. This is a split-phase (ABN) meter.** The header ID is `202`, and every phase-C register —
`AphC`, `PhVphC`, `WphC`, `VAphC`, `VARphC`, `PFphC` and the phase-C energy accumulators — is
fixed at **0**. They exist in the map because the SunSpec model defines them, not because the
meter measures them.

> That last point matters for this project specifically: a non-zero `WphC` from the served side
> is a sign the MITM is emitting garbage, not a sign of a third phase.

## Writes

Write support is narrow, and worth knowing before assuming a register is settable:

* **Only function code `0x10`** (write multiple registers) is supported. Not `0x06`.
* **Energy counters are locked.** To modify or clear them, first write `0xa5a6` to address
  **40108**, which unlocks the operation. Then write `0xaa55` to address **40109** to clear all.
* Everything else in the table is read-only.

## Communication parameters

Held in the meter's own configuration registers rather than the SunSpec block:

* **Address and baud rate** share one register — high byte is the Modbus address (1–247), low
  byte is the baud rate: `2` = 9600, `3` = 4800, `4` = 2400, `5` = 1200.
* **Parity**: `0` = none, `1` = 2 stop bits, `2` = odd, `3` = even.

This installation runs the meter at **9600 8N1, address 2**.

## Event flags

`Evt` at address 40105 is a `bitfield32` for internal storage failure. The vendor table documents
it in pairs, where **both bits of a pair set** means a fault:

| bits | |
|---|---|
| 0–1 | setting parameters |
| 2–3 | setting parameters |
| 4–5 | setting parameters |

The remaining bits are not documented in the source material, and **none of this is confirmed
against hardware** — this config does not poll 40105 at all, so there is no observation either
way. Treat the table as the vendor's claim, not as something checked.
