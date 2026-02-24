# Transparent File System: Joint State-of-the-Art Review

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Platform Constraints](#2-platform-constraints)
3. [VFS & Storage Abstraction](#3-vfs--storage-abstraction)
4. [Content-Addressed Storage](#4-content-addressed-storage)
5. [Tree Structure & Conflict Resolution](#5-tree-structure--conflict-resolution)
6. [WASM Storage](#6-wasm-storage)
7. [Dioxus Integration](#7-dioxus-integration)
8. [OS-Level Mounting (Excluded)](#8-os-level-mounting-excluded)
9. [Sync Architecture](#9-sync-architecture)
10. [Crate Decision Matrix](#10-crate-decision-matrix)
11. [Risks & Mitigations](#11-risks--mitigations)
12. [Implementation Phases](#12-implementation-phases)
13. [File-Type-Specific Storage Strategies](#13-file-type-specific-storage-strategies)
14. [Spec Deliverables](#14-spec-deliverables)
15. [Sources](#15-sources)

> Section 13.5 covers domain-specific file systems (genomic, metabolomic) and their architectural implications.

---

## 1. Executive Summary

No single Rust crate provides an end-to-end transparent file system across WASM, mobile, and backend. The state of the art is a **composed stack** of mature building blocks. Both research tracks independently converged on the same fundamental architecture:

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **App-level VFS** | Custom async trait (backed by OpenDAL where useful) | Uniform file operations across all platforms |
| **Metadata + tree** | SQLite / Postgres (existing sync) + CRDT tree when needed | Directory hierarchy, file attributes, revisions, tombstones |
| **Content storage** | BLAKE3 + FastCDC → chunked CAS blobs | File bytes with dedup, integrity, and efficient sync |

**Key consensus points:**

- Not kernel-FS everywhere — app-level VFS semantics on all platforms, accessed through the application layer.
- File payloads as chunked content-addressed blobs, referenced by hash from metadata.
- Deterministic op-log for file operations, mapped to existing DB sync rather than syncing raw files.
- No single CRDT framework is mandatory today — two valid approaches exist depending on collaboration intensity.

---

## 2. Platform Constraints

| Capability | Backend (Linux) | Desktop | WASM (Browser) | iOS | Android |
|-----------|----------------|---------|----------------|-----|---------|
| Raw filesystem | Full | Full | **No** (OPFS only) | Sandboxed | Sandboxed |
| SQLite | Native | Native | Via `sqlite-wasm-rs` sahpool | Native (`rusqlite`) | Native (`rusqlite`) |
| PostgreSQL | Direct | Client only | No | No | No |
| Networking | TCP/UDP/QUIC | TCP/UDP/QUIC | HTTP/WebSocket only | TCP/UDP | TCP/UDP |
| Background sync | Unrestricted | Unrestricted | Service Worker (limited) | BGProcessingTask (~30s) | WorkManager |
| Rust target arch | x86_64/aarch64 | x86_64/aarch64 | **wasm32** | **aarch64** (native) | **aarch64** (native) |

**Critical Dioxus insight** (joint-verified): Dioxus mobile renders via WebView, but **Rust code runs natively** (not as WASM). On iOS/Android, `target_arch = aarch64`, so `std::fs`, `rusqlite`, and `tokio` all work. Browser APIs are only needed for the WASM target.

**PostgreSQL status**: 18.2 released 2026-02-12, with out-of-cycle 18.3 announced for 2026-02-26.

---

## 3. VFS & Storage Abstraction

### 3.1 Mature Building Blocks

| Crate | Version | WASM | Strength | Limitation |
|-------|---------|------|----------|------------|
| **`opendal`** | 0.51.x | Partial (OPFS, SQLite, memory backends) | 50+ backends, Apache TLP, `services-opfs`/`services-postgresql`/`services-sqlite` | Heavy dependency tree, object-storage mental model |
| **`object_store`** | 0.11.x | Builds but **cloud backends not supported on WASM** | Arrow ecosystem, production S3/Azure/GCP | Cloud-focused, no OPFS/SQLite backend |

### 3.2 SQLite VFS Crates (Codex finding)

| Crate | Status | Notes |
|-------|--------|-------|
| `sqlite-vfs` | Prototype | Trait-based custom VFS; marks itself prototype, lists WAL/shared-cache limitations |
| `sqlite_vfs` | Trait/register focused | Different crate despite similar name; useful for VFS research |

These are useful if you need a **custom SQLite VFS** (e.g., CAS-backed virtual pages) but are not on the critical path — `sqlite-wasm-rs` already provides the OPFS VFS needed for browser, and native platforms use standard `rusqlite`.

### 3.3 Recommended Approach: Custom Trait

Both research tracks agree: define a thin, async, platform-agnostic trait rather than depending on OpenDAL everywhere:

```rust
#[async_trait(?Send)] // ?Send required for WASM compatibility
pub trait FileStore {
    async fn read(&self, path: &str) -> Result<Vec<u8>>;
    async fn write(&self, path: &str, data: &[u8]) -> Result<()>;
    async fn delete(&self, path: &str) -> Result<()>;
    async fn exists(&self, path: &str) -> Result<bool>;
    async fn list(&self, prefix: &str) -> Result<Vec<FileEntry>>;
}
```

Implementations: `OpfsFileStore` (WASM), `NativeFileStore` (mobile/desktop via `std::fs`), `PostgresFileStore` (backend, optionally via OpenDAL).

---

## 4. Content-Addressed Storage

### 4.1 Core Stack (Joint Agreement)

| Component | Crate | Version | WASM | Role |
|-----------|-------|---------|------|------|
| **Hashing** | `blake3` | 1.6.x | Yes (~200 MB/s) | Content addressing — single hash function for everything |
| **Chunking** | `fastcdc` | 3.2.1 | Yes | Content-defined chunking for large file dedup |
| **Verified streaming** | `bao-tree` | 0.13.x | Yes | Optional: chunk-level integrity during transfer |
| **Serialization** | `postcard` | 1.x | Yes (no_std) | Compact binary encoding for chunk metadata |

### 4.2 Chunking Parameters

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Min chunk | 16 KB | Prevents too-small chunks |
| Avg chunk | 64 KB | Good dedup vs. metadata overhead balance |
| Max chunk | 256 KB | Fits comfortably in a DB row |
| Skip chunking | Files < 256 KB | Single chunk, avoid CDC overhead |

### 4.3 Storage Backend

| Scale | Backend Store | Chunk Store |
|-------|--------------|-------------|
| < 100 GB total chunks | Postgres `BYTEA` (backend) / SQLite `BLOB` (client) | In-database |
| > 100 GB | Postgres metadata only | S3 / object storage for blob data |

### 4.4 Database Schemas

**PostgreSQL:**

```sql
CREATE TABLE chunks (
    hash BYTEA PRIMARY KEY,
    data BYTEA NOT NULL,
    size INTEGER NOT NULL,
    ref_count INTEGER DEFAULT 1,
    created_at TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_chunks_hash ON chunks USING hash (hash);

CREATE TABLE files (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    path TEXT NOT NULL UNIQUE,
    root_hash BYTEA NOT NULL,
    size BIGINT NOT NULL,
    mime_type TEXT,
    updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE file_chunks (
    file_id UUID REFERENCES files(id),
    chunk_index INTEGER NOT NULL,
    chunk_hash BYTEA NOT NULL REFERENCES chunks(hash),
    PRIMARY KEY (file_id, chunk_index)
);
```

**SQLite (same structure, adapted types):** `BLOB` for hashes, `TEXT` for UUIDs, `datetime('now')` for timestamps.

### 4.5 Sync Protocol

```
Client                              Server
  |-- "My tree root: X" -----------> |
  |<-- "Mine is Y; diffs: [...]" --- |
  |-- "Need chunks [a, b, c]" ----> |
  |<-- chunk_a, chunk_b, chunk_c --- |
  |-- (verify hashes, reassemble) -- |
```

Only missing chunks transfer. A 100 MB file with a 1 KB edit syncs ~64 KB (one changed chunk).

---

## 5. Tree Structure & Conflict Resolution

This is the hardest problem and where our research was most complementary.

### 5.1 The Core Problem

When Device A moves `/docs/readme.md` → `/archive/readme.md` and Device B moves `/archive` → `/docs/archive` concurrently, naive approaches create cycles or orphan nodes. A production file system must handle this.

### 5.2 Available CRDT Libraries

| Library | Version | Tree CRDT | Move Safe | Cycle Prevention | Stars | Maturity |
|---------|---------|-----------|-----------|-----------------|-------|----------|
| **Loro** | 1.10.6 (Feb 2026) | LoroTree (native) | Yes (atomic) | Proven (Kleppmann) | 5.4k | **1.0+, 117 releases, 38+ contributors** |
| **yrs + yrs_tree** | yrs 0.21.x / yrs_tree 0.4.1 | Extension crate | Yes (move_to, move_after) | Yes (Evan Wallace algorithm) | yrs: large / yrs_tree: 2 | **yrs mature; yrs_tree early (single author, 37 commits)** |
| **Automerge** | 0.5.x (Automerge 3) | No | No | N/A | Large | Mature but no tree |
| **cr-sqlite** | Variable | Via SQL parent_id | LWW only | Manual | Medium | Functional, uncertain maintenance |
| **diamond-types** | 0.8.x | No (text only) | N/A | N/A | Medium | Text-specialized |

**Correction from Codex** (validated by Claude): The original claim "Yrs has no tree CRDT" was incomplete. `yrs_tree` (v0.4.1) exists as a community extension providing tree CRDT with move operations and cycle handling. However, it is early-stage: 2 GitHub stars, single author, 37 commits, 64% doc coverage — versus Loro's 5.4k stars, 38+ contributors, 117 releases, and 1.0+ stability.

### 5.3 Joint Recommendation: Two Valid Paths

We discussed this extensively and present both options honestly:

#### Option A: SQL-First Tree (Pragmatic Start)

```sql
CREATE TABLE file_tree (
    id TEXT PRIMARY KEY,          -- UUIDv7
    parent_id TEXT REFERENCES file_tree(id),
    name TEXT NOT NULL,
    is_directory BOOLEAN DEFAULT FALSE,
    content_hash TEXT,
    size INTEGER DEFAULT 0,
    permissions TEXT DEFAULT 'rw-r--r--',
    modified_at INTEGER,
    revision_id TEXT              -- Links to revision/op-log
);
```

- **Pros**: Works with existing Postgres↔SQLite sync. Simpler. SQL query capability. No new CRDT dependency.
- **Cons**: Concurrent moves resolved by LWW (last writer wins — losing move silently discarded). No cycle prevention. Risk of orphans under heavy concurrent structural edits.
- **Best for**: Teams with low concurrent directory editing, existing sync infra they trust, wanting to ship faster.

#### Option B: Loro LoroTree (Correct by Construction)

```rust
let doc = LoroDoc::new();
let tree = doc.get_tree("fs");
let dir = tree.create(Some(root))?;
tree.get_meta(dir)?.insert("name", "documents")?;
tree.mov(file, Some(dir))?; // CRDT-safe, no cycles ever
```

- **Pros**: Concurrent moves are safe (deterministic resolution, no cycles, no orphans). Time travel / version history for free. Shallow snapshots for compaction. First-class WASM + Swift bindings.
- **Cons**: New dependency (though 1.0+ stable). Loro export/import must be plumbed into your sync layer. Larger architectural investment.
- **Best for**: Teams expecting multi-device concurrent directory restructuring, wanting correctness guarantees, willing to invest in the CRDT layer.

#### Decision Gate

| Factor | Lean SQL-first | Lean Loro |
|--------|---------------|-----------|
| Expected concurrent folder moves | Rare | Frequent |
| Existing sync maturity | High (already handles tree) | Low (building from scratch) |
| Correctness requirements | "Good enough" | Must not lose/orphan files |
| Timeline to production | Short | Longer runway |
| yrs_tree maturity by ship date | If it matures, alternative option | Loro is already 1.0+ |

**Joint agreement**: If you expect frequent concurrent structural edits (multi-user collaboration on folder organization), evaluate Loro and `yrs_tree` early with targeted test cases. If your use case is primarily single-user-per-device with rare folder reorganization, SQL-first is pragmatic.

### 5.4 Conflict Policies (Joint)

| Conflict Type | Resolution |
|---------------|-----------|
| Binary file (same file edited on 2 devices) | Keep both versions (conflict copy) |
| Text file (collaborative) | 3-way merge where possible; fallback to keep both |
| File rename (concurrent renames) | LWW (Lamport timestamp) |
| File move (concurrent moves to different parents) | Loro: deterministic winner / SQL: LWW |
| Same-name creation in same directory | Keep both, suffix second ("file (2).md") |
| Delete vs edit conflict | Keep the edit (resurrection wins over deletion) |

---

## 6. WASM Storage

### 6.1 Joint-Verified: sqlite-wasm-rs

| | |
|---|---|
| **Crate** | `sqlite-wasm-rs` |
| **Version** | 0.5.0 |
| **VFS (recommended)** | `sahpool` — OPFS-backed, sync access handles, near-native speed |
| **VFS (fallback)** | `relaxed-idb` — IndexedDB-backed, broader compatibility |
| **Diesel support** | Yes (`diesel` feature flag) |
| **WAL mode** | Supported via sahpool |
| **Thread safety** | **Non-thread-safe** — must run in dedicated Web Worker |
| **Browser support** | Chrome 102+, Firefox 111+, Safari 15.2+ (~96%) |

Both research tracks verified this independently. This is the critical crate for SQLite in the browser.

### 6.2 OPFS Direct Access

For non-SQLite file storage in the browser (e.g., cached chunks, temporary files), use:

- `opfs` crate for Rust bindings to OPFS APIs
- OpenDAL `services-opfs` backend
- Direct `web-sys` OPFS bindings

### 6.3 Architecture: SQLite in a Web Worker

```
Main Thread (Dioxus UI)          Dedicated Worker
┌──────────────┐                ┌──────────────────┐
│  Dioxus WASM │ ◄──messages──► │ SQLite (WASM)    │
│  Components  │   postMessage  │ + sahpool VFS     │
│  + Signals   │                │ + OPFS (persist)  │
└──────────────┘                └──────────────────┘
```

SQLite MUST run in a Worker because OPFS sync access handles are Worker-only, and blocking the main thread freezes the UI. Call `navigator.storage.persist()` at startup to prevent browser eviction.

---

## 7. Dioxus Integration

### 7.1 Current State

| | |
|---|---|
| **Version** | 0.7.3 (Jan 2026) |
| **Mobile** | First-class — `dx serve --platform ios` / `--platform android` |
| **Web** | Stable — `dx serve --platform web` |
| **Desktop** | Stable — WebView via Wry |
| **Reactivity** | Signals (primary primitive since 0.5+) |

Dioxus provides **no built-in file system or storage abstraction**. You must build the `FileStore` trait (Section 3.3) and platform adapters yourself.

### 7.2 Platform Adapter Pattern

```rust
// Compile-time platform selection
#[cfg(target_arch = "wasm32")]
pub fn create_fs() -> impl FileStore { WasmFs::new() }

#[cfg(not(target_arch = "wasm32"))]
pub fn create_fs() -> impl FileStore { NativeFs::new() }
```

### 7.3 Mobile File Access

- **iOS**: `std::fs` works in app sandbox. SQLite in `Library/Application Support/`. For Files app integration: Swift File Provider Extension (call Rust via C FFI).
- **Android**: `std::fs` works in app data dir. SQLite in `getFilesDir()`. For system file picker: Kotlin DocumentsProvider (call Rust via JNI).
- **No pure-Rust crate** exists for iOS File Provider or Android DocumentsProvider.

### 7.4 Dioxus UI Patterns for File Operations

```rust
// Reactive file list with signals
fn file_explorer() -> Element {
    let files = use_signal(|| Vec::<FileEntry>::new());
    let _sync = use_coroutine(|_rx| {
        let mut files = files.clone();
        async move {
            loop {
                if let Ok(updated) = sync_with_server().await {
                    files.set(updated);
                }
                sleep(Duration::from_secs(30)).await;
            }
        }
    });
    rsx! { /* render files.read() */ }
}
```

---

## 8. OS-Level Mounting (Excluded)

FUSE (Filesystem in Userspace) was evaluated and **excluded from the architecture**. FUSE mounts the VFS as a regular OS drive visible to Finder, Nautilus, and external tools — but it only works on Linux and macOS desktop, which are the platforms that least need it (they already have real filesystems). It cannot work on iOS, Android, or WASM — the three platforms where this architecture must operate.

For OS-level file integration on mobile, the correct (non-FUSE) approaches are:

| Platform | Integration | Requirement |
|----------|------------|-------------|
| iOS | File Provider Extension | Swift bridge (no Rust crate exists) |
| Android | DocumentsProvider | Kotlin bridge (no Rust crate exists) |
| Browser | N/A | Files accessed exclusively through the Dioxus UI |

These native bridges are planned for Phase 5 if needed. The primary file access path on all platforms is the Dioxus application UI backed by the FileStore trait.

> **Research preserved**: See `claude/fuse_research.md` for the full FUSE evaluation. Crates `fuser` 0.14.x and `fuse-backend-rs` remain viable if a desktop/server FUSE mount becomes needed later.

---

## 9. Sync Architecture

### 9.1 Joint Data Model

Both tracks agree on these identity and revision principles:

| Concept | Design |
|---------|--------|
| File identity | `file_id` as UUIDv7 (sortable, timestamp-embedded) |
| Paths | Mutable metadata on the file record, not the identity |
| Revisions | Revision graph with `parent_revision_id` (branch on conflict) |
| Op-log | Idempotent: `(device_id, local_seq, op_id)` |
| Ordering | Server assigns canonical commit ordering |
| Deletions | Tombstones with tiered retention policy (e.g., 30 days) |

### 9.2 Two-Phase Logical Commit (Codex contribution)

```
1. Create `pending` revision
2. Upload/verify chunks (CAS)
3. Atomically mark `committed`
4. Later: GC abandoned chunks from failed commits
```

This prevents partial uploads from appearing as valid file versions.

### 9.3 Sync Flow

```
┌─────────────────────────────────────────────────────────┐
│                      Backend                             │
│  ┌──────────┐    ┌──────────┐    ┌───────────────────┐  │
│  │ Metadata │    │ Postgres │    │  Chunk Store      │  │
│  │ (tree +  │◄──►│ (persist)│    │  (Postgres / S3)  │  │
│  │ revisions)│   └──────────┘    └─────────┬─────────┘  │
│  └────┬─────┘                              │            │
│  ┌────┴────────────────────────────────────┴─────────┐  │
│  │              HTTP / WebSocket API                  │  │
│  │  POST /sync/metadata   (op-log / CRDT updates)    │  │
│  │  POST /sync/chunks     (upload missing chunks)    │  │
│  │  GET  /chunks/{hash}   (download chunk)           │  │
│  │  POST /chunks/need     (which hashes do I need?)  │  │
│  └────┬────────────────────────────────────┬─────────┘  │
└───────┼────────────────────────────────────┼────────────┘
        │                                    │
   ┌────┴────┐                          ┌────┴────┐
   │ Device  │                          │ Device  │
   │ SQLite  │                          │ SQLite  │
   │ + VFS   │                          │ + VFS   │
   └─────────┘                          └─────────┘
```

**Metadata syncs first** (tree structure, revisions), then **content syncs second** (only missing chunks).

### 9.4 Offline Outbox Pattern (Joint)

```sql
CREATE TABLE sync_outbox (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    op_type TEXT NOT NULL,
    payload BLOB NOT NULL,
    device_id TEXT NOT NULL,
    local_seq INTEGER NOT NULL,
    created_at TEXT DEFAULT (datetime('now')),
    sent_at TEXT,
    retry_count INTEGER DEFAULT 0
);
```

Process on connectivity: send in order, exponential backoff on failure, mark `sent_at` on success.

### 9.5 Real-World References

| System | Key Technique | Relevance |
|--------|--------------|-----------|
| Dropbox | 4 MB content-addressed blocks, conflict copies | CAS chunk strategy |
| Git | Content-addressable object store, tree objects as Merkle trees | CAS + tree model |
| iCloud Drive | Change tokens, file eviction/on-demand download | Sync protocol |
| Obsidian Sync | File-level sync for small files, diff-based conflict UI | Simple approach |

---

## 10. Crate Decision Matrix

### Must-Have (Both Agree)

| Crate | Version | WASM | Purpose | Confidence |
|-------|---------|------|---------|------------|
| `blake3` | 1.6.x | Yes | Content hashing | Very High |
| `fastcdc` | 3.2.1 | Yes | Content-defined chunking | High |
| `serde` | 1.x | Yes | Serialization framework | Very High |
| `postcard` | 1.x | Yes (no_std) | Compact binary encoding | High |
| `reqwest` | 0.12.x | Yes (wasm feature) | HTTP client (sync API) | Very High |
| `sqlite-wasm-rs` | 0.5.0 | Yes | Browser SQLite (sahpool) | High |
| `rusqlite` | 0.32.x | Native only | Native SQLite | Very High |
| `sqlx` | 0.8.x | Native only | Async Postgres + SQLite (backend) | Very High |
| `axum` | 0.7.x | No | Backend HTTP API | Very High |
| `tokio` | 1.x | Native only | Async runtime | Very High |

### Platform-Specific

| Crate | Platform | Purpose |
|-------|----------|---------|
| `web-sys` 0.3.x | WASM | Browser API bindings |
| `wasm-bindgen` 0.2.x | WASM | JS interop |
| `dirs` 5.x | Native | Platform directory paths |

### Evaluate Based on Tree Strategy Decision

| Crate | Version | Stars | When to Use |
|-------|---------|-------|-------------|
| `loro` | 1.10.6 | 5.4k | If choosing CRDT tree (Option B) |
| `yrs` + `yrs_tree` | yrs 0.21.x / yrs_tree 0.4.1 | yrs: large / yrs_tree: 2 | Alternative CRDT tree — evaluate maturity at decision time |
| `automerge` | 0.5.x | Large | If needing collaborative rich-text editing |
| `yrs` | 0.21.x | Large | If needing real-time text collaboration (Yjs protocol) |

### Domain-Specific (Genomics / Mass Spectrometry)

| Crate | Version | WASM | Purpose |
|-------|---------|------|---------|
| `noodles` | latest | Yes (pure Rust) | Genomic file parsing: BAM, CRAM, SAM, VCF, BCF, FASTQ, FASTA, BED, htsget client |
| `mzdata` | 0.63+ | Partial (mzML, MGF with `miniz_oxide` feature; no HDF5/vendor RAW) | Mass spec file parsing: mzML, MGF, mzMLb, Thermo RAW |
| `rust-bio` | latest | Yes (pure Rust) | Bioinformatics algorithms: alignment, FM-index, pattern matching |
| `needletail` | latest | Yes (pure Rust) | Fast FASTQ/FASTA parser |

### Consider But Not Required

| Crate | Purpose | When |
|-------|---------|------|
| `opendal` | Multi-backend storage | If you need S3/GCS/Azure interchangeably |
| `bao-tree` | Verified streaming | For large file integrity during transfer |
| `iroh` / `iroh-blobs` | P2P transfer (QUIC) | Device-to-device sync without server (not browser WASM) |
| `uhlc` | Hybrid logical clocks | For ordering events across devices |
| `object_store` | Arrow ecosystem storage | If already using Arrow/DataFusion (note: cloud backends not WASM) |

### Avoid

| Crate/Approach | Reason |
|---------------|--------|
| `cr-sqlite` as primary tree | LWW too simplistic for tree operations, uncertain maintenance |
| `vfs` crate | Sync-only, too simple |
| `virtual-fs` | Wasmer/WASI-specific |
| QUIC-only transport | Doesn't work in browser WASM |
| Fixed-size chunking | Poor deduplication on file edits |

---

## 11. Risks & Mitigations

### Technical Risks

| Risk | Severity | Likelihood | Mitigation |
|------|----------|-----------|------------|
| Loro is younger than Automerge/Yrs | Medium | Low | 1.0+ stable, 5.4k stars, 38+ contributors, 117 releases. Pin version. |
| `yrs_tree` is immature | Medium | Medium | Single author, 2 stars. Evaluate at decision time; Loro is safer today. |
| sqlite-wasm-rs limitations | Medium | Medium | Test early with real workload. sahpool is the mature path. |
| iOS/Android file provider requires native code | High | High | No Rust crate exists. Plan for Swift/Kotlin bridges early. |
| Browser storage quota eviction | Medium | Low | Call `navigator.storage.persist()`. Monitor with Storage Manager API. |
| CRDT memory growth | Medium | Medium | Loro shallow snapshots. Periodic re-snapshot. |
| Chunk GC correctness under partial failures | Medium | Medium | Two-phase commit (pending → committed). GC only uncommitted after timeout. |
| Multi-tab WASM SQLite conflicts | Medium | Medium | Cooperative locking in sahpool. Document single-tab limitation if needed. |

### Operational Risks (Codex contribution)

| Risk | Mitigation |
|------|-----------|
| Conflict UX clarity | Clear UI states: synced / local-only / conflicted |
| E2E encryption complexity | Start without E2EE; add later as a layer on top of CAS |
| AuthZ latency in hot paths | Evaluate OpenFGA; cache permission decisions aggressively |

---

## 12. Implementation Phases

### Phase 1: Core Library

1. Define `FileEntry`, `ChunkHash`, `FileStore` trait types
2. Implement BLAKE3 + FastCDC chunking pipeline
3. Implement Merkle root computation for sync detection
4. Unit tests with in-memory storage

### Phase 2: Native Adapter + Server

5. `NativeFs` adapter (rusqlite + std::fs)
2. Server adapter (sqlx Postgres + axum API)
3. Sync protocol (metadata first, then chunks)
4. Two-phase logical commit for uploads
5. Integration test: two native clients syncing via server

### Phase 3: WASM Adapter

10. `WasmFs` adapter (sqlite-wasm-rs sahpool + OPFS)
2. Web Worker architecture for SQLite
3. Test in Dioxus WASM build
4. Verify offline → online sync cycle in browser

### Phase 4: Tree Strategy Decision

14. Benchmark concurrent edit patterns against real usage expectations
2. If concurrent moves are frequent: integrate Loro LoroTree (or evaluate yrs_tree maturity)
3. If rare: stay with SQL parent_id + existing sync

### Phase 5: Platform Polish

17. iOS and Android testing via Dioxus
2. iOS File Provider bridge (Swift) — if needed
3. Android DocumentsProvider bridge (Kotlin) — if needed
4. Optional: WebDAV server as universal fallback

---

## 13. File-Type-Specific Storage Strategies

Different data types require different chunking, derived-data generation, and sync strategies. A comprehensive analysis is available in `claude/storage_strategies_by_file_type.md`. Key highlights:

### 13.1 Chunking by File Type

| File Type | Strategy | CDC Parameters | Rationale |
|-----------|----------|---------------|-----------|
| Images (JPEG, PNG, WebP) | CDC (standard) or inline | 16/64/256 KB | EXIF-only edits preserve most chunks |
| HEIC, RAW | CDC (large params) | 32/128/512 KB | Very large files, rarely edited in-place |
| Videos (MP4, WebM) | CDC (large params) | 64/256/1024 KB | Reduce manifest overhead for GB-scale files |
| Documents (PDF, DOCX) | CDC (standard) or inline | 16/64/256 KB | Standard approach |
| Text / Markdown / Code | Inline (almost always) | N/A | < 256 KB in vast majority of cases |
| Config (JSON, TOML) | Inline (always) | N/A | Tiny files, always under threshold |
| Archives (ZIP, TAR) | CDC (large params) | 64/256/1024 KB | Compression defeats fine-grained CDC |
| Datasets, ML models | CDC (large params) | 64/256/1024 KB | Very large, benefit from chunked transfer |
| Genomic (BAM/CRAM) | CDC (large params) | 64/256/1024 KB | Compressed binary; immutable pipeline outputs; no sub-file dedup |
| Genomic (FASTQ.gz) | CDC (large params) | 64/256/1024 KB | Compressed reads; write-once, sequential access |
| Genomic (VCF.gz) | CDC (standard) or inline | 16/64/256 KB | Variant calls; byte-range random access via tabix |
| Mass spec (mzML) | CDC (large params) | 64/256/1024 KB | XML + base64 binary; indexed random access; 4-18x vendor size |
| Mass spec (MGF) | Inline or CDC (standard) | 16/64/256 KB | Text peak lists; 1-200 MB; sequential access |
| Mass spec (mzMLb/HDF5) | CDC (large params) | 64/256/1024 KB | HDF5 binary; native cross-spectra chunking; near-vendor size |

**Note on scientific data and CAS dedup**: Compressed binary formats (BAM, CRAM, gzipped FASTQ/VCF, vendor RAW mass spec files) are hostile to sub-file content-defined chunking. Even a single-byte change in the uncompressed stream cascades through the compressed output, so FastCDC finds near-zero cross-file redundancy. **Whole-file dedup** (by content hash) is valuable for shared references, indexes, and annotation databases. For mass spec, the mspack tool demonstrates cross-scan dedup (76% lossless compression) but this operates within a single file, not at the storage layer.

### 13.2 Derived Data (Thumbnails, Previews, Text Extraction)

Store thumbnails and previews as **separate CAS blobs** referenced from file metadata. This enables:

- Sync thumbnails without full files (bandwidth optimization)
- Progressive loading: metadata → thumbnail → preview → full content
- "Files on Demand" model (inspired by Dropbox Smart Sync, iCloud)

| Derived Asset | Format | Size | Generation Location |
|---------------|--------|------|-------------------|
| Thumbnail (256px) | WebP | 5-30 KB | Server, native client, or WASM |
| Preview (1024px) | WebP | 30-150 KB | Server or native client |
| Video thumbnail | WebP (single frame) | 5-30 KB | **Server only** (requires ffmpeg) |
| PDF page renders | WebP | 30-100 KB/page | Server preferred (Pdfium WASM is 3 MB) |
| Extracted text | Plain text | 1-100 KB | All platforms |
| Genomic index (.bai/.crai) | Binary | 200 KB-10 MB | Server (from BAM/CRAM) |
| Variant summary | JSON/CSV | KB | Server (from VCF) |
| Mass spec feature table | CSV/TSV | KB-MB | Server (from mzML via XCMS/MZmine) |
| Spectral library match | MGF/MSP | MB | Server (GNPS/MassBank lookup) |

### 13.3 Sync Priority Tiers

| Priority | What Syncs | Typical Size |
|----------|-----------|-------------|
| **P0** | Tree metadata (Loro / SQL) | KB |
| **P1** | Config files (full content) | KB |
| **P2** | Thumbnails for visible folders | KB per file |
| **P3** | CRDT states for active documents | KB-MB |
| **P4** | Extracted text (for search) | KB per doc |
| **P5** | Preview blobs | KB-MB |
| **P6** | Full content for pinned files | Variable |
| **P7** | Full content for remaining files | Variable |

### 13.4 WASM Processing Limitations

Operations that **cannot** run in WASM (server-side only):

- HEIC/HEIF decoding (C library dependency)
- Camera RAW decoding (CPU-heavy)
- Video transcoding / frame extraction (requires FFmpeg)

Operations that **work in WASM**:

- JPEG/PNG/WebP thumbnail generation (`image` + `fast_image_resize`)
- EXIF extraction (`kamadak-exif`)
- PDF text extraction (`lopdf`)
- DOCX parsing (`docx-rs`)
- All BLAKE3 hashing and FastCDC chunking
- Genomic file parsing: `noodles` (pure Rust BAM/CRAM/VCF/FASTQ) — CRAM needs reference genome via HTTP
- Mass spec parsing: `mzdata` default features (mzML, MGF with `miniz_oxide` zlib) — no HDF5/mzMLb in WASM

See `claude/storage_strategies_by_file_type.md` for the full analysis with crate recommendations, code patterns, and bandwidth optimization strategies.

### 13.5 Domain-Specific File Systems and Storage Architectures

Scientific domains have developed their own storage layers that are relevant to our architecture. The patterns below inform how a transparent file system should handle genomic and metabolomic data.

#### Genomic Data Storage

| System | Type | Architecture | Relevance |
|--------|------|-------------|-----------|
| **HTSlib hfile plugins** | Library-level VFS (C) | Pluggable backends (`hfile_s3`, `hfile_gcs`, `hfile_libcurl`) for transparent remote BAM/CRAM/VCF access. Samtools opens `s3://`/`gs://`/`https://` URLs with random access | Most widely deployed genomic data access layer; our VFS trait should support similar URL-scheme routing |
| **htsget-rs** | GA4GH protocol server (Rust) | Serves BAM/CRAM/VCF slices by genomic region via HTTP. Uses noodles + Tokio | Rust reference for region-based random access serving |
| **Arvados Keep** | Content-addressed storage | MD5 block-level dedup + `arv-mount` (Python FUSE). Collections as POSIX filesystem | Closest architectural precedent to our CAS approach (though we use app-level VFS rather than FUSE) |
| **CernVM-FS (CVMFS)** | Read-only CAS + FUSE | Merkle trees, HTTP delivery, used by Galaxy for reference genomes and tool containers | Content-addressed + Merkle tree model similar to our design |
| **Seqera Fusion** | Workflow FUSE (proprietary) | POSIX over S3/GCS/Azure with lazy download, NVMe caching, non-blocking writes. Purpose-built for Nextflow | Demonstrates FUSE + object storage performance optimization for large scientific files |
| **Mountpoint for Amazon S3** | FUSE (Rust) | AWS S3 FUSE mount using `fuser` fork. Used by Genomics England | Production Rust FUSE reference for object-store-backed filesystems |
| **FUSTA** | FUSE (Rust) | Mounts multiFASTA as directory of individual sequences | Only Rust FUSE in bioinformatics; uses `fuser` crate |
| **iRODS** | Policy-based data mgmt | Data virtualization across heterogeneous storage. Multiple FUSE clients (Go, C++, NFSv4.1). Sanger Institute: tens of PB | Enterprise scientific data management with FUSE + metadata catalog |

**Key architectural insight**: All major genomic platforms (Terra/AnVIL, DNAnexus, Seven Bridges) store files as immutable objects in cloud storage (S3/GCS) and expose them via FUSE mounts or copy-based localization. No platform uses sub-file dedup for compressed genomic data. HTSlib's library-level VFS with pluggable backends is the most successful abstraction — analogous to our `FileStore` trait.

#### Mass Spectrometry / Metabolomics Storage

| System | Type | Architecture | Relevance |
|--------|------|-------------|-----------|
| **R/Bioconductor Spectra MsBackend** | VFS-like abstraction | Backend-agnostic API: `MsBackendMemory`, `MsBackendMzR` (on-disk), `MsBackendSql` (SQLite/DuckDB), `MsBackendMetaboLights` (remote). Swappable backends, same API | Closest thing to a VFS for mass spec; our `FileStore` trait mirrors this pattern |
| **ProteoWizard `pwiz::msdata`** | Vendor abstraction (C++) | Unified Reader over Thermo, Waters, Agilent, Bruker, SCIEX + open formats (mzML, mzXML, MGF) | Reference for multi-format transparent access |
| **mzDB** | SQLite-based storage | 3D indexing (RT, m/z, precursor). 25% smaller than XML, 2-2000x faster access | SQLite as MS data backend; aligns with our SQLite-everywhere approach |
| **mzsql** | Modern SQL/Parquet | Converts mzML to SQLite/DuckDB/Parquet. Queries in <1s on >1 GB files | DuckDB/Parquet for analytical queries over scientific data |
| **Tranche** (defunct) | Content-addressed distributed hash | Content-addressed uploads, multi-institution replication. Data migrated to MassIVE | Only CAS system ever built for mass spec; failed due to funding, not architecture |
| **PRIDE / FIRE** (EBI) | S3-compatible object store | 3 data centres, erasure coding, tape archive. 100+ PB across all EBI archives | Production-scale scientific object storage |
| **DIMSpec** (NIST) | Portable SQLite DB | WAL mode, FAIR principles, single-file shareable database for MS reference data | SQLite as portable scientific data container |
| **mspack** | Cross-scan compression | Exploits redundancy across scans via bucket transform + delta encoding. 76% lossless | Novel dedup approach within individual mass spec files |

**Key architectural insight**: The R `Spectra` MsBackend system demonstrates that a backend-agnostic storage API with swappable implementations (memory, file, SQL, remote) is the right pattern for scientific data — exactly our `FileStore` trait approach. mzDB and mzsql show that SQLite works well as a storage backend for large scientific datasets, validating our SQLite-everywhere strategy. Tranche's failure was operational (funding), not architectural — content-addressed storage for scientific data is sound.

#### Implications for Our File System

1. **Immutable file model**: Both genomic and metabolomic data are write-once pipeline outputs. Our CAS architecture (hash-identified, append-only) maps directly to this pattern without modification.
2. **Byte-range random access**: BAM/CRAM/VCF and indexed mzML all require efficient byte-range reads. Our chunk-based CAS must support range queries, not just whole-file retrieval. Consider aligning chunks to format-specific block boundaries (BGZF 64 KB blocks for BAM, indexed mzML byte offsets for mass spec).
3. **File-level dedup over chunk-level**: For compressed scientific data, dedup at file granularity (whole-file content hash) is more effective than sub-file CDC. Reference genomes, indexes, and annotation databases are duplicated across users/projects and benefit from CAS dedup.
4. **Metadata-heavy access patterns**: Scientific workflows often query metadata (which samples? which regions? which spectra?) before accessing bulk data. Our metadata-first sync approach (§9) maps well to this.
5. **Rust WASM ecosystem ready**: `noodles` (genomic) and `mzdata` (mass spec) are both pure Rust with WASM compatibility, meaning browser-based scientific data access is feasible through our VFS layer.

---

## 14. Spec Deliverables (Recommended Next Steps)

Since this is a review/spec phase, the recommended next deliverables are:

1. **Data model spec** — tables, indexes, invariants, migration strategy
2. **Operation protocol spec** — idempotency guarantees, ordering rules, replay safety
3. **Revision/conflict state machine** — pending → committed → synced → conflicted → resolved
4. **Chunking/integrity spec** — hash algorithm, manifest format, verification steps, GC rules
5. **Content-type processing spec** — per-type chunking params, derived data generation pipeline, WASM vs server processing split
6. **Authorization mapping spec** — OpenFGA object model + folder inheritance rules (if needed)
7. **Failure-mode test matrix** — offline edits, crash during pending commit, double upload, stale tombstone replay, concurrent moves

---

## 15. Sources

### Crate Documentation

- OpenDAL: <https://docs.rs/opendal/latest/opendal/> | Features: <https://docs.rs/crate/opendal/latest/features>
- object_store: <https://docs.rs/crate/object_store/latest>
- sqlite-wasm-rs: <https://docs.rs/sqlite-wasm-rs/latest/sqlite_wasm_rs/>
- sqlite-vfs: <https://docs.rs/sqlite-vfs/latest/sqlite_vfs/>
- sqlite_vfs: <https://docs.rs/sqlite_vfs/latest/sqlite_vfs/>
- Loro: <https://docs.rs/loro/latest/loro/> | <https://github.com/loro-dev/loro> (5.4k stars)
- yrs: <https://docs.rs/yrs/latest/yrs/> | <https://github.com/y-crdt/y-crdt>
- yrs_tree: <https://docs.rs/yrs_tree/latest/yrs_tree/> | <https://github.com/BinaryMuse/yrs_tree>
- Automerge: <https://docs.rs/automerge/latest/automerge/>
- BLAKE3: <https://github.com/BLAKE3-team/BLAKE3>
- FastCDC: <https://crates.io/crates/fastcdc>
- bao-tree: <https://crates.io/crates/bao-tree>

### Platform Documentation

- Dioxus mobile: <https://dioxuslabs.com/learn/0.7/getting_started/mobile/>
- SQLite WAL: <https://www.sqlite.org/wal.html>
- SQLite VFS: <https://www.sqlite.org/vfs.html>
- OPFS spec: <https://fs.spec.whatwg.org/>
- PostgreSQL 18.2: <https://www.postgresql.org/docs/release/18.2/>
- PostgreSQL 18.3 announcement: <https://www.postgresql.org/about/news/out-of-cycle-release-scheduled-for-february-26-2026-3241/>

### Scientific Data

- noodles (Rust genomics I/O): <https://github.com/zaeleus/noodles>
- mzdata (Rust mass spec I/O): <https://github.com/mobiusklein/mzdata>
- rust-bio: <https://rust-bio.github.io/>
- htsget-rs (Rust htsget server): <https://github.com/umccr/htsget-rs>
- HTSlib (C, reference implementation): <https://github.com/samtools/htslib>
- GA4GH htsget protocol: <https://www.ga4gh.org/product/htsget/>
- Arvados Keep (CAS for science): <https://arvados.org/>
- CVMFS (CAS + FUSE for science): <https://github.com/cvmfs/cvmfs>
- Seqera Fusion (workflow FUSE): <https://seqera.io/fusion/>
- FUSTA (Rust FUSE for FASTA): <https://github.com/delehef/fusta>
- iRODS: <https://irods.org/>
- R Spectra MsBackend: <https://rformassspectrometry.github.io/Spectra/>
- mzDB (SQLite mass spec): <https://pmc.ncbi.nlm.nih.gov/articles/PMC4349994/>
- mzsql (SQL/Parquet for MS): <https://github.com/wkumler/mzsql>
- mspack (cross-scan dedup): <https://github.com/fhanau/mspack>
- DIMSpec (NIST SQLite MS DB): <https://github.com/usnistgov/dimspec>
- MetaboLights: <https://www.ebi.ac.uk/metabolights/>
- GNPS/MassIVE: <https://gnps.ucsd.edu/>
- MS-Numpress (mzML compression): <https://pmc.ncbi.nlm.nih.gov/articles/PMC4047472/>
- mzML specification: <https://www.psidev.info/mzML>

### Reference Architectures

- Dolt prolly trees: <https://www.dolthub.com/blog/>
- Kleppmann tree move paper: "Highly-Available Move Operation for Replicated Trees" (2021)
- Evan Wallace tree CRDT: <https://madebyevan.com/algos/crdt-mutable-tree-hierarchy/>
- OpenFGA modeling: <https://openfga.dev/docs/modeling/getting-started>

### Detailed Research (In This Repository)

- `claude/review.md` — Claude's full individual review
- `claude/sync_patterns_research.txt` — Deep sync research (7 solutions, 6 case studies)
- `claude/fuse_research.md` — FUSE deep-dive
- `claude/claude_output.txt` — CAS + CRDT + Dioxus detailed research
- `codex/codex_output.txt` — Codex findings + verification updates
- `claude/storage_strategies_by_file_type.md` — File-type-specific storage strategies (images, video, docs, etc.)
- `cross_review.md` — Cross-review comparing both research outputs

---

*Joint review by Claude + Codex. February 20, 2026.*
*Crate versions verified via docs.rs, crates.io, and GitHub as of this date.*
