# The Briard app catalog

What a [Briard](https://github.com/briardhq/briard) node can install. Nodes read the published copy
at `https://get.briard.io/catalog/`, signed; this repository is where it is edited and tested.

```
catalog/<app>/<version>.json   one version of one app, kept for good
catalog/latest.json            the list: each app's current version, by the SHA-256 of its file
catalog-upstream.json          the upstream tag of an app's release, where it is not the bare X.Y.Z
catalog-images.json            the hash nix checks each pulled image against (tests only)
```

A version file is a Briard app manifest (briard's `shared/manifest`), its images pinned by digest.
Only the list is signed; a node checks each version file against the hash its line names.

**The bytes are the identity.** A node keeps a version file's exact bytes, and any change to them —
a reordered field, a trailing newline — is a new version of the app to every household. Never edit
a published version file; write new ones with `jq -cj` (no trailing newline).

## The list

One entry per line: `name`, `title`, `functionality` (two to five words), `description`, `icon` (an
SVG data URI, at most 16 KB), `homepage`; and for an installable app `available`, `version` and
`identity`. A node offers an update when the listed identity differs from the one it runs, and never
installs one unasked. Apps not yet installable are listed for households to ask for. Icons:
`THIRD-PARTY.md`.

## New versions

A nightly run finds each app's newest upstream release, tests it (its VM tests, the upgrade from the
previous version, an audit of its image, a fresh install on stock Ubuntu) and proposes it; it is
published once the owner merges it in [briardhq/signing](https://github.com/briardhq/signing).
A hand edit to an installable app's line is proposed the same way. Any other list change (a line
removed, or a not-yet-installable app's line edited) goes to briardhq/signing as a pull request by
hand, with the same edit committed here.

## Adding an app

A new entry needs its sizes and memory measured once:

```sh
skopeo copy docker://<image@digest> dir:/tmp/img
jq '[.layers[].size] | add' /tmp/img/manifest.json                       # size
for l in $(jq -r '.layers[].digest' /tmp/img/manifest.json); do zstd -dc "/tmp/img/${l#sha256:}"; done | wc -c   # installedSize
```

`minMemoryMB` is its settled anonymous memory after startup (the cgroup's `anon`). Anything the app
needs beyond what a manifest can express is product code in briard, keyed on the app's name.

## Checks

```sh
nix build github:briardhq/briard#catalog-test --override-input catalog path:.
```

parses every file the way a node does and holds each app to what briard's code for it assumes. CI
runs it on every push.
