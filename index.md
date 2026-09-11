## Jujutsu (jj) workspaces for Claude Code worktrees
### 2026-09-11

The Claude Code desktop app gives every session its own git worktree under `.claude/worktrees/`. That works fine with git, but my repos are colocated jj+git repos, and inside a plain git worktree jj is dangerous: there is no `.jj` in the worktree, so jj walks up, finds the `.jj` of the main checkout and happily treats that as the workspace root.

```
$ cd .claude/worktrees/some-session
$ jj workspace root
/Users/okke/projects/homelab        # <- the main checkout, not the worktree
```

Every `jj st`, `jj commit` or `jj new` that Claude runs in "its" worktree snapshots and rewrites my real working copy. So much for isolation.

#### Not the fix: jj-worktree

[kawaz/jj-worktree](https://github.com/kawaz/jj-worktree) shims `git worktree add` into `jj workspace add`. It only works when you launch through `jj-worktree run claude`, so the desktop app (which spawns its own `claude`) never sees it, and it targets non-colocated repos where `git worktree add` fails outright. Not my problem.

#### The fix: WorktreeCreate / WorktreeRemove hooks

Claude Code has `WorktreeCreate` and `WorktreeRemove` hooks that replace the built-in `git worktree` logic entirely. The docs only show them for SVN and friends, but they fire in git repos too, and the desktop app runs them as well (it even tells you to start a new session after you add one). The hook gets `{"name": ..., "cwd": ...}` on stdin and prints the directory to use.

My create hook makes a normal git worktree *and* a jj child workspace in the same directory. jj 0.43 refuses `jj workspace add` into a non-empty directory and does not have `--colocate` yet (that is in unreleased jj), so it creates the workspace in a sibling temp dir at the same depth and moves only its `.jj` over. The `.jj/repo` link is relative, hence the same-depth trick.

Put these in `~/.claude/hooks/`:

`jj-worktree-create.sh`
```bash
#!/usr/bin/env bash
# WorktreeCreate hook: git worktree + jj child workspace in one directory.
# Falls back to a plain git worktree when the repo is not a jj repo.
set -euo pipefail
IN=$(cat)
NAME=$(jq -r .name <<<"$IN")
CWD=$(jq -r .cwd <<<"$IN")
[[ "$NAME" =~ ^[A-Za-z0-9._-]+$ ]] || { echo "bad worktree name: $NAME" >&2; exit 1; }
GITDIR=$(git -C "$CWD" rev-parse --path-format=absolute --git-common-dir)
ROOT=$(dirname "$GITDIR")
WT="$ROOT/.claude/worktrees"
DIR="$WT/$NAME"
# Branch name == worktree name: the desktop app records branch=<name> for hook-made worktrees.
BRANCH="$NAME"
mkdir -p "$WT"
grep -qxF '.claude/worktrees/' "$GITDIR/info/exclude" 2>/dev/null || echo '.claude/worktrees/' >> "$GITDIR/info/exclude"
grep -qxF '.jj/' "$GITDIR/info/exclude" 2>/dev/null || echo '.jj/' >> "$GITDIR/info/exclude"

git -C "$ROOT" worktree add --quiet --no-track -b "$BRANCH" "$DIR" HEAD >&2

if [ -d "$ROOT/.jj" ]; then
  # jj 0.43 has no `workspace add --colocate`; create the workspace next to the
  # git worktree (same depth, so the relative .jj/repo link stays valid) and
  # move only its metadata in. Files are already checked out by git.
  TMP="$WT/.tmp-$NAME"
  rm -rf "$TMP"
  jj -R "$ROOT" workspace add --name "$NAME" -r "$(git -C "$ROOT" rev-parse HEAD)" "$TMP" >&2
  mv "$TMP/.jj" "$DIR/.jj"
  rm -rf "$TMP"
  jj -R "$DIR" status >/dev/null 2>&1 || { echo "jj workspace at $DIR is not healthy" >&2; exit 1; }
fi

# .worktreeinclude is not processed when a hook creates the worktree; copy gitignored config by hand.
for f in .env .mcp.json .claude/settings.local.json; do
  [ -f "$ROOT/$f" ] && { mkdir -p "$DIR/$(dirname "$f")"; cp "$ROOT/$f" "$DIR/$f"; }
done
echo "$DIR"
```

`jj-worktree-remove.sh`
```bash
#!/usr/bin/env bash
set -euo pipefail
IN=$(cat)
DIR=$(jq -r .worktree_path <<<"$IN")
[ -d "$DIR" ] || exit 0
GITDIR=$(git -C "$DIR" rev-parse --path-format=absolute --git-common-dir)
ROOT=$(dirname "$GITDIR")
NAME=$(basename "$DIR")
if [ -d "$DIR/.jj" ] && [ -d "$ROOT/.jj" ]; then
  jj -R "$ROOT" workspace forget "$NAME" >&2 || true
fi
git -C "$ROOT" worktree remove --force "$DIR" >&2
git -C "$ROOT" branch -D "$NAME" >&2 || true
```

And wire them up in `~/.claude/settings.json`. Use the *user* settings, not the repo's `.claude/settings.json`: the desktop app only auto-trusts hook-made worktrees when the hook comes from user settings, otherwise every new worktree shows a trust prompt.

```json
{
  "hooks": {
    "WorktreeCreate": [{"hooks": [{"type": "command", "command": "bash \"$HOME/.claude/hooks/jj-worktree-create.sh\""}]}],
    "WorktreeRemove": [{"hooks": [{"type": "command", "command": "bash \"$HOME/.claude/hooks/jj-worktree-remove.sh\""}]}]
  }
}
```

#### Things I learned along the way

* The branch is named after the worktree, without the usual `claude/` prefix. The desktop app records `branch = <worktree name>` for hook-made worktrees, so this keeps its bookkeeping honest. PR detection uses `git branch --show-current` in the session directory anyway.
* `.worktreeinclude` is skipped when a hook creates the worktree, so the hook copies `.env` and `.mcp.json` itself.
* The hook adds `.jj/` and `.claude/worktrees/` to `.git/info/exclude`, which is shared by all worktrees.
* The child workspace is only semi-colocated on jj 0.43: jj is fully isolated and correct, but git HEAD in the worktree does not move when jj commits, so the desktop diff pane shows jj commits as uncommitted changes. `jj workspace add --colocate` in the next jj release should fix that and shrink the create hook to two lines.
* When the repo has no `.jj`, the hook just makes a plain git worktree, so it is safe to have on for every repo.
* Removing via Claude's `ExitWorktree` asks for `discard_changes: true` because Claude Code cannot vouch for a worktree it did not create with git itself. The remove hook does `jj workspace forget`, `git worktree remove` and deletes the branch.

