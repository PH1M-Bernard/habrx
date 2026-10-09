# HABRX / HAB Air Protocol Specification

**Status:** Draft v0.1  
**Purpose:** Self-discovering, compact, robust HAB telemetry protocol  
**Scope:** Calling/discovery channel, flight/session identity, air-profile advertisement, telemetry packet families, encoding rules, FEC framing, and receiver behavior

---

## 1. Design goals

The protocol is intended for amateur high-altitude-balloon telemetry and should provide:

1. robust discovery with no per-flight receiver configuration;
2. very compact frequent position packets;
3. less-frequent state and system packets;
4. human-readable flight metadata sent periodically rather than in every frame;
5. multiple modulation/air profiles per flight;
6. support for a fast primary link plus a slower robust fallback link;
7. operation with inexpensive RTL-SDR receivers;
8. tolerance of frequency offset and slow oscillator drift;
9. clean versioning and future extensibility;
10. compatibility with rate-1/2 FEC as a baseline;
11. independence between modulation and telemetry semantics;
12. no requirement for a central allocator of flight IDs.

---

## 2. Terminology

### 2.1 Calling channel

A fixed, well-known robust radio channel used only for flight discovery/bootstrap information.

### 2.2 Working/main channel

The RF frequency on which a flight transmits normal telemetry.

### 2.3 Flight Token

A short session identifier for a specific flight. It is not a permanent payload identity and may be reused by later unrelated flights.

The token is intended to be chosen/derived independently without central coordination.

### 2.4 Air Profile

An immutable registry entry defining the physical-layer waveform parameters needed to transmit and demodulate a working-channel frame. This includes modulation family, symbol rate, BT where applicable, deviation or tone spacing, symbol mapping, pulse shaping, preamble/acquisition guidance, and recommended receive filtering.

Air Profiles do **not** define CRC/FEC/whitening/interleaving. Those are defined by a separate Coding Profile.

### 2.5 Coding Profile

An immutable registry entry defining packet protection/coding behavior such as CRC, whitening, convolutional or other FEC, puncturing, termination/tail-biting, interleaving, and coding bit order.

### 2.6 Air Profile Role

A small scheduling/UI hint associated with an advertised Air/Coding pair, such as PRIMARY or ROBUST. The role does not define the waveform or coding.

### 2.7 Telemetry Protocol Version

A version defining telemetry packet interpretation (POSITION, STATE, FLIGHT_INFO, SYSTEM_STATUS) independently of Air and Coding Profiles.

### 2.8 Flight Info

Human-readable metadata mapping a Flight Token to callsign, flight name, and optional description.

---

## 3. Layering

The protocol is conceptually separated into three layers:

```text
1. DISCOVERY
   Where is the flight and how do I demodulate it?

2. PACKET/PROTOCOL DESCRIPTION
   Which protocol/profile/version is in use?

3. TELEMETRY
   Position, state, system status, and flight metadata.
```

Modulation SHALL be independent from telemetry packet semantics.

A POSITION packet transmitted using PRIMARY and the same POSITION packet transmitted using ROBUST represent the same semantic message.

---

## 4. Calling/discovery channel

### 4.1 Fixed behavior and default Discovery PHY

The calling channel SHALL use one fixed, robust, receiver-known Discovery PHY for each Discovery Protocol version. The calling PHY parameters MUST NOT depend on per-flight discovery information or on the Air/Coding Profile registries.

Discovery Protocol v1 defines the following default PHY:

| Parameter | Discovery v1 default |
|---|---|
| Modulation | 4FSK, unshaped |
| Symbol rate | 100 symbols/s |
| Raw bit rate | 200 bit/s |
| Adjacent tone spacing | 270 Hz |
| Preamble | repeating four-tone sequence `0,1,2,3` |
| Preamble length | 32 symbols |
| Sync length | 32 bits / 16 4FSK symbols |
| Sync value | fixed protocol constant, TBD before golden-vector freeze |
| Payload | fixed 16-byte DISCOVERY v1 payload |
| CRC | CRC-16; exact variant TBD before golden-vector freeze |
| FEC | convolutional K=7, rate 1/2, generators `171/133` octal |
| FEC termination | terminated with 6 zero tail bits |
| Whitening | off |

The four frequency states are centered on the configured calling frequency. With 270 Hz adjacent spacing they are nominally:

```text
-405 Hz, -135 Hz, +135 Hz, +405 Hz
```

The preferred Gray-like dibit mapping is:

```text
00 -> -405 Hz
01 -> -135 Hz
11 -> +135 Hz
10 -> +405 Hz
```

The exact 32-bit sync constant and CRC-16 variant remain to be selected algorithmically/through golden-vector work, but their widths and positions are fixed for Discovery v1.

This PHY is intentionally Horus-like at the modem/acquisition layer, but the DISCOVERY payload, CRC/FEC framing, and sync are HABRX-specific. A standard Horus Binary decoder is therefore not expected to decode HABRX DISCOVERY packets without HABRX protocol support.

### 4.2 Purpose

Calling-channel frames advertise enough information for a receiver to create or update a working-channel decoder.

At minimum, a discovery announcement must convey:

- discovery protocol version;
- Flight Token;
- main/working RF frequency;
- one or more supported Air/Coding Profile pairs;
- telemetry protocol profile/version;
- optional schedule hints.

### 4.3 Calling interval

A nominal interval around two minutes is proposed after normal flight acquisition.

