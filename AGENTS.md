# Repository instructions

## Compatibility contract

This generator is designed to be fully compatible with
[cloudnative-pg/postgres-containers](https://github.com/cloudnative-pg/postgres-containers).
Treat this as a design requirement for code, dependency, workflow, and documentation
changes:

- Preserve the same SBOM and provenance formats and user-facing structure.
- Generate them with the same tools as far as possible, including Syft's SPDX
  conventions and BuildKit's standard attestations and provenance.
- Keep user commands for inspection, extraction, verification, and reporting
  identical wherever possible. Image references, signing identities, source
  repositories, and inventory contents naturally differ by publisher and image.
- Preserve the separation of responsibilities: the scanner scopes package
  evidence to the shipped payload; consumer workflows generate provenance and
  signatures. Compatibility includes those workflows as well as the scanner.

When changing scanner dependencies, SPDX composition, attestation packaging, or
release workflows, compare affected documents and commands with the reference
project. Check Docker Buildx SBOM extraction and provenance inspection, Cosign
verification, and Grype and Grant reporting as relevant to the change. For
consumers publishing separately signed SLSA provenance, also compare its
generation and verification workflow. Record necessary differences and their
compatibility impact.

Existing integration tests cover BuildKit/SPDX/Grype/Grant behavior; they do not
prove complete parity with every image and publishing workflow in the reference
project. Distinguish the compatibility goal from verified behavior when reporting
results. Documentation-only changes need local documentation checks; CI still builds
and tests every change and publishes successful pushes to `main`.

Keep these instructions aligned with the [README compatibility goal](README.md#compatibility)
and the [architecture compatibility contract](ARCHITECTURE.md#compatibility-contract).
