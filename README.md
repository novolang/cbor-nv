# cbor-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

RFC 8949 CBOR: JSON's data model plus byte strings, plus tags, in a
binary encoding where every value is three bits of major type, five bits
of additional information and an argument.  A `{"a":1}` costs seven
bytes as JSON and four here; a small integer costs one; and a producer
that does not know a length in advance can say so rather than buffering
the whole thing first.

Four surfaces, and a reader should know which one they are on.

| surface | module | reach for it when |
| --- | --- | --- |
| the **value tree** | `cborfmt` | the other end is not novo-lang, the document's shape is not a struct, or a tag is involved |
| the **stream** | `cbordec` | the document arrives in pieces |
| the **trait bridge** | `cborserde` | both ends are novo-lang and you would rather not write any code |
| the **head codec** | `cborhead` | you are on a device |

## Adding it, and checking it

```bash
novo pkg add cbor-nv          # into your novo.toml
novo pkg build                # type- and effect-check the package
novo test --isolate tests/cbor_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: cborfmt.<fn>`.  They turn green
one at a time as bodies land.

## The one example that will work

```novo
use std.bytes
use cborfmt

fn main() [io]
    let doc = CborMap([
        CborPair { key: CborText("id"), value: CborUnsigned(7) },
        CborPair { key: CborText("ok"), value: CborBool(true) },
    ])
    println(bytes.to_hex(cborfmt.encode(doc, CborCanonical)))
    // a2626964 07626f6b f5 — a two-entry map in nine bytes, keys sorted
```

## The device half is the point

The grid's row for this package says the embedded tier prefers CBOR, and
this release makes that a claim the compiler checks rather than a
sentence: `tests/embedded_probe.nv` compiles `cborhead` to a Cortex-M4
ELF for `--target=nrf52-qemu`.

What a device can do with a head codec alone is more than it sounds.
RFC 8949 § 3 puts every value behind one shape — major type, additional
information, argument — so a sensor producing CBOR writes an array head,
then a map head and a few integer heads per reading, and **never builds a
tree at all**.  A consumer on a device walks heads and skips what it does
not want, using nothing but the argument in each one.  That is the whole
of CBOR that firmware needs.

`cborfmt`, `cbordec` and `cborserde` are deliberately outside the probe:
they speak `Bytes`, `Str` and a recursive value tree, the embedded
runtime defines no `novo_bytes_*` symbol, a tree is a heap allocation per
node, and one host-only function anywhere in a compilation unit is an
undefined symbol at embedded link time whether or not the firmware calls
it.

**msgpack-nv, this package's sibling, carries no probe**, and the split
is on purpose: the two formats are close enough that an embedded
producer choosing between them should choose the one that compiles.

## serde-nv already has a CBOR module — why this one?

`serde-nv`'s `cbor` module is the **subset its trait walk needs**: a
writer that emits definite lengths for the shapes a novo-lang struct
produces, and an offset cursor that reads them back.  Its own header
says so — "nothing here writes an indefinite-length header".

This package is the whole of RFC 8949.  What it adds:

- **A value tree** for documents whose shape is not a struct.
- **Indefinite lengths**, read and collapsed, which is the form a
  streaming producer writes.
- **The tags a program actually meets**: date/time in both forms,
  bignums, decimal fractions, bigfloats, and the self-described prefix a
  CBOR file on disk begins with.
- **Half-precision floats**, which are three bytes where a double is
  nine and are what an embedded producer sends.
- **Core deterministic encoding** as a named mode, and `is_canonical` to
  check a document against it.
- **A streaming decoder**, and **a head codec that builds for a device**.
- **Named refusals with offsets**: reserved additional information, an
  indefinite head where none is allowed, a stray break, a mismatched
  chunk, non-UTF-8 text, a depth limit, trailing bytes.

Both can be in one program: the module names and the type names are
disjoint on purpose (`cborfmt` and `CborValue` here, `cbor` and
`CborWriter` there).  That disjointness is also why this module is not
called `cbor` and why the trait bridge's types are `CborEmitter` and
`CborCursor` rather than `CborWriter` and `CborReader` — a module name
and a public type name are each unique across the whole assembly, and
serde-nv had all three first.

**This package does not depend on serde-nv.**  The `Serializer` and
`Deserializer` traits are the standard library's (`std.serialize`), and
a dependency would put a second CBOR implementation in every consumer's
assembly.

## Is the serde bridge writable today? Yes — both halves

This is the question the interface milestone exists to answer, and for
this format the answer is yes, where for postcard-nv it was half no.
The difference is entirely that CBOR is self-describing.

