# simplestreams-oci — revised first-draft architecture (v2)

Revision over v1: the simplestreams side is now built on **`github.com/imgoci/go-simplestreams`** (read at `/Users/josh/code/imgoci/go-simplestreams`, module `v0.x`, Go 1.26). Incus protocol claims remain tagged **[V]** (verified in `lxc/incus` source) / **[A]** (assumed). New tag **[C]**: claim cross-checked against `go-simplestreams/schema/incus` CUE. Sources as before, plus go-simplestreams root package (`documents.go`, `items.go`, `builders.go`, `artifact.go`, `path.go`, `metadata.go`, `signing.go`, `mirror.go`, `writer.go`, `source.go`, `options.go`), `adapters/{fsmirror,httpmirror}`, `schema/{embed.go,incus,linuxcontainers}`, `.journal/{SIMPLE_STREAMS,TECH_NOTES}.md`.

## 0. Protocol grounding, re-checked against schema/incus

Everything verified in v1 stands. Cross-check against the CUE incus profile:

| Claim | lxc/incus source | schema/incus CUE | Verdict |
|---|---|---|---|
| `content_id` | never read **[V]** | `content_id!: "images"` required | CUE stricter; we emit `"images"` — no conflict for us |
| product `datatype` | checked only on the **index entry** **[V]** | required `"image-downloads"` on the product file | CUE stricter; comply |
| `aliases`, `release_title`, `variant`, `requirements` | all optional to the client **[V]** | all required (`!`) | CUE stricter; we always emit them (`requirements: {}`, `aliases` may be `""` — client splits only when non-empty **[V]**) |
| combined fields | client also reads `combined_disk1-img_sha256`, `combined_uefi1-img_sha256` **[V]** | **missing** from closed `#Item` — such a document fails validation | ⚠ upstream schema gap (flagged, not designed around; v0 emits only `combined_disk-kvm-img_sha256`, which the CUE models **[C]**) |
| version `label`/`pubname` | client reads both **[V]** | `#Version` is closed with only `items!` — a version carrying `label` fails validation | ⚠ upstream schema gap; v0 emits neither, so unaffected |
| item `md5`/`sha512`/`mirrors` | runtime `Item` supports them | absent from closed `#Item` | consistent with v0 output (sha256 only) |
| ftype vocabulary | client also accepts `disk1.img`, `uefi1.img` roots **[V]** | `#IncusFileType` = `incus.tar.xz \| lxd.tar.xz \| root.tar.xz \| squashfs \| disk-kvm.img \| squashfs.vcdiff` | narrower than client; v0's `incus.tar.xz`/`disk-kvm.img` are covered **[C]** |
| `requirements` keys | client copies arbitrary `requirements.*` **[V]** | closed vocabulary (cgroup/secureboot/cdrom_*) | v0 emits none; noted |

No CUE rule contradicts anything verified in lxc/incus; every divergence is the CUE profile being **stricter than the client** (fine for a producer) or **narrower than the client** (upstream gaps flagged above — worth an issue on go-simplestreams, not silent workarounds). go-simplestreams' own journal warns the Canonical JSON schemas are unreliable and prefers upstream source/live streams — consistent with my method.

Unchanged verified facts driving the design: product key unparsed; serial ≥8 chars, `YYYYMMDD` prefix; fingerprint = `combined_disk-kvm-img_sha256` on the metadata item, checked as `sha256(meta‖root)` post-download; per-file sha256 verified during download; item `path` is an arbitrary relative path, filename = last segment; file GETs join against host root, http-then-https. **[V]**

## 1. Domain model (pure core)

go-simplestreams now owns the product-tree half. Our core keeps only what is OCI-specific:

```go
// catalog.Release — proxy-side view of one fetched, spec-valid imgoci release
// (unchanged from v1: Host, Repository, Name, Version, Annotations, Entries).

// catalog.FileLoc — stateless locator for one stored file (unchanged):
// Host, Repository, ManifestDigest, Compression, Filename, Size.
```

The v1 `catalog.Item` type is **deleted**. Translation now emits go-simplestreams runtime nodes directly:

**Translation rules for `incus-vm` (v0)** — same rules 1–6 as v1, now expressed in library calls:

