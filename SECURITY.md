# Security Policy

HABRX processes untrusted RF data, exposes a local/network API, stores data locally, and may contain credentials for optional upload services. Security issues should therefore be treated seriously even though the primary use case is amateur-radio telemetry reception.

## Reporting

Until a dedicated security contact is configured for the repository, do not publish credential exposure, remote-code-execution, authentication-bypass, or similarly sensitive vulnerabilities as public proof-of-concept issues. Use GitHub private vulnerability reporting if enabled, or contact the repository owner privately.

## Security boundaries

The implementation should assume:

- radio frames can be malformed or deliberately hostile;
- variable-length fields require strict bounds checking;
- API credentials must not be exposed to browser JavaScript, logs, diagnostics bundles, or configuration exports;
- remote upload failures must not affect receiver/DSP availability;
- the embedded web/API server should default to local/LAN use and not be exposed directly to the public Internet;
- TLS certificate verification remains enabled for remote services.

## Supported versions

During pre-1.0 development, only the latest development/release line is expected to receive security fixes unless otherwise stated.
