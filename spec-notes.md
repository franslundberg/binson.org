# spec-notes

Notes for the development of the Binson specification: ideas, candidate
changes, open questions and background for future versions.

Versioning: a clarification that does not change which byte sequences are
valid Binson objects goes into a 1.x version. A real change to the format
(some byte sequences become valid or invalid) needs a new major version.


## 1. Single NaN encoding: 0x7ff8000000000000

Added 2026-10-06. Target: version 2 (a real change to the format).

**Proposal.** A double that is NaN must be stored as the bit pattern
0x7ff8000000000000 (positive quiet NaN). All other NaN bit patterns are
invalid.

**Background.** BINSON-SPEC-1.1 defines a double as its 64-bit pattern.
Every NaN pattern is then a distinct value, and the one-to-one mapping
between objects and bytes holds. But IEEE-754 has 2^53 - 2 NaN patterns,
and different languages produce different ones for "NaN":

- Java, Python, JavaScript, Swift, C, Rust: 0x7ff8000000000000.
- Go: math.NaN() returns 0x7ff8000000000001.
- x86 hardware: 0.0/0.0 gives 0xfff8000000000000 (sign bit set).

So the "same" NaN written from Go and from Java gives different bytes,
and thus different hashes and signatures. A single required encoding
removes this.

**Why 0x7ff8000000000000.** It is the default NaN in most languages and
it is a quiet NaN, so it does not trigger floating-point exceptions.

**Compatibility.** Every v2 object is still a valid v1 object, so v1
readers accept v2 output. v1 writers that emit other NaN patterns produce
output that v2 readers reject. BINSON-SPEC-1.1 recommends (non-normatively)
that writers use 0x7ff8000000000000, so writers that follow that
recommendation are already v2-compliant.

**Implementation survey (2026-10-06).** All linked implementations except
two pass the 8 bytes of a double through unchanged:

- binson-c corrupts negative NaNs whose top 33 bits are all ones (for
  example 0xffffffffffffffff is written as 0x00000000000000ff), because
  binson_util_pack_double() reuses the shortest-form integer packer. This
  is a bug under SPEC-1 already.
- binson-erlang cannot represent NaN or infinity at all (Erlang floats
  cannot hold them); decoding such a double crashes.

**Open questions.**

- Should a reader reject any other NaN pattern, or normalize it to
  0x7ff8000000000000? Rejecting matches the "if and only if" style of the
  rules; normalizing is friendlier but means the reader's output differs
  from its input.
- Is this alone worth a major version, or would a "strict profile" of 1.1
  be enough? Collect other candidates first.


## 2. Ordering of key-value pairs in maps

Added 2026-10-06. Target: a 1.x version (a recommendation, so no change
to which byte sequences are valid).

**Proposal.** Extend recommendation 5 (a map is stored as a single array
of alternating keys and values) with:

> When map keys are strings, key-value pairs should be stored in
> lexicographical order of the UTF-8 bytes of their keys, excluding
> encoding prefixes. Bytes should be compared as unsigned values. If one
> key is a prefix of another, the shorter key should come first.
> Duplicate keys should not occur.

**Background.** This uses the same ordering as rule 3 for field names.
"Excluding encoding prefixes" means that the stringLen bytes (type byte
and length) are not part of the comparison, only the UTF-8 bytes of the
key. With a defined order, a map has one serialization, just like an
object.

**Open questions.**

- How should key-value pairs be ordered when keys are not strings, for
  example integers or bytes? And what if a map mixes key types?