The actual interval SHOULD include jitter to reduce persistent collisions between independently operating balloons.

Example concept:

```text
nominal = 120 s
actual = nominal +/- pseudo-random jitter
```

The jitter MAY be deterministically derived from the Flight Token and frame counter/time.

### 4.4 Initial acquisition cadence

Flights MAY advertise more frequently during startup/launch and reduce discovery cadence later.

Example policy, not normative:

```text
first 10 minutes: approximately every 30 s
afterward:        approximately every 120 s
```

### 4.5 Same-SDR simultaneous capture

The recommended deployment model is to keep normal working frequencies close enough to the calling frequency that a common RTL-SDR capture can contain both simultaneously.

The exact recommended offset window is TBD, but approximately +/-500 kHz is a candidate design target.

Receivers SHALL nevertheless permit hardware retuning when an announced working channel falls outside the current capture span.

### 4.6 Receiver configuration policy

The calling channel SHALL be standardized by default but configurable at the receiver. Normal users should select the standard Discovery v1 profile and normally change only parameters that are operational rather than protocol-defining.

Standard receiver configuration SHOULD expose:

```text
calling frequency
acquisition search width
SDR device/sample rate/gain
enable/disable discovery
```

Recommended initial acquisition search width is **+/-5 kHz**, with **+/-10 kHz** available as an advanced option. The final recommended regional default calling frequency is intentionally TBD and MUST remain user-configurable.

Advanced/experimental mode MAY override protocol PHY parameters such as symbol rate, tone spacing, preamble length, sync word, and FEC. Such overrides are non-standard unless they exactly reproduce a defined Discovery Protocol version.

The UI SHOULD present a simple default such as:

```text
Discovery profile: Standard HABRX v1
Calling frequency: <regional/default configurable value>
Search width:      +/-5 kHz
```

with deeper PHY parameters hidden behind an advanced/experimental control.

---

## 5. Flight Token

### 5.1 Semantics

The Flight Token identifies a flight/session, not permanent hardware.

It can be reused by another flight at another time.

The token should be large enough that independently chosen simultaneous flights have a low collision probability without a central coordinator.

### 5.2 Width

DISCOVERY v1 fixes the Flight Token at **10 bits**, giving 1024 possible values. Working-channel Telemetry Protocol v1 is expected to use the same 10-bit token.

### 5.3 Generation

Recommended methods include:

- random token chosen before/at launch; or
- a short hash of callsign + launch timestamp + flight name.

The generation method is not transmitted and is not required for decoding.

### 5.4 Collision handling

A token collision is possible by design. Receivers and remote storage systems SHOULD use additional evidence such as RF frequency, Flight Info, timing, and geography to distinguish conflicting sessions where necessary.

Future protocol versions MAY extend identity handling if operational experience shows a need.

---

## 6. Air and Coding Profile registries

### 6.1 Design rule: separate waveform from coding

The protocol SHALL use two independent registries:

```text
Air Profile ID     -> how the RF waveform is generated/demodulated
Coding Profile ID  -> how the payload is protected/encoded
```

A transmitter advertises a pair `(Air Profile ID, Coding Profile ID)`. The same Air Profile may therefore be tested with multiple coding schemes without defining duplicate RF profiles, and the same Coding Profile may be used across 2GFSK, 4GFSK, GMSK, or future modulation families.

### 6.2 Immutability and registry versioning

Profile IDs are globally stable within the HABRX protocol. Once a profile definition is published for interoperability, that ID MUST NOT be reassigned or redefined. A corrected or improved definition receives a new ID.

The registry documents/software MAY have release versions (for example `Air Registry 1.3`), but registry version numbers are not transmitted on air. Older receivers simply report unknown profile IDs as unsupported.

During this Draft v0.1 phase, candidate allocations remain provisional until their complete waveform/coding parameters and golden vectors are frozen.

### 6.3 Air Profile contents

An Air Profile SHALL fully define all physical-layer parameters needed for modulation and demodulation, including as applicable:

```text
profile ID
canonical name
status (draft/experimental/stable/deprecated; registry metadata only)
modulation family
symbol rate
bits per symbol
BT / pulse shaping
modulation index or deviation
adjacent tone spacing for multi-level FSK
symbol-to-frequency mapping
frequency polarity/convention
preamble pattern/profile
minimum preamble length
recommended receive bandwidth
recommended output samples/symbol after channelization
```

The status is not transmitted over the air.

### 6.4 Coding Profile contents

A Coding Profile SHALL fully define:

```text
profile ID
canonical name
status (registry metadata only)
CRC algorithm and exact parameters
whitening/scrambling, if any
FEC family
constraint length / generator polynomials where applicable
code rate and puncturing pattern
termination or tail-biting behavior
interleaving, if any
bit ordering through the coding chain
```

### 6.5 Registry namespaces

Suggested Air Profile namespace:

| ID range | Purpose |
|---|---|
| `0x00` | invalid / unassigned |
| `0x01-0x0F` | standardized 2GFSK family |
| `0x10-0x1F` | standardized GMSK/MSK family |
| `0x20-0x2F` | standardized 4GFSK/4FSK family |
| `0x30-0x7F` | future standardized waveform families |
| `0x80-0xEF` | experimental profiles |
| `0xF0-0xFF` | private/local profiles |

Suggested Coding Profile namespace:

