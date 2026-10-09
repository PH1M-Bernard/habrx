# HABRX Architecture Specification

**Status:** Draft v0.1  
**Audience:** HABRX implementers, contributors, and reviewers  
**Primary implementation language:** Rust  
**Primary desktop shell:** Tauri  
**Primary headless interface:** Embedded web UI + HTTP/WebSocket API

---

## 1. Purpose

HABRX is a cross-platform receiver for high-altitude-balloon telemetry. It is designed around a self-discovering radio protocol in which a robust calling channel announces active flights, their working frequencies, and one or more supported Air/Coding Profile pairs. HABRX then creates narrowband receive channels on demand, decodes the advertised telemetry streams, maintains flight state, exposes the information through a browser and optional desktop application, stores data locally, and optionally uploads to remote services such as SondeHub Amateur and BalloonSpotter.

The architecture must work on both:

- desktop systems, especially Windows systems commonly used by radio amateurs; and
- headless Linux systems such as Raspberry Pi receivers, including low-resource installations.

The receiver should operate well with inexpensive RTL-SDR hardware while also taking advantage of wider-bandwidth SDRs such as Airspy R2 when available.

---

## 2. Architectural principles

### 2.1 Receiver engine first

The actual product is the receiver engine. User interfaces are clients of that engine.

The architecture SHALL NOT make the DSP pipeline depend on Tauri, a browser, or any specific UI technology.

Conceptually:

```text
SDR / IQ source
     |
     v
DSP + protocol engine
     |
     +--> local persistence
     +--> remote upload adapters
     +--> HTTP/WebSocket API
               |
               +--> browser UI
               +--> Tauri desktop shell
               +--> CLI / diagnostics
```

### 2.2 One frontend, multiple delivery modes

The same web frontend SHOULD be usable:

- directly from a browser on a headless receiver;
- embedded inside Tauri on Windows/macOS/Linux; and
- remotely from another machine on the LAN.

Tauri SHOULD add desktop conveniences, not unique receiver functionality.

### 2.3 Cross-platform receiver core

DSP, protocol parsing, FEC, flight management, storage abstractions, and most configuration logic SHALL live in cross-platform Rust crates.

Platform-specific code SHOULD be limited to areas such as:

- SDR driver integration where unavoidable;
- service management;
- desktop notifications;
- installer/update integration.

### 2.4 Graceful degradation

The system SHALL remain useful when optional capabilities are unavailable.

Examples:

- No Internet: decode and store locally.
- SondeHub unavailable: BalloonSpotter may continue uploading.
- BalloonSpotter unavailable: SondeHub may continue uploading.
- Tauri closed: headless engine may continue receiving.
- Wideband SDR unavailable: receiver may retune when required.
- Waterfall disabled: decoding continues.
- Flight metadata missing: token and telemetry remain visible.

---

## 3. Major components

```text
+-------------------------------------------------------------+
|                         HABRX Engine                         |
|                                                             |
|  +-------------+      +---------------+                     |
|  | SDR Sources |----->| Wideband IQ   |                     |
|  +-------------+      | Pipeline      |                     |
|                       +-------+-------+                     |
|                               |                             |
|                       +-------v-------+                     |
|                       | Channel       |                     |
|                       | Manager       |                     |
|                       +---+-------+---+                     |
|                           |       |                         |
|              +------------+       +-------------+           |
|              |                                  |           |
|       +------v------+                    +------v------+     |
|       | CALL DDC +  |                    | Dynamic DDC | ... |
|       | Decoder     |                    | + Modem     |     |
|       +------+------+                    +------+------+     |
|              |                                  |           |
|              +----------------+-----------------+           |
|                               |                             |
|                       +-------v-------+                     |
|                       | Frame / FEC   |                     |
|                       +-------+-------+                     |
|                               |                             |
|                       +-------v-------+                     |
|                       | Protocol      |                     |
|                       | Parser        |                     |
|                       +-------+-------+                     |
|                               |                             |
|                       +-------v-------+                     |
|                       | Flight        |                     |
|                       | Manager       |                     |
|                       +--+---------+--+                     |
|                          |         |                        |
|             +------------+         +-------------+          |
|             |                                    |          |
|      +------v------+                     +-------v------+   |
|      | SQLite      |                     | Upload       |   |
|      | + Outbox    |                     | Manager      |   |
|      +------+------+                     +---+-------+--+   |
|             |                                |       |      |
|             |                          SondeHub   Balloon-  |
|             |                                      Spotter |
|      +------v------+                                      |
|      | HTTP / WS   |                                      |
|      | API         |                                      |
|      +------+------+                                      |
+-------------|-----------------------------------------------+
              |
      +-------+---------+
      |                 |
  Browser UI         Tauri
```

