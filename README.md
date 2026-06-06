# github-fork-sync

Fork a public GitHub repo, make it private, and keep it automatically synced with the upstream — from A to Z.

## Why you'd want this

**Concrete example:** Suppose your team uses [Claude Code](https://claude.ai/code) and you want to manage a shared set of skills (for example [obra/superpowers](https://github.com/obra/superpowers)) for your entire organization. To do this, you need to:

1. Fork the public skills repo into your org's private GitHub
2. Keep it **private** so you can add internal customizations without exposing them
3. **Automatically pull upstream improvements** daily so your team always gets the latest skills

The same pattern applies anywhere you want to privately maintain and customize a public open-source repo — internal tooling, config templates, AI prompts, documentation bases, and more.

---

## What's in this repo

| File | Purpose |
|------|---------|
| [`template/sync-upstream.yml`](template/sync-upstream.yml) | GitHub Actions workflow template — you copy this into your fork at `.github/workflows/`, where it runs daily and syncs with upstream |
| [`skills/fork-private-sync/SKILL.md`](skills/fork-private-sync/SKILL.md) | Skill that automates the entire setup — requires the Claude desktop app (or Claude Code) with the Claude in Chrome extension; the Claude.ai web chat does not work |

---

## Option A: Manual setup (step by step)

### Prerequisites

- A GitHub account (personal or org)
- The URL of the public repo you want to fork privately

### Step 1: Fork the repo

1. Go to the public repo on GitHub (e.g., `https://github.com/obra/superpowers`)
2. Click **Fork** in the top-right corner
3. Select your account or org as the destination
4. Click **Create fork**

### Step 2: Leave the fork network

GitHub won't allow changing visibility to private while the repo is still part of the upstream fork network — you must detach it first.

1. In your new fork, go to **Settings**
2. Scroll to **Danger Zone** at the bottom
3. Click **Leave fork network** (or "Convert to independent repository")
4. Type the repo name to confirm, then click the confirmation button

### Step 3: Make the fork private

1. Still in **Settings** → **Danger Zone**
2. Click **Change repository visibility** → **Make private**
3. Type the repo name to confirm, then click the confirmation button

### Step 4: Add the sync workflow

1. In your fork, click **Add file** → **Create new file**
2. Name it: `.github/workflows/sync-upstream.yml`
3. Copy the contents of [`template/sync-upstream.yml`](template/sync-upstream.yml) from this repo
4. Replace `UPSTREAM_URL_HERE` with your upstream URL (e.g., `https://github.com/obra/superpowers.git`)
5. Click **Commit changes…** → **Commit directly to main** → **Commit changes**

### Step 5: Test it

1. In your fork, go to the **Actions** tab
2. Click **Sync upstream** in the left sidebar
3. Click **Run workflow** → **Run workflow** (green button)
4. Wait ~10 seconds — you should see a green ✓

### Step 6: Done

The workflow now runs every day at **06:00 UTC** automatically. No further setup needed — `GITHUB_TOKEN` is provided automatically by GitHub Actions, no secrets to configure.

---

## Option B: Let Claude do it

The [`fork-private-sync` skill](skills/fork-private-sync/SKILL.md) tells Claude exactly what to do — it will open GitHub in Chrome and handle every step: fork, leave fork network, make private, add the sync workflow, and test it.

The only thing the skill needs is **browser control** (the **Claude in Chrome** extension) — that's how Claude clicks through GitHub.

**Where it works:**

| Environment | Works? | Why |
|-------------|--------|-----|
| **Claude desktop app — chat** | ✅ Yes | Has Claude in Chrome. Load the skill by pasting `SKILL.md` into the prompt (below). |
| **Claude desktop app — Code** ([Claude Code](https://claude.ai/code)) | ✅ Yes | Has Claude in Chrome and native skill support. Cleanest option. |
| **Cowork** | ❌ No | No browser control. |
| **Claude.ai web chat** | ❌ No | No browser control. |

Whichever you use, make sure the **Claude in Chrome** extension is connected first.

**Get the skill — pick one:**

- **Copy** — open [`SKILL.md`](skills/fork-private-sync/SKILL.md) on GitHub and click the copy icon in the top-right of the file view to copy the full contents
- **Download** — [download `SKILL.md`](https://github.com/petrocasek/github-fork-sync/raw/main/skills/fork-private-sync/SKILL.md) directly

**Then paste it into Claude with your request, for example:**

```
[paste SKILL.md contents here]

---

Fork https://github.com/obra/superpowers privately into my GitHub account and set up daily upstream sync.
```

Claude will take it from there.

---

## How the sync workflow works

```
Every day at 06:00 UTC (or on-demand via Actions tab):
  1. Checkout your fork's main branch (always latest HEAD)
  2. Fetch all branches from upstream
  3. Merge upstream/main into your main (fast-forward only)
  4. Push to your fork
  → If your fork has diverged (custom commits), the sync skips safely
```

**Key design decisions:**

- **`ref: main` in checkout** — ensures the workflow always operates on the current HEAD, not the SHA that triggered it. Without this, the push fails with a non-fast-forward error when `main` has been updated since the workflow was triggered.
- **`--ff-only` merge** — protects your custom commits. If you've made changes on your fork's `main` that aren't in upstream, the workflow exits cleanly instead of overwriting your work.
- **`actions/checkout@v5`** — uses Node.js 24, avoiding the deprecation warning that affects `@v4` from June 2026.
- **No secrets needed** — `GITHUB_TOKEN` is automatically injected by GitHub Actions.

---

## License

MIT
