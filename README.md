# EROFS Image Layer Format Specification

> **Status: Draft.**
> Media-type strings, annotation keys, and the binary chunk-index layout are subject to change until the first stable release.
> Implementations are encouraged but should expect to track breaking changes during the draft phase.

This repository defines an alternative layer format for [OCI images][oci-spec], using [EROFS][erofs] filesystem images as the on-wire layer payload in place of tar archives.
The format preserves the OCI image manifest, image index, and content descriptor shape; only the layer payload bytes and the layer media types change, and `rootfs.diff_ids` in the image configuration becomes optional.

**The normative specification is in [spec.md](spec.md).**

---

## Overview

An image layer in this format is an EROFS filesystem image — a kernel-native filesystem in a single blob — rather than a tar archive that must be unpacked.
That removes one of the largest contributors to cold container startup latency.
A small optional binary index over each layer's chunks adds two more capabilities: containers can start before all layer bytes are local (missing chunks are fetched on demand and hash-verified before delivery), and identical chunks across different images can deduplicate in a chunk-addressed block store.
A single `org.erofs.role` annotation tells the runtime how each layer fits into the image — overlayfs lower, overlayfs data source, raw data device, or the complete root filesystem.
OCI's manifest, image index, and descriptor shapes are unchanged; the image configuration shape is largely unchanged — `rootfs.diff_ids` becomes optional when the `org.erofs.uncompressed-digest` annotation is present on each layer descriptor.

### EROFS images

The on-disk EROFS format is the same one used elsewhere in Linux, unchanged — this specification only defines how an EROFS image is wrapped, transported, and identified within the OCI image model.
Layers are distributed either as raw EROFS images or zstd-compressed; the runtime reads files directly from the blob with no unpack step and no duplicate on-disk copy.
The layer media type is `application/vnd.erofs`, with the `+zstd` variant for zstd compression.

### Chunk-based distribution

A chunk index is a small binary index over a layer's image data.
It records where each chunk lives in the blob and carries a per-chunk checksum, which serves as the chunk's content address.
Two capabilities follow.
First, containers can begin running before all layer bytes are local: missing chunks are fetched on demand as they are read, and each chunk is hash-verified against its index entry before its bytes are delivered to the kernel.
Second, identical chunks across different images deduplicate naturally in a chunk-addressed block store — by checksum, without re-hashing on ingest.
The chunk index may be embedded as a trailing zstd skippable frame in its parent layer's blob (transparent to standard zstd decoders) or distributed as a standalone layer.
The format media type is `application/vnd.erofs.chunk-index.v1`.
Consumers that do not implement chunk indexes simply skip them; the rest of the image still works, just without these capabilities.

### Roles (`org.erofs.role`)

The `org.erofs.role` annotation on a layer descriptor determines how that layer participates in image composition.
When absent, the layer contributes to an overlay stack or forms the complete image root filesystem in the single-layer case.

| Role | Applies to | What the runtime does |
|---|---|---|
| `overlay-lower` | `application/vnd.erofs[+zstd]` | Mounts the layer read-only and contributes it as an overlayfs `lowerdir`. Whiteouts and opaque directories are honored. |
| `overlay-data` | `application/vnd.erofs[+zstd]` | Supplies the layer as an overlayfs data-only lower. File payloads are referenced by a higher metadata layer via overlayfs metacopy/redirect. |
| `device` | **Any media type** | Decompresses the blob and passes the raw byte stream as a `device=` option to a higher EROFS mount. The higher layer's EROFS metadata references blocks in this byte stream by address. |

EROFS supports multi-device mounts: a single EROFS filesystem image can reference blocks in additional files by absolute offset, with those files attached at mount time via the `device=` option.
The `device` role wires an OCI layer into that mechanism — making a tar archive, a raw `application/octet-stream` file, or any other opaque payload available as a raw byte source to a higher EROFS layer on mount.
The `overlay-lower` and `overlay-data` roles require an EROFS media type.

### Annotations

All EROFS-specific metadata is carried on layer descriptors as annotations.

