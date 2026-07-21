| | |
| --- | --- |
| Created | 2026-07-21 |
| Last updated | 2026-07-21 |
| Status | Draft |
| Reviewers | fmeum |
| Title | Artifact aliasing |
| Authors | Sahin Yort (Aspect) |

## Abstract

Bazel has no way to say "this content is also addressable here." Rulesets that
must present inputs at a canonical layout — rules_js co-locating sources under
`bazel-bin`, rules_python staging `site-packages`, rules_oci assembling layer
trees — express that need by *copying*, because copying is the only primitive
available. This proposal adds artifact aliasing: `ctx.actions.alias(output,
actual)` declares that `output` **is** `actual`'s content under a second exec
path. Both **file-to-file** and **tree-to-tree (directory)** aliases are in scope
and supported; `output` and `actual` must be the same kind (file↔file,
tree↔tree). It is not a copy and not a transformation; it is an identity relation
on *content*. How that identity is realized on disk (symlink, hardlink, copy, or
remote digest reuse) is deliberately left unspecified so the executor can pick
the cheapest correct laydown.

## Background

The recurring need across these rulesets is *re-addressing*: the same bytes must
appear at a different exec path so a tool sees a coherent tree. Today that is
spelled as a per-file copy action — a spawn that moves bytes and pays sandbox and
remote round-trips to produce a second artifact whose content is, by
construction, identical to the first.

**PR #396 ("Copy action")** attacks this as *a cheaper copy*:
`actions.copy(in, out, path)` with deferred materialization. That framing keeps
copy semantics — a new, independent artifact — and so inherits copy-shaped
questions its review has not resolved: whether to copy-as-is / dereference /
error on symlink inputs; whether the result may be empty; whether hardlinks or
copy-on-write are permitted.

This document argues the primitive is one level down. The thing rulesets want is
not a fast copy; it is an **alias** — a statement that two exec paths name the
same content. A copy is merely *one laydown* of an alias. Choosing the alias as
the primitive makes those open questions disappear rather than answering them:

