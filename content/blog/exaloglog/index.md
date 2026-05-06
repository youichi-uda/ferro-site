+++
title = "Up to 43% less memory than HyperLogLog: a Rust implementation of ExaLogLog"
date = 2026-05-06
description = "ExaLogLog (Ertl 2024) reaches the same RMSE as 6-bit HyperLogLog with around 43% less memory. Today I'm releasing a from-scratch pure-Rust implementation, including both the packed and CAS-friendly variants, both estimators, and a reproducible bench harness."

[taxonomies]
tags = ["rust", "launch", "library", "cardinality", "algorithm"]

[extra]
toc = true
+++

If you've ever run `SELECT COUNT(DISTINCT user_id)` on a stream of telemetry,
you've probably leaned on HyperLogLog. It's been the standard distinct-
counting sketch since Flajolet's 2007 paper; Redis, ClickHouse, BigQuery,
Snowflake, Elasticsearch, DuckDB, and Apache DataSketches all ship some
flavor of it. The deal is simple: spend a couple of kilobytes of register
state per stream, feed in well-hashed inputs, and recover the cardinality
to within ~2% relative error.

In early 2024, Otmar Ertl at Dynatrace Research [published][paper] a sketch
that reaches the same RMSE as 6-bit HyperLogLog with around 43% less
in-memory state. The paper is rigorous — theory, experiments, a reference
implementation in Java — but a year later I couldn't find a pure-Rust
implementation with both estimators, mergeability, serialization, and
empirical accuracy validated against the paper's tables. So I wrote one.
Today I'm releasing it as the [`exaloglog`][crate] crate.

This post walks through what changed, the core math, and what the
implementation cost. It's also a sibling release to my main work on Ferro:
ExaLogLog will land in FerroStream's distinct-count aggregations and
FerroStash's log analytics, and there's a series of similar paper-to-crate
releases queued up behind it. More on the roadmap at the end.

[paper]: https://arxiv.org/abs/2402.13726
[crate]: https://crates.io/crates/exaloglog

## The headline

From Table 2 of the [EDBT 2025 paper][edbt-pdf], all configurations
tuned for around 2% RMSE at `n = 10⁶`:

| Algorithm | In-memory bytes | Serialized bytes | Empirical RMSE |
| --- | ---: | ---: | ---: |
| HyperLogLog 8-bit (`p = 11`) | 2296 | 2088 | 2.29% |
| HyperLogLog 6-bit packed (`p = 11`) | 1792 | 1577 | 2.29% |
| HyperLogLog ML (`p = 10`) | 1416 ± 56 | 1067 ± 4 | 2.38% |
| Compressed Probability Counting (`p = 10`) | 1416 ± 56 | **656 ± 4** | 2.38% |
| UltraLogLog (Ertl 2023, `p = 10`) | 1056 | 1024 | 2.38% |
| HyperLogLogLog (`p = 11`) | 1100 ± 13 | 898 ± 16 | 2.30% |
| **ExaLogLog `(t=2, d=20, p=8)`** | **936** | 896 | 2.27% |

[edbt-pdf]: https://www.openproceedings.org/2025/conf/edbt/paper-252.pdf

Two things are worth pulling out:

- **In-memory state**: ExaLogLog is the smallest at this RMSE target,
  43% under HLL-6 packed and 48% under HLL-8.
- **Serialized state**: Compressed Probability Counting (CPC) wins
  at 656 bytes — but pays for it with a fairly expensive variable-length
  encoding step at serialization time. ExaLogLog still beats every
  algorithm whose serialized form is the dense register array.

Asymptotically, the savings come from the memory-variance product
(MVP) ratio: HLL-6 has MVP = 6.48, ExaLogLog at `d = 20` has MVP = 3.67.
Same RMSE means same `MVP × storage`, so ELL needs 3.67 / 6.48 ≈ 57%
of HLL-6's bits — the 43% reduction. In the discrete configurations of
Table 2 the in-memory reduction lands at 48%; in my own 200-trial
empirical search (where the RMSE-target rounds to a nearby `p`) it
lands at ~42%. "Around 43% less in-memory state at the same RMSE" is
the headline I'm comfortable defending.