| ID range | Purpose |
|---|---|
| `0x00` | invalid / unassigned |
| `0x01-0x0F` | uncoded/diagnostic coding profiles |
| `0x10-0x3F` | convolutional coding profiles |
| `0x40-0x5F` | LDPC family |
| `0x60-0x7F` | other future standardized FEC |
| `0x80-0xEF` | experimental profiles |
| `0xF0-0xFF` | private/local profiles |

### 6.6 Initial Si4060-oriented Air Profile candidates

The first experimental transmitter uses an Si4060-C2A. The initial candidate registry therefore concentrates on native 2GFSK/4GFSK while also including the GMSK configuration that has shown strong low-signal decoding performance in controlled testing. These allocations are provisional until all waveform constants and golden vectors are frozen.

| Candidate ID | Name | Definition | Intended role |
|---|---|---|---|
| `0x01` | `2GFSK-100-BT05` | 2GFSK, 100 sym/s, 100 bit/s, BT=0.5, deviation/h TBD | EXTREME ROBUST |
| `0x02` | `2GFSK-200-BT05` | 2GFSK, 200 sym/s, 200 bit/s, BT=0.5, deviation/h TBD | EXTREME ROBUST alternative |
| `0x11` | `GMSK-1200-H05-BT05` | GMSK, 1200 sym/s, 1200 bit/s, h=0.5, BT=0.5 | NORMAL / weak-signal baseline |
| `0x21` | `4GFSK-1200-BT05` | 4GFSK, 1200 sym/s, 2400 raw bit/s, BT=0.5, adjacent tone spacing TBD | FAST / PRIMARY candidate |
| `0x22` | `4GFSK-2400-BT05` | 4GFSK, 2400 sym/s, 4800 raw bit/s, BT=0.5, adjacent tone spacing TBD | FAST high-rate experimental |

The initial three-tier design target is therefore:

```text
FAST            4GFSK, high rate, frequent
NORMAL          GMSK 1200, h=0.5, BT=0.5
EXTREME ROBUST  2GFSK 100 or 200, infrequent
```

The final 4GFSK symbol mapping SHALL be fixed in the profile definition. A Gray-like mapping is preferred so adjacent frequency states differ by one bit.

### 6.7 Initial Coding Profile candidates

The initial coding registry is intentionally small:

| Candidate ID | Name | Definition | Intended use |
|---|---|---|---|
| `0x01` | `CRC16-ONLY` | CRC-16, no FEC | diagnostics/test only |
| `0x10` | `K7-R12-TERM` | CRC-16 + K=7 convolutional rate 1/2, terminated | interoperability candidate |
| `0x11` | `K7-R12-TAIL` | CRC-16 + K=7 convolutional rate 1/2, tail-biting | preferred baseline candidate |
| `0x12` | `K7-R12-TAIL-WHITE` | as `0x11` plus whitening | experimental |
| `0x13` | `K7-R23-TAIL` | punctured K=7 nominal rate 2/3, tail-biting | experimental |
| `0x14` | `K7-R34-TAIL` | punctured K=7 nominal rate 3/4, tail-biting | experimental |

For the K=7 family, the intended generator pair is the conventional octal `171/133`. Exact CRC-16 parameters, puncturing patterns, whitening polynomial, and coding bit order remain TBD until golden vectors are frozen.

The first flight should preferentially test `Coding 0x11` across the FAST/PRIMARY, NORMAL/AUXILIARY, and EXTREME-ROBUST profile candidates.

### 6.8 Air-profile roles

The 2-bit role field used by DISCOVERY is:

```text
00 PRIMARY
01 ROBUST
10 EXPERIMENTAL
11 AUXILIARY
```

Role is a scheduling/UI hint. Receivers MUST use the Air/Coding Profile IDs, not the role, to determine how to demodulate/decode a frame.

The descriptive tier names **FAST**, **NORMAL**, and **EXTREME ROBUST** are design/UI terminology rather than additional on-air role values. In the initial three-tier mapping, FAST normally uses role `PRIMARY`, NORMAL normally uses `AUXILIARY`, and EXTREME ROBUST uses `ROBUST`.

---

## 7. Multiple modulations per flight

A flight SHALL be allowed to use multiple Air Profiles. DISCOVERY v1 advertises one main working frequency shared by all advertised profile slots; future Discovery versions MAY add per-profile frequencies if operational experience justifies it.

The first design target is therefore one announced main frequency with multiple time-multiplexed profiles. The initial three-tier strategy is:

```text
FAST
  4GFSK
  high symbol rate
  frequent short telemetry

NORMAL
  GMSK 1200
  h=0.5, BT=0.5
  strong low-signal decoding baseline

EXTREME ROBUST
  2GFSK 100 or 200
  infrequent long-range/fallback telemetry
```

A receiver that cannot decode FAST may still track the balloon through NORMAL, and a marginal receiver may still obtain periodic fixes through EXTREME ROBUST.

The EXTREME ROBUST profile SHOULD periodically carry enough information to be useful independently, especially POSITION and enough identity metadata to associate the flight.

The receiver SHOULD NOT blindly classify modulation. Discovery tells it which profiles exist and their nominal intervals. Modem instances should be activated only when needed or around learned/expected transmission windows.

---

## 8. Telemetry packet type field

The current design uses a 2-bit telemetry packet type, giving four primary families:

