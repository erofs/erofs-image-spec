# EROFS Image Layer Format Specification

This document is the normative specification of the EROFS image layer format.
It is an extension of the [OCI Image Format Specification][oci-spec]: image manifests, image indexes, and content descriptors are unchanged in shape, the image configuration shape is preserved, and two new media types are introduced alongside the existing tar-based layers.

For the project overview, status, and contributing guidance, see [`README.md`](README.md).

## Table of Contents

- [1. Introduction](#1-introduction)
  - [1.1 Notational Conventions](#11-notational-conventions)
  - [1.2 Conformance Statement](#12-conformance-statement)
  - [1.3 Relationship to the OCI Image Format Specification](#13-relationship-to-the-oci-image-format-specification)
  - [1.4 Relationship to EROFS](#14-relationship-to-erofs)
- [2. Media Types and Annotations](#2-media-types-and-annotations)
  - [2.1 application/vnd.erofs and +zstd](#21-applicationvnderofs-and-zstd)
  - [2.2 application/vnd.erofs.chunk-index.v1](#22-applicationvnderofschunk-indexv1)
  - [2.3 Annotations](#23-annotations)
  - [2.4 Roles](#24-roles)
- [3. Layer Format](#3-layer-format)
  - [3.1 Blob Structure](#31-blob-structure)
  - [3.2 Raw EROFS Filesystem Image](#32-raw-erofs-filesystem-image)
  - [3.3 Zstd-Compressed Filesystem Image](#33-zstd-compressed-filesystem-image)
  - [3.4 Chunk Index](#34-chunk-index)
    - [3.4.1 Header](#341-header)
    - [3.4.2 Chunk Entry](#342-chunk-entry)
    - [3.4.3 Delivery: Embedded or Standalone Layer](#343-delivery-embedded-or-standalone-layer)
    - [3.4.4 Reading a Range](#344-reading-a-range)
    - [3.4.5 Per-chunk Checksums](#345-per-chunk-checksums)
    - [3.4.6 Edge Cases](#346-edge-cases)
  - [3.5 dm-verity Merkle Tree](#35-dm-verity-merkle-tree)
  - [3.6 Whiteouts](#36-whiteouts)
  - [3.7 Hardlinks](#37-hardlinks)
  - [3.8 Layer Ordering](#38-layer-ordering)
- [4. Image Manifest](#4-image-manifest)
  - [4.1 Image Layout Interactions](#41-image-layout-interactions)
- [5. Image Configuration](#5-image-configuration)
  - [5.1 `rootfs.type`](#51-rootfstype)
  - [5.2 `rootfs.diff_ids` (optional)](#52-rootfsdiff_ids-optional)
  - [5.3 ChainID and ImageID](#53-chainid-and-imageid)
  - [5.4 Platform OS Features](#54-platform-os-features)
  - [5.5 Image Identity (informative)](#55-image-identity-informative)
- [6. Image Index](#6-image-index)
- [7. Applying EROFS Layers](#7-applying-erofs-layers)
- [8. Conformance Requirements](#8-conformance-requirements)
  - [8.1 Producer Requirements](#81-producer-requirements)
  - [8.2 Consumer Requirements](#82-consumer-requirements)
  - [8.3 Reproducibility](#83-reproducibility)
- [9. Conversion from Tar Layers (informative)](#9-conversion-from-tar-layers-informative)
  - [9.1 Content Hashing and Cross-Image Reuse (informative)](#91-content-hashing-and-cross-image-reuse-informative)
- [10. Considerations](#10-considerations)
  - [10.1 Extensibility](#101-extensibility)
  - [10.2 Tooling Compatibility](#102-tooling-compatibility)
  - [10.3 Non-Distributable Layers](#103-non-distributable-layers)
- [11. Worked Examples](#11-worked-examples)
  - [11.1 Multi-layer Overlayfs Image](#111-multi-layer-overlayfs-image)
  - [11.2 Single-layer EROFS Image](#112-single-layer-erofs-image)
  - [11.3 Composite Image: Device, Overlay-data, and Standalone Chunk-index](#113-composite-image-device-overlay-data-and-standalone-chunk-index)
  - [11.4 Device Blob Consumed by an Overlay Stack](#114-device-blob-consumed-by-an-overlay-stack)
- [12. Future Work](#12-future-work)
- [Appendix A. Sizing and Layout Guidance (informative)](#appendix-a-sizing-and-layout-guidance-informative)
- [Appendix B. Test Vectors (TBD)](#appendix-b-test-vectors-tbd)

## 1. Introduction

This specification defines a layer format and accompanying image-configuration extensions for distributing container image layers as [EROFS][erofs] filesystem images.
An EROFS layer is a complete read-only EROFS filesystem image that takes the place of a tar archive in the OCI layer model.
The format preserves the OCI image manifest, image index, content descriptor, and image configuration shape; only the layer payload bytes, the layer media types, and the interpretation of `rootfs.diff_ids` change.

The format is built around exactly two media types.
`application/vnd.erofs` (and its `+zstd` compression variant) identifies any valid EROFS filesystem image.
`application/vnd.erofs.chunk-index.v1` identifies a chunk index — a small binary index that maps chunks of a layer's image data to their on-blob locations and optional per-chunk checksums.
How each layer participates in image composition is determined by the optional `org.erofs.role` annotation rather than by the media type: the same `application/vnd.erofs` blob can serve as an overlayfs lower layer, an overlayfs data source, a raw byte-source device for EROFS multi-device addressing, or the root filesystem of a single-layer image.

This design differs from the tar-based layer format in three places only:

- The byte-level layout of the layer payload — an EROFS filesystem image instead of a tar archive (see [§3](#3-layer-format)).
- The interpretation of `rootfs.diff_ids` in the [image configuration][oci-config] (see [§5.2](#52-rootfsdiff_ids-optional)).
- The encoding of [whiteouts](#36-whiteouts), which uses the Linux kernel's [overlayfs][overlayfs] conventions directly rather than the `.wh.` filename convention used by tar layers.

### 1.1 Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119][rfc2119].

The keywords "unspecified", "undefined", and "implementation-defined" are to be interpreted as described in the [rationale for the C99 standard][c99].

Multi-byte integer fields in binary structures defined by this specification are stored in **little-endian** byte order unless otherwise noted.

The terms "media type", "descriptor", "image manifest", "image index", "image configuration", "DiffID", "ChainID", "ImageID", "annotation", and "filesystem changeset" are used as defined in the [OCI Image Format Specification][oci-spec].

The grammar of annotation values defined in [§2.3](#23-annotations) is given in the [EBNF subset defined by the OCI Image Format Specification][oci-ebnf].

### 1.2 Conformance Statement

An implementation is not conformant if it fails to satisfy one or more of the MUST, MUST NOT, REQUIRED, SHALL, or SHALL NOT requirements for the role it implements.
An implementation that satisfies all such requirements for its role is conformant.

The detailed requirements split between producers and consumers are listed in [§8](#8-conformance-requirements).

### 1.3 Relationship to the OCI Image Format Specification

This specification **extends** the [OCI Image Format Specification][oci-spec]; it does not replace any part of it.
Image manifests, image indexes, content descriptors, and the [image-layout][oci-image-layout] file structure are unchanged in shape and semantics.
The image configuration's structural shape is also unchanged; this specification makes `rootfs` and `rootfs.diff_ids` optional for EROFS layers and adds a required `os.features` value (see [§5](#5-image-configuration)).

The discriminator between an EROFS-bearing manifest and a tar-bearing manifest is the **layer media type**.
The platform descriptor's `os.features` array additionally carries the value `erofs` so that hosts that do not implement this specification can identify and skip EROFS-bearing manifests in an image index without parsing layer bytes.

An image manifest SHOULD use a single layer media type family within its `layers` array; mixing EROFS and tar layer media types within one manifest is not defined by this specification.
An image index MAY reference both EROFS-based and tar-based image manifests for the same software, allowing transports and runtimes to select the form they support.

### 1.4 Relationship to EROFS

This specification does **not** define a new on-disk filesystem format.
The layer payload is an EROFS image as defined by the [Linux kernel EROFS documentation][erofs-format]; this specification only defines how an EROFS image is wrapped, transported, and identified within the OCI image model.
Producers MAY use any conformant EROFS toolchain (`mkfs.erofs`, Go-language EROFS writers, etc.) to generate the underlying image.

## 2. Media Types and Annotations

### 2.1 `application/vnd.erofs` and `+zstd`

| Media Type | Description |
|---|---|
| `application/vnd.erofs`      | Raw EROFS filesystem image. The blob bytes are a kernel-mountable EROFS image. A dm-verity merkle tree MAY be appended directly after the EROFS image (see [§3.5](#35-dm-verity-merkle-tree)); when present, the tree covers data range `[0, hash_offset)` of the blob, matching the kernel's single-file dm-verity layout. An optional chunk index MAY be appended after the image data (see [§3.4](#34-chunk-index)). |
| `application/vnd.erofs+zstd` | Zstd-compressed EROFS filesystem image. The blob bytes are one or more zstd frames that decompress to the EROFS image data. A dm-verity merkle tree MAY be appended as part of the decompressed image data stream; it covers data range `[0, hash_offset)` of the decompressed stream. An optional chunk index MAY be embedded as a trailing zstd skippable frame (transparent to a standard zstd decoder). |

The `+zstd` structured suffix follows the convention established by [RFC 8478][rfc8478] and used by the `application/vnd.oci.image.layer.v1.tar+zstd` media type.

The media type alone does not determine how a layer participates in image composition.
The optional `org.erofs.role` annotation on the layer descriptor governs composition role; see [§2.4](#24-roles).
When `org.erofs.role` is absent, the layer is treated as an EROFS filesystem image forming part of an overlay stack or, in the single-layer case, the complete image root filesystem.

A consumer that encounters a layer media type whose subtype is unknown MUST refuse to apply the layer.
A consumer that encounters `application/vnd.erofs[+zstd]` but does not implement this specification MUST refuse to apply the layer; in particular, a consumer that implements only the tar-based layer format MUST NOT attempt to apply an EROFS layer as a tar archive.

Whether an EROFS layer carries inline file data or relies entirely on EROFS data-device addressing is a producer choice and does not affect the media type.

### 2.2 `application/vnd.erofs.chunk-index.v1`

| Media Type | Description |
|---|---|
| `application/vnd.erofs.chunk-index.v1` | A chunk index as defined in [§3.4](#34-chunk-index). Identifies the chunk-index format when referenced from a layer's descriptor annotations and when distributed as a standalone layer. |

The chunk index is a small binary index over a layer's image data.
It maps chunks to their on-blob locations and optionally carries per-chunk checksums that serve as content addresses.
The chunk index is the precondition for **lazy loading** — random-access reads of arbitrary byte ranges — and for **chunk-level deduplication** in a chunk-addressed block store.

This media type is used in two ways:

1. As the value of the `org.erofs.chunk-index.mediaType` annotation on an `application/vnd.erofs[+zstd]` layer descriptor, identifying the format of the chunk index embedded in that layer's blob (see [§3.4.3](#343-delivery-embedded-or-standalone-layer)).
2. As the `mediaType` of a standalone layer descriptor in `manifest.layers[]`, when the chunk index is distributed as its own blob rather than embedded in the parent layer's blob (see [§3.4.3](#343-delivery-embedded-or-standalone-layer)).

A consumer that recognizes the EROFS layer media type but does not recognize `application/vnd.erofs.chunk-index.v1` MUST treat the layer as if no chunk index were present, falling back to sequential reads.
A standalone `application/vnd.erofs.chunk-index.v1` layer that is not recognized by a consumer MUST be silently skipped during composition; the rest of the image remains valid and composable without lazy-loading benefit.
A consumer MUST NOT attempt to parse a chunk index whose media type it does not implement.

### 2.3 Annotations

The following annotations carry metadata that aware readers need to locate the chunk index, configure dm-verity, or determine a layer's composition role.
They are placed on the **layer's [descriptor][oci-descriptor]** in the [image manifest][oci-manifest].
A layer MAY also carry these annotations on the manifest itself for tooling that does not inspect descriptor annotations, but the descriptor annotations are authoritative.

| Annotation | Type | Required when | Description |
|---|---|---|---|
| `org.erofs.chunk-index.range` | `range` | Chunk index embedded in blob | Absolute byte offset and end (exclusive) of the chunk index in the layer blob, denoting the half-open interval `[offset, end)`. For `+zstd` layers: `offset` points to the first byte of the enclosing zstd skippable frame (its 4-byte magic number); the 32-byte chunk-index header begins eight bytes later. For raw layers: `offset` points to the first byte of the chunk-index header directly. `end` MAY be omitted; consumers then take it as the end of the blob (the chunk index, when embedded, is always the trailing section). |
| `org.erofs.chunk-index.digest` | `digest` | RECOMMENDED whenever chunk index is present | Digest of the chunk-index payload bytes (the 32-byte header and all chunk entries; **not** including any enclosing frame header bytes). Used for integrity verification of the index before reading chunk entries. |
| `org.erofs.chunk-index.mediaType` | `mediatype` | Chunk index present (optional; defaults to `application/vnd.erofs.chunk-index.v1`) | Media type identifying the chunk-index format. When absent, consumers MUST assume the default value `application/vnd.erofs.chunk-index.v1`. |
| `org.erofs.chunk-index.target` | `digest` | RECOMMENDED on standalone chunk-index layers | Digest of the layer that this standalone chunk-index applies to. When present, consumers MUST verify it matches the descriptor digest of the immediately preceding `manifest.layers[]` entry; a mismatch indicates a malformed manifest. When absent, the chunk-index applies to the immediately preceding layer per §3.4.3. This annotation makes the composition self-describing and enables integrity verification independent of layer position. |
| `org.erofs.uncompressed-digest` | `digest` | RECOMMENDED on compressed layers (`application/vnd.erofs+zstd`); OPTIONAL on raw layers | Digest of the layer's uncompressed image data — the physical bytes obtained by fully decompressing the layer blob. For `application/vnd.erofs+zstd` this is the SHA-256 of the decompressed data stream. For raw `application/vnd.erofs` this equals the descriptor `digest` and the annotation may be omitted. This value is identical to the layer's DiffID as defined in [§5.2](#52-rootfsdiff_ids-optional). When `rootfs.diff_ids` is also present, a consumer MAY ignore the `rootfs.diff_ids` entry in favour of the annotation. When `rootfs.diff_ids` is absent, this annotation is the sole source of the per-layer uncompressed digest and SHOULD be present on every compressed layer. |
| `org.erofs.dmverity.hash_offset` | `uint` | dm-verity merkle tree present | **Uncompressed** byte offset where the dm-verity merkle tree begins. This is equivalently the size of the EROFS filesystem image. The merkle tree covers data range `[0, hash_offset)` of the uncompressed image data. The tree byte length is calculable from `hash_offset`, `block_size`, and the hash algorithm; no separate size annotation is required. For the raw variant this also equals the layer-blob offset of the tree's first byte. The value MUST be a multiple of `block_size`. |
| `org.erofs.dmverity.root_digest` | `digest` | dm-verity merkle tree present | Root digest of the dm-verity merkle tree. |
| `org.erofs.dmverity.block_size` | `uint` | dm-verity merkle tree present (optional; defaults to `4096`) | dm-verity data and hash block size, in bytes. When omitted, consumers MUST assume `4096`. `hash_offset` MUST be a multiple of this value. |
| `org.erofs.role` | `role` | OPTIONAL on any layer descriptor | Marks the layer's composition role. See [§2.4](#24-roles). When absent, the layer is treated as an EROFS filesystem image forming part of the overlay stack or, in the single-layer case, the complete image root filesystem. |

All annotation values are strings as required by the [OCI Annotations][oci-annotations] specification; numeric values are the decimal text representation of the integer.

The grammar of typed values:

```ebnf
range     = offset [ ":" end ]
offset    = 1*DIGIT
end       = 1*DIGIT
uint      = 1*DIGIT
mediatype = type "/" subtype [ "+" suffix ]
type      = 1*token-char
subtype   = 1*token-char
suffix    = 1*token-char
role      = "device" / "overlay-lower" / "overlay-data"
DIGIT     = "0" / "1" / "2" / "3" / "4" / "5" / "6" / "7" / "8" / "9"
```

The grammar of `digest` is defined by the [OCI Content Descriptors specification][oci-descriptor].
The grammar of `mediatype` follows [RFC 6838][rfc6838]; `token-char` matches the production for `restricted-name-chars` defined there.

Implementations MUST treat any unknown `org.erofs.*` annotation as informational and MUST NOT fail manifest parsing on its presence; see [§10.1](#101-extensibility).

### 2.4 Roles

The `org.erofs.role` annotation on a layer descriptor determines how that layer participates in image composition.
Three values are defined.

#### `device`

A layer carrying `org.erofs.role: device` is a raw byte-source for EROFS multi-device addressing.
The `device` role MAY be applied to **any** media type that yields a usable byte stream once the runtime processes any `+suffix` framing: `application/vnd.erofs[+zstd]`, `application/vnd.oci.image.layer.v1.tar`, `…tar+gzip`, `…tar+zstd`, `application/octet-stream`, and custom media types are all valid carriers.
The runtime decompresses the blob (per the carrier media type's `+suffix` convention) to obtain a byte stream, places that stream at a predictable per-snapshot path, and passes the path to the consuming non-device EROFS layer's mount via the `device=` mount option.
That non-device EROFS layer's EROFS metadata references blocks in the device stream by `(device_id, block_addr)` tuples via EROFS multi-device addressing.

The `device` role does not constrain the media type or internal byte layout of the annotated layer; any content that produces a usable byte stream is acceptable.
Examples:

- A tar+zstd layer whose byte offsets are referenced by a higher-index EROFS metadata layer — matching the way `mkfs.erofs --tar=i` already produces metadata images that reference an original tar as a data device.
- A large opaque blob (e.g. an ML model file, a pre-built VM disk image) distributed as `application/octet-stream` and referenced by an EROFS metadata layer that presents per-file views into the blob.
- An `application/vnd.erofs[+zstd]` image used purely as a block source, with the consuming layer ignoring its filesystem structure and referencing it by absolute block address.

A `device`-role layer MUST be followed in `manifest.layers[]` by a non-device EROFS layer that consumes it via multi-device addressing.
A `device`-role layer MUST NOT be the last entry in `manifest.layers[]`.

#### `overlay-lower`

A layer carrying `org.erofs.role: overlay-lower` is an EROFS filesystem image used as an overlayfs `lowerdir`.
It contributes overlay-changeset semantics: whiteouts and opaque-directory markers within the image are honored by the runtime when composing the overlay stack (see [§3.6](#36-whiteouts)).
MUST imply `application/vnd.erofs[+zstd]` media type; other media types MUST NOT carry `overlay-lower`.

At runtime, each `overlay-lower` layer is mounted as a read-only EROFS filesystem (with any referenced data devices also prepared) and contributed as an overlayfs `lowerdir` in `manifest.layers[]` order (index 0 lowest, increasing index toward the viewer).

#### `overlay-data`

A layer carrying `org.erofs.role: overlay-data` is an EROFS filesystem image whose file payloads are referenced by a higher-index EROFS metadata layer via the overlayfs metacopy/redirect mechanism.
MUST imply `application/vnd.erofs[+zstd]` media type; other media types MUST NOT carry `overlay-data`.

The `overlay-data` layer contains file data but not necessarily meaningful directory metadata for the consuming image.
The higher-index EROFS metadata layer's inodes carry `trusted.overlay.redirect` extended attributes pointing at paths within the `overlay-data` image; when the overlay stack is mounted, the kernel reads file data from the `overlay-data` layer at the redirected paths.
The runtime supplies the `overlay-data` layer to the overlayfs mount as a data-only lower (for example via the `lowerdir=<meta>::<data>` syntax on kernels that support the overlayfs data-only-lower feature).
The exact mount-option construction, path-resolution algorithm, and fs-verity wiring for per-file content addressing are implementation-defined.

An `overlay-data`-role layer MUST be followed in `manifest.layers[]` by a non-device, non-`overlay-data` EROFS metadata layer that references its file payloads via overlayfs metacopy/redirect.
An `overlay-data`-role layer MUST NOT be the last entry in `manifest.layers[]`.

## 3. Layer Format

### 3.1 Blob Structure

A layer blob has the following top-level structure, in order:

```
+------------------------------------------------+  offset 0
|                                                |
|  Image data                                    |
|  ─ EROFS filesystem image                      |
|  ─ optional: dm-verity merkle tree appended    |
|              directly after the EROFS image    |
|                                                |
|  Encoding:                                     |
|  ─ raw variant: raw bytes                      |
|  ─ +zstd variant: one or more zstd frames      |
|    decompressing to the image data above       |
|                                                |
+------------------------------------------------+
|  Chunk Index                            (opt.) |
|  ─ raw variant: raw chunk-index bytes          |
|  ─ +zstd variant: zstd skippable frame         |
|    (uncompressed payload)                      |
+------------------------------------------------+  end of blob
```

The **image data** is the kernel-mountable content.
When dm-verity is used, the merkle tree is appended directly after the EROFS filesystem image; it covers the data range `[0, hash_offset)` of the image data.
For the raw variant the image data spans from offset 0 to the start of the chunk index (or the end of the blob when no chunk index is present); for the `+zstd` variant the image data is the byte sequence obtained by decompressing the zstd frames in order (a standard zstd decoder naturally skips any trailing skippable frame).

The **chunk index** is an optional trailing section described in [§3.4](#34-chunk-index).
It is embedded directly in this blob (via the `org.erofs.chunk-index.range` annotation) or distributed as a separate standalone layer (see [§3.4.3](#343-delivery-embedded-or-standalone-layer)).

The layer blob is **content-addressable as a whole**: the layer descriptor's `digest` is computed over all bytes from offset 0 to the end of the blob, including any embedded chunk-index section.

### 3.2 Raw EROFS Filesystem Image

The image data of a raw `application/vnd.erofs` layer starts at offset 0 and consists of an EROFS filesystem image as defined by the [EROFS on-disk format documentation][erofs-format].
A dm-verity merkle tree MAY be appended directly after the EROFS image (see [§3.5](#35-dm-verity-merkle-tree)); when present it covers `[0, hash_offset)` of the blob.

EROFS itself supports per-file compression of the underlying file data (LZ4, LZMA, DEFLATE, zstd) inside the filesystem image; whether that internal compression is used is a property of the EROFS image and is independent of the outer media-type variant.

A raw layer MAY carry an embedded chunk index as trailing raw bytes (no zstd skippable-frame wrapper), located by the `org.erofs.chunk-index.range` annotation.
The chunk-index header's `CompressionType` field (see [§3.4.1](#341-header)) MUST be `0` (none) for embedded chunk indexes on raw layers.

### 3.3 Zstd-Compressed Filesystem Image

For `application/vnd.erofs+zstd` the image data is encoded as one or more [zstd frames][rfc8878].
The decompressed bytes are the image data: when dm-verity is present, the producer concatenates the EROFS filesystem image and the merkle tree into a single uncompressed byte stream and then encodes that combined stream as zstd frames.
A standard zstd decoder reading the entire blob from offset 0 produces the full image data and silently skips any trailing chunk-index skippable frame.

If the layer does not carry a chunk index, the producer MAY use any framing for the image data, including a single zstd frame for the whole stream.
Such a layer supports only sequential decompression.

If the layer carries a chunk index with `CompressionType=1` (zstd), the image data MUST be encoded so that each chunk occupies exactly one zstd frame; chunk boundaries are determined by the chunk index.
This one-frame-per-chunk constraint is what makes the chunk index's `BlockOffset` values resolvable to standalone zstd frames at random-access read time.

### 3.4 Chunk Index

The chunk index is a small binary index over a layer's image data.
It maps chunks to their on-blob locations and optionally carries per-chunk checksums that serve as content addresses.
The format is identified by the media type `application/vnd.erofs.chunk-index.v1`; this specification defines exactly this one chunk-index format.

The chunk index is OPTIONAL.
A chunk index is REQUIRED to support **lazy loading** — random-access reads of arbitrary byte ranges — and for chunk-level addressing in a chunk-addressed block store.
Layers without a chunk index support only sequential reads, which MUST be integrity-verified end-to-end by comparing the running digest of the decompressed stream against the layer's uncompressed digest (from the `org.erofs.uncompressed-digest` annotation or `rootfs.diff_ids` per [§5.2](#52-rootfsdiff_ids-optional)).

#### 3.4.1 Header

The chunk index begins with a 32-byte header.
All multi-byte fields are stored in little-endian byte order.

| Field             | Offset | Size (bytes) | Description |
|---|---:|---:|---|
| `Magic`           |  0     |  4  | `0xCD 0xE4 0xEC 0x67` |
| `Version`         |  4     |  1  | Format version. MUST be `1`. |
| `CompressionType` |  5     |  1  | Compression applied to each chunk's on-blob bytes. `0` = none (raw byte ranges); `1` = zstd (each chunk is exactly one zstd frame). Other values are reserved. MUST be consistent with the carrying layer's media type: raw `application/vnd.erofs` layers MUST set `0`; `application/vnd.erofs+zstd` layers MUST set `1`. Standalone `application/vnd.erofs.chunk-index.v1` layers MUST set the value explicitly. |
| `Flags`           |  6     |  2  | Reserved bitfield (uint16). All bits MUST be `0` in version `1`. Future revisions MAY assign bits to signal new per-entry or per-index fields. |
| `UncompressedSize`|  8     |  8  | Total uncompressed size of the indexed image data in bytes. When the layer carries a dm-verity merkle tree the tree is part of the image data and is covered by `UncompressedSize`; the EROFS-image / merkle-tree boundary is recorded in `org.erofs.dmverity.hash_offset`. |
| `NumChunks`       | 16     |  4  | Number of chunk entries that follow the header. |
| `HashAlgo`        | 20     |  1  | Hash family for per-chunk checksums. `0` = none (checksums omitted); `1` = SHA-2 family. Other values are reserved. |
| `HashSize`        | 21     |  1  | Length in bytes of each per-chunk checksum. For SHA-2: `32` = SHA-256, `64` = SHA-512. MUST be `0` when `HashAlgo` is `0`. |
| `Reserved`        | 22     | 10  | MUST be all zero in version `1`. |

#### 3.4.2 Chunk Entry

The header is immediately followed by `NumChunks` chunk entries.
All entries have the same shape regardless of image size or chunk size.

| Field                | Size (bytes) | Description |
|---|---:|---|
| `BlockOffset`        |  8  | On-blob byte offset of the chunk's data. When `CompressionType = 0` (none): the first byte of the chunk's raw bytes in the blob; the on-blob length equals the chunk's logical (uncompressed) length. When `CompressionType = 1` (zstd): the first byte of the chunk's zstd frame; the on-blob length is `entry[i+1].BlockOffset - entry[i].BlockOffset` for all but the last entry, and `chunkIndexStart - entry[N-1].BlockOffset` for the last, where `chunkIndexStart` is the absolute offset of the chunk-index section from `org.erofs.chunk-index.range`. |
| `UncompressedOffset` |  8  | Byte offset of the chunk's data within the logical (uncompressed) image data stream. The uncompressed length of chunk `i` is `entry[i+1].UncompressedOffset - entry[i].UncompressedOffset` for all but the last entry, and `UncompressedSize - entry[N-1].UncompressedOffset` for the last. |
| `Checksum`           |  `HashSize` | Present only when `HashAlgo` is non-zero (see [§3.4.5](#345-per-chunk-checksums)). Computed over the chunk's on-blob bytes as stored: the raw bytes for `CompressionType=0`; the compressed zstd-frame bytes for `CompressionType=1`. |

Entry size: `16 + HashSize` bytes.
For no checksum: 16 bytes per entry.
For SHA-256: 48 bytes per entry.
For SHA-512: 80 bytes per entry.

Implementations SHOULD verify per-chunk checksums when reading.
A checksum mismatch MUST cause the layer to be treated as corrupt.

#### 3.4.3 Delivery: Embedded or Standalone Layer

A chunk index may be delivered in either of two ways.

**Embedded in the layer blob:**
The chunk index is appended as the trailing section of the layer blob it indexes.
Its location is described by the `org.erofs.chunk-index.range` annotation on the layer's descriptor.

- For `application/vnd.erofs+zstd` layers: the chunk index is wrapped in a [zstd skippable frame][rfc8878] (magic in range `0x184D2A50`–`0x184D2A5F`).
  The frame's payload is the uncompressed chunk-index bytes (32-byte header + entries); the index is not itself zstd-compressed.
  The `org.erofs.chunk-index.range` value's `offset` is the first byte of the enclosing skippable frame; the 32-byte header begins eight bytes later (after the 8-byte frame header).
  A standard zstd decoder reading the whole blob produces just the image data and silently passes over the index frame.
- For `application/vnd.erofs` (raw) layers: the chunk index is appended as raw bytes with no frame wrapper.
  The `org.erofs.chunk-index.range` value's `offset` is the first byte of the 32-byte chunk-index header.

The chunk-index skippable frame for `+zstd` layers SHOULD begin at a 4-byte-aligned offset.
Producers MAY interpose a zero-padding zstd skippable frame between the image data and the chunk-index frame; padding does not affect filesystem-image decoding.

**Standalone layer:**
The chunk index is distributed as a separate entry in `manifest.layers[]` with `mediaType: application/vnd.erofs.chunk-index.v1`.
The blob bytes are the raw chunk-index payload (32-byte header and all chunk entries); there is no enclosing frame.
A standalone chunk-index layer applies to the immediately preceding `layers[]` entry (`layers[i-1]`).

Producers SHOULD emit the `org.erofs.chunk-index.target` annotation on standalone chunk-index layer descriptors, carrying the descriptor digest of the indexed layer.
Consumers that recognize the annotation MUST verify the named digest matches the immediately preceding `manifest.layers[]` entry; a mismatch indicates a malformed manifest.
When the annotation is absent, consumers MUST treat the chunk index as applying to the immediately preceding layer.

A standalone chunk-index layer participates in `rootfs.diff_ids` as a normal layer: its DiffID is the SHA-256 digest of its blob bytes (which are already uncompressed).
Consumers that do not implement standalone chunk-index layers MUST silently skip them during composition; the rest of the image remains composable without lazy-loading benefit.

An `application/vnd.erofs[+zstd]` layer descriptor that carries `org.erofs.chunk-index.range` has an embedded chunk index; it SHOULD NOT also be immediately followed by a standalone chunk-index layer for the same content.

#### 3.4.4 Reading a Range

Given a request for uncompressed bytes `[off, off+len)` from a layer with a chunk index, a reader proceeds as follows:

1. Parse the 32-byte header to determine `UncompressedSize`, `NumChunks`, `HashAlgo`, `HashSize`, and `CompressionType`. Reject the index if any reserved field is non-zero or if `CompressionType` is unrecognized.
2. Compute entry size: `16 + HashSize` bytes (or `16` when `HashAlgo = 0`).
3. Verify that the payload length accommodates exactly `NumChunks` entries:
   ```
   expectedPayload = 32 + NumChunks * entrySize
   ```
   If the actual payload length (derived from `org.erofs.chunk-index.range` or blob size for standalone) does not equal `expectedPayload`, the index MUST be treated as corrupt.
4. Identify the chunk indices covering `[off, off+len)` by binary-searching or scanning `UncompressedOffset` to find entries whose logical range intersects the request.
5. For each required chunk `i`:
   1. Read `entry[i].BlockOffset` to find the on-blob start.
   2. Compute the on-blob length per [§3.4.2](#342-chunk-entry).
   3. Fetch the bytes from `[BlockOffset, BlockOffset + length)` in the blob.
   4. If `HashAlgo` is non-zero, verify `Checksum` against the fetched bytes.
   5. If `CompressionType = 1` (zstd), decompress the fetched zstd frame. If `CompressionType = 0` (none), use the fetched bytes directly.
6. Assemble the requested `[off, off+len)` from the per-chunk data.

Readers SHOULD process chunk fetches concurrently when multiple chunks are required.
The chunk index is small enough (typically under 100 KiB even for multi-gigabyte layers) to cache in memory after the first read.

#### 3.4.5 Per-chunk Checksums

Per-chunk checksums are RECOMMENDED whenever a chunk index is present.
A producer indicates per-chunk checksums by setting `HashAlgo` to a non-zero family identifier; each chunk entry then carries a `Checksum` of the indicated `HashSize` bytes.
A producer that does not wish to record checksums sets `HashAlgo = 0` and `HashSize = 0`; chunk entries omit the `Checksum` field entirely.

Per-chunk checksums serve two purposes that dm-verity does not subsume:

- **Lazy-loading integrity at chunk granularity.** A consumer fetching a single chunk for a small read can verify it in isolation, without setting up a dm-verity device or replaying the merkle tree.
- **Content addressing for deduplication in chunk-addressed block stores.** The checksum is computed over the chunk's exact on-blob bytes, so a block store can index, deduplicate, and serve chunks under the same hash that the chunk index carries — without re-hashing on ingest.

The chunk-index digest carried in `org.erofs.chunk-index.digest` transitively content-addresses every byte of the image data when per-chunk checksums are present: the digest covers the chunk-index payload bytes; the chunk-index entries cover each chunk via `Checksum`; decompressing or reading all chunks reconstructs the full image data.
Producers SHOULD set `HashAlgo > 0` and emit `org.erofs.chunk-index.digest` whenever a chunk index is present.

#### 3.4.6 Edge Cases

- **Empty layer** (`UncompressedSize = 0`): the header is valid with `NumChunks = 0`. No chunk entries follow. A reader MUST treat this as a zero-length uncompressed stream.
- **Single-chunk layer** (`NumChunks = 1`): `entry[0].UncompressedOffset` MUST be `0`. For `CompressionType=1` the on-blob length is `chunkIndexStart - entry[0].BlockOffset`; for `CompressionType=0` the length equals `UncompressedSize`.
- **Truncated or malformed index**: if `expectedPayload` does not match the actual payload length, or if any reserved header byte is non-zero, the layer MUST be treated as corrupt.
- **Unknown `CompressionType`**: a consumer that encounters a `CompressionType` value other than `0` or `1` MUST treat the chunk index as unreadable and fall back to sequential reads.
- **Unknown `Flags` bits**: a consumer that encounters non-zero `Flags` bits it does not recognize MUST treat the chunk index as unreadable, as those bits signal per-entry fields the consumer does not know how to skip.

### 3.5 dm-verity Merkle Tree

A layer MAY include a [dm-verity][dmverity] merkle tree as part of its image data.
When present, the merkle tree is appended directly after the EROFS filesystem image — exactly the layout the kernel's `dm-verity` target expects, where the hash tree is a contiguous tail to the data it covers.

The merkle tree covers data range `[0, hash_offset)` of the uncompressed image data.
Its byte length is fully determined by `hash_offset`, `block_size`, and the hash algorithm, so no separate size annotation is required.
The dm-verity superblock and salt are stored within the merkle-tree bytes themselves; runtimes that use the standard kernel `dm-verity` target auto-detect them by reading from `hash_offset`.

dm-verity is independent of the chunk index: a layer MAY carry a merkle tree with or without a chunk index.

- For `application/vnd.erofs` (raw): the merkle tree occupies blob bytes `[hash_offset, hash_offset + verity_size)`. Any chunk index follows after. For this variant `hash_offset` also equals the layer-blob offset of the tree's first byte.
- For `application/vnd.erofs+zstd`: the merkle tree is part of the uncompressed image data stream. Decompressing all zstd frames produces `[EROFS image bytes][merkle-tree bytes]` as a contiguous stream, with the merkle tree beginning at `hash_offset` of the decompressed stream. A chunk MAY span the boundary between the EROFS image and the merkle tree.

`hash_offset` MUST be a multiple of the dm-verity `block_size`.
EROFS images are naturally block-aligned and satisfy this constraint without producer effort.

The dm-verity metadata needed by the kernel — the tree location, data block size, and root digest — is carried in descriptor annotations:

- `org.erofs.dmverity.hash_offset` — the uncompressed byte offset where the merkle tree begins.
- `org.erofs.dmverity.root_digest` — root digest of the merkle tree.
- `org.erofs.dmverity.block_size` — dm-verity block size; defaults to `4096` when omitted.

A runtime that mounts the layer with dm-verity MUST supply the data range `[0, hash_offset)` of the uncompressed image data, the block size, and the root digest to the kernel `dm-verity` target.

### 3.6 Whiteouts

EROFS layers express filesystem changes using the same [overlayfs][overlayfs] whiteout conventions the Linux kernel uses to stack filesystems at mount time.

Whiteout conventions:

- A **whiteout** is a character device file with major `0` and minor `0` (rdev `0`). When stacked above a lower layer containing a file or directory at the same path, the whiteout hides the lower entry.
- An **opaque directory** is a directory whose extended attribute `trusted.overlay.opaque` is set to `"y"`. When stacked above a lower layer containing a directory at the same path, the opaque directory hides all children of the lower directory.

**Whiteouts are only meaningful when there is something below to hide.**
A layer at index `i` in `manifest.layers[]` MAY contain whiteouts if and only if at least one layer at index `j < i` carries the `overlay-lower` role.
A layer for which no such predecessor exists MUST NOT contain whiteouts; whiteouts in such a layer are meaningless.

More specifically:

- The bottommost `overlay-lower` layer (index 0 of all `overlay-lower` layers, looking from the bottom) MUST NOT contain whiteouts — there is nothing in the overlay stack below it for them to hide.
- A role-less layer at the top of `manifest.layers[]` MAY contain whiteouts if at least one preceding layer carries `overlay-lower`. That role-less top layer is contributed as the highest-priority `lowerdir` in the overlay assembly, and its whiteouts hide entries from lower siblings.
- A `device`-role or `overlay-data`-role layer MAY physically contain whiteout bytes in its EROFS image; consumers MUST ignore them (those layers never contribute to an overlay stack as `lowerdir`s).

These conventions differ from the `.wh.` and `.wh..wh..opq` filename conventions used by the [OCI tar layer format][oci-layer].
A layer producer that converts a tar layer into an EROFS layer MUST translate tar whiteouts into the overlayfs conventions above (see [§9](#9-conversion-from-tar-layers-informative)), and the EROFS layer MUST NOT contain files whose names begin with `.wh.`.

Producers SHOULD strip whiteouts from layers where they are known to be meaningless.
Consumers MUST NOT fail on the presence of meaningless whiteouts.

### 3.7 Hardlinks

EROFS supports hardlinks natively as part of the on-disk filesystem image.
Hardlinks MUST NOT cross layer boundaries: a hardlink in an upper layer to a lower-layer inode is not representable.
A producer that detects such a cross-layer hardlink in source data MUST either materialize the link target in the upper layer or fail the conversion.

### 3.8 Layer Ordering

Layer ordering in `manifest.layers[]` and `rootfs.diff_ids` follows OCI conventions (index 0 is the base; index N-1 is the top).
Entries are classified by their `mediaType` and `org.erofs.role` annotation.
The following ordering rules apply in addition to OCI's existing ordering semantics:

1. **The last entry in `manifest.layers[]` MUST be a mountable EROFS layer**: `mediaType` is `application/vnd.erofs[+zstd]` AND role is either absent or `overlay-lower`. A layer carrying `device`, `overlay-data`, or media type `application/vnd.erofs.chunk-index.v1` MUST NOT be the last entry.

2. **`device`-role layers** are each consumed by the first subsequent non-device EROFS layer in `manifest.layers[]`.
   An `application/vnd.erofs[+zstd]` layer that itself carries role `device` is not a consumer; it is itself a device source.
   Multiple consecutive `device`-role layers preceding a single non-device EROFS layer are all attached to that layer as `device=` sources in `manifest.layers[]` order.
   Standalone chunk-index layers between a `device` layer and its consumer are passed over.
   When a producer needs the same device blob to back multiple non-device EROFS layers in the same image, it MAY include the `device`-role descriptor more than once in `manifest.layers[]` — once before each consumer. Each occurrence is a distinct attachment site. Multiple instances of the same digest SHOULD be treated as the same object during transfer and storage.

3. **`overlay-data`-role layers** follow the same adjacency rule as rule 2 for positional consumption: each `overlay-data` layer is consumed by the first subsequent non-device, non-`overlay-data` EROFS metadata layer in `manifest.layers[]`.
   Multiple consecutive `overlay-data`-role layers preceding a single metadata layer are all supplied to that one EROFS mount as data-only lowers.

4. **Standalone chunk-index layers** (`mediaType: application/vnd.erofs.chunk-index.v1`) MUST NOT be the first entry in `manifest.layers[]` and MUST NOT be the last entry. They reference the immediately preceding entry.

5. **Whiteout ordering** (see [§3.6](#36-whiteouts)): a layer MUST NOT contain whiteouts unless at least one preceding layer carries `overlay-lower`.

6. **ChainID** is computed from per-layer DiffIDs per [§5.2](#52-rootfsdiff_ids-optional) and [§5.3](#53-chainid-and-imageid), including chunk-index layers and device layers.

## 4. Image Manifest

[Image manifests][oci-manifest] are unchanged in structure.
A manifest carrying EROFS layers uses the standard `application/vnd.oci.image.manifest.v1+json` media type, references its `config` blob normally, and lists layer descriptors in `layers` in stack order (index 0 is the base layer).

The only EROFS-specific obligations are at the descriptor level:

- The `mediaType` of each layer descriptor MUST be one of the values defined in [§2.1](#21-applicationvnderofs-and-zstd) or [§2.2](#22-applicationvnderofschunk-indexv1), or any media type valid for a `device`-role layer.
- Each layer descriptor MUST carry the annotations defined in [§2.3](#23-annotations) that are required for its configuration (chunk index, dm-verity, role).

A manifest SHOULD use a single layer media type family across its `layers` array; mixing EROFS and tar layer media types within one manifest is not defined by this specification.

### 4.1 Image Layout Interactions

EROFS layer blobs follow the existing [OCI image-layout][oci-image-layout] convention: blobs are stored under `blobs/<algorithm>/<encoded>` keyed by their descriptor digest, and `index.json` references the manifest as it does for tar-based images.
No additional on-disk layout is defined by this specification.

## 5. Image Configuration

This specification extends the [image configuration][oci-config] to make the `rootfs` and `rootfs.diff_ids` fields optional for EROFS layers, and adds a required value to `os.features`.
Per-layer uncompressed digests are carried on layer descriptors via the `org.erofs.uncompressed-digest` annotation (see [§2.3](#23-annotations)); `rootfs.diff_ids` serves as a legacy fallback when the annotation is absent.
The structural shape of the image configuration is otherwise unchanged.

### 5.1 `rootfs.type`

When a `rootfs` object is present in the image configuration, `rootfs.type` MUST be `"layers"`.
The `rootfs` object MAY be omitted entirely when all compressed layers carry the `org.erofs.uncompressed-digest` annotation.
The presence of EROFS layers is signalled by the layer media types in the manifest and by the `erofs` value in `os.features`, not by the `rootfs.type` value.

### 5.2 `rootfs.diff_ids` (optional)

#### DiffID definition

The DiffID of an EROFS layer is the digest of the layer's uncompressed content, computed as follows:

- **`application/vnd.erofs`** (raw): the DiffID is the SHA-256 digest of the blob bytes from offset 0 to the end of the blob (including any embedded chunk index and any dm-verity merkle tree). This equals the layer descriptor's `digest`.
- **`application/vnd.erofs+zstd`**: the DiffID is the SHA-256 digest of the decompressed image data — the byte sequence obtained by decompressing the zstd frames in order. The trailing skippable frame wrapping any embedded chunk index is automatically skipped by a conformant zstd decoder and is not included in the DiffID. The dm-verity merkle tree, being part of the decompressed image data stream, is included.
- **`application/vnd.erofs.chunk-index.v1`** (standalone): the DiffID is the SHA-256 digest of the blob bytes (the chunk-index payload, which is already uncompressed).
- **Device-role layers of other media types** (e.g. `application/vnd.oci.image.layer.v1.tar+zstd`): DiffID follows the rules of that media type's specification.

These rules apply uniformly regardless of the layer's `org.erofs.role`.

#### Delivery: `org.erofs.uncompressed-digest` or `rootfs.diff_ids`

The uncompressed digest of each layer is communicated via the `org.erofs.uncompressed-digest` annotation on the layer descriptor (preferred) or via the `rootfs.diff_ids` array in the image configuration (legacy fallback), or both.

When the `org.erofs.uncompressed-digest` annotation is present on a layer descriptor, consumers MUST use it as the layer's DiffID.
`rootfs.diff_ids` MAY be ignored when the annotation is present.

When the annotation is absent, consumers MUST fall back to `rootfs.diff_ids`.
In that case `rootfs.diff_ids` MUST be present and its length MUST equal the length of `manifest.layers[]`.

For raw `application/vnd.erofs` layers the DiffID equals the descriptor `digest`; the annotation is redundant on raw layers but MAY be present.

Producers SHOULD emit the `org.erofs.uncompressed-digest` annotation on every `application/vnd.erofs+zstd` layer descriptor.
Producers SHOULD also emit `rootfs.diff_ids` for compatibility with consumers that do not yet recognize the annotation.
Producers MAY omit `rootfs.diff_ids` (and the `rootfs` object entirely) when all compressed layers carry the annotation.

### 5.3 ChainID and ImageID

The [`ChainID`][oci-chainid] recursion is unchanged: `ChainID(L₀) = DiffID(L₀)` and `ChainID(L₀|...|Lₙ) = Digest(ChainID(L₀|...|Lₙ₋₁) + " " + DiffID(Lₙ))`, with `DiffID` interpreted per [§5.2](#52-rootfsdiff_ids-optional).

Every layer participates in the ChainID recursion regardless of role or media type, including standalone chunk-index layers and device-role layers.
Each layer's DiffID is sourced from the `org.erofs.uncompressed-digest` annotation when present, falling back to `rootfs.diff_ids` per [§5.2](#52-rootfsdiff_ids-optional).

The [`ImageID`][oci-imageid] is the SHA-256 digest of the image configuration JSON.

### 5.4 Platform OS Features

An image whose layers conform to this specification MUST declare `erofs` in `os.features`:

- In the [image index][oci-index] platform descriptor's `os.features` array, when the image is referenced from an image index.
- In the [image configuration][oci-config]'s top-level `os.features` array.

A host that does not implement this specification MUST NOT select or run a manifest whose `os.features` contains `erofs`.

An image index MAY contain both EROFS-bearing and tar-bearing manifests for the same software, distinguished by the presence or absence of `erofs` in their respective platform `os.features` arrays.

### 5.5 Image Identity (informative)

This section is non-normative.

The OCI ImageID — the SHA-256 digest of the image configuration JSON — was historically used as a cryptographically unique, node-local identifier for a container image.
Its uniqueness relied on `rootfs.diff_ids` encoding the full per-layer identity chain inside the configuration, making each distinct set of layers produce a distinct configuration digest.

When `rootfs.diff_ids` is omitted and all other configuration fields remain constant across images (e.g. a common base configuration), the configuration digest is no longer guaranteed to uniquely identify the set of layers.

Consumers that need a cryptographically unique identifier for an image SHOULD use the **manifest digest** — the `digest` of the image manifest descriptor.
The manifest digest covers both the layer descriptors and the configuration descriptor; it is stable, universally available from the OCI distribution layer, and does not depend on the content of the image configuration.

Runtime systems that have historically keyed container state on the ImageID (config digest) should prefer the manifest digest as the primary image identity and treat the ImageID as a legacy secondary key.

## 6. Image Index

[Image indexes][oci-index] are unchanged in structure.
The platform descriptor of any index entry pointing to an EROFS-bearing manifest MUST include `erofs` in its `os.features` array.

When an image index contains both tar-bearing and EROFS-bearing manifests for the same platform, the tar-bearing manifest SHOULD be listed first in `manifests` to preserve backward compatibility with clients that select the first matching manifest without inspecting `os.features`.

## 7. Applying EROFS Layers

Given a manifest whose `layers[]` consists of EROFS-bearing layer descriptors, the runtime root filesystem is produced by the following procedure.
Per-layer DiffIDs are sourced from the `org.erofs.uncompressed-digest` annotation on each layer descriptor when present, falling back to `rootfs.diff_ids` in the image configuration (see [§5.2](#52-rootfsdiff_ids-optional)).

**Step 1 — Resolve and classify.**
Resolve each layer descriptor to a blob.
Classify each entry:

| Classification | Criteria |
|---|---|
| EROFS overlay lower | `application/vnd.erofs[+zstd]`; role `overlay-lower` or absent |
| EROFS overlay data | `application/vnd.erofs[+zstd]`; role `overlay-data` |
| Device source | any media type; role `device` |
| Chunk-index hint | `application/vnd.erofs.chunk-index.v1` |

**Step 2 — Prepare device sources.**
For each `device` layer, decompress the blob per the carrier media type's `+suffix` convention and place the resulting byte stream at a predictable per-snapshot path.

**Step 3 — Prepare overlay-data layers.**
For each `overlay-data` layer, mount it as a read-only EROFS filesystem.
It will be supplied to the final overlayfs mount as a data-only lower alongside the metadata layer it serves.

**Step 4 — Prepare non-device EROFS layers.**
For each non-device EROFS layer (role absent, `overlay-lower`, or `overlay-data`; in `manifest.layers[]` order, index 0 first), mount it as a read-only EROFS filesystem.
Each `device`-role layer is attached to the first subsequent non-device EROFS layer in `manifest.layers[]`.
The runtime passes each attached device blob's local path as a `device=` mount option (in the order the `device`-role layers appear in `manifest.layers[]`) when mounting the consuming non-device EROFS layer.
An `application/vnd.erofs[+zstd]` layer with role `device` is itself a device source, not a mount target; it does not become a `lowerdir`.
`overlay-lower` and role-less layers become overlayfs `lowerdir`s; `overlay-data` layers are supplied as data-only lowers to the metadata layer that consumes them.

**Step 5 — Consult chunk-index hints (optional).**
For each standalone `application/vnd.erofs.chunk-index.v1` layer, the runtime MAY use its chunk index for lazy loading or content addressing of the immediately preceding layer.
This step is optional; skipping it does not affect correctness, only lazy-loading performance.

**Step 6 — Assemble the root filesystem.**
All EROFS layers (both `overlay-lower` and role-less) become `lowerdir`s in the overlayfs assembly.
`manifest.layers[]` order determines `lowerdir` priority: the layer at the highest index in `manifest.layers[]` is the highest-priority `lowerdir`.
`overlay-data` layers are supplied as data-only lowers alongside the metadata layers they serve.

- If only a single EROFS layer exists and no `overlay-lower` predecessors are present: the runtime MAY mount it directly as the root filesystem (single EROFS mount) rather than through overlayfs, as an optimisation.
- Otherwise: the overlayfs mount is assembled from all `lowerdir`s in order, with the `overlay-data` layers supplied via the data-only lower mechanism. The resulting overlayfs mount is the root filesystem. The overlayfs `upperdir` is the runtime's writable scratch directory, not an image layer.

**Step 7 — Optional integrity.**
For any layer carrying `org.erofs.dmverity.hash_offset` and `org.erofs.dmverity.root_digest`, the runtime SHOULD wrap the layer's data range `[0, hash_offset)` in a dm-verity device using the annotated parameters.

### Constraints

- Every `application/vnd.erofs[+zstd]` layer MUST be a complete EROFS filesystem image independently mountable as a read-only filesystem (with any required data devices also present).
- The last layer in `manifest.layers[]` MUST be a mountable EROFS layer per [§3.8](#38-layer-ordering).
- Whiteout semantics apply only when an overlay stack is assembled; see [§3.6](#36-whiteouts).

## 8. Conformance Requirements

### 8.1 Producer Requirements

A producer of an EROFS layer MUST:

1. Emit `application/vnd.erofs[+zstd]` as the layer media type for EROFS filesystem images. Set `org.erofs.role` per [§2.4](#24-roles) when the layer's composition role is `device`, `overlay-lower`, or `overlay-data`.
2. Emit a chunk index per [§3.4](#34-chunk-index) when the layer is intended to support lazy loading or chunk-level addressing. Set `CompressionType` in the chunk-index header explicitly: `1` for `+zstd` layers, `0` for raw layers.
3. Emit per-chunk checksums (`HashAlgo > 0`, every entry carrying `Checksum`) when block-level integrity or chunk-level content addressing is desired per [§3.4.5](#345-per-chunk-checksums).
4. When `CompressionType = 1` and a chunk index is present, encode the `+zstd` image data so that each chunk occupies exactly one zstd frame; chunk boundaries are determined by the chunk index per [§3.3](#33-zstd-compressed-filesystem-image).
5. Emit `org.erofs.chunk-index.range` and `org.erofs.chunk-index.digest` on the layer descriptor when a chunk index is embedded in the layer blob. MAY emit `org.erofs.chunk-index.mediaType`; when absent consumers assume `application/vnd.erofs.chunk-index.v1`. SHOULD emit `org.erofs.chunk-index.target` on standalone chunk-index layer descriptors per [§3.4.3](#343-delivery-embedded-or-standalone-layer).
6. Emit `org.erofs.dmverity.hash_offset` and `org.erofs.dmverity.root_digest` when the layer carries a dm-verity merkle tree per [§3.5](#35-dm-verity-merkle-tree). MAY emit `org.erofs.dmverity.block_size`; when absent consumers default to `4096`.
7. Emit `org.erofs.uncompressed-digest` on every `application/vnd.erofs+zstd` layer descriptor per [§5.2](#52-rootfsdiff_ids-optional). SHOULD also emit `rootfs.diff_ids` for compatibility with consumers that do not yet recognize the annotation. When `rootfs.diff_ids` is omitted, MUST emit `org.erofs.uncompressed-digest` on every compressed layer descriptor.
8. Declare `erofs` in `os.features` per [§5.4](#54-platform-os-features).
9. Respect whiteout rules per [§3.6](#36-whiteouts): do not emit whiteouts in layers where no `overlay-lower` predecessor exists. MUST NOT emit `.wh.`-prefixed filenames in EROFS images.
10. Materialize or reject cross-layer hardlinks per [§3.7](#37-hardlinks).
11. Respect layer ordering rules per [§3.8](#38-layer-ordering): do not place `device`, `overlay-data`, or chunk-index layers as the last entry in `manifest.layers[]`.
12. When producing a multi-device EROFS layer (`erofs_super_block.extra_devices > 0`), SHOULD populate each `erofs_deviceslot.tag` field with the hex form of the referenced device blob's descriptor digest (or its first 64 bytes if the hex form is longer than 64 bytes). This is a metadata aid intended for debugging, validation, or future composition tooling; the tag is not used for device resolution in this revision of the specification.

A producer SHOULD generate filesystem images deterministically per [§8.3](#83-reproducibility).

### 8.2 Consumer Requirements

A consumer of an EROFS layer MUST:

1. Refuse to apply or execute a manifest whose `os.features` contains `erofs` if the consumer does not implement this specification.
2. Derive each layer's DiffID per [§5.2](#52-rootfsdiff_ids-optional): from the `org.erofs.uncompressed-digest` annotation when present, otherwise from `rootfs.diff_ids`. `rootfs.diff_ids` MAY be ignored when the annotation is present.
3. Classify and compose layers per [§7](#7-applying-erofs-layers). Refuse if layer ordering violates [§3.8](#38-layer-ordering).
4. When a chunk index is present (signalled by `org.erofs.chunk-index.range` or a standalone chunk-index layer), verify its integrity using `org.erofs.chunk-index.digest` (when present) before relying on its entries. When `org.erofs.chunk-index.target` is present on a standalone chunk-index layer, verify its value matches the descriptor digest of the immediately preceding `manifest.layers[]` entry; treat a mismatch as a malformed manifest.
5. Honor `org.erofs.chunk-index.mediaType` (defaulting to `application/vnd.erofs.chunk-index.v1` when absent); refuse to parse a chunk index whose media type is not recognized, falling back to sequential reads.
6. Refuse to parse a chunk index whose `CompressionType` is not a recognized value (`0` or `1`) or whose `Flags` contains non-zero bits the consumer does not recognize; fall back to sequential reads.
7. Verify per-chunk checksums (`HashAlgo > 0`) before treating decompressed bytes as authoritative; treat mismatch as layer corruption.
8. When dm-verity is present and integrity verification is requested, supply the data range `[0, hash_offset)`, `block_size`, and `root_digest` to the kernel `dm-verity` target.
9. Honor overlayfs whiteout conventions per [§3.6](#36-whiteouts) when composing an overlay stack; ignore whiteout bytes in `device`-role, `overlay-data`-role, and chunk-index layers.
10. Silently skip standalone `application/vnd.erofs.chunk-index.v1` layers during composition if not implemented.
11. Treat unknown `org.erofs.*` annotations as informational per [§10.1](#101-extensibility).

### 8.3 Reproducibility

EROFS producers SHOULD generate filesystem images deterministically to keep layer descriptor digests and DiffIDs stable across reproducible builds.
In particular, producers SHOULD:

- Set the EROFS UUID deterministically (e.g., derived from the source content digest).
- Set filesystem timestamps to a stable baseline.
- Order directory entries canonically.
- Emit the chunk index byte-for-byte deterministically: identical chunk boundaries, header flags, and per-chunk checksums for the same source content. This stability is required for the chunk-index digest (surfaced in `org.erofs.chunk-index.digest`) to match across rebuilds.
- Emit dm-verity metadata using a stable salt when reproducible root digests are desired.

## 9. Conversion from Tar Layers (informative)

This section is non-normative.
It describes how a tar-based layer can be converted to an equivalent EROFS layer that preserves the changeset semantics defined in the [OCI tar layer format][oci-layer].

A converter reads each tar entry and produces a corresponding EROFS inode:

1. **Regular files**: emit a regular-file inode with the same path, mode, ownership, mtime, and xattrs.
2. **Directories**: emit a directory inode with the same path, mode, ownership, mtime, and xattrs.
3. **Symbolic links**: emit a symlink inode whose target is the tar `linkname`.
4. **Hardlinks**: emit a hardlink to the previously-emitted inode whose path matches the tar `linkname`. Cross-layer hardlinks MUST be either materialized as a duplicate file or rejected per [§3.7](#37-hardlinks).
5. **Special files**: emit the corresponding device or special-file inode with the same major/minor or type, mode, ownership, mtime, and xattrs.
6. **Tar whiteout files** named `.wh.<name>`: emit an overlayfs whiteout (character device, major `0`, minor `0`, named `<name>`) per [§3.6](#36-whiteouts). The `.wh.<name>` filename MUST NOT appear in the EROFS image.
7. **Tar opaque markers** (`.wh..wh..opq` inside a directory): set `trusted.overlay.opaque=y` on the directory inode and omit the marker filename.
8. **PAX and other tar extension records**: translate per the [OCI tar layer format][oci-layer] rules.

A converter SHOULD generate the EROFS image deterministically per [§8.3](#83-reproducibility).

### 9.1 Content Hashing and Cross-Image Reuse (informative)

When a producer chooses chunk boundaries that align with file payload boundaries inside the EROFS image — and lays out the EROFS image deterministically — each chunk's per-chunk checksum is a stable content hash of one file or of a fixed range of a large file.
Two images that contain the same file produce the same chunk checksum at that file's position, and a chunk-addressed block store can serve both images from one stored copy of the chunk.

This specification does not prescribe a chunking strategy.
File-aligned boundaries give per-file deduplication; content-defined rolling-hash boundaries provide stronger deduplication at the cost of producer complexity.
All strategies are expressible through the existing chunk-index format.

The `device` role ([§2.4](#24-roles)) enables a complementary strategy: reusing existing blobs (tar archives, model files, pre-built images) as EROFS data devices without converting their bytes.
A higher-index EROFS metadata layer references them by block address via EROFS multi-device addressing, presenting a structured filesystem view without duplicating the source bytes.

## 10. Considerations

### 10.1 Extensibility

Implementations MUST treat any unknown `org.erofs.*` annotation as informational and MUST NOT fail manifest parsing on its presence.
Future revisions of this specification MAY register additional annotations under the `org.erofs.*` prefix.

A consumer that encounters a layer media type whose subtype is unknown MUST refuse to apply the layer.
A consumer that encounters `application/vnd.erofs[+zstd]` but does not implement this specification MUST refuse to apply the layer; it MUST NOT attempt to apply it as a tar archive.
A consumer that recognizes the layer media type but does not recognize the chunk-index media type in `org.erofs.chunk-index.mediaType` MUST treat the layer as if no chunk index were present (sequential reads only).

### 10.2 Tooling Compatibility

Tooling that reads OCI manifests but does not implement this specification SHOULD continue to operate over the manifest and descriptor without parsing layer payloads, treating the new media types as opaque content per the OCI spec's extensibility rules.

Manifests bearing EROFS layers carry `erofs` in `os.features`, giving tools that inspect only manifests an early signal to refuse the image when they cannot process EROFS layers.

### 10.3 Non-Distributable Layers

This specification does not define non-distributable variants of the EROFS layer media types.
The OCI Image Format Specification has [deprecated][oci-layer] non-distributable layers; this specification follows that direction.

## 11. Worked Examples

The following non-normative examples illustrate manifests and configurations for EROFS-bearing images.

### 11.1 Multi-layer Overlayfs Image

A three-layer image: the lower two layers carry `overlay-lower` and the top layer is a role-less EROFS layer (the highest-priority `lowerdir` in the overlay assembly).

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:c0ffee...",
    "size": 1234
  },
  "layers": [
    {
      "mediaType": "application/vnd.erofs+zstd",
      "digest": "sha256:aaa111...",
      "size": 20971520,
      "annotations": {
        "org.erofs.role":               "overlay-lower",
        "org.erofs.uncompressed-digest":      "sha256:aaa111decomp...",
        "org.erofs.chunk-index.range":  "20963328",
        "org.erofs.chunk-index.digest": "sha256:aaaidx..."
      }
    },
    {
      "mediaType": "application/vnd.erofs+zstd",
      "digest": "sha256:bbb222...",
      "size": 31457280,
      "annotations": {
        "org.erofs.role":               "overlay-lower",
        "org.erofs.uncompressed-digest":      "sha256:bbb222decomp...",
        "org.erofs.chunk-index.range":  "31449088",
        "org.erofs.chunk-index.digest": "sha256:bbbidx..."
      }
    },
    {
      "mediaType": "application/vnd.erofs+zstd",
      "digest": "sha256:ccc333...",
      "size": 52428800,
      "annotations": {
        "org.erofs.uncompressed-digest":           "sha256:ccc333decomp...",
        "org.erofs.chunk-index.range":       "52420608:52428800",
        "org.erofs.chunk-index.digest":      "sha256:cccidx...",
        "org.erofs.dmverity.hash_offset":    "260046848",
        "org.erofs.dmverity.root_digest":    "sha256:deadbeef..."
      }
    }
  ]
}
```

The image configuration:

```json
{
  "architecture": "amd64",
  "os": "linux",
  "os.features": ["erofs"],
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:aaa111decomp...",
      "sha256:bbb222decomp...",
      "sha256:ccc333decomp..."
    ]
  },
  "config": { "Entrypoint": ["/usr/bin/myapp"] }
}
```

At apply time: layers 0 and 1 (`overlay-lower`) are each mounted as read-only EROFS filesystems and supplied as overlayfs `lowerdir`s.
Layer 2 (no role) is mounted as a read-only EROFS filesystem and supplied as the highest-priority `lowerdir`.
All three are `lowerdir`s in the overlayfs assembly; the `upperdir` is the runtime's writable scratch directory.
Whiteouts in layers 1 and 2 hide entries from lower layers; layer 0 must not contain whiteouts.

### 11.2 Single-layer EROFS Image

A single-layer image with dm-verity and a chunk index for lazy loading.
No overlay stack; no role annotation needed.

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:1eaf...",
    "size": 879
  },
  "layers": [
    {
      "mediaType": "application/vnd.erofs+zstd",
      "digest": "sha256:b00b1e...",
      "size": 8388608,
      "annotations": {
        "org.erofs.uncompressed-digest":        "sha256:b00b1edecomp...",
        "org.erofs.chunk-index.range":    "8380416",
        "org.erofs.chunk-index.digest":   "sha256:b00bidx...",
        "org.erofs.dmverity.hash_offset": "41943040",
        "org.erofs.dmverity.root_digest": "sha256:b00bverity..."
      }
    }
  ]
}
```

At apply time: the single layer is mounted directly as the root filesystem.
The runtime configures dm-verity for the range `[0, 41943040)` of the decompressed image data.
The chunk index enables lazy loading; missing chunks are fetched on demand and each is hash-verified before delivery.

### 11.3 Composite Image: Device, Overlay-data, and Standalone Chunk-index

A four-layer image:

- Layer 0: a large opaque blob (`application/octet-stream`) annotated as a `device` — for example a multi-GiB ML model file.
- Layer 1: an `overlay-data` EROFS layer — a flat collection of file payloads referenced by the metadata layer.
- Layer 2: a standalone `application/vnd.erofs.chunk-index.v1` layer — the chunk index for layer 1, distributed separately to allow lazy loading of the data layer without embedding the index inside it.
- Layer 3: the top EROFS metadata layer — no role; references layer 0 via multi-device addressing; references layer 1 via overlayfs metacopy/redirect.

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:c0ffee...",
    "size": 1024
  },
  "layers": [
    {
      "mediaType": "application/octet-stream",
      "digest": "sha256:model0...",
      "size": 4294967296,
      "annotations": {
        "org.erofs.role": "device"
      }
    },
    {
      "mediaType": "application/vnd.erofs+zstd",
      "digest": "sha256:data1...",
      "size": 536870912,
      "annotations": {
        "org.erofs.role":          "overlay-data",
        "org.erofs.uncompressed-digest": "sha256:data1decomp..."
      }
    },
    {
      "mediaType": "application/vnd.erofs.chunk-index.v1",
      "digest": "sha256:idx2...",
      "size": 32768,
      "annotations": {
        "org.erofs.chunk-index.target": "sha256:data1..."
      }
    },
    {
      "mediaType": "application/vnd.erofs+zstd",
      "digest": "sha256:meta3...",
      "size": 524288,
      "annotations": {
        "org.erofs.uncompressed-digest":      "sha256:meta3decomp...",
        "org.erofs.chunk-index.range":  "516096",
        "org.erofs.chunk-index.digest": "sha256:meta3idx..."
      }
    }
  ]
}
```

The image configuration:

```json
{
  "architecture": "amd64",
  "os": "linux",
  "os.features": ["erofs"],
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:model0...",
      "sha256:data1decomp...",
      "sha256:idx2...",
      "sha256:meta3decomp..."
    ]
  },
  "config": { "Entrypoint": ["/opt/model/serve"] }
}
```

At apply time:

- Layer 0 (`device`): the runtime places the raw model bytes at a per-snapshot path and passes it as `device=<path>` to the layer 3 EROFS mount.
- Layer 1 (`overlay-data`): mounted as a read-only EROFS filesystem; supplied to the overlayfs mount as a data-only lower.
- Layer 2 (chunk-index): the runtime MAY use this chunk index to lazy-load layer 1 on demand.
- Layer 3 (no role, top): mounted as the root EROFS filesystem using `device=<model-path>` for multi-device addressing. The overlayfs mount uses layer 3 as the metadata layer and layer 1 as the data-only lower; inodes in layer 3 carrying `trusted.overlay.redirect` are resolved against layer 1's files.

### 11.4 Device Blob Consumed by an Overlay Stack

This example shows a `device`-role blob consumed by a single EROFS layer that participates in a multi-layer overlay stack.
The key question it answers: when `device`-role layers and `overlay-lower` layers are interleaved, which EROFS layer does each `device` blob attach to?

The rule: a `device`-role layer is attached to the first subsequent **non-device EROFS layer** in `manifest.layers[]` (see [§2.4](#24-roles)).
An EROFS layer that itself carries role `device` is not a consumer; it joins the device group and attaches alongside the other `device`-role layers to the next non-device EROFS layer.
Standalone chunk-index layers between a `device` layer and its consuming non-device EROFS layer do not interrupt the lookup.

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:c0ffee...",
    "size": 1024
  },
  "layers": [
    {
      "mediaType": "application/octet-stream",
      "digest": "sha256:model0...",
      "size": 4294967296,
      "annotations": { "org.erofs.role": "device" }
    },
    {
      "mediaType": "application/vnd.erofs+zstd",
      "digest": "sha256:base1...",
      "size": 20971520,
      "annotations": { "org.erofs.role": "overlay-lower" }
    },
    {
      "mediaType": "application/vnd.erofs+zstd",
      "digest": "sha256:middle2...",
      "size": 31457280,
      "annotations": { "org.erofs.role": "overlay-lower" }
    },
    {
      "mediaType": "application/vnd.erofs+zstd",
      "digest": "sha256:top3...",
      "size": 524288
    }
  ]
}
```

Layer 0 (`device`) is consumed by **layer 1** — the first subsequent non-device EROFS layer (role `overlay-lower`).
The runtime passes the model blob's path as `device=<model-path>` when mounting layer 1.
Layers 2 and 3 do not receive the model blob as a device; they have no `device` predecessor of their own.

At apply time:

- Layer 0: decompressed to a local path; attached to layer 1 via `device=`.
- Layer 1 (`overlay-lower`): mounted as a read-only EROFS filesystem with `device=<model-path>`; becomes the lowest `lowerdir`.
- Layer 2 (`overlay-lower`): mounted as a read-only EROFS filesystem; becomes the middle `lowerdir`.
- Layer 3 (no role, top): mounted as a read-only EROFS filesystem; becomes the highest-priority `lowerdir`. The overlayfs `upperdir` is the runtime's writable scratch directory.

If layer 3 also required the model blob, the producer would emit the `device`-role descriptor a second time, immediately before layer 3. Each occurrence is a distinct attachment site. Multiple instances of the same digest SHOULD be treated as the same object during transfer and storage.

**EROFS image as a device source.**
An `application/vnd.erofs[+zstd]` layer MAY itself carry role `device` to be used as a raw block source for a higher EROFS layer (the consuming layer addresses its blocks by absolute offset, ignoring the source's filesystem structure).
In that case the EROFS-device layer is **not** a consumer of any preceding `device`-role blobs; it is itself a device:

```json
"layers": [
  { "mediaType": "application/octet-stream", "digest": "sha256:raw0...",
    "annotations": { "org.erofs.role": "device" } },
  { "mediaType": "application/vnd.erofs+zstd", "digest": "sha256:erofsdev1...",
    "annotations": { "org.erofs.role": "device" } },
  { "mediaType": "application/vnd.erofs+zstd", "digest": "sha256:consumer2..." }
]
```

Both layer 0 and layer 1 are `device`-role layers.
Neither consumes the other.
Layer 2 (no role) is the first non-device EROFS layer; it receives **both** layer 0 and layer 1 as `device=` mount options.

## 12. Future Work

The items below are non-normative and describe directions the authors anticipate but do not specify here.

- Standardization of additional integrity primitives (for example, fs-verity per-file digests) within EROFS layers.
- Signed dm-verity root digests for confidential-container use cases.
- Sub-layer artifact descriptors for separately distributing the chunk index or dm-verity tree from the underlying filesystem image.
- A new `rootfs.type` value, distinct from `layers`, if a future revision changes the layer-stacking model in a way that makes the current value semantically inappropriate.
- Alternative chunk-index formats under a new `application/vnd.erofs.chunk-index.*` namespace — for example indexes that use rolling-hash content-defined chunks, dictionary-shared zstd chunks, or file-path-to-chunk-range mappings.
- Additional `CompressionType` values (e.g. `2` for LZ4-framed chunks) introduced in a future version `1` revision via an existing `Flags` bit, or under a future format version.
- Additional `Flags` bit definitions and accompanying per-entry fields (e.g. prefetch-priority weights) in future revisions.
- Definition of the `overlay-data` runtime mount mechanism for specific kernel overlayfs versions and data-only-lower syntax.

## Appendix A. Sizing and Layout Guidance (informative)

The chunk index occupies `32 + NumChunks × entrySize` bytes, where `entrySize = 16 + HashSize`.

The table below shows index sizes for a 500 MiB layer at 1 MiB chunks (NumChunks = 500):

| Configuration | Entry size | Index payload | Total (with 32-byte header) |
|---|---:|---:|---:|
| No checksums            |  16 B |  ~7.8 KiB  |  ~7.8 KiB  |
| SHA-256 checksums       |  48 B |  ~23.4 KiB |  ~23.5 KiB |
| SHA-512 checksums       |  80 B |  ~39.1 KiB |  ~39.1 KiB |

The chunk index does not grow with file count (filesystem metadata is in the EROFS image, not the chunk index).
Even on multi-gigabyte layers the index remains tens of KiB.

For `+zstd` layers with an embedded chunk index, the on-blob size of the chunk-index section is the payload size above plus 8 bytes for the enclosing skippable-frame header.

dm-verity adds no overhead to the chunk index itself; its metadata is carried in three small descriptor annotations.

## Appendix B. Test Vectors (TBD)

This appendix is reserved for canonical test vectors covering:

- Chunk-index header (32 bytes) and entries for `CompressionType=0` and `CompressionType=1`.
- DiffID computation for raw and `+zstd` variants per [§5.2](#52-rootfsdiff_ids-optional).
- Worked dm-verity wrap of a small EROFS image (data, salt, root digest).

Test vectors will be provided in a future revision; they are not normative.

[c99]:             https://www.open-std.org/jtc1/sc22/wg14/www/C99RationaleV5.10.pdf
[dmverity]:        https://docs.kernel.org/admin-guide/device-mapper/verity.html
[erofs]:           https://erofs.docs.kernel.org/
[erofs-format]:    https://erofs.docs.kernel.org/en/latest/design.html
[erofs-multidev]:  https://erofs.docs.kernel.org/en/latest/design.html#multi-device-support
[oci-annotations]: https://github.com/opencontainers/image-spec/blob/main/annotations.md
[oci-chainid]:     https://github.com/opencontainers/image-spec/blob/main/config.md#layer-chainid
[oci-config]:      https://github.com/opencontainers/image-spec/blob/main/config.md
[oci-descriptor]:  https://github.com/opencontainers/image-spec/blob/main/descriptor.md
[oci-ebnf]:        https://github.com/opencontainers/image-spec/blob/main/considerations.md#ebnf
[oci-image-layout]: https://github.com/opencontainers/image-spec/blob/main/image-layout.md
[oci-imageid]:     https://github.com/opencontainers/image-spec/blob/main/config.md#imageid
[oci-index]:       https://github.com/opencontainers/image-spec/blob/main/image-index.md
[oci-layer]:       https://github.com/opencontainers/image-spec/blob/main/layer.md
[oci-manifest]:    https://github.com/opencontainers/image-spec/blob/main/manifest.md
[oci-media]:       https://github.com/opencontainers/image-spec/blob/main/media-types.md
[oci-spec]:        https://github.com/opencontainers/image-spec/blob/main/spec.md
[overlayfs]:       https://docs.kernel.org/filesystems/overlayfs.html
[rfc2119]:         https://tools.ietf.org/html/rfc2119
[rfc6838]:         https://tools.ietf.org/html/rfc6838
[rfc8478]:         https://tools.ietf.org/html/rfc8478
[rfc8878]:         https://tools.ietf.org/html/rfc8878
