# Phase 1 protoboard layout — XIAO ESP32-C6 + TLE7259-3GE

Companion to the "Phase 1 — protoboard" section of `CLAUDE.md`. That section
gives the wiring *by function* (TXD → D6, etc.) and says "get pin numbers
from the TLE7259-3GE datasheet". This document fills in the actual pin
numbers, lays out where each part sits on the perfboard, and gives a
point-to-point wire list. Build two identical boards per the BOM quantities
(2× transceiver, 2× buck module) unless you only want one unit now.

![Protoboard placement map](phase1-protoboard-layout.svg)

## 1. TLE7259-3GE pinout (verified)

`CLAUDE.md` deliberately left this as a placeholder. Pulled from Infineon's
TLE7259-3GE product brief block diagram (Order No. B124-H9854-X-X-7600,
11/2013) — SOIC-8 / PG-DSO-8, pin 1 marked by the dot on the package:

| Pin | Name | Function here |
|---|---|---|
| 1 | RxD | → XIAO **D7** (UART RX, GPIO17) |
| 2 | EN | Mode select → 10 kΩ pull-up to XIAO **3V3** (forces Normal mode) |
| 3 | WK | Local wake input, **unused** → 10 kΩ pull-down to GND (see §5 caveat) |
| 4 | TxD | ← XIAO **D6** (UART TX, GPIO16) |
| 5 | GND | Common ground |
| 6 | BUS | LIN bus → white wire to aircon |
| 7 | VS | 12 V supply (shared with buck module input) |
| 8 | INH | Regulator-enable output, **not used** → leave open (NC) |

This part is the "high-end" LIN transceiver variant with sleep/wake and
fault-reporting features (per the design notes in `CLAUDE.md`, none of that
is needed here — EN is simply strapped to force Normal mode permanently).