| Value | Type | Purpose |
|---|---|---|
| `00` | POSITION | latitude, longitude, altitude, position validity |
| `01` | STATE | temperature, battery, satellites, other slowly changing state |
| `10` | FLIGHT_INFO | callsign, flight name, optional description |
| `11` | SYSTEM_STATUS | watchdog/reset/fault and system-health information |

Packet type semantics are independent of Air Profile.

---

## 9. Common telemetry header

All working-channel telemetry packets SHOULD begin with enough information to identify their packet family and flight/session.

Current logical proposal:

```text
Packet Type     2 bits
Flight Token    10 bits
```

Additional common version/profile bits MAY be added if required by final framing.

Exact byte alignment is intentionally not frozen in v0.1.

---

## 10. POSITION packet

### 10.1 Purpose

POSITION is the most frequent telemetry packet and must remain compact.

Recommended nominal cadence for PRIMARY is approximately every 5 seconds.

ROBUST may transmit POSITION less frequently, for example every two minutes.

### 10.2 Fields

Current field allocation:

| Field | Bits | Encoding |
|---|---:|---|
| Latitude | 23 | quantized -90..+90 deg |
| Longitude | 24 | quantized -180..+180 deg |
| Altitude | 15 | unsigned metres, 0..32767 m |
| GPS position valid | 1 | boolean |
| GPS 3D fix | 1 | boolean |

Including the 2-bit packet type and a 10-bit Flight Token, the current logical size is:

```text
2 + 10 + 23 + 24 + 15 + 1 + 1 = 76 bits
```

before any additional versioning/sequence/CRC/FEC fields.

### 10.3 Latitude encoding

Latitude uses an unsigned 23-bit quantized value across the legal latitude range.

Recommended mapping:

```text
lat_q = round((latitude_deg + 90) * (2^23 - 1) / 180)
```

Decode:

```text
latitude_deg = lat_q * 180 / (2^23 - 1) - 90
```

Resolution is approximately 0.00002146 degrees, about 2.4 m north/south.

### 10.4 Longitude encoding

Longitude uses an unsigned 24-bit quantized value across the legal longitude range.

Recommended mapping:

```text
lon_q = round((longitude_deg + 180) * (2^24 - 1) / 360)
```

Decode:

```text
longitude_deg = lon_q * 360 / (2^24 - 1) - 180
```

Resolution is approximately 0.00002146 degrees, about 2.4 m at the equator.

### 10.5 Altitude encoding

Altitude is unsigned metres:

```text
0      -> 0 m
1      -> 1 m
...
32767  -> 32767 m
```

15 bits provide 32768 distinct values. The maximum exact representable altitude is therefore 32767 m.

A future protocol profile may extend altitude if flights above this ceiling are required.

### 10.6 Position-validity flags

GPS position validity and 3D-fix state are carried with POSITION rather than only in SYSTEM_STATUS because they qualify the position sample itself.

---

## 11. STATE packet

### 11.1 Purpose

STATE carries low-rate operational measurements that do not need to be repeated in every position packet.

Recommended nominal cadence is approximately once per minute.

### 11.2 Fields

Current field allocation:

| Field | Bits | Encoding |
|---|---:|---|
| Temperature | 7 | 1 degree C resolution |
| Battery | 5 | 1.0..3.2 V, 0.1 V resolution |
| Satellites | 5 | 0..30, reserved invalid/unknown value |

With type + 10-bit Flight Token:

```text
2 + 10 + 7 + 5 + 5 = 29 bits
```

before CRC/FEC and future additions.

### 11.3 Temperature encoding

Current proposed mapping:

```text
0   -> -80 C
1   -> -79 C
...
126 -> +46 C
127 -> invalid/unavailable
```

Exact minimum/maximum may be revisited before final v1 freeze.

### 11.4 Battery encoding

Current proposed mapping:

```text
0  -> 1.0 V
1  -> 1.1 V
...
22 -> 3.2 V
23..30 reserved
31 -> invalid/unavailable
```

### 11.5 Satellite count

Current proposed mapping:

```text
0..30 -> satellite count
31    -> invalid/unknown
```

---

## 12. FLIGHT_INFO packet

### 12.1 Purpose

FLIGHT_INFO maps the compact Flight Token to human-readable flight metadata and removes the need for a central token allocator.

Recommended nominal cadence is approximately every two minutes, potentially more frequently near startup.

### 12.2 Required fields

A FLIGHT_INFO packet SHOULD include:

```text
Flight Token
Callsign
Flight name
Optional description
```

Example semantic content:

```text
Flight Token: 19
Callsign: PH1M
Flight name: Suncatcher
Description: balloon with new payload
```

A receiver can then display later compact telemetry as belonging to:

```text
PH1M / Suncatcher
```

### 12.3 Variable length

FLIGHT_INFO MAY be variable length because it is infrequent.

The exact string encoding is TBD.

Likely design options include:

- length-prefixed IA5/ASCII subsets;
- constrained callsign character set;
- compact UTF-8 only if truly needed.

### 12.4 Proposed limits

Candidate implementation limits:

```text
Callsign:     <= 10 characters
Flight name:  <= 20 characters
Description:  <= 63 characters
```

These limits are not yet normative.

### 12.5 Metadata independence

A receiver SHALL be able to retain and display token-only telemetry before FLIGHT_INFO arrives.

Remote upload adapters that require human-readable identity MAY defer uploading until FLIGHT_INFO resolves the token.

---

## 13. SYSTEM_STATUS packet

