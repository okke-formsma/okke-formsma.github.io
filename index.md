# Jujutsu (jj) workspaces for Claude Code worktrees
## 2026-09-11

The Claude Code desktop app gives every session its own git worktree under `.claude/worktrees/`. That works fine with git, but my repos are colocated jj+git repos, and inside a plain git worktree jj is dangerous: there is no `.jj` in the worktree, so jj walks up, finds the `.jj` of the main checkout and happily treats that as the workspace root.

```
$ cd .claude/worktrees/some-session
$ jj workspace root
/Users/okke/projects/homelab        # <- the main checkout, not the worktree
```

Every `jj st`, `jj commit` or `jj new` that Claude runs in "its" worktree snapshots and rewrites my real working copy. So much for isolation.

### Not the fix: jj-worktree

[kawaz/jj-worktree](https://github.com/kawaz/jj-worktree) shims `git worktree add` into `jj workspace add`. It only works when you launch through `jj-worktree run claude`, so the desktop app (which spawns its own `claude`) never sees it, and it targets non-colocated repos where `git worktree add` fails outright. Not my problem.

### The fix: WorktreeCreate / WorktreeRemove hooks

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

### Things I learned along the way

* The branch is named after the worktree, without the usual `claude/` prefix. The desktop app records `branch = <worktree name>` for hook-made worktrees, so this keeps its bookkeeping honest. PR detection uses `git branch --show-current` in the session directory anyway.
* `.worktreeinclude` is skipped when a hook creates the worktree, so the hook copies `.env` and `.mcp.json` itself.
* The hook adds `.jj/` and `.claude/worktrees/` to `.git/info/exclude`, which is shared by all worktrees.
* The child workspace is only semi-colocated on jj 0.43: jj is fully isolated and correct, but git HEAD in the worktree does not move when jj commits, so the desktop diff pane shows jj commits as uncommitted changes. `jj workspace add --colocate` in the next jj release should fix that and shrink the create hook to two lines.
* When the repo has no `.jj`, the hook just makes a plain git worktree, so it is safe to have on for every repo.
* Removing via Claude's `ExitWorktree` asks for `discard_changes: true` because Claude Code cannot vouch for a worktree it did not create with git itself. The remove hook does `jj workspace forget`, `git worktree remove` and deletes the branch.

# Python3.6 on Macbook Pro M1
## 2021-12-10

Installing python 3.6 on an M1 machine is possible, but it's not trivial. The trick is to use a x86-64 version of python and install all prerequisites using a brew running under x86 too.

The following steps worked for me.

### Set up 'ibrew', or a x86 brew

Install the x86 homebrew version. All x86-64 packages are installed in /usr/local/, while 'normal' brew saves packages in /opt/homebrew/
* `arch -x86_64 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`
* `echo "alias ibrew=\"arch -x86_64 /usr/local/bin/brew\"" >> ~/.zshrc`

### Install anaconda x86-64

* `ibrew install anaconda`
* `/usr/local/anaconda3/bin/conda init zsh` (make sure not to use any arm conda version)

### Create python 3.6 x86-64 venv

* `conda create --name venv_py36 python=3.6`
* `ibrew install libpq`
* `export PKG_CONFIG_PATH="/usr/local/opt/libpq/lib/pkgconfig"`
* `export LDFLAGS="-L/usr/local/opt/libpq/lib"`
* `export CPPFLAGS="-I/usr/local/opt/libpq/include"`
* `pip install psycopg2==2.8.6 --force-reinstall --no-cache-dir`
* `ibrew install imagemagick freetype`
* `pip install python-magic`

# Blackmagic on a bluepill
## 2020-06-30

To flash nrfmicro chips, you can use a bluepill with blackmagic installed.

The bluepill has some issues with wrong resistors (I had to change R3 on the bluepill from 100k to 10k ohm). The blackpill with stm32f103 processor does not have this problem, and is recommended. (Black Magic is not supported on blackpill 1.2 (stm32f401) or blackpill 2.0 (stm32f411) at this time.

There's an excellent article on flashing blackmagic firmware onto a bluepill by joric. Here are some the steps I went through to make it work on ubuntu.

Connect the usb-to-uart converter to the bluemicro and set the bluemicro boot0 jumper to 1.

Find the usb port that the converter is attached to

```
> dmesg | grep tty
[157210.247340] cp210x ttyUSB0: cp210x converter now disconnected from ttyUSB0
[157221.140041] usb 1-2: cp210x converter now attached to ttyUSB0
```

Install the dfu. Run stm32loader.py on python2 with the pySerial package installed:
```
python2 stm32loader.py -p /dev/ttyUSB0 -ewv blackmagic_dfu.bin
```
Install the blackmagic firmware:
```
stm32flash -w blackmagic.bin /dev/ttyUSB0 -S 0x08002000
```
Set the jumper back to 0, disconnect the uart adapter, connect the bluepill to usb.
Check if the blackmagic probe is recognized with lsusb.


# Using a trackball as scrollwheel
## 2020-05-27
Tiny scrollwheel on mice have a few problems. They are small and the movement to scroll is prone to RSI. I've been using an Elecom HUGE trackball, and I love it. Except it doesn't have a nice scroll ring as the Kensingtons have. Today I figured out how to turn the entire trackball into a huge scrollwheel by configuring libinput.

To try it out, run a few xinput commands.

* `xinput list` and find the device id of your mouse. In my case it's 14.
* `xinput set-prop 14 "libinput Scroll Method Enabled" 0, 0, 1` to enable 'button scrolling'
* `xinput set-prop 14 "libinput Button Scrolling Button" 12` to use 'Fn3' on the Huge as the scroll trigger. If you want to use the 'middle mouse button', use 2 here.

If you like it, create the file /usr/share/X11/xorg.conf.d/60-huge.conf to make these settings persist between reboots.

```
Section "InputClass"
    Identifier  "ELECOM TrackBall Mouse HUGE TrackBall"
    Option  "ScrollMethod" "button"
    Option  "ScrollButton" "12"
EndSection
```