**The read half works.**  postcard-nv's `Deserializer` cannot be written
correctly because `field(self, name)` answers a child cursor and leaves
the parent unchanged, and a nameless format's member 2 begins wherever
member 1 ended — which the parent has no way to learn.  CBOR writes a
key in front of every member, so `field("beta")` **scans the map at the
parent's own offset** and needs no threading at all.

**The write half works too.**  postcard-nv cannot write an optional
because the trait announces `None` (as `put_null`) and announces `Some`
not at all.  CBOR needs no discriminant: `null` is `0xf6` and is
distinguishable from every other value by its own head, so `None` is
`put_null` and `Some(x)` is `x`.

So both stdlib defects postcard-nv found are consequences of a
**nameless, untagged** format, and a self-describing one meets neither.

## Where the trait bridge is still short of the format

Five places, and every one has the value tree as its answer.  None of
them blocks the impl.

**One integer hook.**  `put_int(v: Int)` is all there is, so an unsigned
value at or above 2^63 is unreachable, and anything beyond 64 bits needs
a bignum tag this trait cannot write.  Both are built with `cborfmt`.

**One float hook.**  `put_float(v: Float)` writes a double, always.  Half
precision — three bytes for a temperature against nine — is reached
through `CborFloat16`.

**No bytes hook.**  The trait has `put_str` and nothing for `Bytes` —
the standard library declares `Serialize` for `Int`, `Float`, `Bool` and
`Str` and for nothing else — so major type 2 is unreachable from the
walk.  A member that is really bytes travels as text of whatever the
caller encoded it to, or the document is built with `CborBytes`.  **This
is the one of the five that is a standard library gap rather than a
novo-lang/CBOR impedance**: a `Serialize` impl for `Bytes` and a
`put_bytes` hook would close it, and would close the same gap for
msgpack-nv.

**No tag hook.**  A profile that wraps its values in a tag — COSE, CWT,
a self-described file — cannot be written through the walk at all.

**Struct keys are always text.**  `begin_struct` and `field(name)` are
the only way into a map, so a document whose keys are integers — which
every size-constrained CBOR profile uses — has to be built with
`CborMap`.

And one thing that is a **cost** rather than a shortfall: the bridge
writes `CborShortest` and cannot write `CborCanonical`, because the trait
presents members in declaration order and gives the format no chance to
sort them.  A caller who needs the deterministic encoding decodes and
re-encodes with `cborfmt`.

## Indefinite lengths are read and collapsed

A value tree has nowhere to put "this array's length was not announced":
the elements are the same elements either way.  So an indefinite array
decodes to `CborArray`, an indefinite byte string to the concatenation
of its chunks, and `encode` writes definite lengths throughout.

What that costs is that **bytes** do not round-trip for an indefinite
document, only **values** do — and a caller verifying a signature over
the original bytes has to know.  `cbordec.saw_indefinite` is what tells
it, and a producer that needs to write the indefinite form writes heads
with `cborhead`.

## The layer, and why

`core`.  Everything here is arithmetic over bytes the caller already
holds, and no function declares an effect — a wire format has nowhere to
put one.  `datetime_epoch_value` takes its seconds as a parameter for
the same reason a gzip header's mtime is a parameter: **a `core` package
has no clock**.  And the RFC 3339 text of a tag 0 is handed over
unparsed, because a `core` package has no calendar either — turning that
string into a civil date is calendar-nv's work, and a validator that
half-knew the grammar would be worse than one that does not claim to.

## Not leb128-nv, not zigzag-nv

postcard-nv depends on both, and a reader coming from there will look
for them here.  CBOR's argument is **big-endian and fixed width** — zero,
one, two, four or eight bytes chosen by the low five bits of the initial
byte — and is not a variable-length quantity at all.  The negative fold
is `-1 - n`, which looks like zigzag and is not: zigzag interleaves the
signs into one unsigned range, and CBOR gives negatives a major type of
their own.

## The reference implementation

RFC 8949 and `ciborium` (Rust, Apache-2.0) / `cbor2` (Python, MIT) as
the implementations to check against.  Every vector in
`tests/cbor_tests.nv` is from Appendix A — the specification's own table
of diagnostic-notation-and-encoding pairs — or from § 4.2.1's
deterministic encoding rules, so a reader can check the port against the
specification rather than against this package.

## Status

| function | implemented |
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
| `cborfmt.decode`, `.decode_prefix`, `.CborError.message` | no |
| `cbordec.decoder`, `.with_depth_limit`, `.pending`, `.stream_at` | no |
| `cbordec.saw_indefinite`, `.feed`, `.finish` | no |
| `cborserde.emitter`, `.emitter_bytes`, `.cursor`, `.cursor_at` | no |
| `CborEmitter`'s `Serializer` methods | no |
| `CborCursor`'s `Deserializer` methods | no |
| `cborserde.to_bytes`, `.from_bytes` | no |
