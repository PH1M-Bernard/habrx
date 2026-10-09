# HABRX CI and Release Plan

This document supplements `architecture.md`. If this document conflicts with `architecture.md`, the architecture specification wins.

## Pull-request CI

Once the corresponding source manifests exist, CI should run:

```text
cargo fmt --check
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace
frontend lint/test/build
protocol golden-vector tests
short IQ replay/regression tests
```

Build/compile coverage should include:

| Target | CI expectation |
|---|---|
| Windows x86-64 | Required |
| Linux x86-64 | Required |
| Linux ARM64 | Required |
| Linux ARMv6 | Cross-build initially; physical Pi validation later |
| macOS | Optional/later |

The documentation-only bootstrap repository should remain green before the Rust/Tauri workspace exists.

## Release policy

Full distributable packages should be built from semantic version tags, for example `v0.1.0`.

Expected release artifacts eventually include:

```text
Windows x86-64 installer/package
Linux x86-64 package
Linux ARM64 package
Linux ARMv6 package when supported
SHA256SUMS
release notes
```

Public Windows releases should eventually be code-signed. Release generation must not introduce a separate platform-specific DSP implementation.

## Hardware validation

A real original Raspberry Pi Zero/Zero W is recommended later as a low-resource validation host. It should consume prebuilt artifacts rather than compile the project locally and run smoke tests such as:

- package install/upgrade;
- daemon startup;
- local HTTP API availability;
- short IQ fixture decode;
- resource/memory sanity.
