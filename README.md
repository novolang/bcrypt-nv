# bcrypt-nv

bcrypt is a password hashing function, published by Niels Provos and
David Mazieres at USENIX in 1999 and used by OpenBSD ever since. Its
subject is the *cost* of computing it: the work a single hash takes is
a number chosen when the hash is written, and stored beside it. This
package brings it to novo-lang.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What bcrypt is

A **password hash** is not a message digest. A digest is meant to be
fast; a password hash is meant to be slow, because the attacker who
matters is one who has stolen the database and is guessing passwords
against it. Every design decision in bcrypt follows from that.

bcrypt is built on **Blowfish**, a 64-bit block cipher from 1993.
Blowfish's tables — an 18-entry subkey array and four boxes of 256
entries each — are not constants of the cipher. They start at the
digits of pi and are then rewritten by the **key schedule**, which
costs 521 encryptions each time it runs. Provos and Mazieres took that
expensive schedule and made the number of times it runs a parameter.
They called the result **Eksblowfish**, for "expensive key schedule".

The **cost** is that parameter, as an exponent: the schedule runs
`2^cost` times. Each step up doubles the work, for the server once and
for an attacker on every guess. A **salt** is sixteen random bytes
mixed into the schedule, so that two users with the same password have
different hashes and one precomputed table cannot attack them both.

Hashing is then short: set up Eksblowfish from the password, the salt
and the cost, and encrypt the 24-byte string `OrpheanBeholderScryDoubt`
sixty-four times. The result is the hash.

A stored bcrypt hash is sixty characters in the **modular crypt
format**, which carries everything a verifier needs:

```
$2b$12$LongSaltTwentyTwoChars.ThirtyOneCharactersOfHashHere..
 ^^  ^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
 |   |  22 characters of salt, then 31 of hash
 |   the cost, always two digits
 the variant
```

The **variant** is the two characters between the first two dollar
signs. It does not mean "version 1, version 2". Each letter is a marker
added when an implementation was found to be computing something other
than what it meant to, so that hashes written before the discovery can
still be verified.

| Prefix | What it records |
| --- | --- |
| `2` | The original, from the 1999 paper |
| `2a` | The corrected original, and what most stored hashes carry |
| `2x` | `2a` hashes from the PHP implementation that sign-extended bytes above 127 |
| `2y` | PHP's marker after that was fixed; computes what `2b` computes |
| `2b` | OpenBSD's marker after a key-length overflow was fixed in 2014 |

| Quantity | Value |
| --- | --- |
| Salt | 16 bytes, 22 characters |
| Password bytes the key schedule reads | 72 |
| Cost | 4 to 31 |
| Hash computed | 24 bytes |
| Hash stored | 23 bytes, 31 characters |
| Whole string | 60 characters |

## Install

```
novo pkg add bcrypt-nv
```

## Example

```novo
use std.bytes
use bcryptcost
use bcrypthash
use bcryptinput

fn main() [io]
    // The password, checked: at most 72 bytes and no zero byte in it.
    match bcryptinput.password(bytes.from_str("correct horse battery staple"))
        Err(e) => println("that password cannot be hashed: ${e.message()}")
        Ok(pw) =>
            // Sixteen bytes of salt. Your program reads them from a
            // random source; this package never does.
            match bcryptinput.salt(bytes.zeros(16))
                Err(e) => println(e.message())
                Ok(sa) =>
                    // The sixty characters to store.
                    match bcrypthash.hash_text(pw, sa, bcryptcost.default_cost())
                        Err(e) => println(e.message())
                        Ok(stored) => println(stored)

            // Checking one. The salt and the cost come out of the
            // stored string, so nothing else has to be kept.
            match bcrypthash.verify(pw, "\$2b\$12\$" + str.repeat(".", 53))
                Ok(true)  => println("that is the password")
                Ok(false) => println("that is not the password")
                Err(e)    => println("that row is not a bcrypt hash: ${e.message()}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a `not implemented:
bcrypt-nv.<module>.<fn>` panic. The tests are the specification the
implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `bcryptblowfish` | Blowfish and the expensive key schedule: the tables as a value, the round function, block encryption, and the Eksblowfish setup. |
| `bcryptinput` | The password and the salt as values that have already been checked, and the one call that truncates a long password on purpose. |
| `bcryptcost` | The cost as a value that has already been checked: the range, the rounds it means, and the two digits it is written as. |
| `bcryptvariant` | The five prefixes, what each one records, which of them compute alike, and the one this package writes. |
| `bcryptmcf` | The sixty characters a hash is stored as, both ways, and bcrypt's own base64 alphabet. |
| `bcrypthash` | Hashing a password, checking one, and the question of whether a stored hash should be replaced. |
| `bcrypterr` | The seven refusals, and whether each is about the call or about the stored data. |

## How to choose an entry point

**`bcrypthash.hash_text` writes a row.** It takes a checked password, a
salt you generated and a cost, and answers the sixty characters to
store.

**`bcrypthash.verify` checks a row.** It takes a checked password and
the stored string, reads the salt and the cost out of that string, and
compares the hashes in constant time.

**`bcryptmcf.parse` is for looking inside a stored string** without
hashing anything — reading the cost of every row in a database, or
finding the rows that will not parse.

**`bcryptblowfish` is the cipher underneath.** It is published so that
an implementation can be bisected: a hash that is right there and wrong
in the stored string is an encoding problem, and one that is wrong
there is a key-schedule problem.

## The rules a user needs

1. **The salt is a parameter.** This package reads no randomness. Your
   program generates sixteen bytes and passes them in. That makes
   hashing a pure function of the password, the salt and the cost, so
   a test reproduces a published hash exactly and the one line of your
   program that reads a random source is visible.
2. **A password over 72 bytes is refused, not truncated.** The key
   schedule reads 72 bytes and stops, so byte 73 changes nothing. Every
   other bcrypt truncates silently, which makes two passwords sharing
   their first 72 bytes into one password. `bcryptinput.password`
   answers `BcryptPasswordTooLong`;
   `bcryptinput.password_truncating` does what the others do, under a
   name a reviewer sees.
3. **A password containing a zero byte is refused.** The C
   implementations treat the password as a NUL-terminated string, so
   `"abc\0def"` hashes there as `"abc"`. A password accepted here and
   hashed differently would be one nobody could log in with elsewhere.
4. **The salt is sixteen bytes, and not a length you choose.** A
   shorter one is not padded and a longer one is not trimmed.
5. **The cost is between 4 and 31**, and it is an exponent: the
   schedule runs `2^cost` times. Cost 12 is 4096.
6. **A wrong password is `Ok(false)`; a row that is not a bcrypt hash
   is `Err`.** A login endpoint has to tell them apart, or it reports a
   database fault as a bad password.
7. **Compare hashes with a constant-time comparison.**
   `bcrypthash.verify` already does. A program comparing the stored
   strings itself should use `digest.ct_eq` from
   [crypto-nv](https://novo-lang.org/packages/crypto-nv), never `==`.
8. **This package writes `$2b$` and verifies all five prefixes.**
   Asking it to write any other is refused, because writing a marker
   that records somebody else's bug would be recording a bug this
   package does not have.
9. **The cost field is always two digits.** `$2b$4$` is not a bcrypt
   hash; `$2b$04$` is. Every field in the string is fixed-width, and
   there is no separator between the salt and the hash.
10. **bcrypt's base64 is not RFC 4648's.** The alphabet is
    `./ABC…xyz0123456789`, the same sixty-four characters in a
    different order. A hash encoded with a standard base64 encoder is
    sixty plausible characters that verify against nothing.
11. **The stored hash is 23 of the 24 bytes computed.** The last byte
    is dropped. That was an accident in the original implementation and
    it is now part of the format.
12. **A stored hash's cost can only be raised at a successful login**,
    because that is the one moment the password is in hand.
    `bcrypthash.needs_rehash` is the question to ask then.

## Running on a microcontroller

This package makes no device claim and ships no probe.

The arithmetic is the kind a device can run: integer lookups and
additions over a value with nothing allocated. The working set is not.
The Blowfish tables are 4 KiB, every step of the key schedule produces
a new copy of them, and a Cortex-M4's whole memory is measured in tens
of kilobytes. A program that needs to derive a key from a password on a
microcontroller wants a memory-light function, and
[pbkdf2-nv](https://novo-lang.org/packages/pbkdf2-nv) is that.

## Timing behaviour

- **`bcrypthash.verify` compares in constant time.** It reads both
  values whole and folds every difference into one accumulator, so the
  time it takes says nothing about how many bytes matched.
- **The key schedule's time depends on the cost and on nothing else.**
  It does not depend on the password, which is the property that makes
  the cost meaningful: an attacker cannot find a cheap guess.
- **The round function indexes the S-boxes with bytes derived from the
  key.** That is Blowfish's design and it is what a cache-timing attack
  on Blowfish targets. The attack needs measurements from the machine
  doing the hashing, so it is a concern for a shared host and not for a
  stolen database. No claim beyond that is made here.
- **`bcryptmcf.parse` is not constant-time** and does not need to be: a
  stored hash is not a secret.

## What is not included

- **Randomness.** Generating a salt costs `[rand]`, and this package
  declares no effects. See rule 1.
- **A password policy.** Length rules, dictionary checks and breach
  lists are a different subject and a different package.
- **Argon2, scrypt and PBKDF2.** Argon2 and scrypt are memory-hard,
  which bcrypt is not, and are the better choice for a new password
  store. [pbkdf2-nv](https://novo-lang.org/packages/pbkdf2-nv) is the
  one standards name. This package exists for the databases that
  already hold bcrypt hashes, and for the systems that specify it.
- **The other `crypt(3)` formats.** `$1$` (MD5), `$5$` and `$6$`
  (SHA-256 and SHA-512 crypt) share the modular crypt format and are
  different algorithms.
- **A constant-time comparison of its own.** See "Timing behaviour" and
  rule 7.
- **Writing `$2$`, `$2a$`, `$2x$` or `$2y$`.** All four are verified.
  See rule 8.
- **Any input or output.** A password arrives as bytes the caller read
  and a hash leaves as a string the caller stores.

## Related packages

- [pbkdf2-nv](https://novo-lang.org/packages/pbkdf2-nv) is PBKDF2,
  which is what standards specify when they name a password-based key
  derivation. Take it when a specification names it, or when the result
  has to be a key of a length you choose. Take this package when you
  are storing a password.
- [crypto-nv](https://novo-lang.org/packages/crypto-nv) supplies
  `digest.ct_eq`, the constant-time comparison. This package does not
  depend on it, so that a caller who wants a password hash does not
  link SHA-2, SHA-1, MD5 and HMAC as well.
- [base64-nv](https://novo-lang.org/packages/base64-nv) is RFC 4648
  base64. It is not what bcrypt uses; see rule 10.

## Test vectors

The reference implementation is OpenBSD's `bcrypt.c`. The hashes the
suite checks are the ones jBCrypt's `BCryptTest` publishes — the set
that the Rust `bcrypt` crate, Python's `bcrypt` and Go's
`golang.org/x/crypto/bcrypt` all check themselves against, and which in
turn comes from OpenWall's published test data. A hash that matches
there is a hash OpenBSD's own `crypt` produces.

```bash
novo test tests/bcrypt_vectors_tests.nv   # the published hashes and the two refusals
novo test tests/bcrypt_format_tests.nv    # the cost, the alphabet, the sixty characters
novo test tests/bcrypt_cover_tests.nv     # the cipher, and every public function
```

The suite asserts that the published hashes verify, that a wrong
password is `Ok(false)` and a bad row is `Err`, that 73 bytes is
refused and 72 accepted, that a zero byte is refused, that a salt of
any length but sixteen is refused, that hashing the same three inputs
twice gives the same string, that the raw hash is 24 bytes and the
stored one 23, that the alphabet begins `./AB` and not `ABCD`, that the
cost field is two digits, that parse and render round-trip, and that
Blowfish's first subkey is the first 32 bits of pi.

The tests compile today and fail at run, each on the `not implemented:
bcrypt-nv.<module>.<fn>` panic that is its body. That is the expected
state of an interface release. They turn green one at a time as bodies
land.

## Implementation status

| Item | Implemented |
| --- | --- |
| The sixteen `BLOWFISH_*`, `BCRYPT_*` size, cost and format constants | yes (they are constants) |
| `bcryptblowfish.BcryptState`, `.BcryptBlock`, `bcryptinput.BcryptPassword`, `.BcryptSalt`, `bcryptcost.BcryptCost`, `bcryptvariant.BcryptVariant`, `bcryptmcf.BcryptHash`, `bcrypterr.BcryptError` | the types are declared |
| `bcryptblowfish.initial_state`, `.block`, `.p_word`, `.s_word`, `.feistel` | no |
| `bcryptblowfish.encrypt_block`, `.expand_key`, `.expand_key_salted`, `.eks_setup` | no |
| `bcryptinput.password`, `.password_truncating`, `.password_len`, `.password_bytes` | no |
| `bcryptinput.salt`, `.salt_bytes` | no |
| `bcryptcost.cost`, `.default_cost`, `.value_of`, `.rounds_of` | no |
| `bcryptcost.cost_text`, `.cost_of_text`, `.at_least` | no |
| `bcryptvariant.written_variant`, `.prefix_text`, `.variant_of_text`, `.computes_alike`, `.is_writable` | no |
| `bcryptmcf.b64_encode`, `.b64_decode`, `.b64_chars_for` | no |
| `bcryptmcf.parse`, `.render`, `.hash_value` | no |
| `bcryptmcf.variant_of`, `.cost_of`, `.salt_of`, `.digest_of` | no |
| `bcrypthash.hash`, `.hash_text`, `.hash_as`, `.raw_hash` | no |
| `bcrypthash.verify`, `.verify_against`, `.needs_rehash` | no |
| `bcrypterr.is_input_fault`, `.is_format_fault`, `.format_offset`, `.code`, `BcryptError.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
