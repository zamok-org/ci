# zamok-org/ci — shared macOS release pipeline (public GitHub Action)

One reusable workflow that takes **any** macOS app from a git tag to a signed →
notarized → **stapled** artifact, a published Homebrew cask, and (optionally) a Sparkle
appcast. Each product repo carries a ~30-line caller instead of a ~90-line job.

## Two hosts, one pipeline

| | open-source | private |
|---|---|---|
| host | **GitHub** (this repo, public) | **Gitea** (`zamok/ci`, self-hosted) |
| runner | GitHub-hosted macOS (free for public repos) or a self-hosted label | self-hosted Tart macOS VM |
| `uses:` | `zamok-org/ci/.github/workflows/macos-cask-release.yml@v1` | `zamok/ci/.gitea/workflows/macos-cask-release.yml@v1` |

Same inputs, same secrets, same steps — pick the host that matches the product repo.

## Quick start — onboard a product in 8 steps

1. **Repo + org.** Public open-source app → GitHub. Set the signing secrets (below) on the
   product's org (e.g. `bshk-app`) so `${{ secrets.* }}` resolves in the caller.
2. **Cask metadata.** Add `Distribution/cask-<app>.json` (template below).
3. **Entitlements** *(only if needed)*. Add `<App>.entitlements` if Hardened Runtime needs
   it (e.g. mic). Restricted entitlements also need a provisioning profile + the
   `PROVISION_PROFILE` secret + `provisioning-profile: true`.
4. **Caller workflow.** Add `.github/workflows/release.yml` (copy below); fill `scheme`,
   `bundle-id`, `cask-metadata`, `asset-repo: <owner>/<app>`.
5. **Secrets.** Map the canonical secrets explicitly in the caller (see ⚠️ below — cross-org
   `inherit` does **not** forward them).
6. **Sparkle** *(optional)*. Add Sparkle SPM + Info.plist + EdDSA key, set `sparkle: true`.
7. **Release.** `git tag <app>-v0.1.0 && git push --tags`; watch the run under **Actions**.
8. **Install.** `brew tap <owner>/homebrew-tap && brew install --cask <app>`.

## The caller (GitHub)

`.github/workflows/release.yml` in the product repo:

```yaml
on:
  push: { tags: ["<app>-v*"] }
  workflow_dispatch:
    inputs: { version: { description: "e.g. 1.2.3", required: true } }
jobs:
  release:
    uses: zamok-org/ci/.github/workflows/macos-cask-release.yml@v1
    with:
      scheme: <Scheme>
      bundle-id: app.<owner>.<app>
      cask-metadata: Distribution/cask-<app>.json
      asset-repo: <owner>/<app>            # the app's OWN repo holds the artifact
      # entitlements: <App>.entitlements    # if Hardened Runtime needs them (e.g. mic)
      # provisioning-profile: true           # only for restricted entitlements
      # sparkle: true                        # in-app updates
      # runner: macos-15                     # override the runner (Xcode/SDK needs)
      version: ${{ github.event.inputs.version }}
    secrets:                                 # map explicitly — see the cross-org note below
      DEVELOPER_ID_P12:          ${{ secrets.DEVELOPER_ID_P12 }}
      DEVELOPER_ID_P12_PASSWORD: ${{ secrets.DEVELOPER_ID_P12_PASSWORD }}
      NOTARY_KEY_P8:             ${{ secrets.NOTARY_KEY_P8 }}
      NOTARY_KEY_ID:             ${{ secrets.NOTARY_KEY_ID }}
      NOTARY_ISSUER:             ${{ secrets.NOTARY_ISSUER }}
      TAP_GITHUB_TOKEN:          ${{ secrets.TAP_GITHUB_TOKEN }}
```

> **⚠️ Cross-org secrets — don't use `secrets: inherit` here.** This action lives in
> `zamok-org`; when your product repo is in a *different* org (e.g. `bshk-app`), GitHub
> only inherits secrets within the same org, so `inherit` forwards **nothing** and the run
> fails in ~4 s with *"Secret … is required, but not provided"*. **Map each secret
> explicitly** as above. (If your product repo is in `zamok-org` too, `inherit` works and you
> may drop the explicit block.)

