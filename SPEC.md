# SPEC

_Specification version_: 0.0.1

## Structure of this document

This document uses Rust-style pseudocode to represent the various types/typestates in a war.

## Encoding

> [!NOTE]
>
> relish is being used as the encoding format for rapid prototyping/demonstration of
> the design. A more permanent format will likely require a slightly distinct format
> that is more streaming-friendly.

war currently uses [relish] as its encoding format.

See [relish]'s [SPEC][relish-spec] for concrete details on relish's binary serialization.

[relish]: https://github.com/alex/relish

[relish-spec]: https://github.com/alex/relish/blob/main/SPEC.md

## High-level layout

war uses a header-index-heap layout:

- The [_Header_](#header) uniquely identifies a war
- The [_Index_](#index) provides a canonical representation of the archive's metadata
- The [_Heap_](#heap) stores the archived data itself

All three parts of a war are required. Each is encoded as a _discrete_ relish
message.

### Header

The _header_ identifies the file as a valid war.

```rust
#[derive(relish::Relish)]
struct Header {
    /// Fixed: 1
    #[relish(field_id = 0)]
    version: u8, 
    /// Fixed: b"war!"
    #[relish(field_id = 1)]
    magic: u32,
}
```

Observe that, because a war is a valid relish message, the header is prefixed by
its surrounding framing (i.e. the tag and length components for the `Header`
type itself).

#### `header.version`

`header.version` is always 1. 

A parser shall reject any other version.

#### `header.magic`

`header.magic` is always ASCII bytes `war!`, i.e. `0x77617221`.

A parser shall reject any other magic.

### Index

The [_Index_](#index) stores a canonical representation of the archive's metadata,
including references into the [_Heap_](#heap).

```rust
#[derive(relish::Relish)]
struct Index {
    #[relish(field_id = 0)]
    entries: HashMap<Name, IndexEntry>
}

/// The name of an archive entry.
struct Name {
    #[relish(field_id = 0)]
    components: Vec<NameSegment>,
}

/// A single segment of an archive entry's name.
#[derive(relish::Relish)]
struct NameSegment(#[relish(field_id = 0)] String);

/// A single index entry, providing metadata for an archive entry.
#[derive(relish::Relish)]
struct IndexEntry {
    /// The entry's index, within the heap. 
    #[relish(field_id = 0)]
    index: u64,

    /// A hint for the entry's decompressed size.
    /// Observe that this is a _hint_, not a guaranteed size,
    /// as a maliciously-constructed archive can always lie in its index.
    #[relish(field_id = 1)]
    uncompressed_size_hint: u64,

    /// The entry's compression.
    #[relish(field_id = 2)]
    compression: Compression,

    /// The entry's metadata (for unpacking).
    #[relish(field_id = 3)]
    metadata: Metadata,

    #[relish(field_id = 3)]
    integrity: Integrity
}

/// How an archive entry has been compressed.
#[derive(relish::Relish)]
enum Compression {
    /// No compression.
    #[relish(field_id = 0)]
    Store,

    /// Compressed with DEFLATE.
    #[relish(field_id = 1)]
    Deflate,

    /// Compressed with zstd.
    #[relish(field_id = 2)]
    Zstd,
}

#[derive(relish::Relish)]
struct Metadata {
    /// The entry, when extracted, should be marked as "executable" in a
    // host-specific manner.
    #[relish(field_id = 0)]
    executable: bool,
}

#[derive(relish::Relish)]
struct Integrity {
    #[relish(field_id = 0)]
    crc32c: u32,
}
```

#### `index.entries`

`index.entries` is a non-empty mapping of entry names to index metadata for each entry.

A parser shall reject an empty mapping.

#### `index.entries[name]`

The [_Name_](#names) of each index metadatum is the name of the archived entry.

A [_Name_](#names) is a non-empty sequence of _name segments_, where a
[_Name Segment_](#name-segments) is the delimited segment of a host-specific logical path.

The following table shows the relationship between logical paths and name segments:

| Host OS | Logical path | Name segments |
| ------- | ------------ | ------------- |
| Windows | `foo\bar\baz`  | `["foo", "bar", "baz"]` |
| POSIX   | `foo/bar/baz` | `["foo", "bar", "baz"]` |


See [_Names_](#names) and [_Name Segments_](#name-segments) for validation requirements.

### `index.entries.*.index`

`index.entries.*.index` is a given entry's index into the heap.

A parser shall reject any war whose heap index does not reference a heap entry.

### `index.entries.*.uncompressed_size_hint`

`index.entries.*.uncompressed_size_hint` is a _hint_ for the uncompressed entry's size.

### `index.entries.*.compression`

`index.entries.*.compression` is the method used to compress the entry (in the heap).

war defines three compression methods:

- A "stored" entry is stored verbatim, with no compression.
- A "deflated" entry is compressed with the [DEFLATE] algorithm
- A "zstd" entry is compressed with the [zstd] algorithm

[DEFLATE]: https://datatracker.ietf.org/doc/html/rfc1951

[zstd]: https://datatracker.ietf.org/doc/html/rfc8878

### `index.entries.*.metadata`

`index.entries.*.metadata` is an abstract representation of host-specific metadata that
should be preserved during extraction.

- `index.entries.*.metadata.executable`: the entry should be made executable when extracted

### `index.entries.*.integrity`

`index.entries.*.integrity` provides integrity guards for accidental data corruption.

- `index.entries.*.integrity.crc32c`: the Castagnoli CRC-32 (polynomial `0x1EDC6F41`) of the entry's heap representation

CRC-32C is selected for having hardware acceleration in common ISAs (x86_64 with SSE4.2, AArch64).

A parser shall reject any war whose entry does not match its CRC-32C.

### Heap

The [_Heap_](#heap) stores compressed entries, as referenced by the [_Index_](#index).

```rust
#[derive(relish::Relish)]
struct Heap {
    #[relish(field_id = 0)]
    entries: Vec<HeapEntry>,
}

#[derive(relish::Relish)]
enum HeapEntry {
    #[relish(field_id = 0)]
    Data(Vec<u8>),
    #[relish(field_id = 1)]
    Link(Name),
}
```

#### `heap.entries`

`heap.entries` is a non-empty sequence of heap entries.

A parser shall reject an empty sequence.

#### `heap.entries[*]`

Each heap entry is either a _data entry_ or a _link entry_:

- A _data entry_ contains the binary data for its entry, compressed according to the
  referent index entry's compression method.
- A _link entry_ contains a [_Name_](#names) that the referent index entry's name
  shall point to (in a host-specific manner) upon extraction. This is called the "link target."

### Names

A parser shall reject a name sequence that is empty, i.e. has no name segments.

### Name Segments

A parser shall reject an empty name segment, i.e. an empty string.

In addition, the following rules apply to name segments:

- Name segements must be valid UTF-8 strings
- Name segments may not contain only Unicode whitespace
- Name segments may not have leading or trailing Unicode whitespace
- Name segments may not contain ASCII NUL (`\x00`) bytes
- Name segments may not contain ASCII control codes (Unicode's `Cc` category)
- Name segments may not contain conventional path separators (`/` or `\`)
- Name segments may not equal either of the conventional directory references (`.` or `..`)

The following is a short reference table demonstrating these rules:

| Name segment (in quotes) | Valid | Reason |
| ------------------------ | ----- | ------ |
| `"foo"`                  | ✅    | Violates no segment rules |
| `"foo.txt"`              | ✅    | Violates no segment rules |
| `"🦄"`                   | ✅    | Violates no segment rules |
| `"  "`                   | ❌    | Segment only contains whitespace |
| `"  foo"`                | ❌    | Segment has leading whitespace |
| `"foo   "`               | ❌    | Segment has trailing whitespace |
| `"\x00foo"`              | ❌    | Segment contains an ASCII NULL |
| `"foo\nbar"`             | ❌    | Segment contains a control code |
| `""`                     | ❌    | Segment is empty |
| `"foo/"`                 | ❌    | Segment contains a `/` |
| `"/foo"`                 | ❌    | Segment contains a `/` |
| `"foo\"`                 | ❌    | Segment contains a `\` |
| `"\foo"`                 | ❌    | Segment contains a `\` |

## Constructing a war

## Unpacking a war

Every war is unpacked ("extracted", "unarchived", etc.) relative
to a _target directory_.

If any step in the unpacking process fails, the entire procedure fails. 

The sequencing of the unpacking process is such that an unpacking error never
results in an intermediate or partial unpacking state in the _target directory_. 

Unpacking proceeds as follows:

1. The war's [_Header_](#header) and [_Index_](#index) are parsed
   according to the rules above.
1. A _temporary directory_ is created and used as the
   _temporary target directory_.
1. The [_Index_](#index) is iterated; for each index entry:
    1. The index entry's corresponding heap entry is located;
    1. The heap entry is unpacked at a path relative to the _temporary target directory_,
       corresponding to its name. See [_Entry unpacking_](#entry-unpacking).
1. The _temporary target directory_ is atomically renamed to the _target directory_.

### Entry unpacking

While [_Unpacking a war_](#unpacking-a-war), heap entries are handled as follows:

- _Data entries_ are decompressed and written to disk at the path corresponding to their
  [_Name_](#names). For example, if the _temporary target directory_ is `/tmp/abc/` and the
  entry's name is `["foo", "bar.txt"]`, then the entry is written to
  `/tmp/abc/foo/bar.txt`.

- _Link entries_ are created in a host-specific manner. For example, a POSIX host may choose
  to create a symbolic link, whereas a Windows host may choose to create an NTFS junction.
  A war unpacker is _not_ required to enforce that the link's target exists at extraction time.
  However, by construction, a link target will always be relative to the _target directory_
  and will never point outside of it.

    - Caveat: On Windows, NTFS junctions are always absolute, even when created from a relative
      reference. Consequently, an atomic rename operation on a directory (e.g. from
      the _temporary target directory_ to the _target directory_) will result in broken
      junctions. The "hack" for fixing this is to create junctions within the
      _temporary target directory_, but have them point to the _target directory_.
      This effectively produces an invalid intermediate state, but preserves atomicity
      when the _temporary target directory_ is renamed.

## Additional considerations

- There are additional differentials that war could eliminate. For example, war
  could require a specific Unicode normal form, such as NFC. Note that macOS uses
  NFD for normalization within APFS, whereas Windows generally uses NFC (and Linux
  is largely agnostic, but often uses NFC by convention).