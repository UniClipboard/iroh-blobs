# UniClipboard fork patches for iroh-blobs 0.100.0

This branch (`uniclipboard/0.100.0-patched`) sits on top of upstream
[`n0-computer/iroh-blobs`](https://github.com/n0-computer/iroh-blobs)
tag `v0.100.0` and adds three patches. The
[`UniClipboard/UniClipboard`](https://github.com/UniClipboard/UniClipboard)
repository pulls this branch in as a git submodule under
`src-tauri/vendor/iroh-blobs/` and wires it via `[patch.crates-io]`.

Patches 1 and 2 address the same downstream symptom — process-fatal panics
at `bao_file.rs:410` "poisoned storage should not be used" — by closing
two independent sources of that `BaoFileStorage::Poisoned` state.

Patch 3 exposes a hook on `Downloader` that lets downstream callers abort
an in-flight download by tearing down the underlying QUIC connection.

## Patch 1 summary

`HashContext::persist()` in `src/store/fs.rs` unconditionally swaps the
`BaoFileStorage` for `BaoFileStorage::Poisoned` via `guard.take()`. When the
state is anything other than `Partial`, the let-else branch returns without
restoring the original value, leaving the handle permanently poisoned. A
later `observe(hash)` reaches `BaoFileStorageSubscriber::forward`, which
calls `BaoFileStorage::bitfield()`. `bitfield()`'s `Poisoned` arm panics
with `"poisoned storage should not be used"`, taking down the entity-manager
actor task and rendering the in-memory store unusable until process restart.

The fix matches on the variant *before* calling `take()`. Behaviour for
`Partial` states is unchanged.

## Affected upstream code

`src/store/fs.rs`, lines 990–1009 of the 0.100.0 release:

```rust
#[instrument(skip_all, fields(hash = %self.id.fmt_short()))]
async fn persist(&self) {
    self.state.send_if_modified(|guard| {
        let hash = &self.id;
        let BaoFileStorage::Partial(fs) = guard.take() else {
            return false;
        };
        let path = self.global.options.path.bitfield_path(hash);
        trace!("writing bitfield for hash {} to {}", hash, path.display());
        if let Err(cause) = fs.sync_all(&path) {
            error!(
                "failed to write bitfield for {} at {}: {:?}",
                hash,
                path.display(),
                cause
            );
        }
        false
    });
}
```

`BaoFileStorage::take()` (`src/store/fs/bao_file.rs:519`) is documented as
returning the previous value, but its implementation is
`mem::replace(self, BaoFileStorage::Poisoned)` — there is no "undo" if the
caller decides it didn't want to take the state after all.

## Trigger sequence we observed

`uniclipboard` is a clipboard-sync application. Receiver-side it uses
`iroh-blobs` to fetch file blobs published by the sender. Reproduction
sequence (UTC timestamps from the failing run):

1. `13:40:13` — fetch blob `8878b5704b` into the store; the in-memory
   `BaoFileHandle` reaches `BaoFileStorage::Complete(...)`.
2. `13:40:13` — export the blob with `ExportMode::TryReference`. The
   metadata DB transitions to `EntryState::Complete { data_location:
   DataLocation::External(...) }`; the in-memory state is still `Complete`.
3. Some time between `13:40:13` and `13:41:32` the entity manager evicts
   the handle (`ShutdownCause::Idle`) and invokes `persist()`. The handle's
   state is `Complete`, so the let-else returns early, but `guard.take()`
   has already swapped the state to `Poisoned`.
4. `13:41:32` — the user copies the same file again. The receiver re-enters
   the fetch path, calls `store.blobs().observe(hash)` from
   `ensure_blob_in_store`, and the subscriber dispatches `bitfield()` on
   the now-poisoned state. Panic:

```
thread 'iroh-blob-store-2' panicked at iroh-blobs-0.100.0/src/store/fs/bao_file.rs:410:17:
poisoned storage should not be used
```

The full backtrace traces `ObserveRequest` → `fs.rs:925 observe` →
`bao_file.rs:711 BaoFileStorageSubscriber::forward` → `bao_file.rs:410
BaoFileStorage::bitfield` (the `Poisoned` arm).

## Patch 2 summary

`HashContext::load()` in `src/store/fs.rs:286-302` (Action::Load arm)
calls `BaoFileStorage::open(state, self)` to materialise an in-memory
handle from the metadata-DB record. Upstream unconditionally maps any
IO error to `BaoFileStorage::Poisoned`:

```rust
match BaoFileStorage::open(state, self).await {
    Ok(handle) => handle,
    Err(_) => BaoFileStorage::Poisoned,
}
```

This conflates two very different failure modes:

1. **The metadata DB says `Complete{External(path)}` but the path is
   gone.** `BaoFileStorage::open` calls `std::fs::File::open(path)` for
   the External `DataLocation` (`bao_file.rs:596`), which returns
   `io::ErrorKind::NotFound`. The blob isn't physically there, but it
   is recoverable — a fresh fetch from the network produces a new
   `Complete` entry. Poisoning the handle here means the very next
   `observe(hash)` call lands on `BaoFileStorage::bitfield()`'s
   `Poisoned` arm and panics, which kills the `iroh-blob-store` worker
   task.
2. **A real on-disk fault** — permission denied, corrupted outboard,
   disk full. Here surfacing as Poisoned is the right call; we want
   loud failure rather than silent recovery.

uniclipboard's recovery story has this drift built in: clipboard sync
caches files under `file-cache/iroh-blobs/<entry_id>/` via
`ExportMode::TryReference`, and older releases plus user-initiated
cache pruning can remove the target file while the iroh-blobs
metadata DB still points at it. Once it's gone, every subsequent
`observe(hash)` against that blob panics.

The patch routes through a small free function `state_from_open_error`
that special-cases `NotFound` to `NonExisting` and leaves all other
kinds at `Poisoned`. Downstream code sees the blob as absent and goes
through the normal `download → ImportBao` path, which rewrites the
metadata entry to whatever the new fetch produces.

## Companion application-layer fix

The vendor patches close the *symptom* (panic on stale metadata). The
*cause* of the stale metadata — application code removing cache files
without telling iroh-blobs — is addressed in
`uc-application::clipboard_history::reconcile_missing_files`, which
runs at startup before any observe call and routes every entry whose
backing file is missing through the entry-aware delete path. Cleanup
and reconcile walk the cache↔DB drift in opposite directions and
together keep both sides consistent.

## Minimal spike

Two regression tests ship with this vendor copy:

1. **Code-level deterministic**: `src/store/fs/bao_file.rs::persist_regression`
   constructs a `BaoFileStorage::Complete(...)` state inside a
   `watch::channel`, runs (1) the patched and (2) the buggy upstream snippet,
   and asserts that only the buggy version leaves the state `Poisoned` and
   panics on the next `bitfield()` call. The buggy test is marked
   `#[should_panic(expected = "poisoned storage should not be used")]` so it
   passes today and starts failing once upstream merges an equivalent fix —
   that is the signal to retire this vendor copy.

2. **End-to-end smoke**: `src/store/fs::tests::try_reference_then_re_observe_smoke`
   exercises the full public `FsStore` API: 32 iterations of
   `add_bytes → observe → export_with_opts(TryReference) → sync_db → observe`
   on distinct hashes (so the entity-manager actor pool churns and previously
   completed hashes get a chance to go through `on_shutdown → persist`). Passes
   on the patched build.

### Honest note on a "fully deterministic" end-to-end spike

We tried but did not ship a deterministic public-API repro that *panics on
upstream* and *passes on the patch*, for two independent reasons:

* Upstream's buggy `persist()` is `send_if_modified(|guard| ... false)`.
  `tokio::sync::watch` only signals `changed()` when the closure returns
  `true`, so a subscriber parked on `changed()` never wakes up after the
  buggy `take()` swaps the state to `Poisoned`. A concurrent test that hopes
  to see the panic via the subscriber path stalls forever instead.

* Through `FsStore` the production race window is closed by the
  entity-manager: after `on_shutdown → persist` poisons the state, the main
  actor either calls `reset()` on the handle when the inbox is non-empty
  (re-loading from DB), or `recycle()` resets the state before returning the
  actor to the pool. So the only window where a poisoned handle is observable
  is between `take()` and the subsequent `reset()` — a few instructions long
  inside the manager actor. Reliably staging a test on that window from
  outside the manager would require new test hooks in the vendor crate.

The buggy snippet test (1) pins the actual broken semantics; the smoke test
(2) verifies the patched path is healthy under repeated use. A deterministic
end-to-end repro is left as a TODO for the upstream PR.

## Proposed upstream fix

```rust
#[instrument(skip_all, fields(hash = %self.id.fmt_short()))]
async fn persist(&self) {
    self.state.send_if_modified(|guard| {
        let hash = &self.id;
        // Only Partial states have anything to flush. Calling `take()` on
        // a non-Partial state leaves the handle Poisoned for the rest of
        // its lifetime, which later causes `bitfield()` to panic.
        if !matches!(&*guard, BaoFileStorage::Partial(_)) {
            return false;
        }
        let BaoFileStorage::Partial(fs) = guard.take() else {
            unreachable!("variant checked above");
        };
        let path = self.global.options.path.bitfield_path(hash);
        trace!("writing bitfield for hash {} to {}", hash, path.display());
        if let Err(cause) = fs.sync_all(&path) {
            error!(
                "failed to write bitfield for {} at {}: {:?}",
                hash,
                path.display(),
                cause
            );
        }
        false
    });
}
```

A more defensive variant would change `BaoFileStorage::take()` to only
swap when the current variant matches a predicate, but that has a broader
API impact and is left for upstream to decide.

## Patch 3 summary

`Downloader` in `src/api/downloader.rs` hands every `download(...)`
request off to an internal actor (`DownloaderActor`) that owns the
spawned task via a `JoinSet`. Caller-facing future cancellation does
**not** propagate: `handle_download` (line 101–106) wraps the inner
`tx.send(DownloadProgressItem::Error(...)).await.ok()` — a closed
receiver is silently swallowed and the task continues to run against
the network.

The only reliable way to abort an in-flight download from the outside
is to tear down the underlying QUIC connection held by the actor's
`ConnectionPool`. `ConnectionPool::close(id)` already exists
(`src/util/connection_pool.rs:454`) but is unreachable from outside
the actor.

This patch:

1. Refactors `Downloader::new_with_opts` to construct the
   `ConnectionPool` externally and share a clone with both the actor
   and the `Downloader` struct.
2. Adds a `Downloader::shutdown_endpoint(id)` method that forwards to
   `pool.close(id)`.

Downstream (uniclipboard `IrohBlobTransferAdapter`) calls this on
user-initiated cancel and on receiver-side timeout sweeps so the
actor's `execute_get` loop returns `Read(Reset)` / `ConnectionLost`
and unwinds the spawned task.

## Affected upstream code

`src/api/downloader.rs`. See the inline `// UniClipboard patch (P3)`
comments around `pub struct Downloader`, `DownloaderActor::new_with_pool`,
`Downloader::new_with_opts`, and `Downloader::shutdown_endpoint` for
the exact diff hunks.

## Proposed upstream fix

The natural upstream shape is identical to what this vendor copy
ships: expose the pool handle (or just `shutdown_endpoint`) on
`Downloader`. The change is additive — the existing `new` /
`new_with_opts` / `download` / `download_with_opts` signatures are
unchanged, so any existing call site keeps compiling.

Open question for upstream review: whether to also forward a generic
"cancel-by-request-id" rather than "cancel-by-endpoint-id", to support
fan-out scenarios where multiple downloads share the same provider.
For uniclipboard's one-blob-per-ticket model, endpoint granularity is
sufficient.

## Retiring this vendor copy

When upstream merges equivalent fixes:

- For patches 1 / 2: the `upstream_buggy_persist_poisons_complete_state`
  test in `src/store/fs/bao_file.rs` will begin to fail (the panic
  message it expects to see will no longer occur).
- For patch 3: a `Downloader::shutdown_endpoint` (or equivalent) lands
  in a release we're tracking.

When all three are addressed upstream, drop this `[patch.crates-io]`
entry and switch back to the published crate.
