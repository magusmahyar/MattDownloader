# MattDownloader — GitHub relay (workflows template)

This repository holds **GitHub Actions workflows** only: a small “relay” that downloads a file on GitHub’s runners, stores it under `jobs/<job_id>/` on **your** fork, and lets a companion mobile app pull it down later.

## Inspiration

The idea is **inspired by** [soheilditf5-svg/downloader](https://github.com/soheilditf5-svg/downloader): use GitHub Actions as a stable middle step when direct downloads from the phone are slow, blocked, or keep failing mid-file.

This template is maintained for **MattDownloader** and tuned for that app’s manifest format and cleanup flows.

## Why use this?

- **More reliable than browser-only downloads** on bad links or unstable networks; the heavy HTTP fetch runs on Actions, not on the phone.
- **Useful when many CDNs or sites are hard to reach** but GitHub is still reachable — the phone talks mainly to GitHub with a token you control.
- **Your fork, your data** — artifacts live in a private or public repo **you** own; no third-party download server.
- **Clear lifecycle** — prepare job, download to device from the API, optionally remove the job folder when finished.

**What we changed in this relay (vs the general “Actions as downloader” idea):** we standardize on a **`manifest.json`** per job (mode, size, suggested filename, paths), **split large files at ~95 MB** into ordered chunks so the GitHub API stays practical, **validate `job_id`** for safe paths, and ship companion workflows to **`delete-job`** when the user is done plus **scheduled cleanup** of old `jobs/*` folders. Everything is laid out so a **mobile client** can poll, pull blobs or chunks, merge on device, and export to local storage—without you hosting a separate backend.

**Mobile app:** **MattDownloader** is the official companion app. Download the **release APK** from the **Releases** page of **this same repository** (pre-built Android package only—**no** mobile source code lives in this repo). The app stores your token, lists jobs, saves files to your phone, and can dispatch delete/cleanup against your fork.

## MattDownloader app (releases only)

This repository’s **git tree** contains workflows and docs only—not the mobile app source. The **MattDownloader** Android **APK** is published on **this repo’s GitHub Releases** page: open **Releases**, pick the latest version, and download the attached APK. Install it on your phone, then continue with fork setup below. You do **not** need the app source code to use the relay—only your fork of this template and that release build.

## What belongs here

| Included | Not included |
|----------|----------------|
| `.github/workflows/` | Mobile app **source code** |
| This `README.md`, `.gitignore` | *(APK is not in the tree—get it from **Releases** on this repo.)* |

**Created by Actions (on your default branch):** `jobs/<job_id>/` — manifests and file chunks the app reads; remove when you no longer need them.

## Donate

If this template or MattDownloader helps you, optional donations are welcome (always verify chain and address in your wallet):

| Asset | Network | Address |
|-------|---------|---------|
| **USDT** | Polygon | `0xacef000D72eb46Af92F32252fe2bCE146aa9c2E0` |
| **USDT** | TRON (TRC20) | `TFpeKzEakmSXXi7MXBt1VZVfErB3kXHyL7` |

## Folder layout

- `.github/workflows/` — workflow definitions  
- `jobs/<job_id>/` — appears after a successful run (not shipped in the empty template)

## Workflows

- **`download-split.yml`** (*Download and Prepare Job*)  
  **Inputs:** `file_url`, `job_id`  
  Downloads the URL, splits if ≥ 95 MB, writes `jobs/<job_id>/manifest.json`, commits and pushes.

- **`delete-job.yml`** (*Delete Job*)  
  **Input:** `job_id`  
  Deletes `jobs/<job_id>/` and pushes.

- **`cleanup-expired-jobs.yml`**  
  Scheduled cleanup of old `jobs/*` folders (see the workflow file for rules).

## App integration contract

After `download-split.yml` succeeds, the client reads **`jobs/<job_id>/manifest.json`**, then:

- **`suggested_filename`** — safe basename from the source URL; the app saves under its own downloads tree using this name.
- **`mode: "single"`** — fetch `jobs/<job_id>/file.bin`
- **`mode: "chunked"`** — fetch `jobs/<job_id>/chunks/*` in sorted order and merge on device.

When the user is done, the app should dispatch **`delete-job.yml`** with the same `job_id` (for example on “Remove” in the app). The app does **not** auto-delete immediately after a successful save, so users can re-download or inspect until they remove the job.

`job_id` must match workflow validation: **6–64** characters, `[A-Za-z0-9_-]` only.

## Fork setup (each user)

A **fork** is your own GitHub copy of this template, under **your** account. Workflow runs, commits, and `jobs/<job_id>/` data all happen on **your** fork—not on someone else’s repo—so you keep control and quotas.

1. **Fork this repository** on GitHub (use the **Fork** button). Pick yourself or an org as the owner. You can keep the default name or rename the repo; what matters later is the **`owner/repo`** URL (for example `https://github.com/yourname/your-fork`).
2. Open your fork on GitHub, go to the **Actions** tab, and **allow workflows** if GitHub shows a one-time prompt. Without this, prepare and delete jobs cannot run.
3. Download the **MattDownloader** **APK** from the **Releases** page of **this repository**. Install the APK, open the app, then go to **Settings**.
4. **Create a token in the app’s guided flow** (recommended: a **fine-grained personal access token**). The app walks you through GitHub’s screens and the exact **permissions** this relay needs—scoped to **only your fork**, not your whole GitHub account.
5. In the app, enter **your fork’s repository** as `owner/repo` (same as in the browser address bar after `github.com/`) and **paste the token** where the app asks for it. Save. The app uses that pair to dispatch workflows, read `manifest.json`, and download prepared files from **your** fork.

If something fails, re-check that Actions are enabled on the fork, the token is still valid, and `owner/repo` matches the fork you actually forked—not the upstream template URL.

## Security note

- Prefer a **fine-grained PAT** limited to the single fork, or a **GitHub App** installed only on that repo, over a classic PAT with full `repo` across the account.
- The **Actions** run uses `GITHUB_TOKEN` to push job commits; the user’s app token is for dispatch + API reads as you define.
