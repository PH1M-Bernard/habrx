# HABRX Testing Strategy

Testing is part of the radio architecture because a compile-successful DSP change can still alter sensitivity, acquisition behavior, or interoperability.

## 1. Protocol golden vectors

Every frozen protocol/profile release should provide vectors containing, as applicable:

```text
semantic field values
packed uncoded payload bits/bytes
CRC result
coded/FEC bitstream
interleaved/whitened form when applicable
modulation symbol sequence
```

Transmit and receive implementations in Rust, embedded C, Python, or other languages should be able to verify themselves against the same vectors.

## 2. IQ regression fixtures

Keep fixtures short and purpose-specific. Suggested initial corpus:

```text
discovery_nominal.cf32
discovery_cfo_plus_4khz.cf32
discovery_cfo_minus_4khz.cf32
discovery_drift.cf32
gmsk1200_weak.cf32
4gfsk_fast_weak.cf32
2gfsk_robust_weak.cf32
crc_failure.cf32
unsupported_profile.cf32
```

Each fixture should have machine-readable expected outcomes.

## 3. Replay determinism

The same IQ fixture must produce equivalent protocol results on supported OS targets. Minor floating-point quality-metric variation may be tolerated, but packet identity and CRC/FEC success should be deterministic within documented tolerances.

## 4. Performance tests

Track at least:

- dropped SDR samples;
- processing latency;
- CPU per active channel;
- memory use;
- maximum practical simultaneous flight channels;
- Pi Zero low-resource behavior.

UI/spectrum work must be sacrificed before decoding reliability when resources are constrained.

## 5. Real-flight regressions

When a real flight exposes a decoder bug, save the smallest useful sanitized IQ window and turn it into a regression case where practical.