- `ss.NewProductFile("images")` per snapshot; set `DataType = "image-downloads"`, `Updated` (RFC 2822 — cosmetic, unparsed by Incus **[V]**).
- Per architecture: `productFile.SetProduct("os:release:arch:variant", …)` with product `Metadata` keys `os`, `release`, `release_title`, `arch`, `variant`, `aliases`, `requirements` (via `Product.SetMetadata`; the runtime model deliberately keeps consumer-profile fields in the metadata map).
- `product.SetVersion(serial, …)`; two `version.SetItem(name, …)` calls keyed and typed by ftype: `incus.tar.xz` (metadata role) and `disk-kvm.img` (disk role). `Item.FileType`, `Item.Path` (`ss.RelativePath` from `internal/fileurl`), `Item.Size`, `Item.SHA256` come from imgoci `content.*` annotations; the fingerprint is relayed with `item.SetMetadata("combined_disk-kvm-img_sha256", hex)` — unknown metadata is preserved and flattened at marshal time (verified in `documents.go` marshal path).
- Index: `ss.BuildIndex([]ss.BuildIndexEntry{{ContentID: "images", Path: "streams/v1/images.json", Format: ss.ProductsFormat, DataType: "image-downloads", Products: keys}}, updated)`. Non-empty `products` list is what Incus requires **[V]**; `BuildIndex` enforces path validity and duplicate content IDs.
- Serialization: `ss.MarshalJSONDocument(index / productFile)` — deterministic, sorted, trailing newline; snapshot caches the marshaled bytes.
- Duplicate item identity across sources: guarded with `ss.CheckDuplicateItemRefs` over emitted `ss.ItemRef`s; first configured source wins, warn (unchanged policy).

## 2. Annotation namespace and keys

**Unchanged from v1** (contract): namespace `io.github.imgoci.simplestreams.`, index-level `os`/`release`/`variant`/`serial` required + `aliases` optional; entry-level `combined.disk-kvm-img.sha256` on every metadata-role entry. Worked example as in v1. One addition: the annotation-key → simplestreams-field mapping is now normatively checked in tests against `schema/incus.ValidateRuntimeProductFile`, so a key-set change that breaks the Incus profile fails CI.

## 3. Package layout (redrawn)

Deleted relative to v1: **`internal/stream`** (replaced by go-simplestreams document types/builders/marshaling), **`internal/combined`** (replaced by `ss.SHA256Concat`). Kept but shrunk: `internal/fileurl` (the OCI locator scheme is ours; go-simplestreams only validates `RelativePath`, it defines no locator encoding). imgoci side unchanged.

```
cmd/simplestreams-oci/      main
internal/cli/               cobra wiring: root, publish, serve
internal/config/            viper config for serve + publish

# pure core
internal/catalog/           annotation parsing, incus-vm translation → *ss.ProductFile/*ss.Index,
                            merge + skip decisions; imports go-simplestreams root pkg (data-only use)
internal/fileurl/           catalog.FileLoc <-> ss.RelativePath scheme, encode + parse
internal/decomp/            bounded streaming decoders: none/gzip/xz/zstd

# orchestration cores (declare ports; mockery mocks in mocks/ per T2/T3)
internal/proxy/  (+mocks/)  serve service: refresher, atomic snapshot of marshaled docs, request logic
internal/pub/    (+mocks/)  publish service: input validation, combined hash via ss.SHA256Concat,
                            ReleaseSpec assembly

# adapters (A2)
internal/imgsrc/            ReleaseSource over imgoci/go Client.Fetch
internal/imgpub/            Publisher over imgoci/go Client.Publish
internal/tags/              OCI tag listing (/v2/<repo>/tags/list)
internal/blobfetch/         FileStreamer: file-manifest GET by digest + layer stream + decomp
internal/httpapi/           net/http server: routes, cached document bytes, file streaming
```

Rejected: building the serve side on go-simplestreams' `Mirror` — that is a **consumer** (read) model; we are the producer/server. Also rejected: generating a static mirror through `Store`/`AtomicStore` — those are explicitly "writer foundation ports only" today (TECH_NOTES: publish orchestration, artifact writes, signing, atomic updates are future work upstream). A future `mirror` subcommand (render releases into an `fsmirror`-style static tree) would sit exactly on those ports once upstream finishes them — noted, not v0.

## 4. Ports

Unchanged from v1 (`ReleaseSource`, `TagLister`, `FileStreamer` in `internal/proxy`; `Publisher` in `internal/pub`), with one signature change: the refresher's output to the snapshot is now `*ss.ProductFile`/`*ss.Index` built by `catalog`. go-simplestreams' own `Source` port is not implemented by us in v0 (nothing consumes a mirror); it appears only in functional tests (§8).

## 5. `publish` flow

Steps 1–2, 4–5 unchanged from v1. Step 3 now delegates: per architecture, open metadata and disk files and compute the fingerprint with **`ss.SHA256Concat(metaFile, diskFile)`** — order meta-then-disk matches Incus' post-download check **[V]**. v0 still publishes `compression=none`, so file bytes == decoded bytes and no decode pass is needed. Per-file sha256/size are computed by imgoci/go during `Publish`, not by us.

