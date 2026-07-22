---
created: 2026-07-21
last updated: 2026-07-21
status: Draft
reviewers:
  - fmeum
title:  Sandbox Protocol
authors:
  - thesayyn
---

# Abstract

Bazel's sandbox mechanisms are compiled in. Adding one — gVisor, a
microVM, a FUSE overlay, a content-addressed store that materializes trees
without copying — means patching Bazel's Java, landing it upstream, and waiting
for a release. This proposal adds a **sandbox backend protocol**: a wire contract
(`src/main/protobuf/sandbox.proto`) that lets an *external binary* implement a
Bazel sandbox strategy. Bazel registers the binary with
`--sandbox_backend=<name>=<binary>`, selects it with `--strategy=<mnemonic>=<name>`,
and drives it over stdin/stdout with length-delimited protobuf. The action's
input tree is a standard REAPI Merkle tree — identical to what Bazel already
computes for remote execution — shipped once by digest and reused across actions
and builds. The backend owns *how* the tree becomes a filesystem view and *how*
the action is confined; Bazel owns *what* the tree is and *what* policy applies.
This is to sandboxing what REAPI is to execution: a stable protocol boundary that
moves the mechanism out of the Bazel binary.

# Background

Bazel ships a fixed set of sandbox strategies — `linux-sandbox`,
`darwin-sandbox`, `processwrapper-sandbox`, `docker`, `windows-sandbox` — each a
Java `SpawnRunner` that stages inputs into a host directory (a symlink tree, a
hardlink tree, or copies) and launches the action under an OS jail (Linux
namespaces, macOS Seatbelt). Two properties are baked into that design:

1. **The staging mechanism is Bazel's.** Every strategy materializes the input
   tree the same handful of ways. A fundamentally different laydown — a
   content-addressed FUSE mount, a VM disk image, a snapshotting overlay — cannot
   be expressed without a new in-tree `SpawnRunner` and a Bazel release.

2. **The confinement mechanism is Bazel's.** The jail is whatever Bazel was
   compiled to apply. A team wanting gVisor, bubblewrap, firejail, or a bespoke
   seccomp launcher has no seam to plug into short of forking Bazel.

Meanwhile Bazel *already* computes, for remote execution, exactly the artifact a
sandbox needs: a content-addressed Merkle tree of the action's inputs
(`MerkleTreeComputer`, shared with REAPI). Remote execution proved that "describe
the input tree by digest and let something else realize it" is the right
boundary. Local sandboxing never got that boundary; it re-derives the tree as
host-side symlinks every action, and the realization is hardcoded.

The consequence is that sandbox innovation happens *outside* Bazel and *around*
it — wrapper scripts, `--spawn_strategy` shims, custom `local` launchers — none of
which can see the input tree Bazel already knows. This proposal gives that work a
first-class seam.

# Proposal

## Registration and selection

```
bazel build //... \
  --sandbox_backend=vm=/opt/bin/my-sandboxd \
  --sandbox_backend_opt=vm=--pool-size=8 \
  --strategy=CppCompile=vm
```

`--sandbox_backend=<name>=<binary>` registers a backend as a spawn strategy named
`<name>`; `<binary>` is an absolute path or a bare name resolved against `PATH`.
It is selected like any strategy (`--strategy`, `--spawn_strategy`). The binary is
launched lazily, once per Bazel server, as `<binary> serve`. A backend reports
*available* iff its binary exists and is executable; otherwise it declines
per-spawn and Bazel falls through to the next strategy in `--spawn_strategy`, so a
missing backend degrades rather than fails the build.

`--sandbox_backend_opt=<name>=<opt>` passes one opaque token to the named
backend. Tokens are **not** process arguments — they are relayed out of band via
the `Negotiate` handshake (below), so a backend's configuration surface is part
of the protocol, not its command line.

## Roles

- **Bazel (client).** Computes the input Merkle tree, declares outputs and
  confinement policy, mints sandbox ids, and drives the request stream. Bazel
  decides *what* the tree is and *what* policy applies.
- **Backend (server / "the controller").** A long-lived binary in `serve` mode.
  It reconstructs the tree into a filesystem view however it likes, optionally
  brings its own jail, runs the action, and harvests outputs. It decides *how*.

The backend is expected to be a single long-lived process for the duration of a
Bazel server, so it may hold warm state (caches, warm VMs) across actions. The
protocol says nothing about how Bazel starts or stops it beyond `serve` mode and
the session-scoped store below.