---

## 4. Repository and crate layout

A recommended initial repository structure is:

```text
habrx/
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── SECURITY.md
├── CHANGELOG.md
├── .gitignore
├── .editorconfig
│
├── crates/
│   ├── habrx-core/          # common types, events, configuration models
│   ├── habrx-dsp/           # DDC, filtering, resampling, FFT, DSP utilities
│   ├── habrx-modem/         # 2GFSK/4GFSK/GMSK modem abstractions
│   ├── habrx-fec/           # convolutional FEC, CRC, whitening/interleaving
│   ├── habrx-protocol/      # discovery + telemetry wire protocol
│   ├── habrx-sdr/           # SDR abstraction and source implementations
│   ├── habrx-engine/        # runtime orchestration / channel manager
│   ├── habrx-storage/       # SQLite and migrations
│   ├── habrx-upload/        # upload manager and sink interfaces
│   └── habrx-api/           # HTTP/WebSocket API
│
├── apps/
│   ├── habrx-daemon/        # headless Linux/service executable
│   ├── habrx-cli/           # diagnostics, replay, test tooling
│   └── habrx-tauri/         # desktop shell
│
├── web/
│   └── frontend/            # shared browser/Tauri frontend
│
├── tests/
│   ├── vectors/             # protocol and FEC golden vectors
│   ├── iq/                  # short regression IQ captures
│   └── integration/
│
├── docs/
│   ├── architecture.md
│   ├── protocol.md
│   ├── ci-release.md
│   └── testing.md
│
└── .github/
    ├── workflows/
    │   └── ci.yml
    └── pull_request_template.md
```

Crate boundaries may evolve, but dependency direction SHOULD remain one-way toward reusable core libraries. Core crates SHALL NOT depend on Tauri.

---

## 5. SDR abstraction

### 5.1 Goals

HABRX SHALL not assume a single SDR family. The first implementation SHOULD support RTL-SDR as the reference community receiver. Airspy support is strongly desirable. File/replay support is mandatory for testing.

Potential later sources include SoapySDR-compatible devices, SDRplay, PlutoSDR, network IQ, or custom sources.

### 5.2 SDR source interface

The SDR layer SHOULD expose behavior equivalent to:

```rust
trait SdrSource {
    fn capabilities(&self) -> SdrCapabilities;
    fn configure(&mut self, config: SdrConfig) -> Result<()>;
    fn start(&mut self, sink: SampleSink) -> Result<()>;
    fn stop(&mut self) -> Result<()>;
}
```

Required concepts include:

- absolute center frequency in integer Hz;
- sample rate in integer samples/second;
- gain mode and gain value;
- SDR identity/serial where available;
- supported center-frequency range;
- supported sample rates;
- retune capability and expected latency;
- hardware timestamp capability, if any.

### 5.3 Required source implementations

Initial architectural targets:

```text
RtlSdrSource
AirspySource
FileSource
SyntheticSource
```

`FileSource` and `SyntheticSource` are first-class implementations, not test-only hacks.

### 5.4 Frequency representation

All RF frequencies SHALL be represented as integer Hz in core APIs and persistent configuration.

Example:

```text
434_525_000 Hz
```

Floating-point frequency values SHOULD be avoided outside DSP calculations.

### 5.5 SDR hotplug and failure recovery

The engine SHALL tolerate SDR loss. A typical state sequence is:

```text
RUNNING -> SDR_LOST -> RECONNECTING -> RUNNING
```

The backend SHOULD periodically attempt reconnection using persistent SDR identity/serial where possible.

---

## 6. Wideband IQ pipeline

### 6.1 Single capture, many virtual channels

When the SDR bandwidth is sufficient, HABRX SHOULD keep the calling channel and active flight channels inside one continuous IQ capture and create digital narrowband channels instead of retuning hardware.

Retuning is a fallback when an announced working frequency is outside the usable capture span.

### 6.2 Avoiding DC-center assumptions

