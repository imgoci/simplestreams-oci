# simplestreams-oci — first-draft architecture

Draft for prototyping. Incus protocol claims are tagged **[V]** (verified in source) or **[A]** (assumed, needs prototype check). Sources: `lxc/incus` `shared/simplestreams/{products,simplestreams,index}.go` and `client/simplestreams_images.go` (main branch, read 2026-08-18); `imgoci/spec` spec.md §3, §5, §7, §8; `imgoci/go` public API (`client.go`, `publish.go`, `fetch.go`, `resolve.go`, `list.go`, `fetchfiles.go`, `entry.go`, `dest.go`) and its architecture explainer.

## 0. What the Incus client actually requires (grounding)

- `GET <base>/streams/v1/index.json` → `Stream{index: map[string]{datatype, path, products[]}}`. Incus only uses entries with `datatype == "image-downloads"` and a **non-empty `products` list**; `format`, `updated`, `content_id` are never checked. **[V]** (`getImages`)
- Products JSON: Incus **never parses the product key**; identity comes from the `os`/`release`/`arch`/`variant` fields. Key format `os:release:arch:variant` is convention only. **[V]** (`ToAPI` ignores map keys)
- Version key (serial): must be ≥8 chars and `name[0:8]` must parse as `YYYYMMDD`; it becomes the image creation date and the `serial` property. **[V]**
- VM image = items with `ftype: "incus.tar.xz"` (metadata) + `ftype: "disk-kvm.img"` (disk). `disk-kvm.img` root ⇒ `image.Type = "virtual-machine"`. **[V]**
- Fingerprint = `combined_disk-kvm-img_sha256` **on the metadata item**. After download Incus recomputes `sha256(metaBytes ‖ rootBytes)` and rejects on mismatch — so it is the sha256 of the served metadata file bytes concatenated with the served disk file bytes, in that order. **[V]** (`ToAPI` + tail of `GetImageFile`)
- Each item's `sha256` is verified per file during download (`DownloadFileHash`), and `path` is an arbitrary relative path: index/products paths are joined with the stream base URL (`url.JoinPath`), file paths are joined against the **host** (`urlJoinPathAbsolute` on `httpHost`), trying plain `http://` first, then `https://`. **[V]** Consequence: serve the proxy at its own host/port root, not path-mounted under a prefix (file paths appear to resolve from host root — flagged in §9). The last path segment becomes the reported filename. **[V]**
- `arch` must resolve through Incus `osarch.ArchitectureID` (`amd64`, `arm64` work — images.linuxcontainers.org publishes those spellings). **[A]** — prototype check; `arm/v7`-style imgoci values need mapping (`armhf`).
- Content-Length / Range on file GETs: not required for correctness; hash-verified streaming download. **[A]** (did not read `util.DownloadFileHash`).

## 1. Domain model (pure core)

The core is a translation: *annotated release view → simplestreams product tree*. No I/O anywhere in it.

```go
// catalog.Release is the proxy-side view of one fetched, spec-valid imgoci
// release, decoupled from imgoci/go types at the port boundary.
type Release struct {
    Host, Repository string        // where it lives (for file locators)
    Name, Version    string        // io.imgoci.name / oci version
    Annotations      map[string]string // root annotations (incl. ours)
    Entries          []Entry       // file entries (selector, content digest/size, filename, annotations)
}

// catalog.ProductInfo — parsed from our root annotations.
type ProductInfo struct {
    OS, ReleaseName, Variant, Serial string
    Aliases                          []string // optional
}

// catalog.Item — one simplestreams item plus the locator the HTTP layer
// encodes into its path.
type Item struct {
    FType    string   // "incus.tar.xz" | "disk-kvm.img"
    SHA256   string   // hex of io.imgoci.content.digest
    Size     int64    // io.imgoci.content.size
    Combined string   // fingerprint hex; set on the metadata item only
    Loc      FileLoc
}

// catalog.FileLoc names one stored file statelessly.
type FileLoc struct {
    Host, Repository string
    ManifestDigest   string // sha256 hex of the file manifest
    Compression      string // imgoci compression of that stored alternative
    Filename         string // io.imgoci.filename
    Size             int64  // decoded size; best-effort Content-Length
}
```

