# Fujitsu ASTG18KMCA → Home Assistant via ESPHome

Replacing an IR-blaster setup with a wired interface that gives bidirectional
state (changes made on the IR remote must show up in HA).

## Target hardware

- **Indoor unit:** Fujitsu General ASTG18KMCA (~5.0kW wall-mounted, KM series, Australia)
- **Currently:** IR control only, no wired wall controller installed
- **Home Assistant** already in use for automation

## Approach chosen

Tap the Fujitsu 3-wire remote-controller bus (12V / LIN / GND). It is
electrically LIN-like: 12V rail, shared single-wire bus, recessive high /
dominant low. The protocol on top is Fujitsu proprietary — **500 baud, 8E1**.

- **Firmware component:** https://github.com/Omniflux/esphome-fujitsu-halcyon
  - Proper ESPHome `external_components` source, requires ESPHome 2026.3.0+
  - Exposes climate entity + diagnostics (connected, standby/defrost, error
    code, filter timer, louver control); can feed an external HA temperature
    sensor back to the unit
  - `controller_address: 0` (primary) — we have no wired wall controller.
    Fujitsu supports max 2 controllers on the bus.
- **Reference hardware:** https://github.com/FOSV/Fuji-Atom-Interface
  (source of the original design; see "Schematic analysis" below)
- Older/unmaintained: unreality/FujiHeatPump, FujiHeatPump/esphome-fujitsu

### Rejected alternatives

| Option | Why not |
|---|---|
| Official Wi-Fi adapter (UTY-TFNXZ3) + FGLair | Cloud-only via Ayla, laggy, HACS custom component |
| Intesis INMBSFGL001R000 (Modbus) / WMP Wi-Fi | Works and is local, but AU$300–450 |
| Keep IR + receive IR codes | Only catches remote commands, not actual unit state |

## OPEN QUESTION — resolve before spending money

**Does the KMCA expose the 3-wire bus?**

Fujitsu's Oceania optional-parts list shows KM-series wall units needing a
communication kit for 3-wire controllers ("UTY-RVNYN + UTY-XWNX",
"UTY-RSNYN + UTY-XWNX").

Action: breaker off, front cover off, control box open. Look for a spare
multi-pin CN header (often 5-pin, only 3 used — the official Wi-Fi adapter
shorts pins 4+5 as a presence detect) or an existing red/white/black harness
stub.

- Red = 12V, White = Signal (LIN), Black = COM/GND
- If nothing: UTY-XWNX (~AU$13–20 from Singh's Aircon or Wholesale Aircon).
  Note their listings state ASTG30/34CMTA compatibility, not KMCA — confirm
  by phone before ordering.

## Schematic analysis (FOSV V2 R1.5)

The zip in the FOSV repo contains the **full Altium source** (.SchDoc,
.PcbDoc, .PrjPcb) plus a schematic PDF — not gerbers only. KiCad can import it.

Traced from the schematic:

- **Power:** TPS82140 buck, 12V in, EN tied to VIN. R1/R2 set the rail —
  133k/24.9k against the 0.8V feedback = 5.07V (V2 board). R2 = 42.2k gives
  3.32V (V1 board). That divider is the *only* difference between V1 and V2.
  Output feeds J2 pin 3 → module 5V pin. R4 (500R) + green LED tapped by the
  buck's PG pin = power-good indicator.
- **Transceiver (MCP2025-330E/MD):** VBB→12V, VSS+EP→GND, LBUS→bus wire,
  TXD/RXD→J1 pins 2,3.
- **Key finding:** pin 8 VREG (internal 3.3V LDO) feeds a small local rail
  that does only three things — decoupled by C3 (10µF), pulls RESET high via
  R3 (10.7k), holds CS/LWAKE high for permanent normal mode. **VREG does not
  power the MCU.** The buck does. J1 pin 1 (module 3V3) is unconnected.

**Consequence:** the design uses almost none of the MCP2025's features (no
sleep, no wake-on-bus, no fault monitoring, no external reliance on its LDO).
A replacement transceiver needs only:

1. 12V-tolerant open-drain bus driver + comparator receiver
2. **3.3V-compatible TXD/RXD logic** ← the real constraint
3. An enable/mode pin strappable high
4. A supply for that enable pin

Dropping a transceiver without an LDO costs nothing: delete C3 and R3, tie
enable to 3.3V.

## Design decisions

- **MCP2025-330E/MD is obsolete.** DigiKey: no longer manufactured, 0 stock.
  Listed substitutes MCP2021P-330E/MD and MCP2021P-500E/MD also 0 stock.
  LCSC out of stock. Mouser won't ship to region. Broker/AliExpress stock
  carries counterfeit risk.
- FOSV suggest TJA1029TK/3V3J as a drop-in but **have not tested it**, and
  the footprints differ: MCP2025 is 8-DFN 4×4, TJA1029TK is HVSON8 3×3.
- **Chosen replacement: TLE7259-3GE** (Infineon, SOIC-8, explicitly targets
  3.3V MCUs). SOIC-8 is hand-solderable → no stencil, no hotplate.
  Cheaper alternatives TJA1021T / TJA1027 (~$1) if the logic-supply
  arrangement checks out.
- **MCU: Seeed XIAO ESP32-C6.** Chosen for optional Thread later; running
  Wi-Fi first. C6 requires the ESP-IDF framework (Arduino doesn't support the
  chip). The Omniflux example config already uses esp-idf. Has u.FL for an
  external antenna — useful inside a sheet-metal chassis.
