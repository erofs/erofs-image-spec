# EROFS Image Layer Format Specification

> **Status: Draft.**
> Media-type strings, annotation keys, and the binary chunk-table layout are subject to change until the first stable release.
> Implementations are encouraged but should expect to track breaking changes during the draft phase.

This repository defines an alternative layer format for [OCI images][oci-spec], using [EROFS][erofs] filesystem images as the on-wire layer payload in place of tar archives.
The format preserves the OCI image manifest, image index, content descriptor, and image configuration shape; only the layer payload bytes, the layer media types, and the interpretation of `rootfs.diff_ids` change.

## Contributing

Issues and pull requests are welcome.
Please discuss substantial design changes in an issue before sending a pull request, so reviewers can be involved early.

Commits MUST carry a `Signed-off-by` line attesting to the [Developer Certificate of Origin][dco].
Use `git commit -s` to add it automatically.

Markdown files in this repository follow the OCI convention of **one sentence per line** to keep diffs small and avoid line-wrapping disputes.

## Licence

Everything in this repository is distributed under the [Apache License 2.0][apache-2] (see [`LICENSE`](LICENSE)).
The specification text — this README, and any future spec sub-documents — is additionally available under the [Creative Commons Attribution 4.0 International][cc-by-4] licence at the recipient's option, mirroring the practice of the [OCI Image Format Specification][oci-spec].
Contributors to this repository agree to license their contributions under both terms.

[apache-2]:        https://www.apache.org/licenses/LICENSE-2.0
[cc-by-4]:         https://creativecommons.org/licenses/by/4.0/
[dco]:             https://developercertificate.org/
[erofs]:           https://erofs.docs.kernel.org/
[oci-spec]:        https://github.com/opencontainers/image-spec/blob/main/spec.md