### 13.1 Purpose

SYSTEM_STATUS carries health, reset, watchdog, and fault information that does not belong in every POSITION or STATE frame.

It may be sent periodically and SHOULD also be sent soon after an important system-state change.

### 13.2 Candidate flags

Initial candidates include:

```text
watchdog occurred
brownout occurred
software reset occurred
GPS restarted
GPS fault
radio fault
low battery
reserved / future
```

### 13.3 Latched flags

Important one-time events such as watchdog reset SHOULD be latched so that they remain set long enough to be observed even if the first SYSTEM_STATUS packet is missed.

The exact clearing policy is implementation/profile-defined and MUST be specified before final v1.

### 13.4 Bitmap width

An 8- or 16-bit system bitmap is currently preferred, but the final width is TBD.

---

## 14. Packet sequence / time identity

No final decision has been made on an explicit sequence counter.

A small modulo sequence number may be useful for:

- packet-loss statistics;
- duplicate detection;
- diversity reception analysis;
- stable frame fingerprints.

Candidate width: 5 bits.

The sequence value may be derived from GNSS time when available, for example a modulo function of 5-second epochs, but this does not eliminate the need to transmit it if receivers cannot independently infer it.

This field remains TBD for v0.1.

---

## 15. Packet framing

### 15.1 Logical processing order

The final exact processing order is TBD, but the design SHALL explicitly define and freeze the following before v1:

```text
semantic fields
   -> bit packing
   -> CRC
   -> optional whitening
   -> FEC
   -> optional interleaving
   -> preamble/sync/framing
   -> modulation
```

If a different order is chosen, it must be documented unambiguously.

### 15.2 Bit and byte order

The final specification SHALL define:

- field bit order;
- byte order;
- bit numbering convention;
- CRC input order;
- FEC bit order;
- whitening bit order;
- interleaver order.

No implementation may rely on host-language struct layout.

---

## 16. CRC

A CRC SHALL protect the packet before accepting semantic telemetry.

CRC-16 is the current design preference.

Exact polynomial, initial value, reflection rules, and final XOR are TBD and SHALL be frozen with golden vectors before v1.

CRC validity SHALL be the definitive acceptance criterion after FEC decode.

---

## 17. Coding/FEC baseline

### 17.1 Baseline

The preferred initial coding candidate is `Coding Profile 0x11`: K=7 convolutional rate 1/2 with tail-biting, preceded by CRC-16.

The K=7 generator pair is intended to be octal `171/133`.

### 17.2 Termination alternatives

`Coding Profile 0x10` exists as a terminated K=7 rate-1/2 alternative for interoperability/comparison. Tail-biting is preferred for the compact HAB packets because it avoids the six termination input bits of a conventional K=7 encoder.

### 17.3 Puncturing

Experimental Coding Profiles may define punctured rates such as 2/3 or 3/4. The exact puncturing matrices are not yet frozen.

### 17.4 Whitening

Whitening is independent of the Air Profile and, when used, is part of the Coding Profile. The initial preferred baseline has whitening disabled until controlled RF tests demonstrate a benefit.

### 17.5 Soft decoding

Receivers SHOULD use soft-decision Viterbi decoding where practical.

---

## 18. Preamble and sync

### 18.1 Purpose

Preamble/sync is used for:

- packet detection;
- symbol timing acquisition;
- frequency-offset estimation;
- modem/profile discrimination where useful.

It is NOT the primary flight identity.

### 18.2 Frequency-balanced preamble

Preambles SHOULD exercise modulation states symmetrically to support robust CFO estimation.

For 2GFSK, an alternating pattern such as:

```text
01010101...
```

is a strong candidate.

For 4GFSK, a pattern that repeatedly exercises all four symbols is preferred, for example conceptually:

```text
0 1 2 3 0 1 2 3 ...
```

The actual optimized patterns are TBD.

### 18.3 Profile distinction

Different Air Profiles MAY use distinct sync/preamble signatures if this improves rapid profile detection.

---

## 19. Frequency offset and drift handling

### 19.1 Receiver requirement

The announced working frequency is nominal, not guaranteed exact.

Receivers SHALL tolerate modest initial CFO and slow frequency drift.

### 19.2 Recommended receiver strategy

```text
wideband IQ
   -> nominal DDC placement
   -> coarse search / energy or preamble detection
   -> fine preamble-assisted CFO estimate
   -> demodulation
   -> slow AFC / drift tracking
   -> retain estimate for next packet
```

### 19.3 Per-flight state

Receivers SHOULD retain:

```text
nominal frequency
last measured CFO
estimated CFO drift
last acquisition time
```

and use these estimates to accelerate later acquisition.

### 19.4 BT knowledge

BT may improve matched filtering and advanced demodulation performance. It SHOULD therefore be known to the receiver through the selected Air Profile rather than necessarily as a separate discovery field.

---

## 20. DISCOVERY v1 packet

### 20.1 Fixed-size payload

DISCOVERY v1 uses a fixed **16-byte (128-bit) payload**, followed by **CRC-16 (16 bits)**. Therefore the DISCOVERY message is exactly **18 bytes before calling-channel FEC/framing**.

The calling-channel PHY itself is fixed by Discovery Protocol v1 and is not selected through the Air/Coding registries.

### 20.2 Exact payload bit layout

Bits are listed logically from bit 0 through bit 127. Final on-air bit/byte ordering still must be frozen in the common framing rules and golden vectors.

