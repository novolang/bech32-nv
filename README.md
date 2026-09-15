# bech32-nv

Bech32 is a way of writing binary data as text with a checksum on the
end, designed so that a person can read it aloud or copy it by hand and
have a mistake caught. It is specified in
[BIP 173](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki),
and [BIP 350](https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki)
defines bech32m, a revision that changes one constant. This package
implements both, and segwit addresses, which is the one application the
specifications themselves define.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What bech32 is

A bech32 string has three parts. The **human-readable part**, or prefix,
says what the string is for: `bc` for a Bitcoin address on the main
network, `tb` on a test network, `lnbc` for a lightning invoice. The
**separator** is the character `1`. The **data part** follows, and its
last six characters are the **checksum**.

Each data character carries five bits. The charset is
`qpzry9x8gf2tvdw0s3jn54khce6mua7l`, which is thirty-two of the
thirty-six alphanumerics. The four left out are `1`, `b`, `i` and `o`,
the ones most often confused with `l`, `6`, `1` and `0`. The order is
not alphabetical: characters that are easily confused sit where a single
mistake is most likely to be caught.

The checksum is a **BCH code**, an error-detecting code computed over
the prefix and the data together. Feeding it the checksum characters as
well leaves a fixed value called the **residue**, which is how a reader
checks a string. Bech32 and bech32m are the same arithmetic with
different residues: 1 and `0x2bc830a3`.

A payload is bytes and a character is five bits, so an encoder must
regroup eight-bit groups into five-bit ones. BIP 173 calls this
`convertbits`. The two directions are not symmetric. Eight to five pads
the last group with zero bits. Five to eight requires the leftover bits
to number fewer than eight and to be all zero, and discards them.

Segwit addresses are bech32 strings whose data part is a **witness
version** followed by a **witness program**. The version decides which
encoding the address must use, which is the rule BIP 350 exists for.

| Quantity | Value |
| --- | --- |
| Bits one data character carries | 5 |
| Characters in the charset | 32 |
| Checksum characters | 6 |
| Separator | `1`, ASCII 49 |
| Longest bech32 string (BIP 173) | 90 characters |
| Longest human-readable part | 83 characters |
| Characters allowed in a human-readable part | ASCII 33 to 126 |
| Range of a data value | 0 to 31 |
| bech32 checksum constant | 1 |
| bech32m checksum constant | `0x2bc830a3` |
| Witness versions | 0 to 16 |
| Witness program length at version 0 | exactly 20 or 32 bytes |
| Witness program length at versions 1 to 16 | 2 to 40 bytes |

## Install

```
novo pkg add bech32-nv
```

## Example