The calling frequency does not need to equal the SDR hardware center frequency. The engine MAY intentionally offset the hardware center to keep the calling channel away from a tuner/DC spike.

All virtual channels SHALL be described by absolute RF frequency; the channel manager computes the necessary DDC offset.

### 6.3 Channel extraction

For a small number of active channels, the preferred v0.1 strategy is per-channel DDC:

```text
wideband IQ
   |
   v
complex NCO / mixer
   |
   v
low-pass + decimator
   |
   v
narrowband modem-rate IQ
```

Decimation SHOULD happen as early as practical.

A polyphase filter bank/FFT channelizer MAY be introduced later if many simultaneous channels make per-channel DDC inefficient.

### 6.4 Sample-rate-independent modem input

Modems SHOULD operate at a modest, controlled number of samples per symbol rather than at the SDR input rate.

For example, a 2400-symbol/s modem might receive approximately 8-12 samples/symbol after channel extraction/resampling.

---

## 7. Calling channel and dynamic channel manager

### 7.1 Calling channel

The calling channel is always-on when possible. It uses one fixed, robust, well-known demodulation profile defined by the protocol specification.

Its purpose is discovery and bootstrap, not high-rate telemetry.

### 7.2 Dynamic channel creation

A valid calling/discovery frame produces a `DiscoveryEvent` containing, at minimum:

- flight/session token;
- working RF frequency;
- supported air profile(s);
- relevant protocol version/profile information.

The calling decoder SHALL NOT directly instantiate DSP objects. Instead:

```text
Calling decoder
     |
     v
DiscoveryEvent
     |
     v
FlightManager
     |
     v
ChannelManager
     |
     v
Create/update dynamic channel
```

### 7.3 Dynamic channel lifecycle

Recommended states:

```text
DISCOVERED
    |
    v
ACQUIRING
    |
    v
ACTIVE
    |
    v
DEGRADED
    |
    v
STALE
    |
    v
REMOVED
```

Typical meanings:

- **DISCOVERED:** valid discovery announcement received.
- **ACQUIRING:** DDC/modem created; waiting for valid frames.
- **ACTIVE:** valid frames arriving normally.
- **DEGRADED:** RF/preamble activity exists, but decode quality is poor.
- **STALE:** no useful recent activity.
- **REMOVED:** expensive DSP resources freed.

Exact timeout values are configuration defaults, not protocol constants.

Example starting policy:

- acquiring timeout: 20-30 s;
- degraded/stale threshold: 30-60 s without valid telemetry;
- remove expensive decoder resources: 5-10 min without activity.

### 7.4 Flight record vs live channel

HABRX SHALL separate lightweight persistent/long-lived flight state from expensive live DSP state.

```text
FlightRecord
  token
  callsign
  flight name
  description
  last position
  last seen
  last known RF frequency
  last air profiles
  last CFO/drift estimate

LiveChannel
  DDC state
  filters
  resampler
  modem
  sync state
  FEC state
  short DSP buffers
```

A `FlightRecord` may survive after its `LiveChannel` has been destroyed.

---

## 8. Multiple air profiles per flight

### 8.1 Requirement

A flight SHALL be allowed to advertise and use more than one Air/Coding Profile pair on the same main RF frequency. The initial design target is a three-tier strategy:

```text
FAST
  4GFSK
  high symbol rate
  frequent telemetry
  normally role PRIMARY

NORMAL
  GMSK 1200, h=0.5, BT=0.5
  strong weak-signal baseline
  normally role AUXILIARY

EXTREME ROBUST
  2GFSK 100 or 200
  infrequent telemetry
  normally role ROBUST
```

### 8.2 Air/Coding profile model

Air and Coding Profiles are separate immutable registry entries. The per-flight advertisement combines them with role and schedule metadata. The internal model SHOULD resemble:

```rust
struct AdvertisedProfile {
    air_profile: AirProfileId,
    coding_profile: CodingProfileId,
    role: AirProfileRole,
    nominal_interval: Option<Duration>,
}
```

The on-air role values are:

```text
PRIMARY
ROBUST
EXPERIMENTAL
AUXILIARY
```

The descriptive tier names FAST, NORMAL, and EXTREME ROBUST are scheduling/UI terminology and are not additional role values.

### 8.3 Independent protocol payload

Packet semantics SHALL be independent of modulation. A POSITION packet decoded through PRIMARY and a POSITION packet decoded through ROBUST SHALL feed the same protocol parser and flight-state model.

