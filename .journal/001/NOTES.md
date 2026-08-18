---
id: 001
title: New session kickoff
started: 2026-08-17
---

## 2026-08-17 22:25 — Kickoff
Goal for the session: not yet stated; user asked to start a new session and will provide the actual task next.
Current state of the world: fresh repo (`simplestreams-oci`) at initial commit on `master`; journal scaffold just initialized for jmgilman; no prior sessions.
Plan: await the user's request, then plan from there.

## 2026-08-17 22:32 — Context loaded for architecture brainstorm
Goal restated: brainstorm a general design for simplestreams-oci, then spawn an architecture agent to formalize.
Read ~/code/imgoci/spec (README, Scope, spec.md headings + section 5.4) and ~/code/imgoci/go (README, architecture.md, doc.go).
Key facts:
- imgoci spec: OS-image releases as OCI image indexes; 5 selector annotations (architecture/target/representation/usage/role) + compression; standard + BigOCI file manifests; spec Scope explicitly excludes SimpleStreams-style catalog systems and says "an adapter that makes an existing catalog reader understand imgoci objects" is out of spec scope — that adapter is presumably this repo.
- imgoci/go: hexagonal, one public package, ports in internal/transfer; Fetch/List/Resolve/FetchFiles consumer path; Publish producer path; digest-bound selection; CLI in separate module.
- This repo is an untouched template-go clone (Meigma template: cobra/viper CLI, moon, mise, melange/apko, release-please).
- Spec has incus target + incus-vm representation; SimpleStreams is the catalog protocol Incus/LXD consume.
Next: brainstorm design direction with user.

## 2026-08-17 22:48 — Brainstorm settled; architect agent spawned
Decisions from brainstorm with user:
- simplestreams-oci = translation proxy: imgoci releases in OCI registries (via imgoci/go) on the back, simplestreams protocol on the front; Incus is the target consumer.
- Own both sides of delivery: one CLI, `publish` + `serve` subcommands. `publish` wraps imgoci/go Client.Publish, computes Incus combined hashes (sha256 metadata++disk), stamps custom annotations (ReleaseSpec/FileSpec.Annotations already support this — verified in publish.go).
- Custom annotation namespace outside reserved io.imgoci.* (spec-clean; consumers ignore foreign annotations). Index-level: os/release/variant/serial/aliases. Entry-level: combined hashes on metadata entry.
- Granularity: one release index = one os/release/variant/serial; entries fan per-arch.
- v0 scope: incus-vm only (incus.tar.xz + disk-kvm.img ftypes). No containers — Incus pulls OCI app containers natively; system-container repr would be an imgoci spec addendum, not our problem. Un-annotated releases skipped+logged.
- Proxy stateless, metadata-only: sha256/size from imgoci content annotations; no file reads at catalog build; file GETs stream from registry w/ on-the-fly decompression; combined hashes relayed producer-asserted, unverified in v0.
Spawned software-architect agent SsOciArchitect for first-draft architecture (9-point deliverable incl. Incus protocol verification, package layout per AGENTS.md hex rules, ports, flows, verification plan, open questions). Awaiting result.

## 2026-08-17 23:10 — Architecture first draft delivered
SsOciArchitect (software-architect) delivered the draft; saved verbatim at .journal/001/ARCHITECTURE-DRAFT.md.
Highlights (Incus claims verified in lxc/incus source):
- Incus never parses the product key; identity from os/release/arch/variant fields. Serial must be >=8 chars, first 8 = YYYYMMDD. Fingerprint = combined_disk-kvm-img_sha256 on the metadata item = sha256(metaBytes ++ diskBytes); Incus verifies per-file sha256 and the combined hash itself.
- File item paths join from the HOST ROOT (not stream base URL) — proxy must own its host root; open question #2.
- imgoci/go has NO streaming fetch API (Dest is path-backed) → v0 needs a small blobfetch adapter (manifest GET by digest + blob stream + decomp); upstream io.Reader fetch API proposed as follow-up.
- Annotation ns: io.github.imgoci.simplestreams.* (os/release/variant/serial/aliases at index; combined.disk-kvm-img.sha256 on metadata entries).
- Packages: pure core catalog/stream/fileurl/combined/decomp; orchestration proxy/ + pub/ (declare ports, mockery mocks); adapters imgsrc/imgpub/tags/blobfetch/httpapi; cli wiring on template.
- Stateless serve: atomic snapshot refresher, last-known-good per ref on transient failure; /f/<host>/<repo…>/<digest>/<compression>/<filename> path scheme parsed positionally from both ends.
- e2e plan: testcontainers registry + real incus simplestreams client lib in-process; manual: incus remote add.
Top prototype risks: (1) http vs https for remote add, (2) host-root path joining, (3) blobfetch scope, (4) arch spelling mapping.
Status: awaiting user review of draft; next session likely starts prototyping.