**Translation rules for `incus-vm` (v0):**

1. Parse `ProductInfo` from root annotations; any required key missing/invalid ⇒ skip release (§7).
2. Group entries by deliverable key (arch, target=`incus`, representation=`incus-vm`, usage=∅). Per architecture, require exactly the roles `metadata` and `disk`; pick one transport alternative per role — preference order `none, zstd, xz, gzip` (only decoders we ship; unknown compression ⇒ skip that arch).
3. Map imgoci architecture → simplestreams arch: `amd64`/`arm64` pass through; anything else skipped with a warning in v0 (mapping table is an open question).
4. Each architecture ⇒ one **product**: `os`, `release` (= `release_title`), `variant`, `arch`; key `os:release:arch:variant` (cosmetic, **[V]** unparsed). One **version** keyed by `Serial`, with two items: metadata (`ftype incus.tar.xz`, carries `combined_disk-kvm-img_sha256` relayed from our entry annotation) and disk (`ftype disk-kvm.img`).
5. Multiple releases mapping to the same product (same os/release/arch/variant, different serials) merge into one product with several versions. Same product **and** serial from two sources: first configured source wins, warn.
6. `sha256`/`size` per item come straight from `io.imgoci.content.digest`/`.size` — the proxy serves decoded bytes, so per-file hashes hold by construction. The combined hash is relayed producer-asserted, never recomputed (contract #7).

## 2. Annotation namespace and keys

Namespace: **`io.github.imgoci.simplestreams.`** — reverse-DNS over `github.com/imgoci`, which the project controls; safely outside the reserved `io.imgoci.` prefix (spec §5.2 permits foreign keys; consumers must ignore unknowns).

Index-level (root annotations):

| Key | Required | Syntax | Maps to |
|---|---|---|---|
| `…simplestreams.os` | yes | non-empty, no whitespace | `Product.os` |
| `…simplestreams.release` | yes | same | `Product.release`, `release_title` |
| `…simplestreams.variant` | yes | same; `default` for none | `Product.variant` |
| `…simplestreams.serial` | yes | ≥8 chars; first 8 a valid `YYYYMMDD` date; recommend `YYYYMMDD.N` or `YYYYMMDD_HHMM` (avoid `:` in URL paths) | version key / `serial` property |
| `…simplestreams.aliases` | no | comma-separated alias names, no whitespace | `Product.aliases` |

Entry-level, on **every `role=metadata` entry** of an `incus-vm` deliverable (identical across transport alternatives of the same file, mirroring the content-annotation rule):

| Key | Syntax | Maps to |
|---|---|---|
| `…simplestreams.combined.disk-kvm-img.sha256` | 64 lowercase hex | `combined_disk-kvm-img_sha256` → Incus fingerprint |

The key name mirrors the simplestreams field so future combined fields (`combined.squashfs.sha256`, …) extend the same pattern.

Worked example (digests elided):

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.index.v1+json",
  "artifactType": "application/vnd.imgoci.release.v1",
  "annotations": {
    "io.imgoci.name": "acmeos-trixie-cloud",
    "org.opencontainers.image.version": "20260818.1",
    "io.github.imgoci.simplestreams.os": "acmeos",
    "io.github.imgoci.simplestreams.release": "trixie",
    "io.github.imgoci.simplestreams.variant": "cloud",
    "io.github.imgoci.simplestreams.serial": "20260818.1",
    "io.github.imgoci.simplestreams.aliases": "acmeos/trixie/cloud"
  },
  "manifests": [
    { "artifactType": "application/vnd.imgoci.file.v1", "digest": "sha256:…", "size": 427,
      "annotations": {
        "io.imgoci.architecture": "amd64", "io.imgoci.target": "incus",
        "io.imgoci.representation": "incus-vm", "io.imgoci.role": "metadata",
        "io.imgoci.compression": "none",
        "io.imgoci.content.digest": "sha256:…", "io.imgoci.content.size": "1372160",
        "io.imgoci.filename": "incus.tar.xz",
        "io.github.imgoci.simplestreams.combined.disk-kvm-img.sha256": "9c7d…"
      }},
    { "…disk entry: role=disk, filename=disk.qcow2, compression=zstd, content.* = qcow2 digest/size…" }
  ]
}
```

## 3. Package layout

Module renamed to `github.com/imgoci/simplestreams-oci`; `cmd/template-go` → `cmd/simplestreams-oci`. Each package: one job (A3/A4), `doc.go` everywhere (D4).

```
cmd/simplestreams-oci/      main: version stamping, calls internal/cli
internal/cli/               cobra wiring: root, publish, serve (existing template pattern)
internal/config/            viper-backed config types for serve + publish flags (existing package, extended)

