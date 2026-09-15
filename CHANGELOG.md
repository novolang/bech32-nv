# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**, before anyone implements it.  Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`.  Adding this
package works and calling it panics.

- Four modules.  `bech32_core` is the BCH checksum as one integer fed
  five bits at a time; `bechbits` is the eight-to-five regrouping and
  its inverse; `bech32` is those two with a string attached and the
  three whole-string rules a per-character machine cannot see;
  `bechsegwit` is segwit addresses, the one application the
  specification itself defines.
- **The witness version chooses the encoding** — version 0 is bech32,
  1 through 16 are bech32m — and `bechsegwit.variant_for` is that rule
  as one function.  A mismatch is `BechWrongVariant` and not a bad
  checksum, because a taproot address handed to a bech32-only reader is
  a working address in an application that has not been updated.
- **The two regrouping directions are not symmetric.**  Eight to five
  pads the tail with zeros; five to eight requires the tail to be
  shorter than a byte and all zero, and discards it.  `finish_padded`
  and `finish_exact` are those two rules named, with no flag between
  them — accepting a set padding bit would let two different strings
  decode to the same bytes.
- **The decode writes into the caller's buffer** and answers the
  human-readable part as a RANGE into the string the caller already
  has, not as a `Str` that would be an allocation for something already
  visible.
- **The 90-character limit is lifted by a differently named call**, not
  by a flag: `encode_unlimited` and `decode_unlimited` are for BOLT 11
  invoices and anything else that gave up the four-error guarantee on
  purpose, and the separate name puts that choice at the call site.
- **The separator is the last `1`**, which is why BIP 173's
  eighty-three-character prefix containing a `1` is one of the tests.
- Every vector is a BIP's: the seven valid bech32 strings, the eleven
  invalid ones with a stated reason each, BIP 350's bech32m set, and
  the segwit addresses.

**The device claim is built**, and the consumer that needs it is a
hardware wallet: the device that signs a transaction is the device that
has to show the address on its own screen, because an address shown on
the host is an address the host could have replaced.
`tests/embedded_probe.nv` compiles the checksum and the regrouping to a
Cortex-M4 ELF for `--target=nrf52-qemu`, driving a twenty-byte program
straight into the checksum with no intermediate buffer.

**No dependency on bitstream-nv**, which is the one a reader will look
for.  The regrouping is a fixed five bits against eight with a two-value
bound on every step, so a general variable-width reader carries a width
argument through the inner loop for no gain — and the embedded probe is
built from the package's own core modules and nothing they depend on, so
a dependency would cost the device claim on top of that.
