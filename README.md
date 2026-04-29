# Lemur: Scalable Post-Quantum Synchronized Multi-Signatures

This repository is an anonymous artifact for the paper **"Lemur: Scalable
Post-Quantum Synchronized Multi-Signatures"**.

The artifact contains three main deliverables:

| Directory | Purpose |
| --- | --- |
| `lemur-py/` | Python reference implementation of Lemur, including CLI tools, serialization, and cross-checkable test-vector generation. |
| `lemur-rs/` | Performance-oriented Rust implementation, including the CLI, tests, benchmarks, materialized-tree option, and byte-compatible serialization. |
| `parameter/` | Sage-based parameter estimator and notes explaining how the concrete implementation parameters are generated. |

The artifact is intended to let reviewers inspect the scheme implementation,
reproduce byte-level Python/Rust interoperability, run the Rust test suite and
benchmarks, and inspect the parameter-search methodology.

## Quick Start

### Python Reference

Run from `lemur-py/`:

```sh
cd lemur-py
python3 -m pip install -r requirements.txt

python3 ring.py
python3 kots.py
python3 hvc.py
python3 lemur.py

python3 cli.py vectors --tau 3 --signers 2 --slot 0 --msg "artifact check" --out /tmp/lemur-py-vectors.json
python3 cli.py sizes
```

The Python CLI supports `setup`, `keygen`, `sign`, `verify`, `aggregate`,
`batch-verify`, `sizes`, and `vectors`.  See `lemur-py/usage.md` and
`lemur-py/api.md` for details.

### Rust Implementation

Run from `lemur-rs/`:

```sh
cd lemur-rs
cargo test

cargo run --release --bin lemur -- vectors --tau 3 --signers 2 --slot 0 --msg "artifact check" --out /tmp/lemur-rs-vectors.json
cargo run --release --bin lemur -- sizes

cargo run --release --bin bench -- --fast
cargo run --release --bin bench_breakdown
```

The Rust CLI mirrors the Python CLI.  The Rust crate also includes integration
tests for serialization, stateful signing, robustness against malformed inputs,
and Python/Rust-compatible test-vector behavior.

## Benchmarking And Sizes

The artifact benchmarks the fixed `d=256, k=4, tau=20` parameter set.  The
`tau=24` cell is supported by the implementation for parameter coverage, but it
is not used as the default benchmark target.

Run:

```sh
cd lemur-rs
cargo run --release --bin lemur -- sizes
cargo run --release --bin bench -- --fast
```

Representative serialized sizes for `tau=20, N=1024`:

| Object | Size |
| --- | ---: |
| Public parameters | 65 B |
| Secret seed | 32 B |
| Stateful signer cache | 134.4 KB |
| Public key | 3.4 KB |
| Individual signature | 89.5 KB |
| Aggregated signature, `N=1024` | 201.2 KB |

Representative `bench --fast` timings from a 24-thread run:

| Operation | Time |
| --- | ---: |
| Key generation | 1.3 min |
| Online signing, KOTS only | 347 us |
| Full signing, including HVC open | 1.3 min |
| Stateful signing, BDS08 | 4.13 ms |
| Individual pre-verify, `N=1024` | 1.67 s |
| Aggregate after verified inputs, `N=1024` | 2.40 s |
| Secure aggregation, `N=1024` | 567 ms |
| Batch verification, `N=1024` | 30.1 ms |
| Aggregated signature size, `N=1024` | 194.3 KB |
| Individual pre-verify, `N=8192` | 13.6 s |
| Aggregate after verified inputs, `N=8192` | 37.6 s |
| Secure aggregation, `N=8192` | 5.73 s |
| Batch verification, `N=8192` | 223 ms |
| Aggregated signature size, `N=8192` | 216.0 KB |

Timings are machine-dependent.  The `lemur sizes` table reports deterministic
serialized-size formulas; the benchmark's aggregated-signature sizes are the
actual encoded lengths from that run.  Reviewers should regenerate both on
their own hardware.

## Python/Rust Interoperability Check

Both implementations are designed to produce identical byte-level artifacts for
the same public parameters, seeds, messages, slots, and signer counts.  A quick
cross-check is:

```sh
cd lemur-py
python3 cli.py vectors --tau 3 --signers 2 --slot 0 --msg "artifact check" --out /tmp/lemur-py-vectors.json

cd ../lemur-rs
cargo run --release --bin lemur -- vectors --tau 3 --signers 2 --slot 0 --msg "artifact check" --out /tmp/lemur-rs-vectors.json

python3 - <<'PY'
import json

py = json.load(open("/tmp/lemur-py-vectors.json"))
rs = json.load(open("/tmp/lemur-rs-vectors.json"))

for key in ["pp", "signatures", "ivrfy", "avrfy", "agg_attempt", "aggregate"]:
    print(f"{key}: {'MATCH' if py[key] == rs[key] else 'MISMATCH'}")

for i, (p, r) in enumerate(zip(py["signers"], rs["signers"])):
    print(f"pk {i}: {'MATCH' if p['pk'] == r['pk'] else 'MISMATCH'}")
PY
```

## Parameter Generation

The concrete parameter cells used by the implementations are generated and
checked with the Sage estimator in `parameter/`.

Run from `parameter/`:

```sh
cd parameter
sage lemur_param.sage
```

With `verbosity = 1`, the script writes `summary.txt`.  The estimator README
explains the search space, prime-selection conventions, `beta_encode`, the
Chipmunk comparison scripts, and how to map an estimator cell into the Python
and Rust profile definitions:

```text
parameter/README.md
```

## Artifact Scope

This artifact is scoped to the implementation and parameter-generation
deliverables listed above.  Paper drafts, review notes, local development
metadata, and assistant/tooling notes are not part of the artifact.