### 8.4 Scheduling optimization

A low-duty-cycle robust decoder need not consume full DSP resources continuously if the receiver has reliable schedule information. However, scheduling SHALL be advisory rather than exact; the receiver must tolerate drift, resets, and missed schedule hints.

---

## 9. Modem architecture

### 9.1 Separation from protocol

A modem SHALL output transport/frame information and quality metrics, not semantic telemetry fields.

Conceptually:

```rust
struct DemodulatedFrame {
    raw_bits: Vec<u8>,
    soft_bits: Option<Vec<f32>>,
    crc_ok: bool,
    fec_metrics: FecMetrics,
    rf_metrics: RfMetrics,
    sample_index: u64,
}
```

### 9.2 Air and Coding Profiles

Physical modulation parameters SHALL be represented by immutable Air Profile definitions rather than loose runtime combinations. Error protection and bit-processing parameters SHALL be represented independently by immutable Coding Profile definitions.

An Air Profile defines items such as:

- modulation family;
- symbol rate and bits per symbol;
- BT/pulse shaping where applicable;
- modulation index, deviation, or tone spacing;
- symbol/frequency mapping;
- preamble/acquisition expectations;
- recommended receive bandwidth and samples/symbol.

A Coding Profile defines items such as:

- CRC algorithm;
- convolutional/other FEC;
- code rate and puncturing;
- terminated vs tail-biting behavior;
- whitening/scrambling;
- interleaving;
- coding-chain bit ordering.

Discovery advertises an `(Air Profile ID, Coding Profile ID)` pair. Profile IDs are immutable once published for interoperability. Registry release versions are metadata and are not transmitted over the air.

### 9.3 Initial modem support

Architectural targets:

- 2GFSK;
- 4GFSK;
- GMSK.

Other schemes MAY be added later.

### 9.4 Frequency offset and drift

The receiver SHALL tolerate main-channel frequency offset and slow drift. The design SHOULD include:

1. coarse channel placement from discovery;
2. preamble-assisted fine frequency acquisition;
3. modem-level frequency tracking/AFC;
4. per-flight stored CFO and drift estimates for faster reacquisition.

The receiver SHOULD distinguish nominal frequency, measured CFO, and estimated drift.

---

## 10. DSP execution model

### 10.1 Block processing

DSP SHOULD process reusable blocks, for example 1024-4096 complex samples at a time. Avoid passing one sample at a time across abstraction boundaries.

Hot-path code SHOULD minimize allocations and reuse buffers.

### 10.2 Threads and async

DSP processing SHOULD use predictable synchronous worker threads.

Async Rust/Tokio is appropriate for:

- HTTP;
- WebSockets;
- uploads;
- database coordination;
- control-plane tasks.

Async scheduling SHOULD NOT be required for the inner sample-processing loop.

### 10.3 Backpressure

Bounded queues/ring buffers SHALL separate stages.

Data loss policy MUST depend on data class:

| Data class | Overload behavior |
|---|---|
| Raw IQ | avoid dropping if at all possible; report drops |
| Decoded frames | do not drop |
| Flight-state changes | do not drop |
| Upload events | persist to durable outbox |
| Spectrum frames | may drop old frames |
| Waterfall lines | may drop old lines |
| UI statistics | latest value wins |

GUI slowness SHALL never block SDR/DSP processing.

### 10.4 Resource scheduler

The engine SHOULD support resource profiles such as:

```text
AUTO
LOW
NORMAL
HIGH
```

A LOW profile may:

- minimize SDR sample rate;
- disable waterfall by default;
- reduce FFT rate/size;
- cap simultaneous dynamic decoders;
- reduce database metric frequency;
- prioritize PRIMARY/ROBUST decode over visualization.

The original Raspberry Pi Zero W should be treated as a low-resource headless decode target; richer visualization may require stronger hardware.

---

## 11. Time and timestamps

### 11.1 Monotonic runtime timing

Channel timeouts, DSP timing, drift estimation, and internal durations SHALL use monotonic clocks.

Wall-clock changes caused by NTP or user adjustments SHALL not affect channel lifecycle logic.

### 11.2 Sample indexing

Every incoming IQ block SHOULD carry or inherit a monotonically increasing sample index.

Even SDRs without hardware timestamps can provide deterministic relative timing through sample counters.