```novo
use std.bytes
use bechsegwit

fn main() [io]
    // A twenty-byte witness program: the hash of a public key.
    let prog = bytes.from_hex("751e76e8199196d454941c45d1b3a323f1433bd6") ?? bytes.zeros(0)

    // The address for that program on the main network, at witness
    // version 0. The version chooses bech32 rather than bech32m, so the
    // caller does not pass an encoding.
    match bechsegwit.encode("bc", 0, prog)
        Ok(a)  => println(a)
        Err(e) => println(e.message())

    // The same address read back. Upper case is accepted, because an
    // address may be printed that way for a QR code.
    match bechsegwit.decode("BC1QW508D6QEJXTDG4Y5R3ZARVARY0C5XW7KV8F3T4")
        // The prefix, the witness version, and the program's length.
        Ok(a)  => println("${a.hrp} ${a.version} ${bytes.len(a.program)}")
        Err(e) => println(e.message())
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a `not implemented: <module>.<fn>`
panic. The tests are the specification the implementation will have to
satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `bech32_core` | The BCH checksum as one integer fed five bits at a time, the charset both ways, the two variant constants, and the format's numbers. |
| `bechbits` | The eight-to-five regrouping and its inverse, as a value the caller drives, with the padded and the exact way of ending it. |
| `bech32` | Strings: encode and decode, the three rules that are about a whole string, the eleven named refusals, and the size arithmetic. |
| `bechsegwit` | Segwit addresses: the version-to-encoding rule, the program length rules, encode and decode, and the `scriptPubKey` bytes. |

## How to choose an entry point

**`bechsegwit` is the entry point for a Bitcoin address.** It applies
the rules that a general bech32 reader cannot know: which encoding the
version requires, and which program lengths that version allows.

**`bech32.encode` and `bech32.decode` are for every other use.** BOLT 11
lightning invoices, Nostr identifiers and several PSBT fields are bech32
strings that are not addresses.

**`bech32.encode_into` and `bech32.decode_into` write into a `Cursor`
you already own.** `decode_into` answers the prefix as a range into the
string you passed in, so nothing is copied.

**`bech32_core` and `bechbits` are the format for firmware.** They take
and answer integers, and the caller drives the loop. See "Running on a
microcontroller".

## The rules a user needs

1. **The witness version chooses the encoding.** Version 0 is bech32.
   Versions 1 through 16 are bech32m. A version-1 address encoded as
   bech32 is invalid, and a version-0 address encoded as bech32m is
   invalid. BIP 350 is that rule, and `bechsegwit.variant_for` is the
   one function that states it.
2. **A string valid under the other variant is its own refusal.**
   `BechWrongVariant` rather than `BechBadChecksum`. A bech32m address
   handed to a bech32-only reader is a working address in an application
   that has not been updated.
3. **The separator is the last `1` in the string.** A human-readable
   part may contain `1`, and `1` is not in the data charset. BIP 173's
   83-character prefix vector exists to catch a decoder that scanned
   from the left.
4. **Either case alone is accepted and a mixture is refused.** An
   address may be printed in upper case. `BechMixedCase(at)` names the
   first character whose case disagrees. `bech32.encode` lower-cases the
   prefix on the way in.
5. **The checksum covers the prefix twice.** The high three bits of
   every prefix character, then a zero, then the low five bits. That is
   what makes the checksum notice a change of case in the prefix, which
   is BIP 173's invalid vector `A1G7SGD8`.
6. **The 90-character limit applies to the whole string.** Prefix,
   separator, data and checksum. It exists because the code guarantees
   detection of up to four errors only within that length.
7. **The limit is lifted by a different call, not by a flag.**
   `bech32.encode_unlimited` and `bech32.decode_unlimited` do not
   enforce it, which is what BOLT 11 invoices need. A caller using them
   gives up the four-error guarantee, and the name says so at the call
   site.
8. **Five-to-eight refuses a tail whose bits are set.** BIP 173 requires
   the padding bits to be zero. Accepting them would let two different
   strings decode to the same bytes. `bechbits.finish_exact` is that
   rule, and `bechbits.finish_padded` is the eight-to-five one.
9. **BIP 141 fixes the program length by version.** Version 0 is exactly
   20 bytes for a key hash or exactly 32 for a script hash. Versions 1
   to 16 take 2 to 40. `bechsegwit.check_program` answers it, and a
   version-0 string with a 21-byte program is well-formed bech32 and is
   not an address.
10. **The network prefix is the caller's, and is not checked against a
    list.** `bechsegwit.decode` answers the prefix it found.
    `bechsegwit.decode_for` is the call for a program that accepts one
    network, and it compares case-insensitively.
11. **A decode answers the prefix as a range, not a string.**
    `BechRead.hrp_start` and `hrp_end` index the string the caller
    already holds.
12. **`bech32_core` reports through the value, not a `Result`.** It
    answers integers and booleans, because a `@value` struct may not be
    a `Result` payload. `bech32` turns each case into one of its named
    refusals.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device
with no heap allocator, and the compiler checks that claim on every
build. Here the claim covers `bech32_core` and `bechbits`. Both take and
answer `Int` and `u8`, and neither holds a buffer.

`tests/embedded_probe.nv` is that claim as a program that either builds
or does not. It builds today:

```bash
novo build --target=nrf52-qemu tests/embedded_probe.nv
```

The probe produces a Cortex-M4 executable that runs the checksum over
the prefix `bc` and a twenty-byte program regrouped on the fly, reads
the six checksum characters out, and verifies them again in the other
direction. It builds and it is not run: every function it calls is a
`todo()` today.

The consumer for this is a hardware wallet. The device that signs a
transaction is the device that must show the address on its own screen,
because an address shown on the host is an address the host could have
replaced. It has no heap and no room for a second copy of the string.

**A device cannot use `bech32` or `bechsegwit`.** They speak `Str`,
`Bytes` and `Cursor`, and the embedded runtime defines none of those.
One host-only function anywhere in a compilation unit is an undefined
symbol at link time on a device, whether or not the firmware calls it.

## What is not included

- **A list of networks.** The prefix is an argument. A package that
  owned the list would need a release every time a test network
  appeared.
- **Bitcoin script, beyond one function.**
  `bechsegwit.script_pubkey` answers the version opcode and a push of
  the program, because a caller that decoded an address almost always
  wants it next. Nothing else about script is here.
- **Key derivation, transactions and the network protocol.** This
  package is an encoding.
- **Base58Check, the older Bitcoin address encoding.** A different
  alphabet and a different checksum.
- **A dependency on
  [bitstream-nv](https://novo-lang.org/packages/bitstream-nv).** The
  regrouping is a fixed five bits against eight with a two-value bound
  on every step, so a general variable-width bit reader would carry a
  width argument through the inner loop for nothing.
- **Any input or output.** Every function here is arithmetic over
  characters and bytes the caller already holds.

## Related packages

- [base64-nv](https://novo-lang.org/packages/base64-nv) writes bytes as
  text six bits at a time, with no checksum and no prefix.
- [bitstream-nv](https://novo-lang.org/packages/bitstream-nv) reads and
  writes runs of bits of any width, which is the general form of the
  five-bit regrouping here.
- [crc-nv](https://novo-lang.org/packages/crc-nv) is the other
  error-detecting code on the registry. A CRC protects bytes on a wire;
  this checksum protects characters a person may retype.
- [crypto-nv](https://novo-lang.org/packages/crypto-nv) has the hashes a
  wallet needs to produce a witness program in the first place.

## Tests

```bash
novo test tests/bech32_tests.nv       # 51 tests
```

Every vector is from BIP 173 or BIP 350: the valid bech32 strings, the
invalid ones with the reason each is invalid, the bech32m set, and the
segwit addresses. The surface follows the `bech32` crate in Rust and the
reference Python in BIP 173 itself.

The suite asserts that the published valid vectors decode and that
encoding reproduces them, that a prefix containing `1` finds the last
separator, that either case alone means the same string and a mixture
does not, that the two variants are told apart by the checksum rather
than the prefix, that asking for the wrong variant is its own refusal,
that each of the eleven refusals is raised by the vector that earns it,
that the loop BIP 173 writes in prose produces the vector's checksum,
that eight-to-five pads and five-to-eight refuses a set tail, that the
version chooses the encoding, that BIP 141's program lengths are
enforced, that the prefix is not checked against a list, and that the
`scriptPubKey` is the version opcode and a push.

The tests compile today and fail at run, each on the `not implemented`
panic that is its body. That is the expected state of an interface
release. They turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `bech32_core.variant_bech32`, `.variant_bech32m`, `.variant_const`, `.variant_of` | no |
| `bech32_core.checksum_len`, `.max_length`, `.separator` | no |
| `bech32_core.charset_at`, `.charset_index`, `.hrp_high`, `.hrp_low` | no |
| `bech32_core.start`, `.feed`, `.finish`, `.checksum_value`, `.residue`, `.verify` | no |
| `bechbits.to5`, `.to8`, `.feed`, `.finish_padded`, `.finish_exact`, `.out_len` | no |
| `bech32.variant_code`, `.variant_from_code` | no |
| `bech32.encoded_len`, `.decoded_len` | no |
| `bech32.encode`, `.encode_unlimited`, `.encode_into` | no |
| `bech32.decode_into`, `.decode_unlimited`, `.decode`, `.decode_variant`, `.is_valid` | no |
| `bech32.bytes_to_data`, `.data_to_bytes` | no |
| `bech32.BechError.message` | no |
| `bechsegwit.variant_for`, `.check_program`, `.encoded_len` | no |
| `bechsegwit.encode`, `.encode_into` | no |
| `bechsegwit.decode`, `.decode_for`, `.is_address`, `.script_pubkey` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