- "Copy metadata from input to output" (tjgq's flagged gap) is not a capability
  to add — it is the definition of an alias. The output's metadata *is* the
  input's, including its digest.
- The symlink trichotomy dissolves: an alias to `X` names `X`. It is transparent;
  there is no target to dereference or refuse.
- "Empty `ActionResult`" is a non-question: an alias asserts metadata identity
  (a real, digest-bearing value), never emptiness.

Prior art: RBE providers already detect known copy actions and synthesize an
`ActionResult` to skip execution. That is an alias realized by digest reuse —
done today outside Bazel because Bazel lacks the concept.

## Proposal

### API

```python
def _impl(ctx):
    # File alias.
    out = ctx.actions.declare_file(ctx.file.src.basename, sibling = ctx.file.src)
    ctx.actions.alias(output = out, actual = ctx.file.src)   # output IS actual, re-addressed

    # Directory (tree) alias — the same relation over a whole tree.
    out_dir = ctx.actions.declare_directory("staged")
    ctx.actions.alias(output = out_dir, actual = ctx.file.src_dir)

    return [DefaultInfo(files = depset([out, out_dir]))]
```

`output` is an ordinary `DerivedArtifact` (a file) or `SpecialArtifact` (a tree)
and flows through providers into downstream (aliasing-unaware) actions unchanged.
One call per mapping. `output` and `actual` must be the same kind (file↔file,
tree↔tree).

### Observable semantics (the contract)

An alias is defined purely by what an action with `output` as an input can
observe. The laydown is *derived from* this contract, not the other way around.

**Guaranteed.** For any action that has `output` as an input:

1. **Content identity.** Reading `output` yields exactly `actual`'s content for
   that build: same bytes, same content digest, same size. For a **directory**,
   this holds per child: `output` has exactly `actual`'s set of children, each
   child identical in content, digest, and executable bit, with the same tree
   structure.
2. **Cannot diverge.** `output` is not a snapshot; if `actual`'s content changes,
   `output`'s does too, and actions with `output` as an input are invalidated. There is no
   window in which they differ. For a directory this includes children added,
   removed, or modified in `actual`.
3. **Executable bit.** `output` is executable iff `actual` is (per child for a
   tree).
4. **It is a regular file or a directory tree.** From the tool's perspective
   `output` opens and reads like any other input at its own exec path; a tree
   alias presents the same directory with the same children.

**Explicitly *not* guaranteed** (an action must not depend on any of these; they
are executor-internal and may vary by strategy, platform, filesystem, and build):

1. **On-disk representation** — symlink, hardlink, reflink/CoW copy, full copy,
   bind-mount, or remote digest materialization.
2. **`realpath(output)`** — may be `output` itself (content laydown) or resolve
   to `actual`'s location (symlink laydown). A tool that requires a stable
   realpath (e.g. Node.js) must be run under a content-materializing strategy;
   that is a **deployment/configuration** guarantee, not a property of the alias.
3. **inode / shared storage** with `actual`.
4. **Timestamps** (mtime/ctime/atime). A hardlink shares `actual`'s mtime; a
   fresh copy gets a new one — so timestamps are unspecified by construction.
5. **Mutation.** Nothing new: `output` is an input like any other, and Bazel's
   existing contract — actions do not modify their inputs — applies unchanged.
   (That same existing rule is what keeps a shared-inode hardlink laydown safe.)

This fence is load-bearing: the executor's freedom to choose a laydown is only
real if none of (1)–(5) is observable. If any leaked into the contract, Hyrum's
law would pin the implementation and the stated optionality would be illusory.
Timestamps and the on-disk form are the specific things that differ between a
hardlink and a copy, which is exactly why they are excluded here.

### Materialization (derived from the semantics)

Nothing in the contract mentions a mechanism, so each execution strategy is free
to realize `output` however it cheapest preserves content identity. Bazel itself
never copies bytes for an alias; `output` is an *indirection* — metadata carrying
`actual`'s digest and a pointer to `actual`'s terminal real location. Three
questions, kept separate:

**(1) How `output` is obtained from `actual` in the physical exec root.**
As an indirection — a symlink or hardlink at `output`'s exec path pointing at
`actual`'s terminal real location — laid down by whoever materializes it, never a
byte copy performed by Bazel. (A strategy whose only laydown is copying, e.g.
`docker`, will copy; that is the strategy's choice, not something the alias
requests.) On remote execution there is no local exec root for the action;
`output` is a digest entry in the Merkle tree.

For a **directory**, the indirection is carried on the tree-artifact root — the
whole tree resolves to `actual`'s tree — which the resolved-path machinery
already supports (`AbstractActionInputPrefetcher.maybeGetTreeRoot`). A strategy
may realize it as a single directory-level link or as per-child links/digests; on
remote execution the tree's children are digest entries in the Merkle tree. Tree
artifacts already expand per-child during input staging, so parts (2) and (3)
apply per-tree unchanged.

**(2) How `output` is staged when `actual` is also an input.**
Both appear at their own exec paths, each resolving to `actual`'s content. The
strategy lays each down independently (symlink/hardlink/digest); they simply
share a backing.

The strategy decides the indirection itself — symlink, hardlink, copy, or digest.
*And when it chooses a symlink, where does it point?* When `output` (`B`, or `C`)
is a symlink and `A` is also staged in the same sandbox, the target can be either
`A`'s **in-sandbox** staged path — a relative link that resolves *within* the
sandbox — or `A`'s **out-of-sandbox** exec-root path — an absolute link that
escapes it. That locality is likewise the strategy's decision; nominally it is
part of the unobservable on-disk form. In practice a tool can observe it through `realpath`,
and **rules_js does**: its `node_modules` resolution requires the link to resolve
**in-sandbox**, so the coherent tree Node walks is the sandbox's, not the exec
root's. So under a symlink laydown a strategy serving rules_js must point `B`/`C`
at the in-sandbox `A` (a relative link), which in turn requires `A` to be
co-staged. A **content** laydown (hardlink/copy, or a remote digest) sidesteps
the question: `B` is real content in the sandbox and `realpath(B) == B`. This is
the same tension as realpath-stability — link locality is a strategy/deployment
guarantee a ruleset can require, not a free arbitrary choice — and it is called
out in the open questions.

**(3) How `output` is staged when `actual` is *not* an input.**
Only `output` is declared as an input of the action. `actual` is still present in
the exec root — the alias's generating node depends on it — so the indirection
resolves, but `actual` is never staged into the action's environment. The action
sees only `output`. This is the independence property: an action needs only
`output`, and `actual` is fully independent of it.

**(4) How a chain of aliases is staged — `A → B → C`.**
`B` is an alias of `A` and `C` an alias of `B`. `A` holds the real bytes; `B` and
`C` are indirections. `C` resolves to `A`'s **terminal** real location, never to
`B`: its metadata stores the resolved path of the ultimate source, so a
copy-of-a-copy never points at an intermediate that may carry no bytes, and no
chain of links is ever traversed at stage time. `A`'s presence is guaranteed by
the dependency chain (`C`'s generating node depends on `B`, `B`'s on `A`),
independent of what the action declares.

- **Neither `A` nor `B` is an input (only `C`):** `C` is laid down from `A`'s
  terminal location; `A` is present via the chain; `B` need not be materialized
  at all.
- **Both `A` and `B` are also inputs:** `A`, `B`, and `C` are laid down
  independently at their own exec paths, all resolving to `A`'s content (`A`
  real, `B` and `C` indirections straight to `A`). No special handling — every
  alias in the chain already points at the terminal `A`.

