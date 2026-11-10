# Architecture and implementation

The final image is built `FROM scratch`: it retains extension files and libraries,
but no package database. The generator uses the builder's package evidence to
identify the files actually shipped, excluding packages with no matched files.
It runs as a BuildKit scanner; its tools are not added to the extension image.

See [How it works](README.md#how-it-works) for the complete architecture diagram.

Files are copied to different locations in the scratch image: for example,
`/usr/share/doc/gcc-12-base/copyright` becomes `/licenses/gcc-12-base/copyright`.
The final path therefore cannot be used directly to look up Debian ownership.
We match each shipped file to its original builder file, then retain the package
that owned that builder file.

Ownership requires matching SHA256 content. Exact paths and shared path suffixes
rank candidates; ambiguous matches stay unclaimed. The composer retains owning
packages and their relationships, and passes through Syft package identities,
license references, and extracted license text. It updates file paths, IDs, and
checksums, repairs relationships, and recalculates package verification codes for
the shipped subset. Original copyright notices remain intact. Hash-based license
references can be difficult to interpret; see the [Syft license-reference usability work](WORKAROUNDS.md#syft-readability-of-hash-based-license-references)
and [checksum-based notice retrieval](README.md#retrieve-an-original-notice).

Builder scans should mirror BuildKit's default Syft generator as closely as
possible. Review the [Syft call](generator.py) against the BuildKit scanner settings
linked there: scan base path, source alias, `image,file` cataloger defaults, and
added `sbom-cataloger`. Tests exercise installed packages, embedded SBOMs, and
exclusion of source-only Go dependencies.

## Compatibility contract

The reference project is
[cloudnative-pg/postgres-containers](https://github.com/cloudnative-pg/postgres-containers),
as described in the [compatibility goal](README.md#compatibility). Preserve the
same SBOM and provenance formats and user-facing structure, use the same
generation tools as far as possible, and keep inspection and verification
commands interchangeable apart from image and publisher parameters.

When changing scanner dependencies, SPDX composition, attestation packaging, or
release workflows, compare the resulting documents and commands with that
project. Check Docker Buildx SBOM extraction and provenance inspection, Cosign
verification, and Grype and Grant reporting. For consumers that publish separate
signed SLSA provenance, also compare its generation and verification workflow.
Record any necessary differences and their compatibility impact.

The existing integration tests cover BuildKit/SPDX/Grype/Grant behavior. They do
not establish complete parity with every image and publishing workflow in the
reference project; maintainers must assess that compatibility when relevant
behavior changes.

## Code map

| Area | Entry points |
| --- | --- |
| Syft lifecycle, final inventory, BuildKit statement | [generator.py](generator.py) |
| File ownership, package selection, metadata preservation, document identity | [compose.py](compose.py) |
| Optional hook and its context | [hooks.py](hooks.py) |
| Final SPDX conformance check | [spdx_validation.py](spdx_validation.py), [explicit SPDX Tools dependency](Dockerfile) |
| Scanner image and dependencies | [Dockerfile](Dockerfile) |
| Extension integration | [consumer integration](README.md#consumer-integration) |
| Generator checks, publication, and digest updates | [sbom-generator.yml](.github/workflows/sbom-generator.yml), [release.py](release.py), [renovate.json](renovate.json) |
| Unit checks and real BuildKit/SPDX/Grype/Grant fixture | [tests](tests), [integration](tests/integration) |

Consumers expose the `builder` stage with
`ARG BUILDKIT_SBOM_SCAN_STAGE=builder` and select the scanner image through
BuildKit's SBOM attestation option. The generator wraps its SPDX document in an
in-toto statement; BuildKit attaches it to the output image. Consumer image
signing and any separate SLSA provenance are handled by the consumer's workflows.

## Downstream augmentation

A builder may supply `/usr/local/share/cnpg-sbom/augment_spdx.py` with
`augment_spdx(document, context) -> dict`. It receives the scoped document as
`document` and the full, unprocessed Syft result as `context.builder_document`.
[HookContext](hooks.py) also exposes API version 1, extension name, platform, and
builder/final paths. After the hook returns, the generator calculates the final
document namespace and validates the complete SPDX, including hook additions.

The hook executes trusted Python in the generator environment. Builder-installed
dependencies are unavailable there; downstream builds can prepare JSON evidence
for the hook to read. Missing hooks leave the document unchanged; hook errors or
invalid final SPDX fail the build. SPDX validation checks conformance, not whether
added components actually belong to the payload.

PGRX is a downstream consumer of this interface: it supplies Cargo component and
license evidence. The core generator has no Cargo or PGRX processing dependency.
The [fixture hook](tests/integration/fixture/augment_spdx.py) demonstrates adding
a package that Grant can report.

## Development

Build the scanner and run its unit tests from the repository root:

```bash
docker buildx build --load --tag cnpg-sbom-generator:local .
docker run --rm --entrypoint python3 \
  --volume "$PWD:/workspace:ro" --workdir /workspace \
  cnpg-sbom-generator:local -m unittest discover -s tests
```

CI uses one AMD64 runner and QEMU to build Linux AMD64 and ARM64 images.
Unit and integration tests run only on AMD64. Integration checks exercise
real BuildKit attestations, SPDX validation, package ownership, custom hooks,
Grype, and Grant.

### Validate an extracted SBOM

After building the local scanner image:

```bash
# Extract an SPDX document for the validator check below.
IMAGE=ghcr.io/cnpg-extensions/pgsphere@sha256:54e6f0b0a9d9e4c80db8c6edcac98b451159c1fbaa91e429a92d4978f593518d
docker buildx imagetools inspect "$IMAGE" \
  --format '{{ json (index .SBOM "linux/amd64").SPDX }}' \
  > pgsphere.spdx.json
docker run --rm --entrypoint python3 --volume "$PWD:/workspace:ro" \
  cnpg-sbom-generator:local /opt/cnpg-sbom-generator/spdx_validation.py \
  /workspace/pgsphere.spdx.json
```

Delete `pgsphere.spdx.json` after the check, or supply another extracted
SBOM at that path. See the [user guide](README.md#verify-the-image-and-extract-its-sbom)
for publisher verification.

The validator accepts extracted SPDX or the generator's in-toto statement and
exits nonzero on invalid output. The AMD64 CI checks exercise a real BuildKit
attestation, Debian notice ownership, hook
augmentation, Grype distribution detection, and Grant license reporting.

See the [README](README.md#inspect-an-image) for image inspection commands.

## Generator release and test builds

`ghcr.io/cnpg-extensions/cnpg-sbom-generator:latest` identifies the checked generator
published from `main`; `:test` identifies the most recent checked manual test
publication. These are moving tags. Consumers pin a tag **and its image digest**,
so moving a tag does not silently change an extension build. `sha-<full-commit>`
tags retain individual builds; there are no major/minor releases.

### Publication

Every successful push to `main` publishes to
`ghcr.io/<repository-owner>/cnpg-sbom-generator`. Pull requests and manual runs
without a publication option only build and test:

| Trigger | Result |
| --- | --- |
| Pull request | Checks only |
| Any push to `main`, including documentation and Renovate configuration | Publish or reuse commit image; promote the checked digest to `latest` |
| Manual dispatch, `publish=false`, `publish_latest=false` | Checks only |
| Manual dispatch, `publish=true`, `publish_latest=false` | Publish or reuse commit image; promote to `test` |
| Manual dispatch on `main`, `publish_latest=true` | Force a checked publication to `latest`, including runtime maintenance |

[The workflow](.github/workflows/sbom-generator.yml) has one `Build and test`
job. It builds AMD64 and ARM64 together using QEMU, then runs unit tests and
BuildKit/SPDX/Grype/Grant integration checks on the AMD64 image by digest.
There is no second build for publication.

For pull requests and checks-only runs, the image stays in the job's disposable
local registry. Publishing runs build or reuse a `sha-<full-commit>` image in
GHCR, verify both platform configurations and source labels, and test that exact
digest. The commit image can exist before tests pass; only a successful test run
proceeds to Cosign signing and promotion to `latest` or `test`.

Every build includes per-platform BuildKit `mode=max` provenance and a
default-scanner SPDX SBOM. Cosign signs the checked image index, binding the
platform images and their attestations to the workflow. This does not provide
`slsa-verifier` verification for the generator itself.

### Release promotion

Main and manual runs are serialized and are not cancelled by newer runs. Pull
requests have separate concurrency groups so they do not interrupt releases.
Failed checks leave aliases unchanged; manual test publication never moves
`latest`. [release.py](release.py) reuses commit tags and verifies alias digests.
A dependency rebuild requires a new commit.

Once its digest passes checks, a `main` push updates `latest` even if unrelated
commits have since advanced `main`. GitHub does not guarantee queue order, so
serialization alone does not enforce source-commit order for publications.

### Testing from a branch

From a checkout of this repository, push a generator change on a branch named `sbom-license-update`
and publish its test image:

```bash
git switch -c sbom-license-update
# Make and commit the generator change.
git push origin sbom-license-update

gh workflow run sbom-generator.yml \
  --ref sbom-license-update -f publish=true

gh run list \
  --workflow sbom-generator.yml --branch sbom-license-update --limit 1
```

Take the checked `test@sha256:…` reference from the successful run summary and pin
it in a temporary consumer branch. Run the affected extension builds before
accepting the update. Keep test pins out of stable consumers.

### Update consumers

When a checked publication advances `latest`, update the pinned digest in
consumer builds and run their tests. Consumers can configure Renovate to track
`latest@sha256:…` references and merge digest updates after their checks pass.

### Dependency maintenance

Renovate tracks the Python base-image digest, which covers its bundled Debian
packages and pip. Additional Debian packages are tracked in architecture-specific
`dependencies/debian-*.txt` files.
The Dockerfile pins Syft and the `spdx-tools` Python dependency used for SPDX
conformance validation. Renovate also tracks CI actions,
runners, and test tools.

All tracked dependencies share one **SBOM generator dependencies** PR, including
major updates. Renovate creates and updates the grouped branch only on Mondays
from **00:00 to 06:00 UTC** (`* 0-5 * * 1`, `updateNotScheduled: false`). Existing
branches wait for the next window for routine updates and rebases. Non-major
batches retain automerge after required checks pass; a batch containing a major
update requires manual merging. GitHub can complete automerge after the window
closes.

Each dependency batch builds the generator for AMD64 and ARM64 using QEMU and
tests it on AMD64 only. Require the `Build and test` check in branch protection.
Signing and promotion run only after tests pass within that job. ARM64 runtime
tests are not part of this workflow.
After merging, all dependency changes—including Python, Debian packages,
`spdx-tools`, and CI tools—trigger checked publication.
There are no runtime-only publication exclusions. The window batches updates;
it is not a hard limit of one CI run or publication per week. Updates within the
window, rebases, and the merge to `main` can trigger additional runs.

Keep the Python base on Bookworm to match the Debian dependency lists.
New Debian dependencies must also be added to both architecture lists.
The image revision label identifies the generator source commit. The SPDX
document retains Syft's original name, creators, and license metadata.

To roll back, restore the previous checked consumer digest and hold updates until
fixed. Retain the corresponding commit images.
