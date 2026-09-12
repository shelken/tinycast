# Tinycast (fork)

Releases of [abue-ammar/tinycast](https://github.com/abue-ammar/tinycast) built here: upstream's
source, a small patch set, and an update feed that points at this repository.

Tinycast is a native macOS launcher: fuzzy app search, a global hotkey, per-app hotkeys, clipboard
history, an inline calculator, notes, snippets, custom commands, window management, and Raycast
extensions rendered in SwiftUI. The app and its documentation are upstream's:
[feature list](https://github.com/abue-ammar/tinycast#features), [docs/](docs/README.md).

## Install

Apple silicon, macOS 26 or newer:

```sh
brew install --cask shelken/tap/tinycast
```

Or take `Tinycast-<version>.dmg` from
[Releases](https://github.com/shelken/tinycast/releases). Every build is self-signed, so a manual
install needs the quarantine flag cleared once:

```sh
xattr -dr com.apple.quarantine "/Applications/Tinycast.app"
```

The bundle identifier is `com.tinycast.app`, the same as upstream's build, so this one replaces an
upstream install rather than sitting beside it.

## First run

1. **Settings → General** records the global shortcut that summons the palette.
2. Press it anywhere. Type to filter, `↵` launches, `Tab` switches between Apps and Clipboard, `Esc`
   dismisses.
3. **Settings → Shortcuts** binds a hotkey to an app or a custom command.

Pasting and snippet expansion need **Accessibility** (System Settings → Privacy & Security →
Accessibility). macOS prompts the first time a feature needs it.

## How this fork releases

`main` holds `patches/` and the release pipeline, and upstream is never merged into it.
[`.github/workflows/sync-upstream-release.yml`](.github/workflows/sync-upstream-release.yml) runs on
a schedule: it picks the oldest upstream stable release that has no release here yet, creates a
worktree at that tag, applies every patch in `patches/`, runs the gates, then builds and publishes.

- Gates: `./Scripts/run-tests.sh scopes-test`, `./Scripts/run-tests.sh updates-test`, lint, the model
  purity check, and a Debug build.
- The shipped app is an arm64 Release build signed with a self-signed identity.
- Each release carries `Tinycast-<version>.dmg`, `Tinycast-<version>.zip`, and
  `Tinycast-<version>-Source.tar.gz`, the last being the patched source with `website/` left out.
- The release tag points at the `main` revision whose `patches/` built it, not at upstream history.
  Release notes name both commits: the upstream one and that `main` revision.
- A failed run opens an issue named `[upstream-sync] <tag> failed`, and a green run closes it.

## Patches

`patches/` is what publishes. Each patch runs through `git apply` against the upstream tag in a clean
worktree, so it has to stay context-clean against upstream's copy of the file it touches.

| Patch | Touches | Why it exists |
| --- | --- | --- |
| `01-support-soft-link.patch` | `Tinycast/Features/Launcher/Model/SearchScopes.swift`, `Tests/scopes-test.swift` | Upstream's app scan walks real folders only, so apps that a dotfiles manager keeps behind symlinks in `~/Applications` never show up. The patch descends through symlinked folders and symlinked `.app` bundles while keeping logical paths, and covers that with tests. |
| `02-release-feed.patch` | `Tinycast/Features/Updates/Model/ReleaseFeed.swift` | Points `ReleaseFeed.repository` at `shelken/tinycast`, so the in-app update check reads this fork's releases. |

To rebuild a published version by hand:

```sh
git fetch https://github.com/abue-ammar/tinycast.git "refs/tags/vX.Y.Z:refs/remotes/upstream/tags/vX.Y.Z"
git checkout refs/remotes/upstream/tags/vX.Y.Z
git apply patches/*.patch
./Scripts/run-tests.sh scopes-test
./Scripts/run-tests.sh updates-test
```

The `-Source.tar.gz` asset of a release is that same tree if you would rather read it than rebuild it.

## Upstream and contributions

App bugs, feature requests and pull requests go to
[abue-ammar/tinycast](https://github.com/abue-ammar/tinycast), under its [CONTRIBUTING.md](CONTRIBUTING.md)
and its issue-first rule. A change that only makes sense for this fork belongs in `patches/` here.

## License

[AGPL-3.0](LICENSE), from upstream. Copyright (C) 2026 Abue Ammar.
