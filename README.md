# The Briard app catalog

What a [Briard](https://github.com/briardhq/briard) node can install: every app, every version of
it ever published, and the signed lists that say which version is current. This repository is
where the catalog is edited, tested and recorded. Nodes do not read it: they fetch the published
copy from `https://get.briard.io/catalog/`, and trust a list because of its signature, not because
of where it was served from.

```
catalog/<app>/<version>.json   one version of one app; every version ever published, never deleted
catalog/latest.json            every app's newest version: the list a node reads
catalog-images.json            the hash nix checks each pulled image against (tests only)
```

## Version files and lists

A **version file** is one app manifest — the schema is briard's `shared/manifest` — that pins
its images by digest. A **list** names, for each app, the version a node installs and the
`sha256` of that version file's exact bytes. **Only lists are signed**: `latest.json` is published
beside a detached Ed25519 signature over its exact bytes (`latest.json.sig`), made with the release
key. A node fetches the list from its `CATALOG_URL`, verifies the signature *before* parsing, then
fetches `<app>/<version>.json` from beside the list and refuses it unless its hash is the one the
line names. So one signature covers everything down to the image bytes, and an old version file
never needs signing again.

Every version stays published: a node that runs an older version reads its file, and an upgrade
is tested from the previous version — the next lower file — to the new one. Re-issuing a version
replaces its file; git keeps the old bytes. The images stay with their publishers, named by
digest: integrity is exact, availability is theirs.

## ⚠️ The bytes are the identity

A node keeps a version file's raw bytes on its replicated volume, and the app's identity is
`sha256` of exactly those bytes. **Any** change to a version file — a reordered field, an added
trailing newline, a reformat — mints a new app identity. Nodes read that as a new version of the
app and will take the whole upgrade path for it: snapshot, switch, health-gate, and a rollback if
it fails. There is no such thing as a cosmetic edit in this directory.

That is also why these files have **no trailing newline**: it is what the published bytes have,
and matching them is what keeps a version here the same app as the one already installed in every
household. Write them with `jq -cj`.

## The list: what a household can see

`latest.json` is also the catalog's listing — every app the dashboard and the website show, in the
order they show it: the ones a node can install today and the ones a household may ask for. Unlike
a version file its bytes are not an identity: an edit here changes what people read, never what a
node runs.

One entry per line, so a diff reads per app. Each carries:

| Field | What it is |
|---|---|
| `name` | the catalog slug — the key; an installable app's version files are under `catalog/<name>/` |
| `available` | `true` when the line names a version a node can install; omitted otherwise |
| `version` | for an available entry: the version a node installs now, which is also its file's name |
| `identity` | for an available entry: `sha256:` of that version file's exact bytes — the app a node installs now, and what a node running another identity offers as an update; omitted otherwise |
| `title` | the app's name for a person |
| `functionality` | two to five brand-less words on what it does, shown beside the title ("document management system") |
| `description` | a few lines on what it does for a household; no promise, no version |
| `icon` | the app's mark as an SVG data URI, at most 16 KB decoded, with no script, handler, external reference or foreign content |
| `homepage` | the upstream project's site |

**The list says what is installable.** An entry marked `available` names a version file here by
its version and identity; an entry without the mark is listed for a household to ask for. The
checks (below) hold the two together: an app with version files is listed, an entry is marked
exactly when it has them, and an available line's version and identity are its file's. The node
reads this one document (daily, on its update watcher's tick), caches it node-locally, and pushes
it to its dashboard, which renders every card from it and never reaches the network itself. One
signed listing read whole is what every add-on store and package index does.

**The list also says what an update is.** A node compares an available entry's `identity` with the
one it runs and offers the household the update on its page and in Home Assistant; it never takes
one unasked. A new version is a new file plus its list line, published together, the file first. An
entry that needs a newer briard is published only once that briard is `stable` and nodes have had a
night to take it — an update offered sooner is one that is tried, fails its gate, and is put back.

**Choosing what to list.** The list is for apps a household wants, not a mirror of everything
that can be self-hosted. Left out on purpose: anything whose role the platform already fills
(reverse proxies, container managers, remote-access tunnels, auto-updaters, uptime and metrics
stacks). Apps that need a user-data store, or a design pass of their own, are listed although
none is installable yet: the list exists to learn which of them to build.

**Icons** come from [homarr-labs/dashboard-icons](https://github.com/homarr-labs/dashboard-icons)
(Apache-2.0 for the collection; each mark remains its project's) unless the collection's copy is
a bitmap, in which case the project's own vector logo is used — see `THIRD-PARTY.md`. A page
shows them through `<img>`, where an SVG's own styles and scripts cannot reach the page; the
list's validation is the second line.

## The checks

briard reads this repository as a pinned flake input (`catalog`). `nix build .#catalog-test` in
briard lays it at `catalog/` and runs `catalog/catalog_test.go`, which parses every version file
and the list with the same functions a node uses, holds the list to its files, and holds each
app's current version to what briard's own code for that app assumes (Home Assistant's `/config`
and host networking, the broker's ports and data mount, Files' config path). This repository's CI
runs the same check against briard `main`:

```sh
nix build github:briardhq/briard#catalog-test --override-input catalog path:.
```

**The VM tests run these entries.** briard builds each app's test fixture from the version the
list names, so a changed entry is what they boot, and an app's own tests are tagged `app-<name>`
to run as its gate. The one thing they need that no entry says is the hash nix checks the pulled
image against — the hash of the archive nix repacks it into, which no registry digest gives. A
new image's goes in `catalog-images.json`; build once without it and nix prints it.

## What an entry costs: `size` and `installedSize`

Every entry declares two totals, measured once when the entry is made and signed with the rest:
`size` is what a node **downloads** (the compressed layers of every container's image, summed)
and `installedSize` is what the image store **holds** afterwards (the same layers uncompressed —
two to four times more; Home Assistant 2026.9.4 is 646 MB down and 2.47 GB on disk). The first is the
dashboard's progress bar; the second is the free-space check a node runs *before* the first byte
moves. Two scalars rather than a layer table: the digest already commits to every layer, and
podman's own store knows each finished layer's size. To measure a new entry:

```sh
skopeo copy docker://<image@digest> dir:/tmp/img            # then, in /tmp/img:
jq '[.layers[].size] | add' manifest.json                     # size
for l in $(jq -r '.layers[].digest' manifest.json); do zstd -dc "${l#sha256:}"; done | wc -c   # installedSize (zstd layers; gzip -dc for a gzip image -- the mediaType says which)
```

## What an entry needs to run: `minMemoryMB`

The guest starts small and grows while it runs, so an entry does not declare its worst case — only
what it needs from its **first second**, before growth can react: its settled footprint with room
for its own startup. A node boots its guest with at least the system's share plus every installed
entry's `minMemoryMB`, and grows the running guest to that before installing one more (refusing
only when the host cannot give that much at all). Anything an app needs beyond its minimum is
found at runtime, by growth. Measure a new entry from its anonymous memory settled after startup
(the cgroup's `anon` in `memory.stat`) — not from `docker stats`, which counts page cache too.
Omitted, the entry counts for nothing until it needs more.

## What is not in a manifest

The schema deliberately cannot express host binds, privileges, host networking or a command line —
capability comes by omission, so a catalog entry cannot reach the host. Anything an app needs
beyond that is **product-side, keyed on the app name** (briard's `agent/apps`): Home Assistant's
control channel, mosquitto's config file. If a new entry cannot be expressed here, that is the
question to answer, not a field to add. A schema change is two commits in two repositories, the
reader first: an entry using a new field waits for the briard that reads it to reach `stable`.

## Publishing

An entry may only be published once its image has been through whatever gate the catalog demands
of it: for Home Assistant that includes booting under the token wrapper (briard's `agent/hass`),
because an image whose s6 furniture moved would otherwise become a fleet incident rather than a
failed promotion. Publishing uploads the new version files first, then `latest.json` and its
signature; what was published is brought back here.

## Where a new version comes from: `catalog-upstream.json`

The nightly catalog run looks for each app's newest release in the image repository its current
version pulls from, by tag. An app's release `X.Y.Z` is the tag `X.Y.Z` unless
`catalog-upstream.json` gives the app a template: Home Assistant tags its releases bare, while
mosquitto's official image publishes `X.Y.Z-alpine` and Files `X.Y.Z-stable`. Pre-releases and
moving tags (`latest`, `stable`, `2026.11`, `-beta`) never match a template, so they are never
proposed. A candidate is proposed with its VM tests, its upgrade from the previous version, an
audit of its image and a fresh install on a stock Ubuntu machine behind it, and published only
once the owner merges it in the signing repository.