### 11.3 UTC timestamps

UTC timestamps are used for:

- logs;
- storage;
- upload services;
- UI history.

Where exact hardware timestamping is unavailable, the implementation SHALL clearly distinguish host-derived receive time from hardware-derived time.

---

## 12. Signal detection vs decoding

The engine SHOULD separate:

- RF/energy presence;
- preamble correlation;
- modem synchronization;
- FEC success/confidence;
- CRC validity.

This allows states such as:

```text
RF present, no sync
sync present, FEC poor
FEC successful, CRC failed
valid frame
```

This information is useful for diagnostics and receiver-quality metrics.

---

## 13. Quality metrics

Per-channel and per-reception metrics SHOULD include, when meaningful:

- relative RSSI/power;
- SNR estimate;
- CFO in Hz;
- CFO drift in Hz/s;
- timing error;
- preamble/sync correlation score;
- soft-bit confidence;
- Viterbi/path metrics;
- FEC correction/confidence information;
- CRC success/failure;
- packet-loss estimate;
- selected Air/Coding Profile pair;
- actual RF frequency and SDR center frequency.

Not every modem needs to provide every metric.

---

## 14. Flight manager

The Flight Manager owns semantic flight state and SHALL not directly manipulate modem internals.

Responsibilities include:

- map flight/session tokens to known metadata;
- consume POSITION/STATE/FLIGHT_INFO/SYSTEM_STATUS events;
- track active/stale flights;
- expose normalized current state to storage/API/uploaders;
- retain last known RF/profile information for reacquisition;
- coordinate channel-manager create/update/remove requests.

---

## 15. Local storage

### 15.1 Database

SQLite is the preferred local database.

Likely tables include:

```text
flights
frames
receptions
positions
state_samples
system_events
receiver_events
upload_outbox
upload_status
schema_migrations
```

Exact schema is implementation-defined.

### 15.2 Database migrations

Schema changes SHALL use explicit ordered migrations.

Example:

```text
001_initial.sql
002_add_receptions.sql
003_add_upload_outbox.sql
```

The application SHALL never assume a column exists merely because code expects it.

### 15.3 Retention

Decoded telemetry is small and SHOULD have long retention by default.

Large data such as IQ diagnostics and high-rate spectrum history SHALL have configurable storage limits and oldest-first cleanup.

---

## 16. Raw frame retention and reception identity

### 16.1 Raw decoded data

Successful decoded frames SHOULD retain their original packed payload/frame bytes or bits alongside semantic fields.

This enables:

- later reprocessing;
- protocol migration testing;
- interoperability debugging;
- validation of third-party decoders.

### 16.2 Frame vs reception

One transmitted radio frame may be heard by many receivers.

The internal and BalloonSpotter-facing model SHOULD distinguish:

```text
RadioFrame
   +-- Reception(receiver A, SNR, CFO, timestamp...)
   +-- Reception(receiver B, SNR, CFO, timestamp...)
   +-- Reception(receiver C, SNR, CFO, timestamp...)
```

A stable frame fingerprint SHOULD support deduplication and idempotent upload.

---

## 17. Web/API architecture

### 17.1 Embedded server

The receiver engine SHOULD embed an HTTP server, for example using `axum`.

Recommended conceptual endpoints:

```text
GET  /api/v1/receiver
GET  /api/v1/flights
GET  /api/v1/flights/{id}
GET  /api/v1/config
PUT  /api/v1/config
GET  /api/v1/diagnostics
WS   /api/v1/events
```

Exact API design is TBD.

### 17.2 REST vs WebSocket

REST is appropriate for:

- current configuration;
- receiver information;
- history queries;
- explicit control actions.

WebSocket is appropriate for:

- newly decoded packets;
- flight updates;
- channel state changes;
- spectrum/waterfall updates;
- live diagnostics.

### 17.3 Browser independence

Closing or disconnecting a browser SHALL have no effect on reception, persistence, or uploads.

### 17.4 Shared frontend

The same SPA/web frontend SHOULD be served by the headless daemon and embedded in Tauri.

Browser-only and Tauri modes SHOULD expose the same receiver functionality except for platform-specific conveniences.

### 17.5 mDNS

Headless installations SHOULD support local-network discovery such as `habrx.local` where practical.

---

## 18. Tauri desktop application

The Tauri app SHOULD act as a first-class client of the HABRX backend.