If you keep cardinality sketches per-tenant and have a few thousand
tenants, that delta turns into a real budget line.

## What ExaLogLog is, in one paragraph

Like HyperLogLog, ExaLogLog distributes hash values across `m = 2^p`
registers using the low `p` bits of the hash. Unlike HyperLogLog, the
register doesn't just track the longest leading-zero run seen so far —
it splits into a high `q` bits storing the maximum *update value* `u`,
and a low `d` bits acting as a bitmap that records which of the most
recent `d` smaller update values have actually been observed.

The bitmap is the key insight. HyperLogLog throws away information every
time it overwrites a register: knowing that some hash with a leading-zero
count of 7 arrived doesn't tell us whether values 5 or 6 also did.
ExaLogLog keeps a sliding window of that "auxiliary" information, and the
maximum-likelihood estimator that follows is the one that uses it.

The default configuration this crate ships is `t = 2, d = 20`: four
"fine-grained" update values per HLL-style integer level, twenty bits of
bitmap. This puts the per-register width at 28 bits — exactly two
registers per 7 bytes. The memory-variance product (MVP) hits 3.67,
where HLL with 6-bit registers sits at 6.48. The ratio is the 43%
savings.

## Reading Algorithm 2 in earnest

The insert procedure (Algorithm 2 of the paper) looks unassuming on
paper. Given a 64-bit hash `h`:

```
i  = bits[t .. t+p) of h          // register index
a  = h | ((1 << (p+t)) - 1)       // force low p+t bits to 1
k  = nlz(a) * 2^t + low_t(h) + 1  // update value, k ∈ [1, (65-p-t)·2^t]
u  = r_i >> d                     // current max update for register i
Δ  = k - u

if Δ > 0:
    r_i ← (k << d) | floor((2^d + bitmap(r_i)) / 2^Δ)
elif Δ < 0 and d + Δ ≥ 0:
    r_i ← r_i | (1 << (d + Δ))
else:
    no-op
```

Two pieces tripped me up.

The first was `k`. ExaLogLog approximates a base-`b = 2^(2^-t)`
geometric distribution over update values by chunking `2^t` consecutive
values into "plateaus" with equal probability. Translating "compute `k`
from the hash" into a few cycles of bit-twiddling required understanding
*why* the construction `nlz(a) · 2^t + low_t(h) + 1` actually samples
from the right distribution. I spent an hour staring at Eq. 8 before
convincing myself.

The second was the register update rule when `Δ > 0`. The expression
`floor((2^d + bitmap) / 2^Δ)` looked off until I realized: when we bump
`u` by `Δ` levels, the bitmap's highest position now has to record that
the *previous* `u` was hit. The `2^d` term prepends that 1-bit, then
the right-shift drops the bottom `Δ` bits that fall off the end of the
window. It's a one-line implementation of "shift the bitmap, mark the
most recent observation."

## Maximum likelihood, by way of bisection

