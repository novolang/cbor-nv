# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## Unreleased

### Changed

- Internal: ladders rewritten as `match` with range patterns, and errors are propagated with `!`. No public signature changed.

### Fixed

- A head scanned a byte at a time takes a nine-byte argument without overflowing, a cursor stepping over members refuses nesting past the decoder's depth limit, and an indefinite-length string of many chunks is joined in one pass.

## 0.1.0 — 2026-10-11

The implementation.  Every body published as `todo()` in 0.0.1 is
written, and `tests/cbor_tests.nv` — the interface release's own suite,
unchanged but for its stamp — passes as written.

Added:

- `cborfmt.decode_prefix_limited`: `decode_prefix` under a nesting
  limit the caller carries.  `cbordec.feed` decodes one value at a time
  out of a buffer that also holds a partial one, and the limit has to
  be the decoder's own rather than the default's.
- `tests/cbor_nv_edge_tests.nv`: 40 tests over the head arithmetic at
  both ends of its ranges, every refusal the reader and the bridge
  make, the float widths at their edges, the UTF-8 matrix, the
  indefinite forms the value tree collapses, the documents a reader
  accepts and the writer would not produce, the stream's offsets and
  its resumability, and the writer's own paths.

### Changed

- `x == false` is written `not x` at the six sites the lint's
  `redundant-bool` rule names (`tests/cbor_nv_edge_tests.nv`).  No
  signature changed.

### Known

- **Text that decodes to a string containing a NUL loses everything
  from the NUL on.**  A novo `Str` is NUL-terminated and `bytes.to_str`
  stops there, so a document carrying such a text reads back short.
  Nothing refuses it, and `decode` cannot tell.
- **A `CborSimple` outside 0..255 is masked** when it is written back:
  `CborSimple(300)` encodes as `f8 2c`.  Only a value built by hand can
  be out of range; the reader never answers one.
- **A length at or above 2^63 saturates** instead of reporting an
  exact shortfall: a head announcing it answers "needs
  9223372036854775807 more bytes", which is a lower bound, not the
  count.
- **A chunk that is itself indefinite, inside an indefinite string, is
  `CborBadChunk`** rather than a refusal of its own — the spec has a
  name for the rule and not for the case.
- **A missing map key is `CborWrongType`** whose `want` is the key and
  whose `saw` is `"absent"`; the cursor's own answer is
  `no member named <key>`.  A reader has no refusal of its own for
  "the map is fine and the key is not in it".
- **RFC 8949 § 3.3's two-byte `f8 nn` with `nn` below 32 is read, not
  refused.**  Re-encoding writes the one-byte form, so `is_canonical`
  reports such a document as not canonical, which is the right answer
  to a question that has nothing to do with the read.
- **The bridge writes a double for every float.**  The standard
  library's `Serializer` has one float hook, so `CborFloat16` and
  `CborFloat32` are reachable from the value tree and not from the
  walk.  Values are unaffected; the bytes are wider than they need to
  be.
- **A cursor over a malformed map answers `no member named …`** rather
  than failing: it walks member by member and a member it cannot read
  ends the walk.  The value tree's `decode` refuses the same bytes.
- **A variant constructor with arguments cannot be used directly as a
  method receiver.**  The compiler leaves such a receiver unresolved
  and the LLVM leg refuses the module (IR verification, undefined
  `@novo_user_<Enum>_serialize`); the tests bind the value to an
  annotated `let` first, which is the workaround.  Filed as
  `bugs/codegen-llvm/a-variant-constructor-as-a-method-receiver-is-left-unresolved.md`.

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
