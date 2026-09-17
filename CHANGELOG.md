# Changelog

All notable changes to bcrypt-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `bcryptinput` — the decision the package is built around. A
  password over 72 bytes and a password with a zero byte in it are
  REFUSALS, not truncations. Every other bcrypt in the world
  truncates silently, so two passwords sharing their first 72 bytes
  are one password there and nobody is told — which is how a password
  manager's hundred-character passphrase quietly becomes a
  seventy-two-character one. `password_truncating` is the classic
  behaviour, under a name a reviewer sees and a `grep` finds.
- `bcryptcost` — the cost as a checked value rather than an `Int`. A
  library's default work factor is a security decision nobody
  revisits; `BCRYPT_COST_DEFAULT` is a named constant a deployment can
  read and print, and `needs_rehash` is the question a login endpoint
  asks at the one moment a stored hash's cost can be raised.
- `bcryptblowfish` — Blowfish and the expensive key schedule, with the
  1042-word state as a `@value` struct, so a password check allocates
  nothing. `eks_setup` CANNOT FAIL: it takes three already-checked
  values, so by the time the schedule runs there is nothing left to
  check and there is no error channel for a reader to wonder about.
- `bcryptvariant` — the five prefixes as an enum with no arm meaning
  "whatever the string said", `computes_alike` so that a database of
  `$2a$` rows is verifiable without five code paths, and
  `is_writable`, which is true for `Bcrypt2b` alone.
- `bcryptmcf` — the sixty characters both ways, and bcrypt's own
  base64 alphabet as a published constant, because the alphabet is the
  single thing a from-scratch bcrypt gets wrong and a hash encoded
  with RFC 4648's is sixty plausible characters that verify against
  nothing.
- `bcrypthash` — hashing and verifying. A wrong password is
  `Ok(false)` and a row that is not a bcrypt hash is `Err`, because a
  login endpoint that conflated them would report a database fault as
  a bad password.
- `bcrypterr` — seven refusals, with `is_input_fault` and
  `is_format_fault` dividing a bug in the calling code from a bad row
  in the database.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the three API suites reaches `not implemented:
  bcrypt-nv.<module>.<fn>`.
- **The salt is a parameter, and that is what makes this `core`.**
  Generating one costs `[rand]`, which this layer's budget does not
  admit. The by-product is that hashing is a pure function of its
  three inputs, so a test reproduces a published hash with no injected
  generator and the one line of a program that reads a random source
  is visible. No effect row in this package wanted to widen.
- **`BcryptCost`, `BcryptPassword`, `BcryptSalt` and `BcryptHash` are
  plain structs and not `@value` ones.** Each is a `Result` payload,
  and a `@value` struct may not be one (E2015). `BcryptState` and
  `BcryptBlock`, which are never in a `Result`, are `@value`.
- **No device claim, and no probe.** The arithmetic is device-shaped;
  the 4 KiB of Blowfish tables that every step of the schedule copies
  is not. The README says so and points a firmware author at pbkdf2-nv.
- **No dependency on crypto-nv.** bcrypt uses no hash function at all,
  so the only thing it would want is `digest.ct_eq`. Taking it would
  put SHA-2, SHA-1, MD5 and HMAC into the footprint of a caller who
  wanted a password hash and nothing else; `bcrypthash.verify` does
  the constant-time comparison itself and the README says where the
  registry's own lives.
- **Argon2, scrypt, the other `crypt(3)` formats and any password
  policy are named as missing**, not stubbed.