| Bits | Width | Field |
|---|---:|---|
| `0..2` | 3 | Discovery protocol version |
| `3..5` | 3 | Telemetry protocol version |
| `6..15` | 10 | Flight Token |
| `16..32` | 17 | Signed main-frequency offset in 10 Hz units |
| `33..34` | 2 | Profile count encoding |
| `35..56` | 22 | Profile slot 0 |
| `57..78` | 22 | Profile slot 1 |
| `79..100` | 22 | Profile slot 2 |
| `101..122` | 22 | Profile slot 3 |
| `123..126` | 4 | Announcement counter |
| `127` | 1 | Reserved, transmit as 0 |

The following 16 CRC bits are not part of the 128-bit payload.

### 20.3 Flight Token

DISCOVERY v1 fixes the Flight Token width at **10 bits** (`0..1023`). It identifies a flight/session rather than permanent hardware and may be reused by later unrelated flights.

### 20.4 Main-frequency offset

The main working frequency is encoded relative to the fixed calling frequency:

```text
working_frequency_hz = calling_frequency_hz + offset_raw * 10
```

`offset_raw` is a signed 17-bit two's-complement integer. The representable range is therefore approximately:

```text
-655360 Hz .. +655350 Hz
```

The normal design target remains to keep working channels within roughly +/-500 kHz of the calling channel so a typical RTL-SDR can receive both simultaneously.

### 20.5 Profile count encoding

The two-bit field encodes the number of active slots as:

```text
00 = 1 profile
01 = 2 profiles
10 = 3 profiles
11 = 4 profiles
```

Unused slots MUST be transmitted as all zero bits and MUST be ignored by the receiver.

### 20.6 Profile slot layout

Each 22-bit slot contains:

| Width | Field |
|---:|---|
| 8 | Air Profile ID |
| 8 | Coding Profile ID |
| 2 | Role |
| 4 | Nominal interval class |

The role encoding is defined in section 6.8.

### 20.7 Nominal interval classes

The interval is a scheduling hint, not a strict timing promise. Initial mapping:

| Value | Nominal interval |
|---:|---:|
| 0 | unspecified / event-driven |
| 1 | 1 s |
| 2 | 2 s |
| 3 | 5 s |
| 4 | 10 s |
| 5 | 15 s |
| 6 | 30 s |
| 7 | 60 s |
| 8 | 120 s |
| 9 | 180 s |
| 10 | 300 s |
| 11 | 600 s |
| 12 | 900 s |
| 13 | 1800 s |
| 14 | 3600 s |
| 15 | reserved |

Receivers MAY use this value to schedule expensive secondary decoders around expected ROBUST/AUXILIARY transmissions, but SHALL tolerate jitter, reset-induced phase changes, and missed frames.

### 20.8 Announcement counter

The 4-bit announcement counter increments modulo 16 for each new DISCOVERY transmission. It assists duplicate/loss diagnostics but is not part of the flight identity.

### 20.9 Example

Conceptual example:

```text
Discovery version:   1
Telemetry version:   1
Flight Token:        19
Main offset:         +200000 Hz
Profile count:       3

Slot 0:
  Air Profile:       0x21  (4GFSK-1200-BT05 candidate)
  Coding Profile:    0x11  (K7-R12-TAIL candidate)
  Role:              PRIMARY
  Interval:          5 s

Slot 1:
  Air Profile:       0x11  (GMSK-1200-H05-BT05 candidate)
  Coding Profile:    0x11  (K7-R12-TAIL candidate)
  Role:              AUXILIARY
  Interval:          30 s

Slot 2:
  Air Profile:       0x01  (2GFSK-100-BT05 candidate)
  Coding Profile:    0x11  (K7-R12-TAIL candidate)
  Role:              ROBUST
  Interval:          120 s
```

### 20.10 Calling-channel PHY

The calling-channel PHY MUST be fixed and self-contained for each Discovery Protocol version. It MUST NOT depend on Air/Coding Profile registry lookup to decode DISCOVERY itself.

Discovery Protocol v1 uses the standard PHY defined in section 4.1:

```text
4FSK, unshaped
100 symbols/s
270 Hz adjacent tone spacing
32-symbol 0,1,2,3 repeating preamble
32-bit fixed sync constant
CRC-16 over the 16-byte DISCOVERY payload
K=7, rate-1/2 convolutional FEC, 171/133 octal
6 zero termination bits
whitening off
```

The fixed DISCOVERY body before FEC is 18 bytes = 144 bits. With 6 termination bits, the convolutional encoder processes 150 input bits and emits 300 coded bits. 4FSK carries two coded bits per symbol, so the coded body occupies 150 symbols = 1.50 s at 100 symbols/s.

Adding the 32-symbol preamble (0.32 s) and 16-symbol sync (0.16 s) gives approximately **1.98 s total Discovery-v1 airtime**, excluding any implementation-specific TX settling/guard interval.

The receiver SHALL treat the configured calling frequency as nominal. It SHOULD search at least +/-5 kHz by default, estimate the four tone centers/CFO from the balanced preamble, and continue with sync correlation and soft-decision FEC decoding. A wider +/-10 kHz search MAY be enabled in advanced mode.

The modem layer is intentionally Horus-like, but the frame is HABRX-specific and is not a Horus Binary telemetry frame.

---

## 21. Protocol description / profile advertisement on main channel

