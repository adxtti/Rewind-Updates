# Rewind launcher updates

Rewind checks `https://raw.githubusercontent.com/adxtti/Rewind-Updates/main/version.json` on startup and again before either online or offline play. A newer version must be installed before starting NBA 2K19. Failed, missing, or invalid update checks keep both play options locked; Retry checks again. A running game is never interrupted or terminated for an update.

The launcher downloads the full Windows x64 portable executable, shows download progress, checks its declared byte count and SHA-256, and restarts after installation. Its embedded Rewind Client module, helpers, and server connection settings update with it. Local Steam profiles, careers, and saves are retained. The separate required game update downloads the approved installation files from `adxtti/game-repair`, backs up existing files, and replaces them before either online or offline play. See [game update deployment](https://github.com/adxtti/game-repair).

## Publish a release

1. Increase `launcher/package.json`'s version for each release. Rebuild the matching production launcher executable.
2. Run `node tools/export-updates.cjs --settings ../deploy/private/player-settings.json --certificate ../deploy/public/Rewind-public.crt` from the launcher directory. The output is `dist/updates/Rewind-Updates-<version>` under the Rewind project.
3. In the public `adxtti/Rewind-Updates` repository, create GitHub Release tag `v<version>`. Upload only `Rewind-<version>.exe` from the export's `release-assets` folder. The executable contains the player runtime files. Keep client ZIPs, server packages, operator packages, certificate/key ZIPs, server data, signing keys, and account data off the public updates repository and releases.
4. Check that the executable release URL in `version.json` downloads successfully, then copy the exported `version.json`, `.gitignore`, and README into the repository's `main` branch, commit, and push. Publish the manifest last so launchers never see a version whose download is missing.

The prepared `release-assets` directory is deliberately ignored by Git. Portable executables are roughly 100 MB, exceeding GitHub's browser file-upload limit and approaching Git's per-file limit; use GitHub Release assets for binaries. The launcher trusts download links only in this repository (raw main/master files or release assets) and follows HTTPS release redirects only to GitHub's release-asset CDN. It does not accept external update hosts.

Example manifest:

```json
{
  "schemaVersion": 1,
  "version": "0.2.7",
  "platform": "win32-x64",
  "url": "https://github.com/adxtti/Rewind-Updates/releases/download/v0.2.7/Rewind-0.2.7.exe",
  "sha256": "<the exporter supplies the exact 64-character digest>",
  "size": 102421760
}
```

Use the generated manifest without editing its digest or byte count. The public repository's HTTPS connection and ownership are the update trust source; keep control of that GitHub account. Checksums detect incomplete or changed downloads and are not a separate publisher signature. Versions compare numerically using semantic version rules; a server manifest older than the installed launcher is rejected rather than downgrading it. Changing files under an unchanged version does not make that release a newer version.

## Portable installation

Players need a writable folder containing their downloaded `Rewind.exe` (or `Rewind-<version>.exe`). Updates cannot replace the unpacked temporary Electron executable: they replace the original portable executable reported by `PORTABLE_EXECUTABLE_FILE`. A hidden helper first checks the staged download, waits for the exact launcher process to exit and for its portable wrapper to release the file, and rechecks that NBA 2K19 is closed. It stages on the destination disk, atomically replaces the executable, keeps a rollback copy until the updated process starts, and preserves the previous executable when download verification or installation fails. The helper never kills the game, Steam, or other processes and uses no console window.

If the updates repository is empty or unpublished, the production launcher will remain locked by design. Upload the assets and manifest before distributing it. There is no cached permission to bypass a required update in offline mode.

References: [electron-builder portable executable environment variables](https://www.electron.build/docs/nsis/), [GitHub browser upload limits](https://docs.github.com/en/repositories/working-with-files/managing-files/adding-a-file-to-a-repository), [GitHub release assets](https://docs.github.com/en/rest/releases/assets).

## Recovering from 0.2.3 or 0.2.4 installer errors

Those launchers started Windows PowerShell in detached mode, which can exit before executing the install script. Download the latest Rewind executable directly from its GitHub release, close the old launcher, and replace only its portable executable. Keep the existing game folder and local profile data. The corrected installer is included in 0.2.5 and newer for future updates; updating the downloadable version cannot repair an older launcher's already-running installer code.

## Steam connection recovery in 0.2.7

Before online launch, Rewind validates the saved Steam connection with the running server before syncing careers or joining the queue. If the server rejects it, the launcher clears that rejected connection and opens normal Steam sign-in. Cancelled or failed sign-in keeps play blocked and preserves saves. A server timeout does not discard an otherwise valid connection. The fix works with the existing 0.2.6 operator; no operator update is required.
