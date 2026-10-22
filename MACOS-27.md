# yabai on macOS 27 betas (personal fork notes)

Personal notes for running this fork (`sleao/yabai`, branch `macos-27`) on macOS 27
betas, where the official yabai's scripting addition doesn't work. Written after getting
it working on macOS 27.2 beta `26B5112a` on an M5 Pro (2026-10-15).

## Why this fork exists

To switch spaces (`yabai -m space --focus`), yabai injects a scripting addition (SA) into
the Dock. On macOS 27 the official SA:

- rejects major version 27 in `verify_os_version`, so it loads into the Dock but ignores
  every command. `space --focus` still **exits 0** and doesn't switch, which looks like
  a keybind problem but isn't.
- has byte patterns that no longer match the 27.2 Dock. The Dock is now a fat binary with
  `arm64e` and `arm64e.x1` slices. The x1 slice is used on newer chips and starts
  functions with `pacibsppc` instead of `pacibsp`.
- calls `-[Spaces currentSpaceForDisplayUUID:]`, which was replaced by
  `currentSpaceForDisplay:`.

## Where the patch comes from

[asmvik/yabai#2832](https://github.com/asmvik/yabai/issues/2832) by LCS-Dev. The upstream
repo has pull requests disabled, so the patch lives on a fork:

```sh
curl -L https://github.com/LCS-Dev-Ergos/yabai/compare/master...fix/macos-27-scripting-addition.patch | git am
```

Two commits on top of upstream `dd84572`:

1. `feat(osax)`: macOS 27 patterns, offsets and version gates. Bumps `OSAX_VERSION` to
   `2.1.31`.
2. `fix(osax)`: NULL guards in the space handlers (prevents Dock crashes).

**Supported on 27:** focus, create and destroy spaces, and disabling the space-switch
animation.
**Not supported:** moving a space to another display (`space --display`), because
`DPDesktopPictureManager` is gone.

## Remotes

```
origin    git@github.com:sleao/yabai.git    (use SSH; HTTPS push has no credentials)
upstream  https://github.com/asmvik/yabai.git
```

Commits are SSH-signed. If `git am` or `git commit` fails with
`failed to write commit object`, run `ssh-add ~/.ssh/id_ed25519` first.

## Full procedure

### 1. Check the patterns still match the current Dock (optional, before building)

Each new beta can change the Dock binary. The patterns in `src/osax/arm64_payload.m` (the
`majorVersion == 27` branches) must match **exactly once** in both slices, at or after
their scan offset. The scanner searches `0x1286a0` bytes from the offset.

```sh
lipo -archs /System/Library/CoreServices/Dock.app/Contents/MacOS/Dock   # arm64e arm64e.x1
lipo /System/Library/CoreServices/Dock.app/Contents/MacOS/Dock -thin arm64e.x1 -output /tmp/dock_x1
# then search /tmp/dock_x1 for each pattern (a small Python regex scanner works fine)
```

On `26B5112a` all four patterns (`dock.spaces`, `add_space`, `remove_space`,
`animation_time`) matched once in both slices.

### 2. Build

```sh
cd ~/code/sleao/yabai
make install                 # builds ./bin/yabai (does NOT install system-wide)
./bin/yabai --version        # stays v7.1.25; only the SA version changes (2.1.31)
```

### 3. Re-sign the binary (required)

The build only carries a linker signature (`flags=0x20002(adhoc,linker-signed)`). macOS
Accessibility (TCC) never accepts it: System Settings stores cdhashes that don't match,
so yabai asks for permission again on every restart and aborts with
`could not access accessibility features! abort..`.

```sh
codesign -fs - -i com.asmvik.yabai bin/yabai
codesign -dvvv bin/yabai 2>&1 | /usr/bin/grep -E 'Identifier|flags'
# expect: Identifier=com.asmvik.yabai  and  flags=0x2(adhoc)
```

(In the interactive shell, `grep` is aliased to ripgrep and doesn't take `-E`. Use
`/usr/bin/grep`.)

### 4. Install

The launchd plist (`~/Library/LaunchAgents/com.asmvik.yabai.plist`) and the sudoers rule
both point at `/opt/homebrew/bin/yabai`. That path is a plain file, not managed by
Homebrew.

```sh
yabai --stop-service
rm /opt/homebrew/bin/yabai                  # rm first; don't cp over the old file
cp bin/yabai /opt/homebrew/bin/yabai
```

`cp` over an existing binary keeps a stale cached code signature on that file. Remove it
first.

### 5. Update sudoers (in a real terminal; `sudo` needs a TTY for the password)

The rule pins the binary's sha256, so it must be redone after every rebuild or re-sign:

```sh
echo "$(whoami) ALL=(root) NOPASSWD: sha256:$(shasum -a 256 $(which yabai) | cut -d' ' -f1) $(which yabai) --load-sa" | sudo tee /private/etc/sudoers.d/yabai
sudo visudo -cf /private/etc/sudoers.d/yabai   # must say "parsed OK"
```

### 6. Re-grant Accessibility

System Settings → Privacy & Security → Accessibility: **remove** yabai with `–`, then
`+` → `Cmd+Shift+G` → `/opt/homebrew/bin/yabai` → toggle it on. Flipping the existing
toggle isn't enough.

`tccutil reset Accessibility yabai` does **not** work: tccutil only accepts bundle ids.

### 7. Reinstall the scripting addition (in a real terminal)

`--load-sa` only reinstalls the SA when the installed version differs from the built one.
If an older build used the same `OSAX_VERSION`, the stale SA stays in place. (The
previous AhsanFazal fork also used `2.1.31`.) Force it:

```sh
sudo yabai --uninstall-sa        # or: sudo rm -rf /Library/ScriptingAdditions/yabai.osax
killall Dock                     # Dock drops the old payload and restarts
yabai --start-service
sudo yabai --load-sa
yabai -m space --focus 2
```

## Verifying

```sh
pgrep -fl yabai                                  # yabai is running
tail -3 /tmp/yabai_$USER.err.log                 # no new accessibility errors
# Dock has the payload loaded, and it's the new one (check the date):
lsof -p $(pgrep -x Dock) | /usr/bin/grep payload
stat -f '%Sm' /Library/ScriptingAdditions/yabai.osax/Contents/Resources/payload.bundle/Contents/MacOS/payload
yabai -m query --spaces --space | /usr/bin/grep '"index"'
```

Check what TCC actually stored. Reading this needs Full Disk Access for the terminal:

```sh
sqlite3 -readonly "/Library/Application Support/com.apple.TCC/TCC.db" \
  "select client,auth_value,datetime(last_modified,'unixepoch','localtime') from access where service='kTCCServiceAccessibility' and client like '%yabai%';"
```

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `space --focus` exits 0 but nothing happens | SA rejects the OS, or a stale SA is loaded | Steps 1 and 7; check the payload date |
| `could not access accessibility features! abort..` and a permission prompt on every restart | Linker-only signature, or a TCC entry for the old binary | Steps 3, 4 and 6 |
| `yabai-msg: failed to connect to socket` | yabai crashed at startup (usually TCC) | `tail /tmp/yabai_$USER.err.log` |
| `sudo: a terminal is required to read the password` | `sudo` was run from a non-interactive shell | Run it in a real terminal tab |
| `sudo yabai --load-sa` asks for a password | sudoers sha256 is out of date | Step 5 |
| `failed to write commit object` | SSH signing key isn't loaded | `ssh-add ~/.ssh/id_ed25519` |

## Updating from upstream

```sh
git fetch upstream
git rebase upstream/master          # on macos-27
git push --force-with-lease
```

Then repeat steps 2–7. Once an official yabai release supports macOS 27, switch back to
Homebrew (`brew install asmvik/formulae/yabai`) and drop this fork.