Potential desktop-specific responsibilities:

- launch/connect to a local engine;
- discover remote HABRX engines;
- system tray integration;
- desktop notifications;
- installer/update integration;
- local file dialogs for IQ replay/export.

Tauri SHALL NOT contain unique DSP or protocol logic.

A Tauri app MAY connect to a remote receiver such as a Raspberry Pi instead of controlling a locally attached SDR.

---

## 19. Headless deployment

### 19.1 Linux daemon

Linux installs SHOULD support systemd operation.

Typical intended flow:

```text
install package
start/enable service
open browser to habrx.local
configure SDR/station
```

Normal users SHOULD not need to edit configuration files over SSH.

### 19.2 Performance profiles

A low-resource installation SHOULD be able to run without:

- waterfall;
- high-rate spectrum;
- expensive historical graph generation;
- unnecessary dynamic decoders.

Radio decoding always has higher priority than visualization.

---

## 20. Configuration

Configuration SHOULD be separated conceptually into:

### 20.1 System configuration

Examples:

- SDR identity;
- sample rate;
- gain settings;
- calling frequency;
- performance profile;
- web listen address/port;
- retention limits.

### 20.2 Station configuration

Examples:

- uploader/operator callsign;
- receiver description;
- optional station location;
- antenna description;
- enabled upload destinations.

### 20.3 Dynamic flight state

Flight discovery information SHALL generally not become permanent static configuration.

### 20.4 Transactional configuration changes

Configuration changes that affect live reception SHOULD be validated as a complete candidate configuration and then applied atomically where possible.

---

## 21. Secrets and security

API tokens and credentials SHALL be kept separate from ordinary exported configuration and SHALL be excluded from diagnostics bundles and logs.

The local web service SHOULD bind only to intended local/LAN interfaces by default.

Internet exposure SHOULD not be assumed. Remote Internet access is better provided through a VPN/reverse proxy than by making HABRX itself a public authentication gateway.

TLS certificate verification SHALL remain enabled for remote upload destinations.

---

## 22. Upload architecture

### 22.1 Normalized events

Remote services SHALL consume normalized telemetry/reception events rather than raw DSP objects.

Potential normalized event types:

```text
FlightDiscovered
Position
State
SystemStatus
FlightInfo
ReceptionQuality
FlightLost
```

### 22.2 Pluggable sinks

The upload layer SHOULD define a sink abstraction such as:

```rust
trait TelemetrySink {
    fn name(&self) -> &'static str;
    fn accepts(&self, event: &TelemetryEvent) -> bool;
    async fn upload_batch(&self, events: &[TelemetryEvent]) -> Result<UploadResult>;
}
```

Initial sinks:

```text
SondeHubSink
BalloonSpotterSink
```

Potential future sinks:

```text
MQTT
APRS
Webhook
File/export
```

### 22.3 Durable per-destination outbox

Decoded events SHALL be persisted before upload.

Each destination has independent state, retries, and backoff.

Example:

```text
event 18273
  SondeHub:      ACKED
  BalloonSpotter PENDING
```

An unavailable remote service SHALL not consume unbounded RAM and SHALL not block reception.

### 22.4 Retry behavior

Use exponential retry with jitter and sane retry limits/circuit-breaking behavior.

When a service recovers, a large receiver population SHOULD not reconnect in lock-step.

### 22.5 SondeHub adapter

SondeHub output is optional.

The adapter may need to delay uploads until required human-readable payload identity metadata is known.

The adapter SHALL translate HABRX normalized events into the externally required SondeHub representation.

### 22.6 BalloonSpotter adapter

BalloonSpotter output is optional.

The recommended direction is an efficient versioned HTTPS batch protocol, likely using CBOR and optional compression such as zstd.

The protocol SHOULD carry both telemetry and reception-quality metadata.

Receiver identity and operator callsign SHALL be distinct concepts.

---

## 23. Receiver/station identity

HABRX SHOULD distinguish:

```text
Flight identity
Receiver installation identity
Uploader/operator callsign
```

A persistent randomly generated receiver UUID is recommended.

One operator may own multiple receiving stations.

---

## 24. Replay, simulation, and regression testing

### 24.1 Replay mode

Live SDR and IQ replay SHALL pass through the same downstream DSP/protocol pipeline.

Example CLI use:

```text
habrx-cli replay capture.cf32 --center 434400000 --rate 2400000
```

