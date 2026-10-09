# Contributing to HABRX

HABRX is currently in a design-first implementation phase. Contributions are welcome, but interoperability and deterministic RF behavior take priority over implementation convenience.

## Normative documents

The following files are normative:

- `docs/architecture.md`
- `docs/protocol.md`

A code change that changes architecture or wire behavior MUST update the appropriate document in the same pull request. Do not silently make protocol behavior implementation-defined.

## Development principles

- Keep DSP, FEC, protocol parsing, and flight state cross-platform.
- Do not create separate Windows and Linux DSP implementations.
- Keep GUI code out of core crates.
- Do not block SDR/DSP processing on UI, database, or network upload work.
- Treat malformed/corrupted radio data as untrusted input.
- Preserve unknown/unsupported profile handling rather than guessing parameters.
- Avoid system-wide dependency installation from project scripts unless explicitly documented and necessary.

## Tests expected for protocol/DSP changes

Depending on the change, add or update:

- unit tests;
- protocol golden vectors;
- FEC/CRC vectors;
- short IQ replay fixtures;
- frequency-offset/drift cases;
- unsupported-profile cases;
- cross-platform compile checks.

A bug found from a real recording should preferably become a regression fixture.

## Pull requests

Keep pull requests focused. Explain:

1. what changes;
2. whether architecture/protocol behavior changes;
3. how it was tested;
4. whether new IQ/vector fixtures were added;
5. any compatibility impact.

## Profile registries

Published Air and Coding Profile IDs are immutable. Never reuse or redefine an assigned interoperable ID. A corrected/improved profile receives a new ID.
