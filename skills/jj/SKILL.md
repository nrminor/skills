---
name: jj
description: Use when working in a jj repository, coordinating agents with jj workspaces, changing commit graphs, managing bookmarks or remotes, or creating stacked PRs. Encodes the user's snapshot-first, curate-before-handoff workflow.
---

# Jujutsu (jj) Version Control

jj is a version control system built on a mutable directed acyclic graph of
commits. Treat commits as cheap snapshots while work evolves, then mold the
provisional graph into a small, comprehensible set of review units.

The graph may be messy during exploration. It must be curated before handoff:

```text
explore                       curate                         present
-------                       ------                         -------
snapshot uncertain work ->    abandon / split / squash  ->  describe
branch by dependency          rebase / reorder               verify each unit
avoid premature labels        preserve real topology         bookmark PR/RC heads
```

## Read the Graph First

Run all three commands before changing the graph.

```bash
jj status
jj log
jj workspace list
```

Establish:

- what is in the current working-copy commit (`@`)
- which commits this agent created for the current assignment and which came
  from the user, another agent, collaborators, or `jj git fetch`
- dependencies, parallel branches, bookmarks, remotes, conflicts (`×`), and
  divergent changes or bookmarks
- whether the apparent base is merged, released, or otherwise immutable

## Mutability Boundary: Clean Up After Yourself

Automatically rewrite only commits this agent produced for the current
assignment. Treat commits from the user, another agent, collaborators, fetched
remotes, merged history, and released history as immutable unless the user
explicitly delegates them.

Pre-release feature work that has not merged into main is normally mutable.
History merged into main is normally immutable. Some solo projects deliberately
work directly on main, so infer the actual convention from repository context
rather than treating a branch name as proof.

```text
current agent's exploratory work      freely mutable
current agent's pre-merge feature     mutable for curation
other-owned / fetched / merged        preserve
released                              preserve
```

If ownership or publication status is ambiguous, ask before surgery.

Inspect the complete effect of a rewrite, not only the revision named on the
command line. Abandoning, squashing, splitting, or rebasing an ancestor may
rewrite descendants; bookmarks may follow, move, or disappear; another
workspace may become stale. If the affected closure includes protected work,
an active workspace owned by someone else, or a presentation label outside the
assignment, coordinate or ask instead of proceeding automatically.

## Explore with Scratch Commits, in Moderation

Use new, usually undescribed commits to ring-fence revisions that may be kept,
reworked, or abandoned. They are especially valuable when:

- an approach is uncertain
- a coherent hunk may be discarded independently
- a risky change deserves a known-good checkpoint
- parallel agents need independent lines of work
- starting a hunk over would cost fewer tokens than manually undoing it

```bash
jj new                         # snapshot current @; continue in a new commit
jj new <parent>                # start an independent line at a chosen parent
jj abandon <revision>          # discard a failed scratch revision
```

Leave scratch commits undescribed by default. This keeps routine squashing
non-interactive because, when one side has an empty description, jj retains the
other description without asking how to combine them. Give scratch commits
short navigational descriptions only when several concurrent or parallel
revisions would otherwise be hard to distinguish.

This is not a mandate to commit after every action. Keep work in the current
commit when its changes belong to one review unit and will not need independent
discard or rollback.

Be unsentimental about failed scratch work. Inspect its descendants and
bookmarks as well as what the revision contains, then `jj abandon` it or revert
the operation that introduced it rather than spending tokens painstakingly
reversing an approach that is cheaper to redo. `jj abandon` rebases descendants
onto the abandoned revision's parents and deletes bookmarks pointing directly
to it; use `--restore-descendants` or `--retain-bookmarks` only when those are
the intended semantics.

## Let Dependencies Determine Topology

Branches are an emergent property of the commit graph, not named containers.
Make a dependent change a child of its prerequisite. Make independent changes
siblings of the same base unless repository convention requires a linear stack.

```text
A depends on B:              B -- A

A independent of B:         +-- A
                            |
                        base+
                            |
                            +-- B

integration:                A --+
                                +-- combined child
                            B --+
```

Do not linearize independent branches merely to resemble Git history. Respect
the project's review conventions, however: some teams expect a linear PR stack.

A commit with multiple parents is already a merge commit. Create one directly;
there is no need for a separate Git-style merge ceremony.

```bash
jj new <parent1> <parent2>
```

## Workspaces for Parallel Agents

Use jj workspaces when agents need independent working copies and must traverse
or rewrite their own commits without fighting over `@`. Workspaces are useful,
but each may duplicate disk-heavy state such as `node_modules`, virtual
environments, or Rust `target` directories.

**Ask before creating a workspace.** Name it for its purpose and pass its
explicit starting revision with `-r`.

```bash
jj workspace list
jj workspace add --name <purpose> -r <revision> <path>
```