# pure core (stdlib + codecs only)
internal/catalog/           annotation parsing, incus-vm translation, product-tree merge, skip decisions
internal/stream/            simplestreams wire structs (Stream, Products, Product, Version, Item) + JSON encoding
internal/fileurl/           FileLoc <-> URL path scheme, encode + parse (tiny, pure)
internal/combined/          streaming combined-hash computation over io.Readers (pure w.r.t. side effects)
internal/decomp/            bounded streaming decoders: none/gzip/xz/zstd (io.Reader in, io.Reader out)

# orchestration cores (declare the ports)
internal/proxy/             serve service: refresher, atomic catalog snapshot, request handling logic
internal/proxy/mocks/       mockery output for proxy ports (T2/T3)
internal/pub/               publish service: input validation, combined-hash pass, ReleaseSpec assembly
internal/pub/mocks/         mockery output for pub ports

# adapters (one purpose each, A2)
internal/imgsrc/            ReleaseSource adapter over imgoci/go Client.Fetch → catalog.Release
internal/imgpub/            Publisher adapter over imgoci/go Client.Publish
internal/tags/              TagLister adapter: OCI /v2/<repo>/tags/list (oras-go remote)
internal/blobfetch/         FileStreamer adapter: file-manifest GET by digest + layer blob stream, composed with decomp
internal/httpapi/           driving adapter: net/http server, routes, content types, error mapping
```

Rejected alternatives: (a) folding `stream` into `catalog` — kept separate so wire shape changes don't churn translation tests; (b) one `internal/oci` mega-adapter — violates A2; (c) reusing imgoci/go for file streaming — impossible today, its retrieval is path-backed (`Dest`), see §4/§9.

## 4. Ports

Declared by the orchestration cores, mocked with mockery into `mocks/` subpackages (T2/T3).

```go
// internal/proxy

// ReleaseSource fetches and spec-validates one release index.
type ReleaseSource interface {
    Fetch(ctx context.Context, ref string) (catalog.Release, error)
}

// TagLister lists tags in one OCI repository (for tag-pattern sources).
type TagLister interface {
    Tags(ctx context.Context, host, repository string) ([]string, error)
}

// FileStreamer opens the decoded content stream for one stored file.
// size is the decoded length when known, else -1.
type FileStreamer interface {
    Open(ctx context.Context, loc catalog.FileLoc) (rc io.ReadCloser, size int64, err error)
}
```

```go
// internal/pub

// Publisher publishes an assembled release spec and returns the index digest.
type Publisher interface {
    Publish(ctx context.Context, ref string, spec imgoci.ReleaseSpec) (digest.Digest, error)
}
```

`Publisher` intentionally speaks `imgoci.ReleaseSpec` — it is a plain data struct and inventing a parallel spec type buys nothing (tradeoff noted; if it grows I/O-coupled fields, introduce our own). `imgsrc` maps `*imgoci.Release`/`Index`/`FileEntry` onto `catalog.Release` so the core never imports imgoci/go.

## 5. `publish` flow

```
simplestreams-oci publish \
  --ref ghcr.io/acme/acmeos:trixie-20260818.1 \
  --os acmeos --release trixie [--variant cloud] [--serial 20260818.1] \
  [--alias acmeos/trixie/cloud]... \
  --image arch=amd64,metadata=./amd64/incus.tar.xz,disk=./amd64/disk.qcow2 \
  [--image arch=arm64,...]... \
  [--name acmeos-trixie-cloud] [--version 20260818.1]