This is the reason the metadata must store the *terminal* resolved path rather
than the immediate target.

**(5) How a tree alias's contents are staged — symlinks are already content.**
A tree alias never has to stage or relocate a symlink, because a tree artifact's
value model contains none. A tree is a content-addressed, relocatable unit: its
digest is a fingerprint over `(childPath, childContentDigest)` pairs, which is
what lets any strategy materialize it at the alias's exec path (or a remote
Merkle tree) from the digest alone. To preserve that,
`TreeArtifactValue.visitTree` — the canonical tree capture — *dereferences* every
on-disk symlink into the regular file or directory it points to (recursively)
and *errors* on a dangling or tree-escaping link; the visitor is never handed a
`SYMLINK` entry, so every child is a regular file with content and a digest.
Consequently a tree alias re-addresses a child set that is already pure content:
there is no symlink target to point in-sandbox vs. out-of-sandbox (the tension of
part (1)), and parts (2)–(3) apply per child unchanged. This is *why* tree-to-tree
aliasing is over regular files only — not a limitation of the alias, but a
property of tree artifacts that the alias inherits for free.

Illustrative (not normative) laydowns per strategy:

| Strategy | `output` realized as | Bytes moved by Bazel |
|---|---|---|
| Remote execution (RBE) | `actual`'s digest at `output`'s path in the Merkle tree | none |
| Sandbox — symlinked (default `linux-sandbox`, `process-wrapper`, `darwin`) | symlink → `actual` | none |
| Sandbox — hardlinked (`--experimental_use_hermetic_linux_sandbox`) | hardlink to `actual`'s content | none |
| Sandbox — copying (`docker`) | copy (strategy's only laydown) | one copy — strategy's choice |
| Sandbox — Windows (`windows-sandbox`) | read-only map at `output`'s path | none |
| Local, non-sandbox (`standalone`/`local`) | symlink to `actual` in the exec root; **or** a copy/reflink if a realpath-stable file is required (no sandbox to hardlink into) | none for the symlink; bytes for the copy fallback |

**Local execution has no sandbox directory** — this is the one strategy where an
alias may cost a copy. `output` is materialized directly in the *shared exec
root*, so the generating action's laydown there is either a **symlink** to
`actual` (cheap, content-identical, but a followable link — not realpath-stable),
or, when a realpath-stable real file is required, a **copy** (or reflink where the
filesystem supports it). Local has no sandbox to hardlink into, and Bazel does
not hardlink declared outputs in the output tree, so the cheap in-sandbox
hardlink other strategies use is unavailable. The no-bytes path therefore holds
on local only when a symlink is acceptable; otherwise it degrades to a copy.
Remote execution (digest reuse) and content-materializing sandboxes (hardlink)
keep the no-bytes path. Whether a symlink is acceptable on local, or Bazel should
gain a way to link declared outputs into the exec root, is left open below.

Laydown caveats (reasons the form must stay unobservable): a **hardlink** shares
`actual`'s mtime and inode and cannot be used for executables on macOS (Gatekeeper
kills hardlinked executables), so it is opt-in; **reflink/copy** are used where a
hardlink is unsafe or cross-filesystem; a **symlink** laydown is not
realpath-stable. None of these differences is observable per the contract above.