Coordinate graph surgery across active workspaces. Before rewriting a shared
ancestor, tell each affected agent which change IDs will move. After the
rewrite, each affected agent must run `jj status` and `jj log`. If a workspace's
working-copy commit was rewritten or abandoned elsewhere and its files are now
stale, enter that workspace and run `jj workspace update-stale`. If jj reports
divergent changes, compare both diffs and agree which change to preserve before
abandoning either one.

When a workspace is no longer needed, ask before forgetting it or deleting its
working tree. The user may want to reuse the installed dependencies or the
workspace itself.

```bash
jj workspace forget <name>    # only after permission; does not delete files
```

## Curate Before Handoff

A normal agent session is not finished while its completed work remains a pile
of scratch commits. Before handoff, curate this agent's commits inside the
mutability boundary. Ask first when an operation would discard a meaningful
alternative, obscure authorship, cross that boundary, or change external state
that jj cannot restore.

Curate semantically, not by commit count:

1. Inspect every agent-owned revision, its descendants, active workspaces, and
   affected bookmarks.
2. Abandon failed or superseded experiments.
3. Use `jj squash`, `jj split`, and `jj rebase` to form conceptual review units.
4. Preserve parallel topology for truly independent work; linearize real
   dependencies.
5. Describe the resulting review units in the repository's existing style.
6. Run every repository-configured build, test, and lint command required for
   the changed code against each final unit, not only the stack tip.
7. Read the final graph and diff before reporting or presenting it.

```bash
jj abandon <revision>
jj squash -r <revision>        # move a revision into its parent
jj squash --from <src> --into <dst>
jj split -r <revision>
jj split -r <revision> --parallel
jj rebase -r <revision> -A <after>
jj rebase -r <revision> -B <before>
jj rebase -s <root> -o <destination>
```

Know the reach of each rebase form before using it: `-r` moves only the selected
revisions and reconnects their descendants; `-s` moves the selected roots and
all descendants; `-o` leaves the destination's existing descendants where they
are; `-A` inserts after a target and rebases its descendants onto the inserted
work; `-B` inserts before a target and rebases the target and its descendants.

`jj squash` and `jj rebase` are the primary sculpting tools. If curation
combines too much, recover through the operation log or use `jj split` to
restore the meaningful boundary.

### Final Commit Standard

Each final commit is a conceptual review unit rather than an arbitrary bundle
of files. Prefer smaller units when they stand alone, but keep coupled changes
together when splitting would produce misleading or broken intermediates. A
large commit can be correct when one feature or fix genuinely spans many parts
of the codebase; a grab bag of unrelated changes is not.

Unless repository policy conflicts, every final commit should:

- express one coherent feature, fix, refactor, or supporting concept
- build and pass the relevant tests and checks
- leave the repository in a useful state for bisection
- describe what changed and why

Review recent descriptions before writing new ones. Match the local style
instead of imposing a conventional-commits format or a fixed body length.

```bash
jj log -r '::@' --limit 10
jj describe -r <revision> -m '<description>'
```

Use a concise first line that scans well in `jj log`. Add a body when the
reasoning, tradeoffs, or review context would otherwise be lost. Do not require
every small commit to read like a blog post.

If the session ends with incomplete work that cannot honestly be curated into
bisectable review units, preserve it as a clearly labeled scratch stack and
report its status, risks, and next step. Do not manufacture tidy history that
misrepresents unfinished work.

## Bookmarks Are Presentation Labels

Commits are cheap and temporary; bookmarks are deliberate labels. Keep the
graph free to accumulate temporary commits, but do not litter it with half-baked
bookmarks.

Create or move a bookmark only for an established repository boundary, a
release candidate, or a commit intended as a PR head.
Bookmarks do not advance merely because a child commit is created, but they do
follow their target when jj rewrites that change. They may also be deleted when
their target is abandoned. Inspect bookmarks after graph surgery.

```bash
jj bookmark list -a
jj bookmark create <name> -r <revision>
jj bookmark move <name> --to <revision>
jj bookmark track <name>@<remote>
jj bookmark forget <name>              # local-only removal; untracks remotes
jj bookmark delete <name>              # schedule remote deletion on next push
```

Resolve divergent bookmarks (`??`) before pushing. Do not push without the
user's permission. Moving a bookmark backward or sideways requires
`--allow-backwards`; use it only after confirming the target.

### Release Candidates Stay at the Tip

A release-candidate commit or bookmark should sit at the tip of the branch of
changes it presents:

```text
base -- feature -- fix -- release candidate
```

If later fixes or features appear above a release-candidate commit, rebase them
below it while preserving their real dependencies, then leave the release
candidate at the tip. Keep all affected commits within the mutability boundary.
If the release candidate is only a bookmark rather than a distinct commit, move
the bookmark to the curated tip instead.

## The Operation Log Is the Safety Net

Every jj graph operation records an operation. When a rebase, squash, abandon,
workspace interaction, or other mutation behaves unexpectedly, stop and inspect
the operation log instead of piling more graph surgery on top.