| Annotation | Purpose |
|---|---|
| `org.erofs.uncompressed-digest` | Digest of the layer's uncompressed image data; allows consumers to derive ChainIDs without `rootfs.diff_ids` in the image configuration |
| `org.erofs.chunk-index.range` | Byte range of an embedded chunk index in the blob |
| `org.erofs.chunk-index.digest` | Digest of the chunk-index payload, for integrity verification |
| `org.erofs.chunk-index.mediaType` | Format identifier for the chunk index (defaults to `application/vnd.erofs.chunk-index.v1`) |
| `org.erofs.chunk-index.target` | Digest of the layer a standalone chunk-index applies to; consumers verify it matches the preceding layer |
| `org.erofs.dmverity.hash_offset` | Uncompressed offset where the dm-verity merkle tree begins; the tree covers `[0, hash_offset)` and its length is calculable from `hash_offset`, `block_size`, and the hash algorithm |
| `org.erofs.dmverity.root_digest` | Root digest of the dm-verity merkle tree |
| `org.erofs.dmverity.block_size` | dm-verity block size in bytes (default `4096`) |
| `org.erofs.role` | Composition role: `overlay-lower`, `overlay-data`, or `device` |

### containerd 2.3 compatibility

containerd 2.3 emits and recognizes the media types `application/vnd.erofs.layer.v1` and `application/vnd.erofs.layer.v1+zstd`.
New producers should emit `application/vnd.erofs[+zstd]` as defined here.
containerd 2.4 and later continue to accept the 2.3 media types as equivalent to the canonical ones, ensuring that images built against containerd 2.3 remain consumable.

---

## Quick example

The smallest interesting example is a single layer descriptor showing the canonical media type, a chunk index for lazy loading, and dm-verity for integrity:

```json
{
  "mediaType": "application/vnd.erofs+zstd",
  "digest": "sha256:b1ade1...",
  "size": 52428800,
  "annotations": {
    "org.erofs.uncompressed-digest":        "sha256:b1ade1decomp...",
    "org.erofs.chunk-index.range":    "52420608:52428800",
    "org.erofs.chunk-index.digest":   "sha256:facade...",
    "org.erofs.dmverity.hash_offset": "260046848",
    "org.erofs.dmverity.root_digest": "sha256:deadbeef..."
  }
}
```

No `org.erofs.role` annotation, so this is the complete image root filesystem.
The `org.erofs.uncompressed-digest` annotation carries the SHA-256 of the layer's decompressed image data, enabling consumers to derive ChainIDs without parsing `rootfs.diff_ids` from the image configuration.
Add `"org.erofs.role": "overlay-lower"` to make it a lower layer in an overlay stack.

---

## Use cases

### 1. Layered container images (OCI tar drop-in)

Replace tar-based layers with EROFS layers in a standard OCI image.

Lower layers carry `org.erofs.role: overlay-lower`; the top layer may omit the role annotation.
All layers — including the top — become overlayfs `lowerdir`s in the runtime's overlay assembly, with the top being the highest-priority lower.
The runtime's writable scratch directory is the overlayfs `upperdir`; no image layer is ever writable.

```json
"layers": [
  { "mediaType": "application/vnd.erofs+zstd",
    "digest": "sha256:base...",
    "annotations": { "org.erofs.role": "overlay-lower" } },
  { "mediaType": "application/vnd.erofs+zstd",
    "digest": "sha256:middle...",
    "annotations": { "org.erofs.role": "overlay-lower" } },
  { "mediaType": "application/vnd.erofs+zstd",
    "digest": "sha256:top..." }
]
```

The top layer may contain whiteouts that hide entries from lower layers.
The bottom layer should not contain whiteouts — there is nothing below it for them to hide.

Replacing tar layers with EROFS layers gives direct benefits: the kernel mounts EROFS images without unpacking, the optional chunk index enables lazy loading, and each fetched chunk is hash-verified before delivery.

### 2. Single-layer EROFS image

A single EROFS layer carrying the complete image root filesystem.
No overlay stack needed; the runtime mounts it directly (or places it as the sole `lowerdir` in a thin overlayfs for a writable container).

This is useful for smaller images that benefit from skipping overlay assembly altogether, and for appliance or firmware-style images where the producer assembles the full filesystem ahead of time.

