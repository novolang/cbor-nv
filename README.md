# cbor-nv

The Concise Binary Object Representation (CBOR) is a binary data format
for small messages and small code, specified in
[RFC 8949](https://www.rfc-editor.org/rfc/rfc8949). Its data model is
JSON's, with byte strings and tagged values added. This package
implements the format for novo-lang: a value tree, a streaming decoder,
a bridge to the standard library's serialization traits, and a head
codec that builds for a microcontroller. It depends on nothing.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What it is

A CBOR document is one **data item**. Every data item begins with an
**initial byte**: three bits of **major type**, which say what kind of
thing follows, and five bits of **additional information**. The
additional information is either the value itself, for the values 0 to
23, or it announces an **argument** of one, two, four or eight further
bytes, big-endian. RFC 8949 section 3 defines this shape, and every
value in the format has it.

| Major type | What it holds | The argument is |
| --- | --- | --- |
| 0 | An unsigned integer | the value |
| 1 | A negative integer | `n`, standing for `-1 - n` |
| 2 | A byte string | its length in bytes |
| 3 | A text string, UTF-8 | its length in bytes |
| 4 | An array | its element count |
| 5 | A map | its pair count |
| 6 | A tag | the tag number |
| 7 | Simple values and floats | the simple value or the float |

A **tag** is a number in front of one data item that says how to read
it. RFC 8949 section 3.4 defines them, and this package reads and
writes the ones a program meets.

| Tag | Meaning |
| --- | --- |
| 0 | A date and time as RFC 3339 text |
| 1 | A date and time as seconds since 1970-01-01T00:00:00Z |
| 2 | An unsigned bignum, as a big-endian byte string |
| 3 | A negative bignum, whose byte string `n` stands for `-1 - n` |
| 4 | A decimal fraction: `mantissa * 10 ^ exponent` |
| 5 | A bigfloat: the same pair, base 2 |
| 55799 | Self-described CBOR, the three-byte file prefix `d9d9f7` |

A string, an array or a map may be written with an **indefinite
length**: additional information 31 in place of a count, then the
contents, then the **break** byte `0xff`. RFC 8949 section 3.2 defines
the form. A producer that does not know how long a thing will be writes
it this way rather than buffering the whole thing first.

The **core deterministic encoding** of RFC 8949 section 4.2.1 is the
set of rules that makes two encoders produce identical bytes for equal
values: the shortest head for every argument, definite lengths
throughout, floats written at the narrowest width that round-trips, and
map keys sorted by the bytes of their own encoding. A document that is
hashed, signed or used as a cache key has to be in that form.

| Quantity | Value |
| --- | --- |
| Head length, shortest form | 1, 2, 3, 5 or 9 bytes |
| Argument widths | 0, 1, 2, 4 or 8 bytes |
| Largest value in a one-byte head | 23 |
| The break byte | `0xff` |
| Reserved additional information | 28, 29 and 30 |
| Indefinite-length additional information | 31 |
| Default nesting limit for a decode | 64 |
| Longest head an emitter writes | 9 bytes |

## Install

```
novo pkg add cbor-nv
```

## Example

```novo
use std.bytes
use cborfmt

fn main() [io]
    // A document as a tree: a map of two entries, built by hand.
    let doc = CborMap([
        CborPair { key: CborText("id"), value: CborUnsigned(7) },
        CborPair { key: CborText("ok"), value: CborBool(true) },
    ])

    // Encode it with the deterministic rules, so the same value always
    // gives the same bytes. The keys come out sorted.
    let wire = cborfmt.encode(doc, CborCanonical)
    println(bytes.to_hex(wire))

    // Read the bytes back and take one entry out by its text key.
    match cborfmt.decode(wire)
        Err(e) => println(e.message())
        Ok(v)  =>
            match cborfmt.get(v, "id")
                Err(e2) => println(e2.message())
                Ok(id)  => println(cborfmt.type_name(id))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a `not implemented:
cborfmt.<fn>` panic. The tests are the specification the
implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `cborfmt` | The value tree, the tags, the errors, the accessors, the encoder and the one-shot decoder. |
| `cbordec` | The same decoder fed a chunk at a time, for a document that arrives in pieces. |
| `cborserde` | The bridge to `std.serialize`, so a novo-lang struct writes and reads itself as CBOR. |
| `cborhead` | The initial byte and the argument, as a value type that allocates nothing. Builds for a microcontroller. |

## How to choose an entry point

**`cborfmt` is the format itself.** Build a `CborValue`, encode it,
decode bytes back into one. Use it when the other end is not novo-lang,
when the document's shape is not a struct, or when a tag is involved.

**`cbordec` is the same decoder for a document that arrives in
pieces.** The host feeds chunks in with `feed` and takes finished
values out. Use it when you are reading a socket and have no whole
document to hand over.

**`cborserde` writes a novo-lang type with no code to write.**
`to_bytes` and `from_bytes` take any type that implements the standard
library's `Serialize` and `Deserialize`. A struct becomes a map whose
keys are the member names.

**`cborhead` is the half a device can use.** It takes and returns
integers, holds no buffer and allocates nothing. See "Running on a
microcontroller".

## The rules a user needs

1. **An indefinite length is read and collapsed.** An indefinite array
   decodes to `CborArray` and an indefinite byte string to the
   concatenation of its chunks. The values round-trip and the bytes do
   not, so a caller verifying a signature over the original bytes must
   ask `cbordec.saw_indefinite` first. RFC 8949 section 3.2.
2. **Nothing in `cborfmt` writes an indefinite length.** Both encoding
   modes write definite lengths. A producer that needs the indefinite
   form writes heads with `cborhead.emit_indefinite` and closes them
   with `cborhead.emit_break`.
3. **A map is a list of pairs, not a keyed collection.** A CBOR key is
   any data item and not only a string, and a document may carry
   duplicate keys, which RFC 8949 section 5.6 calls invalid while
   leaving both on the wire. A validator has to be able to see both, so
   nothing here drops one.
4. **`cborfmt.get` looks up text keys only.** Whether the keys `1` and
   `1.0` are the same key is left to the application by RFC 8949
   section 5.6, so this package defines no equality over keys. A caller
   with non-text keys walks `as_map`.
5. **`CborCanonical` is the mode to encode in before hashing or
   signing.** It applies the core deterministic encoding of RFC 8949
   section 4.2.1. `CborShortest` writes the shortest heads and leaves
   the map order alone. `cborfmt.is_canonical` checks a document that
   arrived from somewhere else.
6. **`CborNull` and `CborUndefined` are different values.** They are
   major type 7 values 22 and 23. Where CBOR is used as a patch format,
   null means the member is absent and undefined means it is unchanged.
7. **A text string must be UTF-8 and a byte string need not be.** A
   decoder refuses non-UTF-8 text with `CborBadUtf8`. That distinction
   is what major type 2 exists for, and a `CborBytes` is not text even
   when its bytes happen to be valid UTF-8.
8. **The three float widths are kept apart.** `CborFloat16`,
   `CborFloat32` and `CborFloat64` record the width the document used,
   because the width is on the wire. `as_float` answers all three.
9. **An integer too large for a signed `Int` arrives as
   `CborWideInt`.** It carries the argument's 64-bit pattern and the
   major type it came from, never a reinterpreted negative number. A
   value beyond 64 bits is a bignum, tag 2 or tag 3, and `bignum_of`
   reads it.
10. **A decode refuses bytes after the document.** `cborfmt.decode`
    answers `CborTrailingBytes`. A caller reading one document out of a
    longer buffer uses `decode_prefix`, whose `consumed` field is the
    only way to find the next one, because CBOR has no framing.
    RFC 8742 calls that a CBOR sequence.
11. **A decode is limited to 64 levels of nesting.** An array head is
    one byte and opens a level, so a five-byte message can ask for
    thousands of them. Past the limit the decoder answers
    `CborDepthExceeded`, and `cbordec.with_depth_limit` moves the line.
12. **Additional information 28, 29 and 30 is reserved.** RFC 8949
    section 3 forbids it, and a document carrying it is refused with
    `CborReservedInfo`.
13. **Tag 0's text is handed over unparsed, and tag 1's seconds are an
    argument.** This package has no calendar and no clock. Turning an
    RFC 3339 string into a civil date is
    [calendar-nv](https://novo-lang.org/packages/calendar-nv)'s work.
14. **The serde bridge writes `CborShortest` and cannot write
    `CborCanonical`.** The trait presents members in declaration order
    and gives the format no chance to sort them. A caller who needs the
    deterministic encoding decodes the bridge's output and re-encodes
    it with `cborfmt`.
15. **Every failure carries the byte offset it was found at.** For
    `cbordec` that offset is counted from the start of the stream and
    not from the start of the chunk.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device
with no heap allocator, and the compiler checks that claim on every
build. Here the claim covers `cborhead` and nothing else.

`tests/embedded_probe.nv` is that claim as a program that either builds
or does not. It builds today:

```bash
novo build --target=nrf52-qemu tests/embedded_probe.nv
```

The probe produces a Cortex-M4 executable that writes a map head and an
integer head and walks a document's heads back. That is what a sensor
producing CBOR does, and it never builds a tree at all. `CborHeadScan`
and `CborEmit` are value structs, so they live in the caller's stack
frame, and the emitted head is a fixed nine-byte inline array rather
than a list.

The scan takes one byte at a time, because a device reading from a UART
has one byte and nothing else. `scan_need` says how many more bytes the
head wants, so a caller can wait for exactly that many.

**A device cannot depend on this package as a whole.** `cborfmt`,
`cbordec` and `cborserde` speak `Bytes`, `Str` and a recursive value
tree. The embedded runtime defines no `novo_bytes_*` symbol, a tree is
a heap allocation per node, and one host-only function anywhere in a
compilation unit is an undefined symbol at link time on a device,
whether or not the firmware calls it.

## What is not included

- **Writing an indefinite length from the value tree.** A value tree
  has nowhere to record that a length was not announced. `cborhead`
  writes the form; see rule 2.
- **Arithmetic on bignums.** `CborBigNum` carries the magnitude bytes
  and the sign, which is what the tags carry.
  [bigint-nv](https://novo-lang.org/packages/bigint-nv) is where
  addition lives.
- **A check that a tag 0 string is a valid RFC 3339 date.** This
  package has no calendar, and a validator that half knew the grammar
  would be worse than none.
- **A clock.** `cborfmt.datetime_epoch_value` takes its seconds as a
  parameter, which is also what makes a document reproducible.
- **Byte strings through the serde bridge.** The standard library
  declares `Serialize` for `Int`, `Float`, `Bool` and `Str` and for
  nothing else, so major type 2 is unreachable from the trait walk. A
  document that needs it is built with `CborBytes`.
- **Tags through the serde bridge.** The trait has no hook for one, so
  a profile that wraps its values in a tag, such as COSE or CWT, is
  built with `cborfmt`.
- **Integer map keys through the serde bridge.** `begin_struct` and
  `field(name)` are the only way into a map, so a profile with integer
  keys is built with `CborMap`.
- **Half-precision and oversized integers through the serde bridge.**
  The trait has one float hook, which writes a double, and one integer
  hook, which takes a signed `Int`. `CborFloat16` and the bignum tags
  are reached through `cborfmt`.
- **An equality over CBOR values.** See rule 4.
- **CDDL, COSE and CWT.** They are schema and security layers over this
  format, and each is a package of its own.

## Related packages

- [msgpack-nv](https://novo-lang.org/packages/msgpack-nv) is
  MessagePack, the other compact binary format with JSON's data model.
  It carries no device half, so an embedded producer choosing between
  the two chooses the one that compiles.
- [postcard-nv](https://novo-lang.org/packages/postcard-nv) is a
  nameless, untagged format for two ends that already share the type.
  CBOR carries a key in front of every member, which is why the read
  half of the bridge here can be written and postcard's cannot.
- [serde-nv](https://novo-lang.org/packages/serde-nv) has a `cbor`
  module of its own: the subset its trait walk needs, with definite
  lengths and no tags. Both packages can be in one program, because the
  module names and the public type names are disjoint. This package
  does not depend on it, and depending on it would put a second CBOR
  implementation in every consumer's assembly.
- `std.serialize` in the standard library declares the `Serializer` and
  `Deserializer` traits that `cborserde` implements. They are the
  standard library's traits and not serde-nv's.
- `std.json` in the standard library is the text format with the same
  data model, for the places a document is read by a person.

## Tests

```bash
novo test --isolate tests/cbor_tests.nv   # 56 tests: the Appendix A vectors, and more
```

The expected bytes are RFC 8949 Appendix A's own table of diagnostic
notation and encoding pairs, taken one per shape the interface has to
get right, and RFC 8949 section 4.2.1 supplies the deterministic
encoding rules. `ciborium` in Rust and `cbor2` in Python are the
implementations to check a port against.

The suite asserts that a one-byte head reaches 23 and 24 needs an
argument, that major type 1's argument means `-1 - n`, that both
integer majors read through one accessor, that an indefinite string is
collapsed to the concatenation of its chunks, that a chunk whose major
type differs is refused, that a break with nothing open is refused,
that reserved additional information is refused, that non-UTF-8 text is
refused, that the canonical mode sorts map keys by their encoded bytes,
and that a document nested past the limit answers `CborDepthExceeded`.

The tests compile today and fail at run, each on the `not implemented`
panic that is its body. That is the expected state of an interface
release. They turn green one at a time as bodies land.

`tests/embedded_probe.nv` is the device claim; see "Running on a
microcontroller".

## Implementation status

| Item | Implemented |
| --- | --- |
| `cborhead.major_*`, `.break_byte` | no |
| `cborhead.head_len`, `.initial_byte`, `.negative_value`, `.negative_arg` | no |
| `cborhead.scan`, `.push`, `.scan_need` | no |
| `cborhead.emit_head`, `.emit_indefinite`, `.emit_break` | no |
| `cborfmt.type_name`, `.initial_byte`, `.default_depth_limit` | no |
| `cborfmt.as_int`, `.as_bool`, `.as_float`, `.as_text`, `.as_bytes` | no |
| `cborfmt.as_array`, `.as_map`, `.get`, `.untagged` | no |
| `cborfmt.tag_*` | no |
| `cborfmt.datetime_text_value`, `.datetime_epoch_value`, `.datetime_of` | no |
| `cborfmt.bignum_value`, `.bignum_of`, `.decimal_value`, `.decimal_of` | no |
| `cborfmt.encoded_len`, `.encode`, `.encode_into`, `.is_canonical` | no |
| `cborfmt.decode`, `.decode_prefix`, `CborError.message` | no |
| `cbordec.decoder`, `.with_depth_limit`, `.pending`, `.stream_at` | no |
| `cbordec.saw_indefinite`, `.feed`, `.finish` | no |
| `cborserde.emitter`, `.emitter_bytes`, `.cursor`, `.cursor_at` | no |
| `CborEmitter`'s `Serializer` methods | no |
| `CborCursor`'s `Deserializer` methods | no |
| `cborserde.to_bytes`, `.from_bytes` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
