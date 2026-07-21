| | |
| --- | --- |
| Created | 2026-07-21 |
| Last updated | 2026-07-21 |
| Status | Draft |
| Reviewers |  |
| Title | Artifact aliasing |
| Authors | Sahin Yort (Aspect) |

## Abstract

Bazel has no way to say "this content is also addressable here." Rulesets that
must present inputs at a canonical layout — rules_js co-locating sources under
`bazel-bin`, rules_python staging `site-packages`, rules_oci assembling layer
trees — express that need by *copying*, because copying is the only primitive
available. This proposal adds artifact aliasing: `ctx.actions.alias(output,
actual)` declares that `output` **is** `actual`'s content under a second exec
path. It is not a copy and not a transformation; it is an identity relation.
How that identity is realized on disk (symlink, hardlink, copy, or remote digest
reuse) is a laydown detail the executor chooses, not part of the semantics.

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

Prior art from that thread: RBE providers already detect known copy actions and
synthesize an `ActionResult` to skip execution. That is an alias realized by
digest reuse — done today outside Bazel because Bazel lacks the concept.

## Proposal

### API

```python
def _impl(ctx):
    out = ctx.actions.declare_file(ctx.file.src.basename, sibling = ctx.file.src)
    ctx.actions.alias(output = out, actual = ctx.file.src)   # output IS actual, re-addressed
    return [DefaultInfo(files = depset([out]))]
```

`output` is an ordinary `DerivedArtifact` and flows through providers into
downstream (aliasing-unaware) actions unchanged. One call per mapping.

### Semantics: identity, not equality

`output` and `actual` name the same content. This is an identity relation, with
three consequences that distinguish it from a copy:

- **They cannot diverge.** There is no moment at which `output` is "produced"
  from a snapshot of `actual`; it is `actual`, re-addressed.
- **The digest is shared, not coincidental.** `output`'s metadata delegates to
  `actual`'s (`FileArtifactValue.getDigest()` already delegates for
  resolved-symlink values today). Change tracking rides the existing dependency
  edge on `actual`; no bytes are ever hashed or moved for the alias.
- **Aliasing is transitive.** An alias to an alias names the ultimate content.

The action is a *declaration* recorded as a Skyframe node (giving invalidation
and `aquery` visibility), not a spawn. It moves no bytes and produces a real,
digest-bearing output value — not an empty result.

### Laydown is the executor's choice, not the semantics

Because an alias is an identity relation, no materialization mechanism is baked
into it. Each strategy realizes it however it best presents the underlying
content:

- **Remote execution**: the Merkle tree places `actual`'s digest at `output`'s
  exec path. The worker writes a real file; zero extra bytes. (This already
  happens for `SymlinkAction` outputs.)
- **Local / sandbox**: the strategy may symlink, hardlink, reflink/`clonefile`,
  bind-mount, or copy. `SymlinkAction` today always plants a *followable
  symlink*, and the sandbox strategies have no handling of resolved paths at all
  — which is why aliased content escapes the tree under Node's `realpath` and
  rulesets fall back to copies. Aliasing makes a realpath-stable laydown
  (content, not a followed symlink) available and cheap.

Which laydown a strategy uses, and whether a given tool tolerates it, stays a
configuration concern — Bazel makes correct-and-cheap laydowns *possible*, it
does not promise any tool is realpath-safe. Practical constraints noted in the
#396 thread carry over: default to copy or copy-on-write; hardlinks opt-in;
never hardlink executables (macOS Gatekeeper).

### Why this subsumes the copy action

`actions.copy` from #396 is the alias primitive pinned to a single laydown
(copy). If aliasing lands, "copy" is expressible as "alias with a copy laydown,"
and the directory-extraction `path=` case becomes an alias into a tree. The
alias is the smaller, more fundamental surface.

## Backward-compatibility

Additive. No change to `ctx.actions.symlink` or existing copy behavior; aliasing
is opt-in per call. It composes with `--experimental_output_paths=strip` (config
path rewriting applies on top of the aliased path). Downstream consumers see an
ordinary derived artifact with a normal, shared digest.

## Open questions

1. **Naming.** `ctx.actions.alias` collides conceptually with the BUILD-level
   `alias()` rule (target aliasing). Alternatives: a laydown-selecting mode on
   `ctx.actions.symlink`, or a new verb.
2. **Laydown policy surface.** Default laydown (copy vs CoW), and whether it is
   selected per-action (execution requirement) or per sandbox strategy;
   executable outputs need Gatekeeper-safe handling.
3. **Scale.** At 10k+ aliases, do per-mapping declarations add meaningful
   scheduling/BES overhead despite doing no work, or is a multi-output form
   eventually warranted?
4. **Directories / tree artifacts.** File-to-file first; aliasing a directory
   (and the `path=` extraction case) deferred.
5. **Relationship to #396.** Land aliasing as the primitive and express copy on
   top, or ship them independently?
6. **Workers and dynamic execution.** Behavior of aliased inputs under
   persistent workers and dynamic scheduling.
