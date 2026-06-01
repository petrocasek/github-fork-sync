---
name: fork-private-sync
description: Automates A-to-Z: fork a public GitHub repo, make the fork private, and set up automatic daily sync with the upstream. Use this whenever someone wants to privately maintain a copy of a public repo — for example forking obra/superpowers (Claude Code skills) for their org, or any case where someone says "fork X privately", "keep my fork synced with upstream", "private fork with auto-sync", or "maintain a private copy of a public repo". Especially relevant for teams managing shared Claude Code skills or any shared tooling they want to customize without exposing to the public. If someone mentions forking + privacy + automation in the same breath, use this skill.
---

# Fork Private Sync

Automate forking a public GitHub repo, making it private, and keeping it synced daily with upstream — from a single request.

## What you need from the user

Before starting, confirm you have:
1. **Upstream repo URL** — e.g., `https://github.com/obra/superpowers`
2. **Target GitHub account or org** — where the private fork should live

If either is missing, ask. Don't proceed without both.

## Process

Use browser tools (Claude in Chrome) to click through GitHub. Open the repo in a tab and execute each step.

---

### Step 1: Fork the repo

1. Navigate to the upstream repo URL on GitHub
2. Click **Fork** (top right corner)
3. Select the target account or org from the dropdown
4. Leave "Copy the default branch only" checked (default)
5. Click **Create fork**
6. Wait for the fork to finish loading

---

### Step 2: Leave the fork network

GitHub won't let you make a fork private while it's still connected to the upstream fork network. You must detach it first.

1. In the newly created fork, click **Settings** (top nav)
2. Scroll all the way down to **Danger Zone**
3. Click **Leave fork network** (or "Convert to independent repository")
4. Type the repository name to confirm
5. Click the confirmation button

---

### Step 3: Make the fork private

1. Still in **Settings** → **Danger Zone**
2. Click **Change repository visibility**
3. Select **Make private**
4. Type the repository name to confirm
5. Click the confirmation button

---

### Step 4: Create the sync workflow

Navigate to the fork root. Create the file `.github/workflows/sync-upstream.yml`:

1. Click **Add file** → **Create new file**
2. Type the filename: `.github/workflows/sync-upstream.yml`
   — GitHub auto-creates the folder path as you type the slashes
3. Paste the workflow content below, replacing `UPSTREAM_URL_HERE` with the actual upstream URL (e.g., `https://github.com/obra/superpowers.git`)

```yaml
name: Sync upstream

on:
  schedule:
    - cron: '0 6 * * *'
  workflow_dispatch:

jobs:
  sync:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v5
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0
          ref: main

      - name: Sync with upstream
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git remote add upstream UPSTREAM_URL_HERE
          git fetch upstream
          if git merge upstream/main --ff-only; then
            git push origin main
          else
            echo "Cannot fast-forward — upstream and fork have diverged. Skipping."
            exit 0
          fi
```

4. Click **Commit changes…** → **Commit directly to main** → **Commit changes**

---

### Step 5: Test the workflow

1. Click the **Actions** tab in the fork
2. In the left sidebar, click **Sync upstream**
3. Click **Run workflow** → confirm **Branch: main** → click green **Run workflow**
4. Wait ~10–15 seconds and reload
5. Verify the run shows a green ✓

---

### Step 6: Confirm to the user

Report back with:
- **Fork URL** (e.g., `https://github.com/their-org/superpowers`)
- Confirmation it's **private**
- Workflow runs **daily at 06:00 UTC** automatically
- They can trigger a manual sync anytime via Actions → Sync upstream → Run workflow

---

## Technical notes (why the workflow is written this way)

These details help if the user asks why certain choices were made, or if troubleshooting is needed:

- **`ref: main`** in checkout is critical. Without it, the checkout action pins to the exact commit SHA that triggered the workflow (`GITHUB_SHA`). If the fork's `main` has moved forward since the trigger (e.g., from previous edits), the push gets rejected as non-fast-forward. With `ref: main`, checkout always fetches the current HEAD of main.

- **`--ff-only`** is intentional. If the user has made custom commits on their fork's `main` that diverge from upstream, the workflow exits cleanly instead of force-pushing. This protects custom changes.

- **No secrets to configure.** `GITHUB_TOKEN` is automatically available in every GitHub Actions workflow — no manual setup needed.

- **`actions/checkout@v5`** runs on Node.js 24 and avoids the Node.js 20 deprecation warning that affects `@v4` from June 2026 onward.
