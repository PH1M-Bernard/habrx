# HABRX Initial Roadmap

This is a convenience summary. `architecture.md` remains normative.

1. **Offline deterministic core** — Rust workspace, protocol/FEC, FileSource/SyntheticSource, golden vectors, compact IQ fixtures.
2. **RTL-SDR live receiver** — Discovery v1 decoder, dynamic DDC/channel lifecycle, CLI diagnostics.
3. **Headless web application** — engine service, SQLite/migrations, REST/WebSocket, browser UI, systemd packaging.
4. **Desktop application** — shared frontend in Tauri, Windows packaging, local/remote receiver connection.
5. **Remote services** — durable upload outbox, BalloonSpotter and SondeHub sinks.
6. **Optimization/extended hardware** — Airspy, Pi Zero tuning, SIMD/profiling, optional advanced channelizer, mature tagged multi-OS releases.