### 24.2 Synthetic source

A synthetic source SHOULD support controlled generation of:

- valid discovery frames;
- multiple simultaneous flights;
- configurable CFO;
- frequency drift;
- AWGN/noise;
- packet loss;
- varying air profiles;
- malformed packets.

### 24.3 Golden vectors

Protocol/FEC implementations SHALL have golden vectors containing at least:

- semantic field values;
- exact packed bits/bytes;
- expected CRC;
- expected FEC output;
- expected symbol sequence where practical.

### 24.4 Regression IQ captures

Real-world failures SHOULD become short deterministic replay fixtures where licensing/storage permits.

---

## 25. Diagnostics and observability

### 25.1 Structured logging

Use structured logs, for example with Rust `tracing`.

Example fields:

```text
flight_token=19
frequency_hz=434525000
cfo_hz=1237
snr_db=4.2
event=decoder_locked
```

### 25.2 Diagnostics UI

Useful runtime diagnostics include:

```text
SDR
  device/serial
  center frequency
  sample rate
  dropped samples
  buffer occupancy

DSP
  active channels
  CPU/channel
  processing latency

Per flight
  current air profile
  CFO/drift
  SNR
  sync confidence
  packet rate
  CRC success rate
```

### 25.3 Diagnostics bundle

A support export SHOULD include:

- HABRX version/build commit;
- OS/CPU information;
- SDR information;
- sanitized configuration;
- recent structured logs;
- recent decoder statistics.

It SHALL exclude secrets.

---

## 26. Spectrum and waterfall

Spectrum/waterfall is an optional visualization feature, not part of the critical decoding path.

The backend SHOULD reduce FFT data before forwarding it to UI clients.

Recommended behavior:

- no raw IQ streaming to browser by default;
- moderate FFT update rates such as 5-20 frames/s depending on resource profile;
- demand-driven or reduced FFT processing when no UI client needs it;
- drop old visualization frames rather than block DSP.

---

## 27. Packaging targets

Primary release targets:

| Target | Product | Requirement |
|---|---|---|
| Windows x86-64 | Tauri desktop + receiver engine | Required |
| Linux x86-64 | Headless daemon + embedded web UI | Required |
| Linux ARM64 | Headless daemon + embedded web UI (Pi 3/4/5, Zero 2 W class) | Required |
| Linux ARMv6 | Headless low-resource build for original Pi Zero/Zero W | Best-effort, validated separately |
| macOS | Tauri desktop | Optional/later |

Windows SHOULD eventually have a signed installer. Linux SHOULD provide standard packages such as `.deb` where practical. The headless packages SHALL NOT require Tauri or desktop GUI dependencies.

---

## 28. Source control, CI, and release engineering

### 28.1 Canonical repository

GitHub SHOULD be the canonical source repository for HABRX. `docs/architecture.md` and `docs/protocol.md` are normative project documents and SHALL be reviewed like source code. Architectural or wire-protocol changes MUST NOT be introduced silently through implementation changes.

The initial branch model SHOULD remain simple:

```text
main
  ^
  +-- pull requests / feature branches
```

Once implementation work begins, `main` SHOULD be protected so required CI checks pass before merge.

### 28.2 Pull-request CI

CI SHALL run automatically for pull requests and changes to `main`. Once the corresponding source trees exist, required checks SHOULD include:

```text
cargo fmt --check
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace
frontend lint/test/build
protocol golden-vector tests
short IQ regression/replay tests
Windows x86-64 compile/build
Linux x86-64 compile/build
Linux ARM64 compile/build
```

The same Rust DSP, FEC, and protocol implementation SHALL be used across operating systems. Platform builds MUST NOT fork into independent DSP implementations.

ARMv6/Pi Zero builds MAY initially be cross-compiled. A real Pi Zero/Zero W SHOULD eventually be used as a self-hosted or manually invoked hardware-validation runner for installation, startup, API, and short IQ-decode smoke tests.

### 28.3 Release builds

Full distributable packages SHOULD be produced from version tags rather than on every commit. HABRX SHOULD use semantic version tags such as:

```text
v0.1.0
v0.2.0
v1.0.0
```

A release SHOULD eventually publish:

```text
Windows x86-64 installer/package
Linux x86-64 package
Linux ARM64 package
Linux ARMv6 package when supported
SHA-256 checksums
release notes/changelog
```

