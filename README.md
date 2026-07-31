# mago-supervisor — build harness

**This repository contains no application source code, and never should.**

It exists only to run GitHub Actions builds for **InstaFarms/mago-supervisor**, which is
private. GitHub Actions minutes on standard runners — including macOS — are free
for public repositories, so keeping the harness public and the code private
gives unlimited iOS and Android builds at no cost.

## How a build works

1. You start a build from the **Actions** tab (builds are manual only).
2. The runner clones the private source repo using a **read-only deploy key**
   held in `SOURCE_REPO_SSH_KEY`. A deploy key is scoped to exactly one
   repository and cannot write, so a leak cannot reach anything else.
3. Signing credentials are written from repository secrets into an EAS
   `credentials.json` that exists only for the life of the job.
4. `eas build --local` compiles on the runner. This consumes **no EAS build
   credits** — the build never goes to Expo's servers.
5. The artifact is encrypted and uploaded as a downloadable build artifact.
   **Nothing is ever submitted to a store from here** — this harness only
   produces files.

Nothing from the source tree is committed here, and the job only uploads the
artifact paths it names explicitly.

## Branches

| Branch | Workflow | Runner | Produces |
|---|---|---|---|
| `android` | Build Android | `ubuntu-latest` | `.aab` or `.apk` |
| `ios` | Build iOS | `macos-15` | `.ipa` |

Both workflows are present on both branches. That is not redundancy: GitHub
only allows `workflow_dispatch` for workflows that exist on the repository's
DEFAULT branch, so an iOS workflow living solely on `ios` would be invisible and
un-runnable. The `ios` branch remains the place to edit iOS build logic.

## Running a build

Actions → *Build Android* / *Build iOS* → **Run workflow**, then pick:

- **profile** — an EAS profile as named in the app's own `eas.json`
  (`production` by default)
- **format** — `aab` or `apk` (Android only)
- **ref** — the source branch, tag or SHA to build

## Getting the artifact

**Artifacts on a public repository can be downloaded by anyone who can see the
run.** A signed binary embeds the whole JS bundle and every `EXPO_PUBLIC_*`
value, so artifacts are encrypted before upload and the job fails outright if
`ARTIFACT_PASSPHRASE` is missing.

To decrypt:

```bash
openssl enc -d -aes-256-cbc -pbkdf2 -in app.aab.enc -out app.aab -pass pass:'<ARTIFACT_PASSPHRASE>'
sha256sum app.aab      # compare against sha256.txt from the same artifact
```

Artifacts are kept for 30 days, then deleted by GitHub automatically.

An iOS build signed with an **App Store distribution** certificate will not
install directly on a device. If you want a sideloadable IPA, point the run at
an eas.json profile that uses an ad-hoc or enterprise provisioning profile.

## Configuration

See [`SECRETS.md`](SECRETS.md) for every secret and variable, and how to produce
each value.

## Security rules for this repo

- **Never commit application source, `.env` files, keystores, `.p8`/`.p12` keys
  or provisioning profiles.** Everything sensitive belongs in Actions secrets,
  which are encrypted and are not exposed to forks.
- Builds are `workflow_dispatch` only. Do not add `pull_request` or
  `pull_request_target` triggers — a fork could then run modified workflow code
  against this repository's context.
- Keep `permissions: contents: read`.
- Workflow **logs are public**. Never `echo` a secret; pass values via `env:`
  rather than inline `${{ }}` interpolation inside a shell command.