```

1. `internal/cli` parses flags into a pure `pub.Input`; defaults: `variant=default`, `serial` = UTC `YYYYMMDD_HHMM` now, `name` = slug of os-release-variant, `version` = serial.
2. `pub` validates: serial grammar (date-prefixed), ≥1 image, arch uniqueness, alias syntax.
3. Per arch, one streaming pass per file through `internal/combined`: `sha256(metadataBytes ‖ diskBytes)` (order verified in §0). v0 publishes `compression=none` for both roles, so source bytes == decoded bytes and no decode pass is needed (compressed transport at publish is deferred, §9).
4. Assemble `imgoci.ReleaseSpec`: root `Annotations` = the §2 index keys; per arch two `FileSpec`s — metadata (`target=incus, representation=incus-vm, role=metadata, compression=none`, filename `incus.tar.xz`, annotation = combined hash) and disk (`role=disk`, filename `disk.qcow2`). imgoci/go computes content digest/size itself and enforces incus-vm producer rules (roles, `incus` target).
5. `Publisher.Publish`; print the canonical index digest and a per-arch summary to `Out`.

## 6. `serve` flow

Config (viper: file + env + flags):

```yaml
listen: ":8080"
refresh: 15m               # 0 disables periodic refresh (refresh once at start)
sources:
  - ref: ghcr.io/acme/acmeos:trixie-20260818.1   # explicit reference
  - repo: ghcr.io/acme/acmeos                     # tag pattern via TagLister
    tags: "trixie-*"                              # path.Match glob
