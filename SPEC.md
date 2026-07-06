# SPEC

_Specification version_: 0.0.2

## Structure of this document

This document uses Rust-style pseudocode to represent the various types/typestates in a war.

## Encoding

war does not currently define a binary encoding; the type definitions below
are intended to be abstract.

## High-level layout

war uses a header-index-store layout:

- The [_Header_](#header) uniquely identifies a war
- The [_Index_](#index) provides a canonical representation of the archive's names
- The [_Store_](#store) stores the archived data itself

All three parts of a war are required, and must be encoded contiguously.

### Header

The _header_ identifies the file as a valid war.

```rust
struct Header {
    /// Fixed: b"war!"
    magic: u32,
    /// Fixed: 1
    version: u8, 
}
```

#### `header.magic`

`header.magic` is always ASCII bytes `war!`, i.e. `0x77617221`.

A parser shall reject any other magic.

#### `header.version`

`header.version` is always 1. 

A parser shall reject any other version.

### Index

The [_Index_](#index) stores a canonical representation of the archive's names,
with references into the [_Store_](#store) for the metadata and contents corresponding
to each name.

Internally, the index is a [finite state transducer] that efficiently represents
the archive's names (by deduplicating common prefixes) and _transduces_ (i.e. maps)
each archive name to an offset into [_Store_](#store).

[finite state transducer]: https://en.wikipedia.org/wiki/Finite-state_transducer

```rust
struct Index {
    entries: Fst<Name>,
}

/// The name of an archive entry.
struct Name {
    components: Vec<NameSegment>,
}

/// A single segment of an archive entry's name.
struct NameSegment(String);
```

### Names

A [_Name_](#names) is a non-empty sequence of _name segments_, where a
[_Name Segment_](#name-segments) is the delimited segment of a host-specific logical path.

The following table shows the relationship between logical paths and name segments:

| Host OS | Logical path | Name segments |
| ------- | ------------ | ------------- |
| Windows | `foo\bar\baz`  | `["foo", "bar", "baz"]` |
| POSIX   | `foo/bar/baz` | `["foo", "bar", "baz"]` |

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
- Name segments may not match [names reserved by Windows] such as `CON`, `PRN`, and `AUX`

[names reserved by Windows]: https://learn.microsoft.com/en-us/windows/win32/fileio/naming-a-file

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
| `"CON"`                  | ❌    | Segment contains a forbidden name |


#### `index.entries`

`index.entries` is a non-empty mapping (in the form of an FST) of entry [_Names_](#names) to
storage offsets for each entry.

Each offset is a `u64`, which in turn is an offset relative to the start of the [_Store_](#store).

A parser shall reject an empty mapping.

### Store

The [_Store_](#store) stores compressed entries, as referenced by the [_Index_](#index).

```rust
struct Store {
    entries: Vec<StoreEntry>,
}

enum StoreEntry {
    /// A "regular file" entry.
    File(File),
    /// A directory entry.
    Directory(())
    /// A link entry.
    Link(Link),
}

/// A regular file's metadata and contents in the store.
struct File {
    /// A hint for the entry's decompressed size.
    uncompressed_size_hint: u64,

    /// The entry's compression.
    compression: Compression,

    /// The entry's metadata (for unpacking).
    metadata: Metadata,

    /// The file's data, compressed according to the compression field.
    data: Vec<u8>,
}

/// How a storage entry has been compressed.
enum Compression {
    /// No compression.
    Store,

    /// Compressed with DEFLATE.
    Deflate,

    /// Compressed with zstd.
    Zstd,
}

struct Metadata {
    /// The entry, when unpacked, should be marked as "executable" in a
    // host-specific manner.
    executable: bool,
}

struct Link {
    /// The link's target.
    target: Name,
}
```

#### `store.entries`

`store.entries` is a non-empty sequence of store entries.

A parser shall reject an empty sequence.

#### `store.entries[*]`

Each store entry is either a [_File entry_](#file-entries), a
[_Directory entry_](#directory-entries), or a
[_Link entry_](#link-entries):

### File entries

A file entry represents a "regular" file, and contains the file's (compressed binary data)
along with metadata needed to unpack the file.

#### `uncompressed_size`

`uncompressed_size` describes the size, in bytes, of the file entry after decompression.

#### `compression`

`compression` is the method used to compress the file entry.

war defines three compression methods:

- A "stored" entry is stored verbatim, with no compression.
- A "deflated" entry is compressed with the [DEFLATE] algorithm
- A "zstd" entry is compressed with the [zstd] algorithm

[DEFLATE]: https://datatracker.ietf.org/doc/html/rfc1951

[zstd]: https://datatracker.ietf.org/doc/html/rfc8878

#### `metadata`

`metadata` is an abstract representation of host-specific metadata that
should be preserved during unpacking.

- `metadata.executable`: the file entry should be made executable when unpacked

### Directory entries

A directory entry represents a directory. It is a marker (i.e. empty) entry that contains
no additional data besides being associated with a [_Name_](#names) via the [_Index_](#index).

Directory entries exist to explicitly mark a directory for creation during unpacking,
such as when a directory is not implied to be created through the creation of dependent
file entries. This occurs primarily when an archive wishes to create an empty directory.

### Link entries

A link entry represents a host-specific link between two [_Names_](#names) (a "source"
and a "target"). The source is described by the [_Index_](#index), and the target
is described within the link entry.

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
1. The [_Index_](#index) is iterated in order; for each index entry:
    1. The index entry's corresponding store entry is located;
    1. The store entry is unpacked at a path relative to the _temporary target directory_,
       corresponding to its name. See [_Entry unpacking_](#entry-unpacking) for how
       each kind of entry is unpacked.
1. The _temporary target directory_ is atomically renamed to the _target directory_.

### Entry unpacking

While [_Unpacking a war_](#unpacking-a-war), store entries are handled as follows:

- [_File entries_](#file-entries) are decompressed and written to disk at the path corresponding to their
  [_Name_](#names). For example, if the _temporary target directory_ is `/tmp/abc/` and the
  file entry's name is `["foo", "bar.txt"]`, then the entry is written to
  `/tmp/abc/foo/bar.txt`.

  An unpacker shall enforce that the decompressed size of an entry corresponds to the entry's
  [_`uncompressed_size`_](#uncompressed_size).

- [_Directory entries_](#directory-entries) are created by creating a directory at the path
   corresponding to their [_Name_](#names). For example, if the _temporary target directory_ is `/tmp/abc/`
   and the directory entry's name is `["def", "ghi"]`, then the directory is created as
   `/tmp/abc/def/ghi/`.

   Observe that this implies the creation of intermediate directories as well, in this case `/tmp/abc/def/`.

- [_Link entries_](#link-entries) are created in a host-specific manner. For example, a POSIX host may choose
  to create a symbolic link, whereas a Windows host may choose to create an NTFS junction.
  A war unpacker is _not_ required to enforce that the link's target exists at unpack time.
  However, by construction, a link target will always be relative to the _target directory_
  and will never point outside of it.

    - Caveat: On Windows, NTFS junctions are always absolute, even when created from a relative
      reference. Consequently, an atomic rename operation on a directory (e.g. from
      the _temporary target directory_ to the _target directory_) will result in broken
      junctions. The "hack" for fixing this is to create junctions within the
      _temporary target directory_, but have them point to the _target directory_.
      This effectively produces an invalid intermediate state, but preserves atomicity
      when the _temporary target directory_ is renamed.

Entries are created with host-specific permissions, modulo the "executable" marker on a
file entry. When a file entry is marked as "executable," the resulting file should include
the appropriate execute permission.

## Additional considerations

- There are additional differentials that war could eliminate. For example, war
  could require a specific Unicode normal form, such as NFC. Note that macOS uses
  NFD for normalization within APFS, whereas Windows generally uses NFC (and Linux
  is largely agnostic, but often uses NFC by convention).