## 6. `serve` flow

Config, refresh model, authorization, and the stateless file-path scheme are unchanged from v1:

```
/f/<host>/<repository…>/<manifest-sha256-hex>/<compression>/<filename>
```

`internal/fileurl` renders this as an `ss.RelativePath` and validates with `RelativePath.Validate()` (non-empty, relative, no `..`/`...`/backslash traversal — matching the protocol's own path rules), then our parser applies the positional grammar (host first, filename last, 64-hex digest third-from-last).

Catalog build now ends in: build documents via §1, validate the assembled product file with `schema/incus.ValidateRuntimeProductFile` (**debug/test builds and an opt-in `--validate` flag only** — it spins a CUE context per call, too heavy for every refresh on the hot path; P1), marshal once with `ss.MarshalJSONDocument`, store bytes in the snapshot. Routes serve the cached bytes:

| Route | Serves |
|---|---|
| `GET /streams/v1/index.json` | snapshot's marshaled `ss.Index` (path constant `ss.DefaultIndexPath`) |
| `GET /streams/v1/images.json` | snapshot's marshaled `ss.ProductFile` |
| `GET /f/...` | file streaming via `FileStreamer` (manifest GET by digest → layer blob → `decomp` → response), unchanged |
| `GET /healthz` | snapshot age + source counts |

Signing: not in v0. go-simplestreams handles *verification* of `.sjson`; **producing** signed metadata is unfinished upstream (flagged). Incus consumes unsigned `streams/v1/index.json` **[V]**, so nothing blocks; serving `index.sjson` becomes trivial once upstream grows signing.

## 7. Failure and skip semantics

Unchanged from v1 (table stands). One addition: if the assembled product file ever fails `ValidateRuntimeProductFile` under `--validate`, the snapshot swap is refused and the previous snapshot keeps serving (fail-closed on our own output, fail-open on upstream releases).

## 8. Verification plan

- **Unit (pure):** `catalog` golden tests — annotated release views in, `MarshalJSONDocument` bytes out — plus `schema/incus.ValidateRuntimeProductFile` over every golden output; `fileurl` round-trips incl. hostile paths (leaning on `RelativePath.Validate`); `decomp` per codec; serial grammar. Combined-hash vector test moves upstream's way: `ss.SHA256Concat` is already tested there; we test only our call ordering (meta before disk).
- **Integration (mock ports):** unchanged (refresher, httpapi, pub with mockery mocks).
- **Functional/e2e:** testcontainers registry + `publish` + `serve`, then consume the running proxy **twice**: (a) with go-simplestreams itself — `httpmirror.New(proxyURL)` → `ss.NewMirror` → `Index`/`ProductFile`/`Items` → `ArtifactRef.VerifyReader` over the file GETs (checks per-file sha256 + size exactly as a strict client would); (b) with the real Incus client library (`shared/simplestreams` + `client.ProtocolSimpleStreams.GetImageFile`) for the fingerprint check. Manual acceptance: `incus remote add … --protocol simplestreams` + `incus launch --vm`.

## 9. Open questions, risks, upstream flags (by risk)

1. **TLS/scheme for `incus remote add`** — unchanged, still the top prototype experiment.
2. **Host-root file path joining** (`urlJoinPathAbsolute`) — unchanged; serve at host root until disproven.
3. **imgoci/go streaming gap** — unchanged; `blobfetch` stays here, upstream `io.Reader` fetch API proposed.
4. **Upstream (go-simplestreams) gaps found, belong upstream:** (a) `schema/incus` `#Item` omits `combined_disk1-img_sha256`/`combined_uefi1-img_sha256` and `#Version` omits `label`/`pubname` — closed defs reject documents the Incus client accepts; (b) metadata **signing** and `Store`/`AtomicStore` publish orchestration are unfinished (self-declared); (c) typed combined-hash fields on the runtime `Item` would beat `SetMetadata` string keys. None block v0.
5. **Belongs here, not upstream:** the OCI file-locator path scheme, HTTP serving, OCI blob streaming/decompression, tag listing, imgoci annotation contract.
6. **Architecture spellings / `arm/v7`→`armhf` mapping** — unchanged **[A]**.
7. **Compressed publish inputs**, **refresh model**, **requirements/EOL keys** — unchanged deferrals; note `#Requirements`' closed CUE vocabulary when requirements keys do land.
8. **CUE validation cost** — `ValidateRuntimeProductFile` builds a CUE context per call; if `--validate` becomes always-on, cache the loaded schema value (or push a reusable-context API upstream).