The main channel SHOULD periodically carry enough protocol/profile information that a receiver which has already switched to or digitally extracted that channel can continue operation without continuously monitoring the calling channel.

This may take the form of:

- a compact `PROTOCOL_INFO` extension;
- repeated Air Profile identifiers;
- version/profile bits in working-channel framing.

The exact mechanism is TBD.

The design goal is:

```text
CALL tells receiver where/how to start.
MAIN contains enough repeated information to stay synchronized and self-consistent.
```

---

## 22. Suggested transmission schedule

The following is the current three-tier design example and is not yet a strict wire-protocol requirement:

```text
FAST / PRIMARY (4GFSK)
  POSITION      every ~5 s
  STATE         may be carried periodically as convenient

NORMAL (GMSK 1200, h=0.5, BT=0.5)
  POSITION      every ~30 s
  may also carry periodic state/identity information

EXTREME ROBUST (2GFSK 100/200)
  POSITION      every ~120 s
  FLIGHT_INFO   periodically, perhaps every few robust cycles
  STATE         optionally less often

Common semantic cadence across profiles
  STATE         approximately every ~60 s
  FLIGHT_INFO   approximately every ~120 s
  SYSTEM_STATUS approximately every ~60 s and after important change
```

The calling-channel discovery beacon may also be approximately every 120 s with jitter.

Transmitters SHOULD avoid intentional overlap between time-multiplexed profiles. Receivers MAY learn the phase of periodic profile transmissions and activate the NORMAL/EXTREME ROBUST demodulators only around expected windows, while tolerating jitter and reset-induced schedule changes.

Exact scheduling remains transmitter policy rather than a strict wire-protocol constant.

---

## 23. Receiver behavior with multiple profiles

A receiver SHOULD create modem resources only when needed and SHOULD share one narrow DDC/channel extraction per flight when multiple time-multiplexed profiles use the same RF frequency.

Possible strategy:

```text
Always:
  CALL decoder

After discovery:
  create one FlightChannel/DDC for the announced main frequency
  keep FAST decoder active when appropriate
  enable NORMAL decoder around expected windows
  enable EXTREME ROBUST decoder around expected windows

After inactivity:
  destroy expensive DSP/modem resources
  retain lightweight FlightRecord and last CFO/drift estimates
```

The receiver SHOULD prioritize RF decoding over spectrum/waterfall work on resource-constrained systems. Blind modulation classification is not required.

A receiver MUST NOT assume that all profiles are present in every transmission interval.

---

## 24. Compatibility and versioning

### 24.1 Discovery protocol version

The 3-bit Discovery Protocol Version selects the fixed DISCOVERY payload/framing semantics. Version 1 is defined by section 20. Unsupported versions MUST be reported explicitly.

### 24.2 Telemetry protocol version

The 3-bit Telemetry Protocol Version identifies the interpretation of working-channel POSITION/STATE/FLIGHT_INFO/SYSTEM_STATUS packets. Changing a telemetry packet layout requires a new telemetry version, but does not require new Air or Coding Profiles.

### 24.3 Registry evolution

Air and Coding registries may receive new document/software releases without changing on-air version fields. Existing profile IDs never change meaning. Unknown IDs are unsupported, not guessed.

### 24.4 Independent evolution

These dimensions evolve independently:

```text
Discovery Protocol Version  -> bootstrap packet semantics
Telemetry Protocol Version  -> telemetry packet semantics
Air Profile ID              -> RF waveform
Coding Profile ID           -> CRC/FEC/whitening/interleaving
```

For example, a new 4GFSK Air Profile does not require a new Telemetry Protocol Version; a changed POSITION layout does not require a new Air Profile.

### 24.5 Unsupported profiles

A receiver that discovers an unsupported Air, Coding, or Telemetry profile/version SHOULD report this explicitly rather than silently ignoring the flight.

Example:

```text
Flight 19 discovered
Frequency: ...
Air Profile 0x47 unsupported
Coding Profile 0x11 supported
```

---

## 25. Error handling and malformed frames

All packet parsers SHALL treat RF input as untrusted/corruptible data.

Variable-length fields MUST be bounds-checked.

Unknown enum/profile values MUST NOT crash the receiver.

Invalid CRC frames MUST NOT update authoritative semantic flight state, though they MAY contribute diagnostic statistics.

---

## 26. Interoperability and golden vectors

Before declaring protocol v1 stable, publish golden vectors containing:

```text
semantic input values
exact packed pre-CRC bits/bytes
CRC value
post-CRC frame
FEC output bits
interleaved/whitened output if applicable
symbol sequence for selected Air Profile
```

At minimum, vectors SHALL cover:

- POSITION;
- STATE;
- FLIGHT_INFO with and without description;
- SYSTEM_STATUS;
- DISCOVERY;
- boundary values;
- invalid/reserved encodings.

The vectors must be sufficient for independent C, Rust, Python, embedded, and other implementations to prove compatibility.

---

## 27. Remote-service mapping

Remote upload formats are not part of the RF wire protocol.

The receiver SHALL decode RF packets into normalized events, then adapters map these events to services such as:

```text
SondeHub Amateur
BalloonSpotter
```

This avoids coupling the over-the-air protocol to a specific web service.

---

## 28. Privacy and station metadata

Receiver/station location is not part of the balloon RF protocol.

Ground receiver software may optionally provide station metadata to remote services, but this must remain separate from balloon telemetry semantics.

