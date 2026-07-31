# Secrets and variables

Set these under **Settings → Secrets and variables → Actions**.

Secrets are encrypted, are never printed in logs, and are **not** available to
workflows triggered from forks — which is what makes a public harness repo safe.
Variables are plain text and visible to anyone; only non-sensitive values go
there.

## Variables (Settings → Variables)

| Name | Value for this repo |
|---|---|
| `SOURCE_REPO` | `InstaFarms/mago-supervisor` |

## Secrets (Settings → Secrets)

### Both branches

| Name | What it is |
|---|---|
| `SOURCE_REPO_SSH_KEY` | Private half of a **read-only deploy key** on `InstaFarms/mago-supervisor` |
| `EXPO_TOKEN` | Expo access token (`expo.dev` → Account → Access tokens) |
| `RCLONE_CONFIG_ONEDRIVE` | An rclone config whose `[builds]` remote is pinned to the OneDrive `app-builds` folder |

### `android` branch

| Name | How to produce it |
|---|---|
| `ANDROID_KEYSTORE_BASE64` | `base64 -w0 release.keystore` |
| `ANDROID_KEYSTORE_PASSWORD` | Keystore password |
| `ANDROID_KEY_ALIAS` | Key alias inside the keystore |
| `ANDROID_KEY_PASSWORD` | Key password |

### `ios` branch

| Name | How to produce it |
|---|---|
| `IOS_DIST_CERT_P12_BASE64` | `base64 -w0 dist-cert.p12` |
| `IOS_DIST_CERT_PASSWORD` | Password set when exporting the `.p12` |
| `IOS_PROVISIONING_PROFILE_BASE64` | `base64 -w0 profile.mobileprovision` |

## Exporting the existing credentials from EAS

The signing material is already held by EAS. Pull it out rather than generating
new keys — **a new Android upload key cannot be swapped in on an app Google Play
already has**, and replacing it means a support request to Google.

```bash
cd <app-source-checkout>
eas credentials            # choose the platform, then "Download credentials"
```

For Android this yields the keystore plus its passwords and alias. For iOS it
yields the distribution certificate (`.p12`) and provisioning profile.

## The OneDrive remote

`RCLONE_CONFIG_ONEDRIVE` holds a complete rclone config with a single `[builds]`
remote. Its `root_folder_id` pins the remote to `/onedrive/app-builds`, so the
workflow can only see and write that subtree.

Be aware of what that does and does not buy you: `root_folder_id` constrains
**this client**, not the token. The underlying OAuth token is a delegated
OneDrive-for-Business token and is not scoped server-side, so anyone holding it
could point a different config at the rest of the drive. It is defence in depth,
not a permission boundary — keep the secret tight and rotate it if in doubt.

## The deploy key

`SOURCE_REPO_SSH_KEY` holds the private half of an ed25519 deploy key registered
on `InstaFarms/mago-supervisor` with **write access disabled**. This is deliberately preferred
over a personal access token:

- it is bound to one repository and cannot reach any other,
- it is read-only, so it cannot push, tag or delete,
- it belongs to no user, so revoking it locks out nobody.

Revoke or rotate it under `InstaFarms/mago-supervisor` → Settings → Deploy keys.