Pin `@v1` (moving tag) or a SHA.

## Inputs

| input | required | default | notes |
|---|---|---|---|
| `scheme` | ✓ | — | Xcode scheme |
| `cask-metadata` | ✓ | — | path to the zamokctl cask JSON (repo-root-relative) |
| `bundle-id` | | — | informational / Sparkle naming |
| `project-path` | | `.` | dir holding the Tuist project (e.g. `apps/macapp`) |
| `workspace` | | `<scheme>.xcworkspace` | override if it differs |
| `asset-repo` | | `bshk-app/homebrew-tap` | **set to the app's own repo** for the canonical layout |
| `tap` | | `bshk-app/homebrew-tap` | where `Casks/<app>.rb` lands |
| `entitlements` | | — | repo-root-relative plist |
| `provisioning-profile` | | `false` | decode `PROVISION_PROFILE` + embed (restricted entitlements only) |
| `sparkle` | | `false` | generate + push the appcast |
| `build-flags` | | — | extra `xcodebuild` settings, e.g. `ARCHS=arm64 ONLY_ACTIVE_ARCH=YES SWIFT_ENABLE_EXPLICIT_MODULES=NO` |
| `runner` | | `macos-latest` | GitHub-hosted (free for public repos) or a self-hosted label |
| `version` | | from tag | override; else `<app>-v1.2.3` → `1.2.3` |

## Secrets

Set these on the **product's org** (so `${{ secrets.* }}` resolves in the caller), then
**map them explicitly** in the caller (cross-org `inherit` doesn't forward — see above).

| secret | required | for |
|---|---|---|
| `DEVELOPER_ID_P12` / `DEVELOPER_ID_P12_PASSWORD` | ✓ | Developer ID signing |
| `NOTARY_KEY_P8` / `NOTARY_KEY_ID` / `NOTARY_ISSUER` | ✓ | `notarytool` (App Store Connect API key) |
| `TAP_GITHUB_TOKEN` | ✓ | push the artifact + cask to the tap (needs access to BOTH the tap and `asset-repo`) |
| `PROVISION_PROFILE` | — | base64 `.provisionprofile` (restricted entitlements) |
| `SPARKLE_ED_PRIVATE_KEY` | — | sign the appcast (EdDSA) |

`DEVELOPER_ID_P12` is the base64 of an exported Developer ID Application `.p12`; the
notary trio is an App Store Connect API key.

## Per-app files

**`Distribution/cask-<app>.json`** — zamokctl cask metadata:

```json
{
  "token": "<app>",
  "name": "<App>",
  "desc": "…",
  "homepage": "https://github.com/<owner>/<app>",
  "autoUpdates": true,
  "appBundle": "<App>.app",
  "dependsOnMacOS": ">= :sequoia",
  "livecheck": true,
  "livecheckUrl": "https://raw.githubusercontent.com/<owner>/<app>/main/appcast.xml",
  "zapTrash": ["~/Library/Application Support/<App>", "~/Library/Caches/app.<owner>.<app>"]
}
```

**`<App>.entitlements`** — only if Hardened Runtime requires it (e.g.
`com.apple.security.device.audio-input`). Hardened Runtime itself is applied by zamokctl.
Restricted entitlements (e.g. `keychain-access-groups`) also need
`provisioning-profile: true` + a `PROVISION_PROFILE` secret.

## Notes

- **GitHub-hosted Xcode.** The default `macos-latest` runner carries Apple Silicon + a
  recent Xcode. If your app needs a newer Xcode/SDK than the image ships, set
  `runner: macos-15` (or a newer label / self-hosted runner with the right Xcode).
- **zamokctl** is installed from the tap at build time — **≥ 1.2.4** verifies the staple
  actually attached and fails the build otherwise.
- **Sparkle status:** the `sparkle: true` appcast step is currently a stub; the first
  Sparkle app implements `generate_appcast`. Until then use `sparkle: false`.

## Release

```bash
git tag <app>-v1.2.3
git push --tags          # → CI builds → signs → notarizes → staples → publishes
```

…or trigger via `workflow_dispatch`.