---

## 29. Items intentionally TBD in v0.1

The following are intentionally unresolved pending controlled RF testing and implementation experiments:

1. actual calling frequency;
2. exact 32-bit Discovery-v1 sync constant and CRC-16 variant (lengths and framing are fixed);
3. final recommended simultaneous-capture working-frequency window (DISCOVERY encoding itself is fixed at approximately +/-655 kHz);
4. final Air Profile parameters for the Si4060 candidates, especially 2GFSK deviation/modulation index and 4GFSK adjacent tone spacing/symbol mapping;
5. which candidate Air Profiles are promoted to STABLE;
6. exact CRC-16 parameters used by Coding Profiles;
7. exact puncturing patterns for Coding Profiles 0x13/0x14;
8. whitening polynomial/initialization for Coding Profile 0x12;
9. final working-channel Air Profile sync/preamble constants (Discovery-v1 preamble/sync lengths are fixed);
10. final working-channel packet sequence/time field;
11. final SYSTEM_STATUS bitmap width and clearing semantics;
12. exact FLIGHT_INFO string encoding;
13. exact main-channel protocol-info repetition mechanism;
14. final bit/byte ordering through CRC/FEC/framing, to be frozen with golden vectors.

The following items are now fixed for Draft DISCOVERY v1:

```text
16-byte fixed DISCOVERY payload
CRC-16 appended -> 18 bytes before calling-channel FEC
3-bit Discovery version
3-bit Telemetry version
10-bit Flight Token
17-bit signed main-frequency offset in 10 Hz units
4 fixed profile slots
separate 8-bit Air and Coding Profile IDs
2-bit profile role
4-bit nominal interval class
4-bit modulo-16 announcement counter
1 reserved payload bit
Discovery-v1 PHY: unshaped 4FSK, 100 sym/s, 270 Hz adjacent spacing
32-symbol repeating 0,1,2,3 preamble
32-bit sync length
K=7 rate-1/2 terminated FEC, 171/133 octal, whitening off
default receiver acquisition search width +/-5 kHz
```

These remaining TBD items do not block implementation of the overall architecture, registry system, DISCOVERY parser/serializer, replay tests, dynamic channel lifecycle, or web/application infrastructure.

---

## 30. Current proposed core field summary

### POSITION

```text
Packet type          2 bits
Flight Token        10 bits
Latitude            23 bits
Longitude           24 bits
Altitude            15 bits
GPS pos valid        1 bit
GPS 3D fix           1 bit
```

### STATE

```text
Packet type          2 bits
Flight Token        10 bits
Temperature           7 bits
Battery               5 bits
Satellites            5 bits
```

### FLIGHT_INFO

```text
Packet type          2 bits
Flight Token        10 bits
Callsign           variable
Flight name        variable
Description        optional variable
```

### SYSTEM_STATUS

```text
Packet type          2 bits
Flight Token        10 bits
Status bitmap      TBD
Optional status extensions TBD
```

---

## 31. Example conceptual flight

```text
Calling channel:
  Discovery v1
  4FSK, 100 sym/s, 270 Hz spacing
  32-symbol four-tone preamble
  32-bit sync
  terminated K7 rate-1/2 FEC

DISCOVERY v1 says:
  Discovery version: 1
  Telemetry version: 1
  Flight Token: 19
  Main frequency: Fc + 200 kHz
  Profiles: 3

  Slot 0:
    Air Profile:    0x21  4GFSK-1200-BT05 candidate
    Coding Profile: 0x11  K7-R12-TAIL candidate
    Role:           PRIMARY
    Interval:       5 s

  Slot 1:
    Air Profile:    0x11  GMSK-1200-H05-BT05 candidate
    Coding Profile: 0x11  K7-R12-TAIL candidate
    Role:           AUXILIARY / NORMAL
    Interval:       30 s

  Slot 2:
    Air Profile:    0x01  2GFSK-100-BT05 candidate
    Coding Profile: 0x11  K7-R12-TAIL candidate
    Role:           ROBUST
    Interval:       120 s

FAST traffic:
  POSITION approximately every 5 s

NORMAL traffic:
  GMSK POSITION approximately every 30 s

EXTREME ROBUST traffic:
  2GFSK POSITION approximately every 120 s
  periodic identity/state information

Common slower semantics:
  STATE every ~minute
  FLIGHT_INFO every ~2 minutes
  SYSTEM_STATUS periodically/on change
```

A receiver with enough capture bandwidth can monitor calling + main simultaneously. A narrower receiver may retune according to discovery.

---

## 32. Summary

The protocol uses a fixed robust Discovery-v1 calling PHY (unshaped 4FSK, 100 symbols/s, 270 Hz adjacent spacing) to discover flights, their working frequency, and advertised Air/Coding Profile pairs. Flight identity is session-oriented through a reusable short Flight Token supplemented by periodic human-readable FLIGHT_INFO packets. Telemetry is split into compact POSITION, STATE, FLIGHT_INFO, and SYSTEM_STATUS packet families. Multiple modulations per flight are explicitly supported through a three-tier FAST 4GFSK / NORMAL GMSK / EXTREME ROBUST 2GFSK strategy. A K=7 rate-1/2 Coding Profile with CRC protection is the preferred baseline candidate. Receiver calling frequency and acquisition search width are operationally configurable, while advanced overrides permit experimental non-standard discovery PHY settings.
