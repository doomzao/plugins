# Releasing a plugin update (the correct way)

Maintainer notes. Users do not need anything in this folder.

This is the process for shipping any change to users, for every plugin distributed through this repository. When done right, users just click **Update** in the Dalamud installer (or get the update automatically on login).

## How updates reach users

Dalamud re-reads `repo.json` periodically and on every game login. When the `AssemblyVersion` in `repo.json` is higher than the installed one, the plugin shows an **Update** button in `/xlplugins`. Users with auto-update enabled get it without clicking anything.

This means the `repo.json` push is the trigger. A GitHub release alone does nothing for users until `repo.json` says there is a new version.

## Step by step

### 0. Before releasing

- Build is clean: `dotnet build -c Release` with 0 errors and 0 warnings.
- The change was tested in game as a dev plugin.
- README/docs updated if the change affects users.

### 1. Bump the version and build

In the plugin's `.csproj`, raise `<Version>` (four-part, e.g. `0.2.0.0`):

- Bug fix: bump the third number.
- New feature: bump the second number.
- Breaking change or rewrite: bump the first number.

Then:

```powershell
dotnet build -c Release
```

DalamudPackager writes the distributable zip to `<Plugin>/bin/Release/<InternalName>/latest.zip`. The zip embeds the version, so always rebuild after the bump.

### 2. Create the GitHub release (as doomzao)

```powershell
gh auth switch --user doomzao
gh release create vX.Y.Z "<path to latest.zip>" -R doomzao/<plugin-repo> --title "<Plugin Name> vX.Y.Z" --notes "<what changed, user-facing wording>"
gh auth switch --user matheuspgamba
```

Rules:

- Tag format is `vX.Y.Z` (three numbers; the csproj's fourth digit stays 0 in tags).
- The asset must be the file named exactly `latest.zip`. The download links in `repo.json` use `/releases/latest/download/latest.zip`, which always resolves to the newest release, so the links never need to change.
- Never edit or delete an existing release to "fix" it. If a release is bad, ship a new higher version; `releases/latest` will point at it automatically.

### 3. Update repo.json and push (this is what triggers the update)

**Run `git pull --rebase` first.** The download counter workflow also commits to `repo.json`, so the remote may be ahead of your local copy.

In this repository, edit the plugin's entry in `repo.json`:

| Field | Set to |
|---|---|
| `AssemblyVersion` | The new four-part version (e.g. `0.2.0.0`) |
| `LastUpdate` | Current unix timestamp (`[int][double]::Parse((Get-Date -UFormat %s))` in PowerShell) |
| `DalamudApiLevel` | Only if Dalamud bumped its API level since the last release |

Then commit and push:

```powershell
git add repo.json
git commit -m "<Plugin Name> vX.Y.Z"
git push
```

Done. Users see the update within minutes (next installer refresh or next login).

## Adding a brand new plugin

1. The plugin repo lives under the doomzao account, follows the identity rules (doomzao git config, github-kefka remote), and publishes a release with `latest.zip` as above.
2. Add a new entry to the `repo.json` array: copy an existing entry and fill in `Name`, `InternalName`, `Punchline`, `Description`, `Tags`, `RepoUrl`, `AssemblyVersion`, `DalamudApiLevel`, `LastUpdate` and the three download links pointing at `https://github.com/doomzao/<repo>/releases/latest/download/latest.zip`.
3. Add a row to the plugin table in this repository's `README.md`.
4. Commit and push. Everyone who already added the repository URL gets the new plugin in their installer automatically.

## Common mistakes to avoid

- **Pushing repo.json before the release exists**: users would download the old zip under the new version number. Always release first, repo.json last.
- **Forgetting the csproj bump**: the zip would carry the old version and Dalamud would not offer the update even with repo.json bumped.
- **Renaming the release asset**: the `/releases/latest/download/latest.zip` links break. The asset name is always `latest.zip`.
- **Committing with the wrong identity**: these repos must only ever see `doomzao <292908125+doomzao@users.noreply.github.com>`. Check with `git config user.email` before committing.