## Transport

Varint length-delimited `Request`/`Response` protos over stdin/stdout
(`writeDelimitedTo` / `parseDelimitedFrom`). Each `Request` carries a
caller-assigned `rid`; a single reader thread drains stdout and matches each
`Response` back to the issuing Bazel thread by `rid`, so **responses may arrive
out of order** and many actions multiplex over the one pipe. The request pipe is
ordered: a `Push` written before a `Create` is guaranteed to reach the controller
first.

## The input tree: content-addressed, pushed out of band

The heart of the design is that **no host path is ever a durable reference**.
Everything is addressed by digest and immutable by digest, so reuse across
actions and builds is by digest alone.

- **`Push`** (fire-and-forget, no response) loads the controller's session store:
  `blobs` (serialized REAPI `Directory` messages, shipped verbatim — the hasher
  already produced the bytes) and `content` (leaf file bytes, either `inline` or a
  host `location` the controller reads *on receipt*). Both are content-addressed,
  so an entry is pushed once per controller lifetime; Bazel tracks what it has
  pushed to *this* controller instance and batches only novel entries — no
  discovery round-trip.
- **`Create`** ships a `Manifest`: the action mnemonic, the `exec_root`, the hash
  function, the `input_root_digest` (entry point into the pushed tree), the
  declared `outputs`, writable dirs, and the `ConfinementSetting`. The tree is
  referenced *only* by digest.
- **Default content location.** A leaf the store has no `content` entry for is
  read from `exec_root/<tree path>`, where Bazel has already staged it. `Push`
  covers exactly the leaves that live elsewhere — runfiles and fileset entries (at
  their target's exec path) and virtual inputs like param files (inline).

`Content` carries the load-bearing subtlety. A `location` is a **momentary
assertion**, true only at the instant the `Push` is sent — a generated path is
reused with different bytes after a rebuild; a source path is a symlink that may
be re-planted. So the controller **must capture** (snapshot / clone / copy into
its own store) *while processing the Push* and serve the digest only from that
captured copy forever after. It must never defer the read to `Create` time or
re-read the path later. This mirrors exactly the assumption Bazel already makes
for local execution (a source edited mid-build is not guarded by default).

## Recovery: `MissingContent`, never re-reading a path

A `Create` whose tree references a digest the store lacks — never pushed, or
evicted — comes back as `Create.Result.MissingContent` listing the missing digest
hashes. Bazel re-pushes exactly those, **from the current action's tree** (so a
re-pushed `location` is captured from a path valid for *this* build), and retries.
This is the single safety net for eviction or a stale client belief, and it is
kept a distinct outcome from `Error` precisely so the client retries rather than
failing the action. `Error` is always terminal.

A small attempt bound (3) applies: attempt 1 rides the proactive push, each
`MissingContent` drives one more retry after pushing what was named. More than a
single retry means content is being evicted between push and re-`Create`, which
Bazel fails rather than spins on.

## Confinement is Bazel's policy, the backend's mechanism

`Create.Result.Ok` states how the action is confined, and the split is
deliberate: **the backend never picks a Bazel mechanism.** It only states which of
three it is doing:

- **unset** → Bazel applies its platform-default built-in jail (Seatbelt on
  macOS, Linux namespaces via the `linux-sandbox` helper). The common case: a
  backend that just wants to be confined returns nothing and gets a jail for free.
- **`CustomConfinement`** → the backend brings its own jail, delivered as an argv
  `wrapper` (plus extra `env`) that Bazel prepends in the same outermost position
  it applies a built-in — `wrapper(process-wrapper(action))`. This lets a backend
  ship gVisor, bwrap, firejail, or a seccomp launcher **without a Bazel release**.
  The wrapper argv is built from `ConfinementSetting` (writable paths,
  inaccessible paths, network policy) plus the sandbox mount root the backend
  already minted. It MUST be non-empty — an empty wrapper is no jail, which is
  only expressible via `Unconfined`.
- **`Unconfined`** → the backend confines itself (its own VM/namespace) and Bazel
  applies nothing. A *present, empty* message, distinct from unset.

Absence is the safe default (confined). An unconfined run is always a present,
named choice and can never happen by omission.

## Operation lifecycle

