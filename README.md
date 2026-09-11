# bech32-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

BIP 173 bech32 and BIP 350 bech32m — the checksummed encoding a Bitcoin
address is written in, and the one a lightning invoice and a Nostr
identifier are written in too.

Four modules, and the first two build for a Cortex-M:

- **`bech32_core`** — the BCH checksum as one integer, fed five bits at
  a time, in the same loop BIP 173 writes in prose.
- **`bechbits`** — the eight-to-five bit regrouping and its inverse, as
  a value the caller drives.
- **`bech32`** — those two with a string attached, plus the three rules
  that are about the whole string and that a per-character machine
  cannot see: the mixed-case refusal, the 90-character limit, and where
  the separator is.
- **`bechsegwit`** — segwit addresses, which is the one application the
  specification itself defines.

## The one rule BIP 350 exists for

**The witness version chooses the encoding.** Version 0 is bech32;
versions 1 through 16 are bech32m. A version-1 address encoded as bech32
is invalid, and a version-0 address encoded as bech32m is invalid.

A length-extension weakness was found in the original checksum for
strings whose final character is `p`, and splitting by version is how
the fix was deployed without invalidating every address that already
existed. `bechsegwit.variant_for` is that rule as one function, and
everything else in the module calls it rather than writing
`if version == 0` a second time.

`bech32.decode_variant` turns the mismatch into its own refusal —
`BechWrongVariant`, not `BechBadChecksum` — because a taproot address
handed to a bech32-only reader is a *working address in an application
that has not been updated*, and telling the user "bad address" would
send them looking in the wrong place.

## The layer, and why

`core`. A polymod and a shift over characters the caller already holds,
and no function declares an effect.

It carries `tests/embedded_probe.nv`, so the device claim is **built**
rather than asserted: `bech32_core` and `bechbits` speak `Int` and `u8`
and nothing else, and the probe compiles to a Cortex-M4 ELF for
`--target=nrf52-qemu` — driving the checksum over the prefix `bc`, a
twenty-byte program regrouped and fed straight in with no intermediate
buffer, the six checksum characters out, and the verify direction back.

**The consumer that needs it is a hardware wallet.** The device that
signs a transaction is the device that has to show the address on its
own screen, because an address shown on the host is an address the host
could have replaced. It has no heap and no room for a second copy of the
string, which is exactly what a `@value` checksum and a `@value`
regrouper are for.

`bech32` and `bechsegwit` are deliberately outside the probe: they speak
`Str`, `Bytes` and `Cursor`, and one host-only function anywhere in a
compilation unit is an undefined symbol at embedded link time whether or
not the firmware calls it.

## Adding it, and checking it

```bash
novo pkg add bech32-nv        # into your novo.toml
novo pkg build                # type- and effect-check the package
novo test --isolate tests/bech32_tests.nv
```

`novo test` is red today and that is the point of the release: all
fifty-one assertions fail with `not implemented: <module>.<fn>`. They
turn green one at a time as bodies land.

## The one example that will work

```novo
use std.bytes
use bechsegwit

fn main() [io]
    let prog = bytes.from_hex("751e76e8199196d454941c45d1b3a323f1433bd6") ?? bytes.zeros(0)

    // The encoding is chosen by the version, not by the caller.
    match bechsegwit.encode("bc", 0, prog)
        Ok(a)  => println(a)   // bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kv8f3t4
        Err(e) => println(e.message())

    match bechsegwit.decode("BC1QW508D6QEJXTDG4Y5R3ZARVARY0C5XW7KV8F3T4")
        Ok(a)  => println("${a.hrp} ${a.version} ${bytes.len(a.program)}")   // bc 0 20
        Err(e) => println(e.message())
```

## The load-bearing interface

`BechChk`, in `bech32_core`:

```novo
pub @value
struct BechChk
    chk: Int
```

One integer in the caller's frame — not the prefix, not the data, not a
position. The caller feeds five bits at a time and **the order it feeds
them in IS the algorithm**:

```novo
var c = bech32_core.start()
for each byte of the hrp:  c = bech32_core.feed(c, bech32_core.hrp_high(byte))
c = bech32_core.feed(c, 0)
for each byte of the hrp:  c = bech32_core.feed(c, bech32_core.hrp_low(byte))
for each data value:       c = bech32_core.feed(c, value)
```

then `finish` to produce a checksum or `verify` to check one. BIP 173's
"Checksum" section is that loop in prose, and keeping the shape
identical is what makes the port reviewable against the specification
rather than against this package.

Four consequences a reviewer should push on:

- **The prefix goes into the checksum twice** — high three bits of every
  character, a zero, then the low five. That is what makes the checksum
  notice a change in a character's *case* even though the data part is
  case-insensitive, which is BIP 173's `A1G7SGD8` invalid vector and a
  test here.
- **The two variants are one algorithm and two constants**, 1 against
  0x2bc830a3. Everything else about them is identical, which is why
  `variant_of` can read a finished residue and say which encoding a
  string used rather than the caller having to guess.
- **The decode writes into the caller's buffer and answers the
  human-readable part as a RANGE.** `BechRead` carries `hrp_start` and
  `hrp_end` into the string the caller already has — not a `Str`, which
  would be an allocation for something the caller can already see.
- **The two regrouping directions are not symmetric, and that asymmetry
  is what a wrong implementation gets wrong.** Eight to five pads the
  tail with zeros; five to eight requires the tail to be shorter than a
  byte and *all zero*, and discards it. `finish_padded` and
  `finish_exact` are those two rules named, with no flag between them —
  accepting a set padding bit would let two different strings decode to
  the same bytes.

## The 90-character limit, and the pair that lifts it

BIP 173 caps a bech32 string at 90 characters, because the BCH code only
guarantees detection of up to four errors within that length. It is a
property of the whole string — prefix, separator, data and checksum.

Some uses lift it deliberately: BOLT 11 lightning invoices are bech32
and are routinely longer. So `encode` and `decode` enforce it, and
`encode_unlimited` and `decode_unlimited` do not. **Two names rather
than a flag**, because a caller lifting the limit is choosing to give up
the error-detection guarantee, and that choice should appear at the call
site rather than in a variable.

## The separator is the LAST `1`

A human-readable part may contain `1` — `1` is not in the data charset,
so there is no ambiguity once you scan from the right. BIP 173's
`an83characterlonghumanreadablepartthatcontainsthenumber1and…` vector
exists to catch a decoder that scanned from the left, and it is a test
here.

## Why this does not depend on bitstream-nv

A reader who has seen bitstream-nv will expect it in the dependency
list, because bech32 is five bits at a time and that is what a bit
reader does.

The first reason is arithmetic: the regrouping is a **fixed** five bits
against eight with a two-value bound on every step, so the whole
converter is one shift, one mask and a count. A general reader that can
take any width from 1 to 64 carries a width argument through that inner
loop for no gain.

The second is the device claim: the shard audit builds the embedded
probe from the package's own core modules and nothing they depend on, so
a probe that reached a dependency could not be built at all today. That
limitation is filed against the toolchain rather than designed around
here — but with the first reason standing on its own, there is nothing
to design around.

## What is not here

No script interpreter, no network table, no key derivation. `hrp` is a
caller's argument — `"bc"`, `"tb"`, `"bcrt"` — because the set of
networks is not this package's to decide, and a consumer on a signet
with its own prefix should not have to wait for a release.
`bechsegwit.decode_for` is the call for a caller that does have one
network in mind.

The single piece of Bitcoin script is `bechsegwit.script_pubkey`, which
is here because a caller that has decoded an address almost always wants
it next and would otherwise write the version-to-opcode table itself.

## The reference implementation

`bech32` (Rust, MIT) and the reference Python in BIP 173 itself, with
**BIP 173** and **BIP 350** as the specifications. Every vector in
`tests/bech32_tests.nv` is from one of those two documents' test-vector
sections — the seven valid bech32 strings, the eleven invalid ones with
a stated reason each, the bech32m set, and the segwit addresses — so a
reader can check the port against the specification rather than against
this package.

## Status

| function | implemented |
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