### Why this subsumes the copy action

`actions.copy` from #396 is the alias primitive pinned to a single laydown
(copy). If aliasing lands, "copy" is expressible as "alias with a copy laydown,"
and the directory-extraction `path=` case becomes an alias into a tree. The
alias is the smaller, more fundamental surface.

## Backward-compatibility

Additive. No change to `ctx.actions.symlink` or existing copy behavior; aliasing
is opt-in per call. It composes with `--experimental_output_paths=strip` (config
path rewriting applies on top of the aliased path). Downstream actions see an
ordinary derived artifact with a normal, shared digest.

## Open questions

1. **Naming.** `ctx.actions.alias` collides conceptually with the BUILD-level
   `alias()` rule (target aliasing). Alternatives: `ctx.actions.copy` (the
   implementation currently uses this), a laydown-selecting mode on
   `ctx.actions.symlink`, or a new verb. The proposal text and the implementation
   should converge on one name.
2. **Transitive aliases** (correctness requirement, specified in Materialization
   (4)): the metadata must store the *terminal* resolved path, not the immediate
   target, so a copy-of-a-copy never points at a byteless intermediate. Flagged
   because the current implementation stores the immediate target and trips the
   `ResolvedSymlinkArtifactValue` invariant — a bug to fix.
3. **Local execution laydown (unresolved).** Local has no sandbox directory, so
   `output` is materialized directly in the shared exec root. A symlink gives
   content-identity cheaply but is not realpath-stable; a realpath-stable file
   appears to require a copy (or reflink) into the exec root, because there is no
   sandbox to hardlink into and Bazel does not hardlink declared outputs in the
   output tree today. Open: is a symlink acceptable on local (accepting no
   realpath-stability there), or must local copy when an action requires a
   realpath-stable input?
   Should Bazel gain a way to hardlink/reflink declared outputs into the exec
   root? This is the one strategy where an alias may cost a copy.
4. **Directory laydown granularity.** Tree aliasing is in scope. Open: whether to
   realize a tree alias as a single directory-level link or as per-child
   links/copies — a laydown detail (unobservable), but with different cost and
   platform limits (a directory symlink vs. reflinking/hardlinking each child).
   Also the `path=` sub-case: aliasing a single file that lives inside a directory
   input to a file output. (Tree artifacts cannot contain symlinks, so a tree
   alias is always over a set of regular files.)
5. **Scale.** At 10k+ aliases, do per-mapping declarations add meaningful
   scheduling/BES overhead despite doing no work, or is a multi-output form
   eventually warranted?
6. **Workers and dynamic execution.** Behavior of aliased inputs under persistent
   workers and dynamic scheduling.
7. **Indirection and symlink-target locality are the strategy's decision.**
   Whether an alias is realized as a symlink, hardlink, copy, or digest — and,
   for a symlink, whether it targets the in-sandbox `A` (relative, resolves
   inside the sandbox) or the out-of-sandbox exec-root `A` (absolute) — is decided
   by the strategy, not the alias. rules_js requires in-sandbox resolution for
   `node_modules` under a symlink laydown. Open: should "in-sandbox / relative
   links" be a guarantee a ruleset can require of a strategy (and does requiring
   it force `A` to be co-staged), or is a content laydown the expected way to
   satisfy such tools?