```bash
jj op log
jj undo
jj --at-op=<operation-id> log
jj --at-op=<operation-id> status
jj op revert <operation-id>
jj op restore <operation-id>
```

The operation log is shared by the repository's workspaces. Use `jj undo` only
when the latest operation is the one this agent intends to reverse and no
concurrent agent operation would be caught with it. Otherwise identify the
agent-owned operation and prefer `jj op revert <operation-id>`. Inspect candidate
states with `--at-op`; reserve `jj op restore` for a deliberate repository-wide
rollback because it reverses all later operations, potentially including work
from other workspaces.

Restoring jj's local state does not reverse external effects. A Git push, merged
PR, deleted on-disk workspace, ignored file, installed dependency, generated
artifact, external service, or deployment remains external. After a push, use
the operation log to understand local state, then reconcile the remote
deliberately rather than assuming `jj undo` has un-pushed anything. If a
post-push restore is truly needed, `jj op restore <operation-id> --what repo`
avoids also restoring stale remote-tracking bookmarks.

## Rebasing and Remote Changes

Fetch before deciding whether agent-owned feature work needs a new base, but do
not treat fetched history as material for cleanup. Fetch is not always purely
additive: jj may abandon local commits that a remote rewrite or deletion leaves
unreachable. Inspect the resulting graph and operation log when a remote has
changed history.

```bash
jj git fetch
jj rebase -s <stack-root> -o main@origin
```

Before rebasing a stack onto an updated remote base, inspect active workspaces
and every descendant of each ancestor that the rebase will rewrite.

Aliases such as `jj tug` and `jj tush` are configuration-dependent. Inspect
`jj config list` before relying on their semantics, and treat any alias that
pushes like a push: ask first. Use jj rather than raw Git commands in a jj
repository; jj owns the Git-backed graph.

## Stacked PRs

Stacked PRs are a presentation of a dependency chain. Curate and verify the
commits first, then place a deliberate bookmark at each PR head.

```text
main@origin
  +-- base change          (bookmark, PR 1)
        +-- dependent API  (bookmark, PR 2)
              +-- UI       (bookmark, PR 3)
```

For review feedback in the middle of a mutable stack, create a scratch child of
the commit to fix, make and verify the correction, then squash it into the
target. This is authorized when updating that open stack is the assignment;
otherwise ask before rewriting remotely presented work. Inspect automatically
rebased descendants and bookmarks afterward.

```bash
jj new <commit-to-fix>
# make and verify the fix
jj squash                   # non-interactive if the scratch child is undescribed
jj log
```

After a base PR merges, fetch and rebase the remaining agent-owned stack onto
the new remote base. Use `jj bookmark forget` for local-only cleanup; use
`jj bookmark delete` only when remote deletion is intended. Either cleanup
requires appropriate user permission and repository convention.

## Conflicts and Divergence

Conflicts live in commits and are marked with `×` in `jj log`. Resolve them in a
scratch child, verify the result, and squash the resolution into the conflicted
commit when that commit is within the agent's mutability boundary.

```bash
jj new <conflicted-revision>
# resolve and verify
jj squash
```

Parallel workspaces and concurrent rewrites may create divergent changes. Do
not panic and do not guess which side to discard. Inspect the graph, provenance,
diffs, workspace ownership, and operation log; coordinate with other active
agents before choosing a resolution.

Resolve and verify conflicted commits before pushing. `--allow-conflicts` is an
explicit exception for unusual workflows, not a routine escape hatch.

## Useful Revsets and Inspection Commands

```bash
@                              # current workspace's working-copy commit
@-                             # parent of @
::@                            # ancestors of @
@::                            # descendants of @
main@origin..@                 # commits ahead of remote main
bookmarks()                    # bookmarked commits
description("fix")             # descriptions containing "fix"

jj log -r '::@' --limit 20
jj show <revision>
jj diff -r <revision>
jj workspace list
jj op log
```

The default `jj log` revset may hide commits already present on a remote. Use an
explicit ancestor or other targeted revset when a commit appears to have
disappeared.

## GitHub (`gh`)

Use `gh` alongside jj for PRs, checks, and issues. They are complementary: jj
shapes and publishes the commit graph; `gh` manages GitHub objects.

```bash
gh pr list
gh pr view <number>
gh pr checks <number>
gh pr create --base <base> --head <bookmark>
gh issue list
gh issue view <number>
```

Before creating a PR:

1. Curate the agent-owned commit graph.
2. Verify each review unit and inspect the final stack.
3. Put deliberate bookmarks at the intended PR heads.
4. Ask before pushing.
5. Use each commit's first description line as a title candidate. Write a PR
   body that states scope, rationale, test evidence, and stack dependencies.

For stacked PRs, use the parent PR's bookmark as the child PR's base. Never use
a scratch commit or incidental bookmark as a PR head.
