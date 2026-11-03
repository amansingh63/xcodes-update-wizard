# xcodes-update-wizard

An interactive, step-by-step Xcode updater for macOS, built on
[`xcodes`](https://github.com/XcodesOrg/xcodes). It finds the newest Xcode,
installs it, makes it the system-wide default, offers to remove old versions,
and sets up simulator runtimes. **You choose what happens at every step**:
nothing is installed, switched or deleted until you say yes.

> This is an independent project, not part of [XcodesOrg](https://github.com/XcodesOrg).
> It's a wrapper around their `xcodes` command-line tool.

```
━━ Step 7/9 · Remove old Xcode versions ━━
• Other installed versions (the active and newly installed Xcodes are never offered):
    27.0 Release Candidate (27A266a)       11G  /Applications/Xcode-27.0.0-Release.Candidate.app
    26.4 (17E258)                          10G  /Applications/Xcode-26.4.0.app

What should happen to them?
   1) Uninstall all of them
   2) Ask me about each one
   3) Keep them all — just use the new version
Choose 1-3 [3]:
```

## Quick start

```bash
git clone https://github.com/amansingh63/xcodes-update-wizard.git
cd xcodes-update-wizard
./xcode-update                # main menu
./xcode-update update         # straight into the update wizard
./xcode-update --dry-run      # walk through everything; only print commands
```

To run it from anywhere, link it into a directory on your `PATH`, such as
`~/.local/bin` or `/usr/local/bin`. Run this from inside the cloned folder:

```bash
ln -s "$PWD/xcode-update" ~/.local/bin/xcode-update
```

**Requirements:** macOS and Homebrew. If `xcodes` is missing, the script offers
to install it. `aria2` is optional, but downloads are 3-5x faster with it; the
script offers to install that too.

## Commands

| Command | What it does |
|---|---|
| *(none)* | Main menu with every action below |
| `update` | Full 9-step update wizard |
| `switch` | Choose which installed Xcode is the system-wide default |
| `cleanup` | Uninstall old Xcode versions (all, one by one, or none) |
| `runtimes` | Download or delete simulator runtimes |
| `status` | Installed Xcodes, the active one, and the `Xcode.app` symlink |
| `signout` | Forget the Apple ID that xcodes saved in the Keychain |

| Option | |
|---|---|
| `-n`, `--dry-run` | Print each command that would change something, without running it. Read-only lookups still run, so you see real data. |
| `-h`, `--help` | Usage |
| `-V`, `--version` | Script version |

## The update wizard

| # | Step | What it asks you | Command it runs |
|---|---|---|---|
| 1 | Preflight | Upgrade xcodes? Install aria2? Continue if disk space is low? | `brew upgrade xcodes`, `brew install aria2` |
| 2 | Current state | — (read-only) | `xcodes update`, `xcodes installed` |
| 3 | Choose version | Latest release / latest beta (only if newer) / pick from list / type a version / install nothing | — |
| 4 | Install | Fast unxip? Delete the `.xip` or move it to the Trash? Start the download? | `xcodes install <ver> --experimental-unxip --empty-trash` |
| 5 | Make default | Repoint the `/Applications/Xcode.app` symlink, select the app directly, or keep the current one | `sudo ln -sfn …` + `sudo xcode-select --switch …` |
| 6 | First launch | Accept the license and install components? *(skipped automatically if already done)* | `sudo …/xcodebuild -runFirstLaunch` |
| 7 | Remove old versions | Uninstall all / one by one / keep all. Move to the Trash or delete permanently? | `xcodes uninstall <ver> [--empty-trash]` |
| 8 | Simulator runtimes | Download iOS / another platform / a specific version, or prune outdated runtimes | `xcodebuild -downloadPlatform iOS`, `xcodes runtimes install`, `simctl runtime delete --outdated` |
| 9 | Summary | — | `xcode-select -p`, `xcodebuild -version` |

### Safety rails

- **The active Xcode and the one just installed are never offered for removal.**
- Uninstalled Xcodes go to the **Trash** by default, so you can restore them until you empty it.
- `--select` is not passed to `xcodes install`; step 5 decides how the new Xcode becomes the default.
- If you remove the version the `Xcode.app` symlink points to, the script notices and offers to repoint the symlink or remove it.
- A real `Xcode.app` folder (for example, one from the Mac App Store) is never replaced by a symlink.
- Real runs need a terminal, because xcodes prompts for your Apple ID, 2FA code and `sudo` password. Ctrl-C stops the wizard cleanly; steps that already finished are kept.
- Running it again is safe. Versions that are already installed, already selected or already set up are detected and skipped.

## Configuration

| Variable | Default | Purpose |
|---|---|---|
| `XCODES_DIRECTORY` | `/Applications` | Where Xcodes are installed. xcodes reads this variable too. |
| `XCODE_SYMLINK` | `$XCODES_DIRECTORY/Xcode.app` | Fixed path that scripts and CI can depend on |
| `MIN_FREE_GB` | `40` | Warn below this much free disk space |
| `XCODES_USERNAME` / `XCODES_PASSWORD` | — | Apple ID for xcodes (otherwise it prompts once and saves it to the Keychain) |
| `NO_COLOR` | — | Turn off colored output |

## Other things xcodes can do

The wizard covers the common flow. These are the other xcodes commands
(v2.1.0) you can run directly.

### Xcode versions

```bash
xcodes list                                   # every version for this Mac (marks Installed/Selected)
xcodes list --architecture universal          # Intel-compatible builds
xcodes update --data-source apple             # read Apple's site directly (newest, but more fragile)
xcodes installed                              # what's on disk
xcodes installed 27.2                         # print the path of one version

xcodes install 27.2                           # a specific version
xcodes install "27.3 Beta 2"                  # a beta (quote versions with spaces)
xcodes install --latest                       # newest release
xcodes install --latest-prerelease            # newest beta / RC
xcodes install 27.2 --path ~/Downloads/Xcode_27.2.xip   # use a .xip you already downloaded
xcodes install 27.2 --directory /Volumes/External       # install somewhere else
xcodes install 27.2 --no-superuser            # skip the sudo post-install steps

xcodes download 27.2                          # only download the .xip (to ~/Downloads)
xcodes uninstall                              # interactive picker
xcodes uninstall 27.0 --empty-trash           # delete permanently, skipping the Trash
```

### Switching

```bash
xcodes select 27.2                            # by version (runs xcode-select for you)
xcodes select /Applications/Xcode-27.2.0.app  # by path
xcodes select -p                              # print the selected Xcode
xcodes select                                 # uses ./.xcode-version if present, otherwise asks
```

**`.xcode-version` file:** put a version such as `27.2` in a file named
`.xcode-version` at a project's root. Then `xcodes select`, fastlane and most CI
setups use that Xcode for the project.

### Simulator runtimes

Since Xcode 15, simulator runtimes are downloaded separately from Xcode.

```bash
xcodes runtimes                               # available runtimes (marks Installed / Bundled)
xcodes runtimes --include-betas
xcodes runtimes install "iOS 27.2"            # download + install
xcodes runtimes install "iOS 26.4" --keep-archive        # keep the .dmg afterwards
xcodes runtimes download "iOS 27.2" --directory ~/Archive # download only
```

### Account and download options

```bash
xcodes signout                                # forget the saved Apple ID
xcodes install 27.2 --use-fastlane-auth       # reuse a fastlane/spaceship session (CI)
xcodes install 27.2 --no-aria2                # turn off aria2 even if it's installed
```

**GUI alternative:** `brew install --cask xcodes-app` gives you the same
features in a menu-bar app.

## Related Apple tools

These are built into Xcode and don't need an Apple ID:

```bash
xcode-select -p                               # active developer directory
sudo xcode-select --switch <path>             # set it system-wide
xcodebuild -version                           # active Xcode version
xcodebuild -license check                     # exit 0 = license accepted
xcodebuild -checkFirstLaunchStatus            # exit 0 = first-launch setup done
sudo xcodebuild -runFirstLaunch               # accept license + install packages

xcodebuild -downloadPlatform iOS              # runtime matching the active Xcode
xcodebuild -downloadAllPlatforms              # iOS, watchOS, tvOS and visionOS
xcodebuild -importPlatform ~/iOS.dmg          # install a runtime .dmg offline
xcodebuild -downloadComponent MetalToolchain  # optional components
xcodebuild -showComponent MetalToolchain

xcrun simctl runtime list                     # installed runtime disk images
xcrun simctl runtime delete --outdated --dry-run   # preview cleanup
xcrun simctl runtime delete --notUsedSinceDays 90  # delete runtimes unused for 90 days
xcrun simctl runtime delete --unusable              # delete broken runtimes
```

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `xcodes installed` never shows `(Selected)` | `xcode-select` points at the `Xcode.app` symlink, and xcodes doesn't follow symlinks. The script resolves them, so `status` shows the correct active version. |
| A shell still uses the old Xcode | `DEVELOPER_DIR` is set in that shell and overrides `xcode-select`. Run `echo $DEVELOPER_DIR` and remove it from your shell profile. |
| `xcrun: error: invalid active developer path` | The selected Xcode was deleted outside this tool. Run `xcode-update switch`. |
| Asked for an Apple ID every time / wrong account | Run `xcode-update signout`, or set `XCODES_USERNAME` for one run. |
| Download failed partway through | Run `xcode-update update` again and pick the same version. |
| Disk space didn't come back after uninstalling | You chose "move to Trash". Empty the Trash, or choose permanent deletion next time. |
| Simulator list is empty in Xcode | No runtimes are installed. Run `xcode-update runtimes` → option 1. |

## Project layout

```
xcodes-update-wizard/
├── xcode-update   # the wizard (bash 3.2 compatible, one file, no dependencies besides xcodes)
├── README.md
└── LICENSE        # MIT
```

The script is organised top to bottom as: configuration → output and prompt
helpers → read-only lookups → pickers → the nine wizard steps → commands → `main`.
Each wizard step is one `step_*` function, so you can reorder steps or add one
in `cmd_update`.

## License

MIT. See [LICENSE](LICENSE).
