# HABRX

HABRX is a cross-platform, self-discovering high-altitude-balloon telemetry receiver.

The project is designed around a fixed robust calling/discovery channel. Active flights advertise a working frequency and one or more Air/Coding Profile pairs. HABRX then creates narrowband receive channels only when needed, decodes the advertised telemetry, stores it locally, exposes it through a browser/API, and can optionally upload receptions to services such as SondeHub Amateur and BalloonSpotter.

## Current status

**Design / pre-implementation (Draft v0.1).**

The normative design documents are:

- [`docs/architecture.md`](docs/architecture.md) — software/system architecture.
- [`docs/protocol.md`](docs/protocol.md) — over-the-air discovery and telemetry protocol.

Implementation must not silently redefine either document. Protocol or architectural changes should update the relevant specification in the same pull request.

## Design highlights

- Rust receiver/DSP core.
- Shared browser UI for headless installations.
- Tauri desktop shell for Windows/macOS/Linux.
- RTL-SDR as the reference community receiver; Airspy and additional SDRs can be added behind the SDR abstraction.
- Headless Linux/Raspberry Pi support, including a low-resource mode.
- Fixed 4FSK/100-symbol/s Discovery v1 calling PHY with configurable calling frequency.
- Dynamic per-flight channels created on discovery and discarded after inactivity.
- Multiple Air/Coding Profile pairs per flight, including FAST, NORMAL, and EXTREME-ROBUST tiers.
- Local SQLite persistence and durable independent upload queues.
- Optional SondeHub and BalloonSpotter upload adapters.
- Replay, protocol golden vectors, and IQ regression fixtures as first-class test assets.
- Cross-platform CI and tagged release builds.

## Intended targets

| Platform | Intended application |
|---|---|
| Windows x86-64 | Tauri desktop + receiver engine |
| Linux x86-64 | Headless daemon + web UI |
| Linux ARM64 | Raspberry Pi/headless daemon + web UI |
| Linux ARMv6 | Best-effort original Pi Zero/Zero W headless build |
| macOS | Optional/later Tauri desktop target |

## Repository bootstrap

This repository is intentionally documentation-first. Codex or contributors should scaffold the Rust/Tauri workspace according to `docs/architecture.md` rather than allowing the initial implementation to define the architecture implicitly.

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md).

## Security

See [`SECURITY.md`](SECURITY.md).

## License

This starter repository uses the MIT License. If a different licensing model is desired, change it before accepting external contributions.
