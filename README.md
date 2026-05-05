# xmtp-cache-apple

Composite GitHub Action that pins Apple's Xcode at a deterministic
`/nix/store/<hash>-Xcode.app` path on macOS runners. Used by flake-based iOS
builds that need a content-addressed Xcode reference for `xcodebuild`,
`fastlane`, and Nix devShells.

## Why

Apple's Xcode license forbids public redistribution, so we cannot put Xcode in
a shared substituter like `cache.nixos.org`. This action keeps Xcode in private
storage you control — either WarpBuilds' GHA-style cache (default) or a
private FlakeHub binary cache — and exposes it at a stable store path the rest
of your build can depend on.

## Usage

### WarpBuilds cache (default, branch-scoped)

```yaml
- uses: xmtplabs/xmtp-cache-apple@v1
  with:
    xcode-path: /Applications/Xcode_26.3.app
    xcode-nix-path: /nix/store/x9hdz5mfp44i9b05sswp271jdv68r8vx-Xcode.app
```

Capture `xcode-nix-path` once per Xcode version locally:

```bash
nix store add --name Xcode.app /Applications/Xcode_26.3.app
# → /nix/store/x9hdz5...-Xcode.app
```

Bump both `xcode-path` and `xcode-nix-path` together when Xcode upgrades.

### FlakeHub binary cache (cross-branch, paid past free tier)

```yaml
- uses: xmtplabs/xmtp-cache-apple@v1
  with:
    xcode-path: /Applications/Xcode_26.3.app
    xcode-nix-path: /nix/store/x9hdz5mfp44i9b05sswp271jdv68r8vx-Xcode.app
    backend: flakehub
    flakehub-cache-name: xmtplabs/private-xcode
    flakehub-token: ${{ secrets.FLAKEHUB_CACHE_TOKEN }}
```

## Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `xcode-path` | yes | — | Runner-local Xcode.app path. |
| `xcode-nix-path` | yes | — | Pinned `/nix/store/<hash>-Xcode.app`. |
| `backend` | no | `warpbuilds` | `warpbuilds` or `flakehub`. |
| `cache-key` | no | derived | Override the WarpBuilds cache key. |
| `flakehub-cache-name` | for `flakehub` | — | `org/cache` slug. |
| `flakehub-token` | for `flakehub` | — | FlakeHub auth token. |

## Outputs

| Name | Description |
|---|---|
| `cache-hit` | `true` when Xcode was hydrated from the backend without rehashing. |
| `developer-dir` | Same as `xcode-nix-path`; set as `DEVELOPER_DIR` on `xcodebuild`. |

## How it works

1. Probe `/nix/store` — skip everything if Xcode is already present.
2. Try the chosen backend (cache restore or substituter pull).
3. If still missing, bootstrap: `nix store add` from the runner-local Xcode.
   Slow (~2-3 min) but only runs on a cold cache.
4. After a bootstrap, push the result back to the backend so future runs hit.

## Backend tradeoffs

- **`warpbuilds`**: included with Warp runners, no extra cost. Cache scope is
  per-branch — pre-warm `dev`/`main` on a schedule if you want PR branches to
  hit on first run.
- **`flakehub`**: cross-branch substitution, SOC II audited. Paid past the
  free tier; Xcode is ~11 GiB, factor that into your plan.

## License

Internal action. Inherits Apple's Xcode license terms — do not point this at
a public substituter.