```json
"layers": [
  {
    "mediaType": "application/vnd.erofs+zstd",
    "digest": "sha256:rootfs...",
    "annotations": {
      "org.erofs.chunk-index.range":    "...",
      "org.erofs.chunk-index.digest":   "sha256:...",
      "org.erofs.dmverity.hash_offset": "...",
      "org.erofs.dmverity.root_digest": "sha256:..."
    }
  }
]
```

### 3. Large ML model files as devices

Large model files (multi-GiB) are rarely unique to a single image, yet copying them into a tar layer means every image that uses the same model carries its own redundant copy.

EROFS supports multi-device mounts: its filesystem metadata can reference blocks in additional files by absolute byte offset, with those files attached at mount time.
With the `device` role, the model file is distributed as its own OCI layer in a native format (e.g. `application/octet-stream`) and attached to a higher EROFS layer on mount.
A small EROFS metadata layer uses this multi-device addressing to present per-file views into the model blob at conventional paths.

```json
"layers": [
  {
    "mediaType": "application/octet-stream",
    "digest": "sha256:model0...",
    "size": 4294967296,
    "annotations": { "org.erofs.role": "device" }
  },
  {
    "mediaType": "application/vnd.erofs+zstd",
    "digest": "sha256:meta1...",
    "size": 524288
  }
]
```

The runtime passes the model blob's path to the EROFS metadata mount via `device=`.
The metadata layer's EROFS filesystem exposes paths like `/models/llama3/model.gguf` that reference byte ranges inside the model blob.

Multiple images that use the same model reference the same blob digest; the registry stores it once.
Lazy loading is typically not the value here — runtimes generally want to maximize disk-IO throughput into VRAM rather than optimize pull time.
The value is that the model blob is a single OCI-addressed object shared by all images that reference it.

### 4. Reusing existing artifacts as devices (build cache, tar archives)

Existing blobs in a registry — tar layers, build-cache exports, pre-built disk images — can be referenced as EROFS data devices without converting or re-encoding their bytes.

Apply `org.erofs.role: device` to an existing layer in a new manifest.
A higher EROFS layer references its blocks via multi-device addressing, presenting a structured filesystem view without duplicating the source bytes.

```json
"layers": [
  {
    "mediaType": "application/vnd.oci.image.layer.v1.tar+zstd",
    "digest": "sha256:buildcache...",
    "size": 67108864,
    "annotations": { "org.erofs.role": "device" }
  },
  {
    "mediaType": "application/vnd.erofs+zstd",
    "digest": "sha256:meta...",
    "size": 262144,
    "annotations": {
      "org.erofs.chunk-index.range":  "253952",
      "org.erofs.chunk-index.digest": "sha256:metaidx..."
    }
  }
]
```

This is how `mkfs.erofs --tar=i` already works: it produces a metadata-only EROFS image that references an original tar as its data device.
This spec formalizes that relationship as first-class OCI layers, making it available to any OCI-compatible registry and runtime.

### 5. File-level deduplication

For workloads that share many individual files across images — Python package caches, build environments, dataset collections — the `overlay-data` role enables file-level deduplication without per-image duplication of file bytes.

An `overlay-data` layer is a flat EROFS image containing only file payloads, keyed by content hash.
A metadata EROFS layer describes the directory tree: each file inode carries a `trusted.overlay.redirect` extended attribute pointing at the corresponding file in the data layer.
When the overlay stack is mounted, the kernel resolves each file's data from the data layer at the redirected path.

```json
"layers": [
  {
    "mediaType": "application/vnd.erofs+zstd",
    "digest": "sha256:data0...",
    "size": 536870912,
    "annotations": { "org.erofs.role": "overlay-data" }
  },
  {
    "mediaType": "application/vnd.erofs.chunk-index.v1",
    "digest": "sha256:dataidx...",
    "size": 32768,
    "annotations": {
      "org.erofs.chunk-index.target": "sha256:data0..."
    }
  },
  {
    "mediaType": "application/vnd.erofs+zstd",
    "digest": "sha256:meta2...",
    "size": 524288
  }
]
```