| Op | Direction | Response? | Purpose |
|---|---|---|---|
| `Negotiate` | Bazel → backend | yes | One-time handshake: relay `--sandbox_backend_opt` tokens and agree a protocol `Version`. |
| `Push` | Bazel → backend | **no** | Load session store with blobs / leaf content by digest. |
| `Create` | Bazel → backend | yes | Build a sandbox from a `Manifest`; returns the sandbox root path + confinement, or `MissingContent`, or `Error`. |
| `Collect` | Bazel → backend | yes | After a successful action, move each declared output from its in-sandbox path to its destination under `exec_root`. |
| `Destroy` | Bazel → backend | yes | Tear the sandbox down. |

`sandbox_id` is minted by Bazel on `Create` and shared by `Collect` / `Destroy`;
`Push` and `Negotiate` carry none (the store is session-scoped).

## Outputs and path mapping

`Manifest.outputs` is keyed by the output's **in-sandbox** path (where the action
writes it); the `Output` value carries its `type` (`"dir"` vs `"file"` — a dir
output gets the directory itself materialized so `tar --directory` can chdir in)
and a `dest`. Under `--experimental_output_paths=strip`, the key holds the
*mapped* path (config segment stripped) while `dest` holds the *unmapped* exec
path Bazel expects — `Collect` bridges the two. Without path mapping the two
coincide and `dest` is empty.

The input tree is the standard REAPI Merkle tree, so a backend and remote
execution consume the exact same content-addressed inputs — the protocol
deliberately reuses REAPI's `Digest` and `Directory` types rather than inventing
its own.

# Backward-compatibility

Fully additive. No existing strategy, flag, or sandbox behavior changes. The new
surface is three flags (`--sandbox_backend`, `--sandbox_backend_opt`, and
strategy selection, which already exists) and one new proto. With no
`--sandbox_backend` registered, Bazel behaves exactly as today. A registered
backend whose binary is missing declines per-spawn and Bazel falls through the
`--spawn_strategy` list, so adding a backend cannot harden a build against a
misconfigured environment. The proto is versioned (`Version`, `VERSION_1` today)
and negotiated per handshake, so future protocol evolution is append-only and a
version mismatch surfaces as a clean `Negotiate` error rather than a wire crash.

# Open questions

1. **Store eviction policy and cross-build persistence.** `Retention`
   (`RETENTION_REUSE` vs `RETENTION_SINGLE_USE`) is a memory/eviction *hint*
   today, never a correctness signal (validity is the digest's alone). Left open:
   whether Bazel should communicate a memory budget, whether the controller's
   store should survive across Bazel servers on disk, and how eviction interacts
   with the `MissingContent` retry bound.

2. **Concurrency and backpressure.** Actions multiplex over one stdin/stdout pipe
   with out-of-order `rid` matching. Open: whether the protocol needs an explicit
   in-flight / flow-control signal, or whether Bazel's existing execution
   scheduling (`--jobs`, local resource estimation) is sufficient to bound the
   controller.

3. **Failure taxonomy.** `Error` is a single human-readable string, terminal for
   its op. Should some errors be retryable or carry a machine code (e.g. transient
   controller fault vs. malformed manifest), the way `MissingContent` is already
   split out as recoverable?

4. **Controller crash mid-action.** If the controller dies with sandboxes live,
   the next call respawns a fresh one with an empty store. In-flight actions fail
   and retry; open whether that is acceptable or whether sandbox state should be
   recoverable / handed off, given the "exit on its own to hand warm state to the
   successor" shutdown contract.

5. **`Collect` semantics for missing / partial outputs.** The contract says
   `Collect` moves each declared output; open how a declared-but-absent output is
   reported (does it surface as `Error`, or does Bazel's normal missing-output
   handling apply after a clean `Collect`?).

6. **Confinement expressiveness.** `ConfinementSetting` covers writable paths,
   inaccessible paths, and a network boolean. Is that enough for real jails
   (resource limits, syscall filters, user/uid mapping, tmpfs sizing), or does the
   setting need to grow — and if it grows, does it stay Bazel-policy or leak into
   backend-specific opts?

7. **Windows.** The confinement defaults name Seatbelt and Linux namespaces. What
   is the built-in default (the unset arm) on Windows, and does `windows-sandbox`
   fit this model or stay separate?

8. **Interaction with dynamic execution and persistent workers.** A backend
   strategy racing against a remote branch, and worker actions whose inputs are a
   sandbox tree, are both unspecified here.
