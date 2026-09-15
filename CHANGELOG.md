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

- Four surfaces.  `cborfmt` is the value tree, the tags and the
  encoder; `cbordec` is the same decoder fed a chunk at a time;
  `cborserde` is the bridge to the standard library's `Serialize` and
  `Deserialize`; `cborhead` is the initial byte and argument as a
  `@value` state machine, which is the half a device can use.
- All seven major types, all three float widths kept apart because the
  width is on the wire, `null` and `undefined` as the different things
  they are, and the unassigned simple values carried rather than
  refused.
- Indefinite lengths read and collapsed, with every rule RFC 8949 § 3.2
  attaches to them checked: an indefinite head only on the four major
  types that may have one, a break only where something is open, and
  every chunk of an indefinite string matching the string's own major
  type.
- The tags a program actually meets: date/time as text and as an epoch,
  unsigned and negative bignums, decimal fractions, bigfloats, and the
  self-described prefix a CBOR file on disk begins with.  `untagged`
  peels all of them, because tags nest.
- Core deterministic encoding as `CborCanonical`: shortest heads,
  definite lengths, floats shrunk to the narrowest width that
  round-trips, and map keys sorted by the bytes of their own encoding.
  `is_canonical` is the check a verifier runs before it hashes.
- Named refusals with offsets: reserved additional information, a bad
  indefinite head, a stray break, a mismatched chunk, non-UTF-8 text,
  nesting past the limit, trailing bytes, an accessor given the wrong
  type, a tag whose payload is the wrong shape, and a destination
  buffer too small.

**The device claim is built.**  `tests/embedded_probe.nv` compiles
`cborhead` to a Cortex-M4 ELF for `--target=nrf52-qemu`, driving the
head emitter and the byte-at-a-time head scanner the way firmware would.
A sensor producing CBOR writes heads and integers and never builds a
tree at all, which is what the grid's row means when it says the
embedded tier prefers this format.

**The serde bridge is writable today, both halves.**  postcard-nv
publishes a read half that is not, because a nameless format's member 2
begins where member 1 ended and the trait's cursor is not threaded; CBOR
writes a key in front of every member, so `field` scans the map at the
parent's own offset and the parent never has to move.  And postcard
cannot write a `Some` because the trait announces only `None`; CBOR
needs no `Some` marker, since `null` has its own head.  Both of
postcard's blockers are consequences of being nameless and untagged.

**Five places the bridge is still short of the format**, each with the
value tree as its answer: one integer hook, so an unsigned value at or
above 2^63 is unreachable; one float hook, so everything goes out as a
double; no bytes hook at all, so major type 2 is unreachable — the
standard library declares `Serialize` for `Int`, `Float`, `Bool` and
`Str` and nothing else; no tag hook, so a COSE or CWT profile cannot be
written through the walk; and no way to open a map whose keys are not
text.  The bridge also writes `CborShortest` and cannot write
`CborCanonical`, because the trait presents members in declaration order
and gives the format no chance to sort them.

**No dependencies.**  Not leb128-nv or zigzag-nv — CBOR's argument is
big-endian fixed width, and its `-1 - n` fold is not zigzag — and not
serde-nv, because the traits are the standard library's and a dependency
would put a second CBOR implementation in every consumer's assembly.
