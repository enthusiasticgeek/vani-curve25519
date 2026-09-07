# curve25519

`field25519` (GF(2^255-19) field arithmetic), `X25519` (RFC 7748
Diffie-Hellman), and `Ed25519` (RFC 8032 EdDSA sign/verify) — pure
vāṇी, hardware-agnostic and heap-free.

Extracted from [Dhruva OS](https://github.com/enthusiasticgeek/dhruvaos)'s
Pi 4/5 port. Field elements are `[u32; 8]` little-endian limb arrays;
every field/bignum function returns its result **by value** (an owned
local can always supply either `ref` or `mut ref` at its own call
site) rather than the out-parameter convention a heap-based port would
use — see `src/lib.vani`'s own header comment for the specific
vāṇी-language facts (an aliasing XOR rule, no `ref` of a call
temporary) that make the by-value convention the right one here, not
just a style choice.

## Dependency

Depends on [`vani-crypto-hash`](https://github.com/enthusiasticgeek/vani-crypto-hash)
for SHA-512 (Ed25519's deterministic nonce derivation and challenge
hash). Vendored at `./vendor/crypto_hash` and pulled in via a direct
relative `use` in `src/lib.vani` — **not** declared as a `vani.toml`
`[deps]` entry, because doing so trips a real vani-compiler bug (see
`src/lib.vani`'s header comment and `vani.toml`'s own `[deps]`
comment for the specifics, and `vani-compiler/docs/TODO_CURRENT.md`
BUG-234 upstream).

## Two variants: `src/lib.vani` vs `src/core.vani`

`src/lib.vani` is the full API (below) — use this unless you have a
specific reason not to. `src/core.vani` is the SAME functionality
minus `x25519_self_test`/`ed25519_self_test`, for a consumer whose
own code already defines a function of one of those exact names (or
`sha256_hash`/`sha512_hash`/etc, transitively — `core.vani` depends
only on `crypto_hash/src/core.vani`, never `crypto_hash/src/lib.vani`)
and would otherwise collide when `use`-ing this package — e.g.
DhruvaOS's Pi 1 kernel, which has its own pre-existing `x25519_
self_test`/`ed25519_self_test`. See `core.vani`'s own header comment
for the full story. Kept in sync with `lib.vani` by construction
(subset, not a separate implementation).

## API

Structs:

```
struct Ed25519Point { x: [u32; 8], y: [u32; 8], z: [u32; 8], t: [u32; 8] }
struct Ed25519RecoverXResult { ok: i64, x: [u32; 8] }
struct Ed25519DecompressResult { ok: i64, x: [u32; 8], y: [u32; 8], z: [u32; 8], t: [u32; 8] }
struct Ed25519ExpandResult { a: [u32; 8], prefix: [u8; 32] }
```

Field arithmetic (all `fn f(a: ref [u32;8], ...) -> [u32;8]`):
`field25519_add`, `field25519_sub`, `field25519_mul`, `field25519_sqr`,
`field25519_invert`.

X25519:
```
fn x25519_scalarmult(k_bytes: ref [u8; 32], u_in_bytes: ref [u8; 32]) -> [u8; 32]
fn x25519_self_test() -> i64   // 1 = pass, 0 = fail; no I/O side effects
```

Ed25519:
```
fn ed25519_secret_to_public(secret: ref [u8; 32]) -> [u8; 32]
fn ed25519_sign(secret: ref [u8; 32], msg: ref [u8; 512], msg_len: i64) -> [u8; 64]
fn ed25519_verify(pubkey: ref [u8; 32], msg: ref [u8; 512], msg_len: i64, sig: ref [u8; 64]) -> i64
fn ed25519_self_test() -> i64   // 1 = pass, 0 = fail; no I/O side effects
```

`msg` is capped at 512 bytes (v0.2.0, up from 64 in v0.1.0) — a
practical stack-buffer size, not an algorithmic limit: the hashing
itself streams (`ed25519_hash_2block_msg`, internal) and has no
length cap at all. Sign a pre-hashed digest yourself for something
larger than 512 bytes.

The self-test functions return `1`/`0` and print nothing, same
convention as `crypto_hash`'s own.

## Verification

Both self-tests reuse real, independently-verified test vectors from
DhruvaOS's own kernels: 2 X25519 Diffie-Hellman vectors (base-point
derivation, arbitrary-u key agreement) and a full Ed25519 pubkey/
sign/verify/tamper-rejection check (seed `0x00..0x1f`, a fixed test
message, and a pinned expected public key + signature, all originally
checked against the real `cryptography` Python library before ever
being ported). `test/host_test.vani` runs both under the host harness.