- **Thread/Matter status:** ESPHome OpenThread since 2025.6, matured through
  2026 releases. ESPHome's Matter integration was still experimental as of
  2025.6, wrapping sensor/switch/binary_sensor only; thermostat device-type
  coverage was tracked for mid-2026 — verify current status. Known issue:
  C6 + ESPHome OpenThread forming its own partition and detaching the border
  router (esphome#15645, ESPHome 2026.3.3, April 2026) — check if fixed.
  Since HA speaks the ESPHome native API directly, Matter adds nothing here;
  Thread is only a transport swap and the device is mains-powered anyway.

## Build plan

### Phase 1 — protoboard (~AU$45), this IS a valid finished product

| Item | Qty | Source | ~AUD |
|---|---|---|---|
| Seeed XIAO ESP32-C6 | 1 | Core Electronics | 14 |
| TLE7259-3GE SOIC-8 | 2 | element14 / Mouser | 6 |
| SOIC-8 → DIP-8 adapter | 5pk | AliExpress | 3 |
| Buck module 12V→5V (mini-360 / MP1584) | 2 | AliExpress / Jaycar | 3 |
| JST ZH 1.5mm 3-pin + pigtail kit | 1 | AliExpress | 5 |
| Perfboard 5×7cm | 5pk | AliExpress | 3 |
| 100µF 25V electrolytic | few | Jaycar | 2 |
| 100nF ceramics, 10k resistors | few | Jaycar | 2 |
| Headers, hookup wire | — | on hand | 3 |
| Small ABS enclosure | 1 | Jaycar | 5 |

Wiring (get pin numbers from the TLE7259-3GE datasheet, wire by function):

| Function | To |
|---|---|
| VS | 12V from aircon |
| GND | common |
| LIN | bus wire (white) |
| TXD | XIAO D6 |
| RXD | XIAO D7 |
| EN | XIAO 3V3 pin via 10k |
| INH/WK if present | per datasheet |

Plus 100nF at the transceiver supply pin, **100µF across the 12V input**
(the bus was designed for a wall thermostat drawing tens of mA, not a Wi-Fi
radio with 300mA TX bursts).

Steps:

1. Solder transceiver to DIP adapter. Check for bridges with a loupe.
2. **Set buck output to 5.0V before connecting the XIAO.** These ship with
   the trimmer anywhere.
3. Assemble on perfboard, XIAO on female headers.
4. Flash and verify over USB with nothing connected to the aircon. Cold
   ESP-IDF builds take ~20 min.
5. Add the Fujitsu component, reflash.
6. Connect to aircon with breaker OFF.
7. **Unplug USB before running from aircon 12V.**

Starting config (verify platform name/keys against the component README, and
D6/D7 GPIO numbers against Seeed's pinout):

```yaml
esp32:
  board: seeed_xiao_esp32c6
  framework:
    type: esp-idf

external_components:
  - source: github://Omniflux/esphome-fujitsu-halcyon

uart:
  id: fuji_uart
  tx_pin: GPIO16   # D6
  rx_pin: GPIO17   # D7
  baud_rate: 500
  parity: EVEN
```

**Success criteria:** climate entity in HA, connected = true, initialisation
complete, and IR remote changes reflected in HA within seconds.

Then: exercise every mode/setpoint/fan/swing; check the "Supported Features"
text sensor; confirm the IR remote still works alongside; test power-cycle
ordering (ESP should be powered before or with the AC per FOSV; error codes
need a power cycle to clear). If feature probing misbehaves, set
`autoconf: false` and specify `supported_modes` etc. manually. If frames look
wrong, enable the TZSP option and decode in Wireshark with the Fujitsu
dissector.

### Phase 2 — PCB (optional, ~AU$115 for 5 boards)

Only if the protoboard annoys you. Nobody sells these assembled — checked
Sept 2026, there was an unanswered request on the HA forum in May 2026.

Changes from FOSV V2:

- XIAO ESP32-C6 footprint replacing Atom Lite headers
- TLE7259-3GE (SOIC-8) replacing MCP2025; **delete C3 and R3**; strap EN
- Replace TPS82140 (~$7, a third of board cost) with **AP63205WU-7**
  (SOT-23-6, fixed 5V, 3.8–32V in, 2A, ~$0.40) + 1 inductor + 2 caps.
  Fixed output ⇒ **R1/R2 disappear entirely**, along with the 0.1% resistors.
- Single rail: 5V to the XIAO's 5V pin, then take 3.3V back out of the XIAO's
  3V3 pin to reference the transceiver EN. Do NOT inject 3.3V into the 3V3
  pin — bypasses the onboard regulator and conflicts with USB.
- Add PTC fuse + TVS on the bus side (shared bus with an expensive appliance)
- 100µF bulk cap on 12V
- Antenna keepout under the module; u.FL pigtail clearance
- Mounting holes

Print the layout 1:1 and check fit physically before ordering. JLCPCB, no
stencil needed. Global Standard Direct shipping (~$12, 2–3 weeks) vs DHL (~$35).

Bench bring-up: assemble with no module fitted, power from current-limited
12V, verify 5V rail and sane current draw, then fit the XIAO, loop TX→RX to
confirm the UART, then connect to the aircon.

### Phase 3 — install

Enclosure inside the indoor unit if there's room, away from the heat
exchanger and drain path; otherwise the wall cavity behind. Strain-relief the
harness. Label the board with the repo URL and date. Keep the IR blaster
wired for a month as fallback.

## Notes

- All prices are indicative AUD as of Sept 2026, unverified, ±30%.
  Re-check at order time.
- Verify every footprint against its datasheet — that's where these projects die.
