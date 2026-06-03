# war

> [!IMPORTANT]
>
> This is a **very early draft** for an archive format design.
> It does not represent a final state, and almost certainly contains
> mistakes, dead ends, etc.

Paperware for a [Way better ARchive format] for Python packaging.

[Way better ARchive format]: https://www.youtube.com/watch?v=ztZI2aLQ9Sw

See [SPEC.md](./SPEC.md).

## Goals

* Encode/decode performance: war should be fast to encode and decode.
  An idiomatic Python implementation should be measurably faster than
  the equivalent ZIP or tar-encoded archive would be with Python's
  `zipfile` or `tarfile`.
* Indexability/incrementality: it should be possible to efficiently stream and
  read only the relevant parts of a war, without having to decompress unrelated
  parts of the archive.
* Unambiguity: unlike tar, there should be only one way to express any given concept
  in a war. Consumers of a war should not have to be aware of different extensions
  or variants in order to properly encode or decode a war.

## Anti-goals

* Generality: war is not a general purpose archival format. It's specialized to
  Python packaging's needs, although it might also work (or serve as a reference)
  for other packaging ecosystems.
* Maximal compression: war does not attempt to squeeze every possible byte
  of compression out of its members. It trades some compressibility
  for indexability and per-member compression support.

## Inspirations

* [Nix Archive] (NAR)
* [eXtensible Archive] (XAR)

[Nix Archive]: https://nix.dev/manual/nix/2.22/protocols/nix-archive

[eXtensible Archive]: https://en.wikipedia.org/wiki/Xar_(archiver)