Layer 0 is the data blob; layer 1 is a standalone chunk index for lazy loading of layer 0; layer 2 is the metadata layer (no role; the top).

With content-hashed file chunking in the data blob — chunk boundaries aligned to file boundaries, per-chunk checksums in the chunk index — identical files across images share a single copy in a chunk-addressed block store.
A second image that uses the same Python packages produces the same per-file chunk checksums and hits the block store's deduplication without any additional effort.

---

## What this specification does not change

- The image **manifest**, **index**, and **descriptor** schemas.
- The image **configuration** structural shape — `rootfs.type` remains `"layers"` when present; `rootfs.diff_ids` becomes optional rather than required.
- Existing **tar layer** semantics — tar-bearing manifests are unaffected, and an image index may carry both EROFS and tar variants of the same software.

---

## Relationship to the OCI Image Format Specification

This specification **extends** the [OCI Image Format Specification][oci-spec]; it does not replace any part of it.
The discriminator between an EROFS-bearing manifest and a tar-bearing manifest is the layer media type.

The platform descriptor's `os.features` array carries the value `erofs` so non-implementing hosts can skip EROFS-bearing manifests in an image index without parsing layer bytes.
A non-implementing host picks the tar-bearing manifest (listed first by convention); an EROFS-capable host picks the EROFS-bearing manifest.

## Relationship to EROFS

This specification does **not** define a new on-disk filesystem format.
The layer payload is an EROFS image as defined by the [Linux kernel EROFS documentation][erofs-format]; this specification only defines how an EROFS image is wrapped, transported, and identified within the OCI image model.
Producers may use any conformant EROFS toolchain (`mkfs.erofs`, Go-language EROFS writers, etc.) to generate the underlying image.

## Known consumers

containerd's in-tree EROFS snapshotter, EROFS differ, and `ctr image convert --erofs` emit and consume the `application/vnd.erofs[+zstd]` media types specified here; this repository formalizes the on-wire format that those components produce and consume.

The format is independent of any single runtime.
Other OCI-conformant runtimes may adopt the format on their own timeline; the `os.features=erofs` selection mechanism allows EROFS-bearing images to coexist with tar-bearing images in the same image index.

## Repository layout

| Path | Purpose |
|---|---|
| [`spec.md`](spec.md) | The normative specification. |
| `README.md` | This file. |
| `LICENSE` | Apache License 2.0 covering everything in this repository. |

A future revision may split `spec.md` into per-topic documents in the style of `opencontainers/image-spec` (e.g. `media-types.md`, `layer.md`, `config.md`); it is initially authored as a single document for ease of review.

## Contributing

Issues and pull requests are welcome.
Please discuss substantial design changes in an issue before sending a pull request, so reviewers can be involved early.

Commits must carry a `Signed-off-by` line attesting to the [Developer Certificate of Origin][dco].
Use `git commit -s` to add it automatically.

Markdown files in this repository follow the OCI convention of **one sentence per line** to keep diffs small and avoid line-wrapping disputes.

## Licence

Everything in this repository is distributed under the [Apache License 2.0][apache-2] (see [`LICENSE`](LICENSE)).
The specification text — this README, [`spec.md`](spec.md), and any future spec sub-documents — is additionally available under the [Creative Commons Attribution 4.0 International][cc-by-4] licence at the recipient's option, mirroring the practice of the [OCI Image Format Specification][oci-spec].
Contributors to this repository agree to license their contributions under both terms.

[apache-2]:        https://www.apache.org/licenses/LICENSE-2.0
[cc-by-4]:         https://creativecommons.org/licenses/by/4.0/
[containerd]:      https://github.com/containerd/containerd
[dco]:             https://developercertificate.org/
[dmverity]:        https://docs.kernel.org/admin-guide/device-mapper/verity.html
[erofs]:           https://erofs.docs.kernel.org/
[erofs-format]:    https://erofs.docs.kernel.org/en/latest/design.html
[oci-spec]:        https://github.com/opencontainers/image-spec/blob/main/spec.md
[overlayfs]:       https://docs.kernel.org/filesystems/overlayfs.html
