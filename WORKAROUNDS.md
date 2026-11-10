# Dependency workarounds

This register records dependency workarounds and adaptations needed to preserve
the [compatibility goal](README.md#compatibility) with
[cloudnative-pg/postgres-containers](https://github.com/cloudnative-pg/postgres-containers):
the same SBOM and provenance formats, the same tools wherever possible, and the
same user commands wherever possible.

On dependency updates, check the linked issue, pull request, or release contract,
verify the replacement, and remove temporary measures and their entries together
when they are no longer needed. Name the responsible dependency explicitly and
record any compatibility impact. Some entries describe packaging adaptations
rather than pending bug fixes. Where no issue is linked, say so instead of
implying that a fix is scheduled. Dependency status below was last reviewed on
2026-11-10.

## Renovate: Debian security package lookups

- **Renovate tracking:** [issue #44330](https://github.com/renovatebot/renovate/issues/44330)
  and [PR #35865](https://github.com/renovatebot/renovate/pull/35865), both open;
  the PR is not merged.
- **Review result (2026-11-10): keep.** Renovate's latest release,
  [44.225.1](https://github.com/renovatebot/renovate/releases/tag/44.225.1), still
  [selects gzip unconditionally](https://github.com/renovatebot/renovate/blob/44.225.1/lib/modules/datasource/deb/packages.ts).
  Live requests to Bookworm security's `main/binary-amd64` and `main/binary-arm64`
  indexes returned HTTP 404 for `Packages.gz` and HTTP 200 for `Packages.xz`.
  Debian package-page version selectors still matched all 30 package/architecture
  pairs in our dependency lists. This checks the website assumptions; it is not
  an end-to-end hosted Renovate run.
- **Problem:** the native `deb` datasource requests `Packages.gz`, but Bookworm
  security indexes are published as `Packages.xz`. Renovate skips the failed
  component and can report no updates even when a newer security package exists.
- **Workaround:** the two SBOM generator Debian rules in [renovate.json](renovate.json)
  use `custom.debian-bookworm` to read the full version from Debian's package
  details page for the requested architecture. The `plain` format lets JSONata
  inspect the HTML version heading, or the architecture's version table cell
  when Debian lists different binary rebuilds across architectures.
  This works with hosted Renovate and keeps Debian version ordering, existing
  weekly grouping and the existing merge policy.
  Package details include security updates and version epochs; download filenames
  omit epochs and must not be used as the version source. Architecture-independent
  packages also resolve through the architecture-specific details URLs.
- **Limits:** this depends on Debian's website layout and refresh cadence. A
  missing or changed version heading or ambiguous version cell fails the lookup
  instead of producing a guessed version. Package availability and dependency
  compatibility still need validation by the generator build.
- **Remove when:** the hosted Renovate release supports the security indexes.
  Restore the `deb` registry URLs for `bookworm`, `bookworm-updates`, and
  `bookworm-security` on each architecture, remove the datasource overrides and
  custom datasource, and verify a security-only update is proposed on both
  architectures with the correct epochs.

## Syft: readability of hash-based license references

- **Syft tracking:** [issue #3527: Unable to classify complex licenses](https://github.com/anchore/syft/issues/3527) remains open.
  This [comment](https://github.com/anchore/syft/issues/3527#issuecomment-6295475239)
  discusses the usability of hash-based results.
  [PR #5448](https://github.com/anchore/syft/pull/5448), currently open, proposes
  more descriptive custom references for possible license matches below the
  configured coverage threshold.
- **Review result (2026-11-10): keep the guidance and tracking.** The latest
  [Syft release, v1.57.1](https://github.com/anchore/syft/releases/tag/v1.57.1),
  matches our Dockerfile pin; PR #5448 remains unmerged. Scanning the local
  generator filesystem with that version still produced hash-based custom
  license references (96 extracted license definitions in the scan).
  This entry describes a reporting limitation and notice-retrieval guidance,
  not a local Syft patch that can be removed.
- **Related Grant issues:** [#157](https://github.com/anchore/grant/issues/157)
  (custom `LicenseRef` support) is closed, but that does not resolve descriptive
  naming of hash-based references. [#217](https://github.com/anchore/grant/issues/217)
  (option to ignore `licenseConcluded`) remains open. Both are referenced by
  Syft PR #5448; neither provides a replacement for the guidance below.
- **Problem:** complex Debian copyright notices can appear in license reports
  as hash-based custom references. The license evidence is retained, but a hash
  alone gives users little help understanding it.
- **Current approach:** the SBOM generator passes through Syft's license
  references and extracted text while scoping package and file records to the
  shipped payload. Original notices are shipped in `/licenses`. Users can start with a checksum in Grant and retrieve the
  original notice. See [reading license results](README.md#reading-license-results).
- **Limits:** a hash identifies text; it does not explain its terms. PR #5448
  proposes `LicenseRef-Possible-<ID>-<hash>` names containing candidate license
  IDs and a shortened source-text hash. These remain custom references, not
  assertions of standard SPDX licenses. Complex notices may still need manual
  review, and scanner accuracy remains a Syft concern.
- **Revisit when:** a released Syft version includes the license-reference usability
  improvement. Upgrade the pin and check how candidate references and their
  extracted text appear in SPDX and Grant, then update the checksum-based
  reading guidance as appropriate.