The paper presents two estimators. The martingale (HIP) estimator is
incremental and trivially fast — every state-changing insert
increments the running estimate by `1/μ` and adjusts `μ`. The catch
is that it lives entirely in auxiliary state alongside the registers:
it cannot be recomputed from the register array alone, so once you
merge two sketches (where the auxiliary state has no consistent
meaning) or deserialize one (in this crate's wire format the auxiliary
state isn't carried), you're stuck. So you also want a maximum-
likelihood estimator that works from the register state alone.

Algorithm 3 in the paper computes `(α, β_u)` coefficients of the
log-likelihood, which has the simple shape

```
ln L = -(n/m) · α + Σ β_u · ln(1 - exp(-n / (m · 2^u)))
```

Setting the derivative to zero gives the ML equation `g(y) = α` where
`y = n/m`, and the paper devotes Algorithm 8 to a robust Newton solver
with a recursive product computation to avoid floating-point overflow.

I shipped bisection in `log₂(y)` instead. It converges to f64 precision
in under 200 iterations and sidesteps the overflow concerns by using
`exp_m1` for the denominator. Newton would be substantially faster
for the estimate call (the paper reports 5-7 iterations), but
`estimate()` is a query-time operation, not on the per-insert hot
path; for the workloads I'm targeting the call cost is in the noise.
If you're calling `estimate()` thousands of times per second, file an
issue and a Newton solver gets bumped up the list.

## A bug I caught with a sanity check

My first pass through Algorithm 3 was wrong. Section 3.3 of the paper
defines `h(r) = (1/m)(ω(u) + Σ (1 - l_{u-k}) · ρ_update(k))` — bit set
means "we observed update value k" means *no contribution* to the
state-change probability. I had it inverted: my `compute_alpha_beta`
contributed `1/2^φ(k)` to `α` when the bit was set and to `β` when
clear.

The smoking gun was that `h(0) = 1/m` is supposed to hold — the
state-change probability of an empty register equals the probability
that the next inserted element lands on it. With my bug, that identity
broke. Once I fixed the inversion, every other property fell into
line: the empirical RMSE matched theory to within statistical noise,
and the martingale and ML estimators agreed to within 2% of `n` on
fresh sketches at `p = 12`.

This is the kind of error that's invisible in spot-tests. A small bias
on the cardinality estimate is well below most sanity thresholds. The
way to catch it is to grind theory invariants — `h(0) = 1/m`,
`ω(0) = 1`, "`h` strictly decreases on every register transition the
insert algorithm produces" — and turn each one into a test that runs
on every build. The repo has a handful of such property tests; they're
cheap insurance and they paid out before the code shipped.

## Two variants, two tradeoffs

The crate ships both `ExaLogLog` (packed, `d = 20`, MVP = 3.67) and
`ExaLogLogFast` (32-bit aligned, `d = 24`, MVP = 3.78).

The packed variant is the headline 43% memory savings configuration.
Two registers share a 7-byte chunk, so reads are an unaligned 4-byte
load and a mask, and writes are read-modify-write of the byte that's
shared with the sibling register. That shared byte is the reason
packed cannot be CAS'd safely for lock-free concurrent updates.

`ExaLogLogFast` trades 3% extra memory for `u32`-aligned registers —
one register per machine word, individually CAS'd via
`add_hash_atomic(&self, hash)`, and around 15-30% faster on the
scalar insert path (see the throughput table below). If you're
feeding events into a sketch from many cores at once, this is the
variant you want; the registers are stored as `Box<[AtomicU32]>` so
multiple threads can ingest into a shared sketch without external
synchronization.

The two share their math via a `math` module parameterized by `d`.
The ML solver, the (α, β) coefficient computation, the per-register
insert rule, and the merge rule are all the same code, called with
different `d`. It would have been easy to implement the packed
variant by copy-pasting the aligned one, but then every bug fix would
have needed two patches, and divergence over time is inevitable.

### Sparse mode for low cardinalities

The dense register array at `p = 12` takes 14 KB. If you're keeping
sketches per-tenant and most tenants only have ten distinct items,
that 14 KB is mostly zero registers. Both variants start in
**sparse mode** — a sorted list of 32-bit hash tokens (paper §4.3) —
and auto-promote to the dense array at the per-variant break-even
point.

For 100 distinct items at `p = 12`, sparse mode uses ~424 bytes vs
the dense 14336 — about 33× smaller. Below break-even, the ML
estimator is *exact*: the token set IS the data, no statistical
inference involved. Above break-even the dense estimator takes over
and storage is fixed. `new_dense(p)` skips sparse mode if you know
`n` will be large, or if you need to start handing the sketch out to
multiple threads for `add_hash_atomic` (sparse storage is `Vec`-
backed and can't be safely shared via `&self`).

### Reducing precision after the fact

Both variants implement `reduce(new_p)` (Algorithm 6 of the paper),
returning a sketch at lower precision identical to one built directly
at `new_p`. Useful for migration scenarios — you committed to too
high a `p` for some tenants and need to compact existing sketches
before they ship to slower storage.

## Throughput

Single-threaded scalar code, criterion bench on AMD Ryzen 9 9950X,
Linux, rustc 1.95.0 stable, `--release` (default codegen). Bench
source: [`benches/insert.rs`][bench]. Hashes are pre-computed via
SplitMix64 so the measurement isolates sketch work from hashing
cost.

| variant | `p = 8` | `p = 12` | `p = 16` |
| --- | ---: | ---: | ---: |
| `ExaLogLog` (packed) | 101 M inserts/s | 46 M inserts/s | 39 M inserts/s |
| `ExaLogLogFast` (aligned) | 146 M inserts/s | 53 M inserts/s | 45 M inserts/s |

A textbook HLL in the same harness ([source][hh]) hits ~100 M
inserts/s; ExaLogLog is in the same regime despite doing strictly
more bookkeeping per insert. The drop as `p` grows is cache-bound:
at `p = 16` the register array is 64 KB and falls out of L1.

[bench]: https://github.com/abyo-software/exaloglog-rs/blob/main/benches/insert.rs
[hh]: https://github.com/abyo-software/exaloglog-rs/blob/main/examples/head_to_head.rs

A SIMD-accelerated batch-insert path is on the v0.3 list. The
random-access register updates cap the win — gather/scatter is
needed and AVX-512 is the only reasonable target — so I'd rather
ship correct scalar code now and verify I'm not breaking accuracy
with the SIMD version later.

## Using it

```rust
use exaloglog::ExaLogLog;

let mut sketch = ExaLogLog::new(12);   // m = 2^12 = 4096 registers
for item in stream {
    sketch.add(&item);
}
println!("distinct: {}", sketch.estimate());
```

The API closely mirrors what you'd expect from a HyperLogLog crate.
`add(&T)` for any `Hash` value, `add_hash(u64)` if you've already
hashed (use this with a fast hasher like xxhash3 or wyhash if hashing
is your bottleneck), `merge(&other)` to union two sketches, and
`to_bytes()` / `from_bytes()` for transport. With `--features serde`
you also get `Serialize` / `Deserialize` impls that go through the
same byte format, so JSON, MessagePack, bincode, and CBOR work
without re-encoding. Both the ML and the martingale (HIP) estimators
are available; `estimate()` defaults to ML, which is the one you
want after merges or deserialization.

## What I'm doing next

ExaLogLog is the first in a series of paper-to-crate releases I'm
shipping through the [`abyo-software`][abyo] organization. The next
ones queued up are RaBitQ for vector quantization, Ribbon +
InfiniFilter for approximate-membership filters, Eg-walker CRDTs,
ChalametPIR for private information retrieval, and BMP + Seismic for
learned sparse retrieval. Where it makes sense each will also land
inside Ferro: RaBitQ and BMP feed FerroSearch's vector and learned-
sparse paths, ExaLogLog joins FerroStream and FerroStash, ChalametPIR
unlocks the private-search use case.

ExaLogLog took roughly two engineer-days end to end including the
writeup. I don't want to overgeneralize from one data point, but
mining the recent distributed-systems literature for papers with
rigorous theory and reference implementations in languages other
than the one your stack actually runs on looks like a high-yield
activity right now.

If you're already reaching for HyperLogLog, ExaLogLog is worth a
look — especially if you're storage-constrained per-sketch. Source
on [GitHub][gh]. Crate on [crates.io][crate]. Docs on [docs.rs][docs].
Issues, benchmarks against your own workloads, and PRs welcome.

[abyo]: https://github.com/abyo-software
[gh]: https://github.com/abyo-software/exaloglog-rs
[docs]: https://docs.rs/exaloglog