Release artifacts SHOULD become reproducible and signed where practical. Code signing is particularly desirable for public Windows releases.

### 28.4 CI and RF regression tests

Protocol golden vectors and compact IQ fixtures are architectural test assets, not optional developer conveniences. A DSP change that still compiles but changes decode behavior MUST be detectable in CI.

Representative fixtures SHOULD cover at least:

- Discovery v1 nominal reception;
- calling-channel frequency offset near acquisition limits;
- GMSK 1200 weak-signal reception;
- 4GFSK FAST profile reception;
- 2GFSK EXTREME-ROBUST reception;
- slow frequency drift;
- CRC/FEC failure cases;
- unsupported Air/Coding Profile handling.

### 28.5 Initial repository bootstrap

The repository MAY exist before the Rust/Tauri workspace is scaffolded. Initial CI SHALL therefore tolerate the documentation-only bootstrap state and enable code checks automatically once `Cargo.toml` / frontend manifests are present.

---

## 29. Updates

Automatic updates SHALL not unexpectedly stop reception during an active flight.

Desktop updates SHOULD be user-controlled or deferred until idle.

Headless package updates SHOULD follow normal OS administration expectations.

---

## 30. Explicit non-goals for v0.1

The first implementation does not need to provide:

- arbitrary generic user-defined demodulators;
- arbitrary runtime-defined telemetry schemas;
- a large multi-tenant public server;
- advanced PFB channelization;
- perfect cross-receiver time synchronization;
- adaptive transmitter control from ground stations;
- every SDR family.

The architecture should permit future evolution without requiring these in v0.1.

---

## 31. Initial implementation milestones

### Milestone 1: deterministic offline core

- Rust workspace and core crates.
- CI enabled on Windows/Linux and cross-platform compile checks.
- FileSource/SyntheticSource.
- modem abstraction.
- basic 2GFSK/4GFSK/GMSK experiments.
- frame/FEC/CRC implementation.
- protocol parser.
- golden vectors.
- compact IQ regression fixtures.

### Milestone 2: RTL-SDR live receiver

- RTL-SDR source.
- fixed calling-channel decoder.
- dynamic per-flight DDC.
- channel lifecycle.
- CLI diagnostics.

### Milestone 3: headless web application

- engine service.
- SQLite + migrations.
- REST/WebSocket API.
- browser UI.
- Linux/systemd packaging.

### Milestone 4: desktop application

- shared frontend in Tauri.
- Windows packaging.
- local/remote receiver connection.

### Milestone 5: remote services

- durable outbox.
- BalloonSpotter sink.
- SondeHub sink.
- retry/backoff and diagnostics.

### Milestone 6: optimization and extended hardware

- Airspy source.
- low-resource Pi tuning.
- SIMD/CPU profiling.
- optional advanced channelizer if justified.
- tagged multi-OS release automation and signing improvements.

---

## 32. Decisions intentionally left open

The following details remain intentionally open or provisional:

- final recommended regional calling frequency;
- exact recommended main-channel offset/window around the calling frequency;
- Discovery v1 32-bit sync constant;
- exact CRC-16 variant before golden-vector freeze;
- final stable Air Profile constants such as 2GFSK deviation/h and 4GFSK tone spacing/mapping;
- final stable Coding Profile details such as telemetry interleaving/whitening variants;
- exact BalloonSpotter ingest schema and authentication lifecycle;
- final release-signing/update infrastructure.

The following are no longer architectural TBDs and are defined in `protocol.md`: separate immutable Air/Coding registries, Discovery v1 fixed payload layout, 10-bit Flight Token, four fixed Discovery profile slots, and the default 4FSK 100-symbol/s calling PHY structure.

---

## 33. Summary

HABRX is designed as a service-oriented, cross-platform Rust receiver with a shared browser/Tauri frontend. A fixed robust calling channel discovers flights and their one or more Air/Coding Profile pairs. Working channels are created only when needed, decimated early, and removed after inactivity. The system retains lightweight flight records while freeing expensive DSP resources. All critical reception logic remains independent of GUI and remote services. Local persistence and durable independent upload queues ensure that decoding remains reliable even when browsers, networks, or remote services fail. GitHub-based source control, cross-platform CI, protocol golden vectors, and IQ replay regression tests are part of the architecture so that RF behavior remains reproducible across Windows, Linux, and Raspberry Pi targets.
