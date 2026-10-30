# Fluveo build of Diesel 2.2.10 with an 86-column tuple limit

This repository holds the two crates the Fluveo router takes from Diesel, patched to add one Cargo feature.
Fluveo consumes it as a git dependency pinned to a full commit SHA, through `[patch.crates-io]` in
`services/router/Cargo.toml` of fluveohq/fluveo. The decision record is the Diesel tuple-limit ADR in its `docs/adr`.

| Crate | Version | crates.io package SHA-256 | Upstream source commit |
| --- | --- | --- | --- |
| `diesel` | 2.2.10 | `ff3e1edb1f37b4953dd5176916347289ed43d7119cc2e6c7c3f7849ff44ea506` | `2a6752b2b91c55b9968b7b8e3df44d5ff7eca166` |
| `diesel_derives` | 2.2.3 | `e7f2c3de51e2ba6bf2a648285696137aaf0f5f487bcbea93972fe8a364e131a4` | `4c0e1f765c1216d6f829940ecc857af74b18d841` |

Upstream: https://github.com/diesel-rs/diesel. License: MIT OR Apache-2.0, both license files are kept in each crate.

## History

1. The first commit unpacks both published `.crate` files from static.crates.io with no change.
   Reproduce it: download both files, check the SHA-256 above, unpack, and compare with `git diff`.
2. The second commit is the whole Fluveo patch, 17 added and 2 removed lines in four files:
   - `diesel/Cargo.toml` and `diesel_derives/Cargo.toml` add the feature `86-column-tables`. It enables
     `64-column-tables` and forwards to `diesel_derives/86-column-tables`.
   - `diesel_derives/src/diesel_for_each_tuple.rs` sets `MAX_TUPLE_SIZE` to 86 under that feature, so Diesel
     emits its tuple impls for every width from 1 to 86 instead of 1 to 128.
   - `diesel_derives/src/table.rs` names the new limit in the error `table!` raises for a wider table.

No Rust source of `diesel` changes. No public API changes. `128-column-tables` still exists and wins when a
crate in the build enables it.

## Why

Diesel with `128-column-tables` is the longest single crate at the start of the router's cold build. The
router's widest table has 86 columns. Measured on a c8a.4xlarge, the 86 limit shortens the cold router build by
about 37 s with parallel rustc and about 98 s without it.

## Updating

Never move or rewrite this branch: Fluveo pins commits by SHA. A new Diesel version is a new branch named
`fluveo/v<version>-<width>col`, built the same way: one verbatim import commit, then the patch commit.
