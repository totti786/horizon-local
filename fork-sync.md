# Syncing Horizon Local with upstream

The fork follows `horizon-rust/horizon` and keeps a few launcher patches of its own. An upstream
sync replaces `.github/workflows/` wholesale, so every sync must restore the fork's workflow again.

## Steps

1. `git fetch upstream`
2. `git merge upstream/main` on `main` (a merge, not a rebase: the fork's history and its
   published commits stay as they are).
3. Resolve conflicts:
   - `.github/workflows/`: keep only the fork's `launcher.yml`; `git rm` every upstream workflow.
   - `launcher/lang/*.json`: take upstream's file and re-add the fork's key
     `"couldn't check for updates: {error}"` after the opening `{`.
   - The launcher patches below: take upstream's code where it moved, and re-apply the fork's
     behaviour in its new home (upstream restructures the launcher often).
4. Bump `launcher/Cargo.toml` (`version = "0.2.X"`) and the `horizon-launcher` entry in
   `launcher/Cargo.lock`. The workflow tags `launcher@X.Y.Z` from that version.
5. `cargo fmt`, `cargo check --all-targets`, `cargo test --lib --bins` in `launcher/`.
6. Commit the merge, push `main`, and let the workflow publish.
7. If a build fails with "The job was not acquired by Runner of type hosted even after multiple
   attempts" (GitHub's macOS capacity), rerun just that job; the version is not released yet, so
   nothing gets published twice:
   ```sh
   gh run rerun <run-id> -R totti786/horizon-local --failed
   ```

## Launcher patches the fork keeps

- `backend::LAUNCHER_REPO = "totti786/horizon-local"`: self-update reads the fork's releases.
- `launcher_releases()` falls back to the fork's `releases.atom` when GitHub's API refuses
  (60 anonymous reads an hour; Horizon's mirror does not carry the fork), and a check that fails
  is reported (`Updates::launcher_error`) instead of reading as "up to date".
- `backend::OFFLINE_PLAY = true`: offline play in the window and the CLI, no Microsoft account
  that owns the game.
- `account::session_for_play()` is the gate both the window (`launcher_play.rs`) and the CLI/MCP
  (`backend/developer_play.rs`) use: when the owner gate refuses, the game starts offline instead.
  Downloads still go through `account::launch` and need an owner. Upstream removed
  `Settings::dev_offline` and the boolean argument to `session_for_game`; the fork does not need
  them.
- Offline names and their skins are kept though upstream removed them: the `Accounts.offline`,
  `offline_selected` and `looks` fields, `account::Player`, `play_as_player`, `look`, `look_env`
  and `name_arg`, `launcher/src/skins.rs`, the CLI's `account offline NAME`, `account select
  NAME`, `account skin` and `account forget-offline`, and the top bar's quick switch and
  Settings' offline rows (`switch_page.rs`, `blocks_settings.rs::offline_rows`). The game still
  reads `--name`, `HORIZON_OFFLINE_SKIN` and `HORIZON_OFFLINE_CAPE`, so this stays compatible.
- Signature check disabled for fork releases (`signed_digest` returns nothing, `download()`
  skips the check when the digest is empty). `fill_stage` then stores the archive's own SHA-256,
  so `place_checked` still refuses bytes that changed after the download; keep that.
- No Microsoft plates: `frame.rs` top-bar sign-in, `glass_account.rs` Microsoft card,
  `blocks_play.rs` prepare sign-in and the gate row, `blocks_settings.rs` `account_rows` (offline
  names instead), and the switch page's "who plays" chip.
- `Horizon Local` link in the glass sidebar, above the Discord button (`blocks_chrome.rs`).
- The README warning block and the "Not affiliated" rows in Settings' About (`blocks_settings.rs`
  and `glass_settings.rs`).

`launcher/tests/owner_gate.rs` is upstream's test of the gate: the fork rewrote its two cases to
assert the fork's policy (the CLI and the MCP server get past the gate and only ask for a
prepared game), so `cargo test` stays green. `cargo test --lib --bins --tests` is the suite the
fork checks.

The game itself still updates from `horizon-rust/horizon` (`backend::REPO`): the fork builds and
releases the launcher only.