# registry auth: reuse docker login (imgoci WithDockerCredentials) — opt-in flag
```

**Catalog build (refresher):** on start and every `refresh`: expand pattern sources through `TagLister`; `ReleaseSource.Fetch` each ref (imgoci/go validates the index fully); run `catalog` translation; merge into one `stream.Products` + `stream.Stream`; swap into an `atomic.Pointer[Snapshot]`. Per-ref transient failure reuses that ref's last successful `catalog.Release` when one exists; validation/skip failures drop it (§7). The proxy holds no disk state — restart rebuilds from the registries.

**Routes** (`internal/httpapi`):

| Route | Serves |
|---|---|
| `GET /streams/v1/index.json` | one index entry, `datatype: image-downloads`, `path: streams/v1/images.json`, `products`: current keys |
| `GET /streams/v1/images.json` | the snapshot's products document |
| `GET /f/...` | file streaming (below) |
| `GET /healthz` | snapshot age + source counts |

**File path scheme** — stateless, parsed positionally from both ends (no marker segments; the digest's fixed 64-hex shape disambiguates the variable-depth repo):

```
/f/<host>/<repository…>/<manifest-sha256-hex>/<compression>/<filename>
e.g. /f/ghcr.io/acme/acmeos/9c7d…e1/zstd/disk.qcow2
```

`host` = first segment, `filename` = last (Incus reports it as the downloaded name **[V]**), `compression` = second-to-last, digest = third-to-last, repo = the middle. Encodes registry + repo + manifest digest + role's stored alternative with no server state, so paths survive restart and Incus's on-disk products cache.

**File GET path:** parse → authorize: `(host, repo)` must be in configured sources (prevents becoming an open relay/decompression proxy) → `FileStreamer.Open`: GET file manifest by digest (verify manifest-byte digest, validate standard-form §3.1), GET the layer blob, wrap in the `decomp` decoder named by the path (bounded decoder window, imgoci-style) → stream to the response (`io.Copy`, no buffering, P2). `Content-Length` set best-effort from the snapshot's `FileLoc.Size` (always when `compression=none`); otherwise chunked. v0 supports standard file manifests only; BigOCI entries are skipped at catalog build (§7). No server-side content-digest verification — Incus verifies per-file sha256 and the fingerprint itself **[V]**; a corrupted stream fails safely on the client.

## 7. Failure and skip semantics (serve)

| Condition | Action |
|---|---|
| Release fails imgoci validation (`ErrInvalidIndex`) | skip release, log warn with ref + reason |
| Missing/invalid `…simplestreams.*` required annotation | skip release, log warn |
| No `incus-vm` deliverable in a valid annotated release | skip release, log info |
| Arch with missing role, unsupported compression, or non-standard (BigOCI) manifest type only | skip that arch, log warn |
| Metadata entry missing `combined.disk-kvm-img.sha256` | skip that arch (no fingerprint ⇒ Incus rejects it anyway **[V]**) |
| Duplicate product+serial across sources | first configured source wins, warn |
| File GET: malformed path / (host,repo) not configured | 404 |
| File GET: upstream registry error / digest mismatch on manifest | 502 (or mid-stream abort once bytes are sent) |

## 8. Verification plan

- **Unit (pure):** `catalog` golden tests — annotated release view in, products JSON out, including merge, skip, and duplicate-serial cases; `fileurl` encode/parse round-trips incl. hostile paths; `combined` against a fixed vector; `decomp` per codec incl. trailing-byte rejection; serial grammar.
- **Integration (mock ports, mockery):** refresher with mocked `ReleaseSource`/`TagLister` — snapshot swap, last-known-good on transient failure, skip logging; `httpapi` with mocked `FileStreamer` — route/status/header contract; `pub` with mocked `Publisher` — assembled `ReleaseSpec` shape incl. annotations.
- **Functional/e2e:** testcontainers `registry:2` (or zot): `publish` a tiny fixture release, run `serve`, then consume with the **real Incus client library** (`github.com/lxc/incus/v7/shared/simplestreams` + `client.ProtocolSimpleStreams` in a test) — `ListImages`, `GetImage`, `GetImageFile` into temp files; that exercises Incus's exact parsing, per-file sha256, and fingerprint check without a daemon. Manual acceptance: `incus remote add test <url> --protocol simplestreams && incus launch test:<alias> --vm` against a real Incus.

## 9. Open questions and prototype experiments (by risk)

1. **TLS/scheme for `incus remote add --protocol simplestreams`** — file downloads try http-then-https **[V]**, but whether the remote URL itself may be plain `http://` (dev) or needs a valid cert is unverified. *Experiment: remote add against the prototype over http and self-signed https.*
2. **Path-mounted base URLs** — file paths appear joined from the **host root** (`urlJoinPathAbsolute(httpHost, path)`), unlike index/products paths (base-URL join) **[V-ish]**; I did not read `urlJoinPathAbsolute`. If confirmed, the proxy must own its host root — document it, or emit absolute file URLs? (Incus splits on `/` for filename; full-URL paths unverified.) *Experiment: serve under a sub-path and watch the file GETs.*
3. **Streaming gap in imgoci/go** — `blobfetch` reimplements manifest GET + blob streaming + auth that imgoci/go has internally but doesn't export. Right call for v0; propose an upstream `io.Reader`-based fetch API and collapse `blobfetch` onto it later.
4. **Architecture spellings** — `amd64`/`arm64` through `osarch` **[A]**; mapping for `arm/v7`→`armhf` etc. deferred. *Experiment: list images from real Incus for both arches.*
5. **`incus:` alias resolution details** — alias dedup/preference (`sortedImages`) read but not exercised; verify aliases surface as expected in `incus image list`.
6. **Compressed publish inputs** — v0 publishes `compression=none`; accepting pre-compressed sources requires a decode pass for the combined hash. Defer until registry-size pressure is real (qcow2 already compresses internally).
7. **Refresh model** — fixed interval + last-known-good is v0; webhook/on-demand invalidation and per-source intervals only if refresh cost shows up.
8. **Requirements/EOL metadata** (`requirements.*`, `support_eol`) — deliberately out of the v0 key set; add keys when a consumer needs them.