Sources: [Infineon TLE7259-3GE product brief](https://www.infineon.com/dgdl/TLE7259-3GE_PB.pdf) (block diagram, page 2).
I only had the product brief, not the full datasheet's electrical/application
section — cross-check the WK and INH handling below against the full
datasheet before first power-up.

## 2. XIAO ESP32-C6 pins used

| XIAO pin | GPIO | Function |
|---|---|---|
| D6 | GPIO16 | UART TX → TLE7259 pin 4 (TxD) |
| D7 | GPIO17 | UART RX ← TLE7259 pin 1 (RxD) |
| 3V3 | — | Reference for EN pull-up (R2) — do **not** feed power in here |
| 5V | — | ← buck module output |
| GND | — | Common ground |

Pin *names* (D6/D7/3V3/5V/GND) are silkscreened on the XIAO board — wire to
those labels directly rather than trusting the left/right layout drawn in
the diagram above (I could confirm the pin **functions** D6=TX/D7=RX from
Seeed's docs, but not their physical left-vs-right side from what I could
fetch, so the diagram's exact header side is illustrative).

## 3. Board zones (left → right)

Rationale for the placement in the diagram:

1. **J3 (bus connector) + C1 (100 µF bulk cap)** at the edge closest to the
   aircon harness — the bulk cap catches ripple right where the 12 V enters,
   before it reaches either the transceiver or the buck converter's
   switching input.
2. **U1 (TLE7259-3GE)** immediately next to J3, so the LIN bus trace/wire
   stays as short as possible — it's the most noise-sensitive net on the
   board. R2 (EN pull-up), R3 (WK pull-down) and C2 (100 nF VS↔GND
   decoupling, right at the IC's pins 5/7) sit next to it.
3. **U2 (buck converter)** placed away from U1 and from the LIN wire —
   switching regulators are the main EMI source on this board and you don't
   want that coupling into the bus signal. Leave it accessible: you need a
   screwdriver on the trimpot *before* the XIAO is ever plugged in.
4. **XIAO ESP32-C6** on the far edge, on female headers (never solder it
   directly — you'll want to pull it for reflashing/debugging), with the
   USB-C connector overhanging the board edge so you can plug in USB without
   unmounting the board from its enclosure.

## 4. Point-to-point wire list

Perfboard has no copper strips, so every net below is a hand-run wire (or a
shared row of header pins acting as a rail). Group by net, use the legend
colour if you have the wire on hand — it makes later debugging much easier.

| Net | Colour | From | To |
|---|---|---|---|
| 12V | Red | J3 pin 1 | C1 (+) |
| 12V | Red | C1 (+) | U1 pin 7 (VS) |
| 12V | Red | C1 (+) | U2 IN+ |
| GND | Black | J3 pin 3 | C1 (−) |
| GND | Black | C1 (−) | U1 pin 5 (GND) |
| GND | Black | U1 pin 5 | U2 IN− |
| GND | Black | U2 IN− | XIAO GND |
| GND | Black | R3 (WK pull-down) | GND rail |
| LIN | White | J3 pin 2 | U1 pin 6 (BUS) |
| 5V | Orange | U2 OUT+ | XIAO 5V |
| TXD | Yellow/brown | U1 pin 4 (TxD) | XIAO D6 |
| RXD | Green | U1 pin 1 (RxD) | XIAO D7 |
| EN | Purple | XIAO 3V3 | R2 → U1 pin 2 (EN) |
| — | — | U1 pin 8 (INH) | *not connected* |
| — | — | J3 pin 2 (WK path) | *n/a, WK only ties to R3 above* |

Passives: R2 = R3 = 10 kΩ; C2 = 100 nF ceramic at U1's VS/GND pins; C1 =
100 µF/25V electrolytic at the 12V entry. Add a second 100 nF across the
XIAO's 5V/GND pins if you have spares — cheap insurance for the Wi-Fi TX
current bursts `CLAUDE.md` already flags as the reason for the 100 µF bulk
cap.

## 5. Open items / assumptions to confirm

- **WK pin (3):** tied to GND through a 10 kΩ resistor rather than left
  floating, on the general LIN-transceiver principle that an unused active
  wake input should be held in a defined state. I did not have the full
  TLE7259-3GE datasheet's application section to confirm Infineon's
  recommended treatment for this pin when local wake is unused — check it
  before powering the board from the aircon bus. If the datasheet calls for
  a pull to VS instead, swap R3's other end accordingly.
- **INH pin (8):** left open. It's a push-pull output (drives an external
  regulator enable), not an input, so leaving it unconnected is safe
  regardless of the above — flagging it only so you don't assume it needs a
  pull resistor like EN/WK do.
- **XIAO header physical side layout** in the SVG (which pins are drawn on
  the "top" vs "bottom" row) is the common XIAO-family convention, not
  independently confirmed against the ESP32-C6 variant's silkscreen — wire
  by pin *label*, not by matching the picture's geometry.

## 6. Build sequence (bench order, matches `CLAUDE.md` §Phase 1 steps 1-4)

1. Solder U1 onto the SOIC-8→DIP-8 adapter; inspect for solder bridges
   between the tight SOIC pitch pins with a loupe before it goes anywhere
   near the board.
2. Populate C1, R2, R3, C2 and the DIP adapter onto the perfboard per the
   zone map, then the female headers for the XIAO — **XIAO itself stays out
   of its socket for now.**
3. Wire the 12V/GND/LIN net from J3 through to U1 and U2's inputs per §4.
4. Power U2 from a bench supply (or the aircon 12V with the bus wire
   disconconnected) and **trim it to exactly 5.00 V into a multimeter before
   the XIAO ever goes in the socket** — these buck modules ship with the pot
   anywhere.
5. Socket the XIAO, wire D6/D7/3V3/5V/GND per §4, then flash and verify over
   USB with the aircon bus still disconnected (per `CLAUDE.md`: cold
   ESP-IDF builds take ~20 min).
6. Only once USB-side bring-up is clean, connect J3 to the aircon with the
   breaker off, then power up. Unplug USB before running from aircon 12V.

Repeat for the second board if building both now.
