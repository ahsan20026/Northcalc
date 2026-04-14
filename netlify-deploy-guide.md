# NorthCalc — Netlify Automated Deployment Guide
### Publish One New Calculator Per Week, Automatically

---

## The Strategy

You have 8 calculators built. Instead of releasing them all at once, you'll drip one per week to maximize SEO momentum and social sharing. The approach:

- All 8 HTML files live in your GitHub repo from day one
- A `config.json` file controls which tools are "live"
- Every Monday, a GitHub Actions workflow automatically flips the switch for the next tool
- Netlify rebuilds and deploys the updated site within seconds

---

## Step 1 — Push NorthCalc to GitHub

1. Go to [github.com](https://github.com) → **New repository**
2. Name it `northcalc` (or `northcalc.ca`), set it to **Public**
3. On your computer, open Terminal in your `northcalc/` folder and run:

```bash
git init
git add .
git commit -m "Initial NorthCalc build — 8 calculators"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/northcalc.git
git push -u origin main
```

Your files are now on GitHub.

---

## Step 2 — Connect to Netlify

1. Go to [netlify.com](https://netlify.com) → **Add new site** → **Import an existing project**
2. Choose **Deploy with GitHub**
3. Authorize Netlify, then select your `northcalc` repo
4. Build settings (for a plain HTML site, leave these blank):
   - **Build command:** *(leave empty)*
   - **Publish directory:** `.`  *(a single dot — publish the root folder)*
5. Click **Deploy site**

Netlify will deploy in ~10 seconds. You'll get a URL like `https://random-name.netlify.app`.

6. In Netlify → **Domain settings** → **Add custom domain** → type `northcalc.ca`
7. Update your domain registrar's nameservers to Netlify's (shown in the dashboard)
8. Netlify handles HTTPS/SSL automatically (free)

---

## Step 3 — Create a Netlify Deploy Hook

A deploy hook is a secret URL that triggers a new deploy when you POST to it.

1. In Netlify → your site → **Site configuration** → **Build & deploy** → **Build hooks**
2. Click **Add build hook**
3. Name it `Weekly release` → Branch: `main` → **Save**
4. Copy the hook URL — it looks like:
   ```
   https://api.netlify.com/build_hooks/abc123xyz
   ```
5. Go to your GitHub repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**
   - Name: `NETLIFY_DEPLOY_HOOK`
   - Value: paste the full Netlify hook URL
6. Click **Add secret**

---

## Step 4 — Set Up the Weekly Release Schedule in GitHub

Create this folder structure in your repo:

```
northcalc/
  .github/
    workflows/
      weekly-release.yml
  release-schedule.json
```

**`release-schedule.json`** — defines which tool goes live each week:

```json
{
  "releases": [
    { "week": 1,  "date": "2025-02-03", "tool": "mortgage-calculator",      "label": "Mortgage Calculator" },
    { "week": 2,  "date": "2025-02-10", "tool": "hst-gst-calculator",       "label": "HST / GST Calculator" },
    { "week": 3,  "date": "2025-02-17", "tool": "tfsa-calculator",          "label": "TFSA Room Calculator" },
    { "week": 4,  "date": "2025-02-24", "tool": "land-transfer-tax-calculator", "label": "Land Transfer Tax Calculator" },
    { "week": 5,  "date": "2025-03-03", "tool": "income-tax-calculator",    "label": "Income Tax Calculator" },
    { "week": 6,  "date": "2025-03-10", "tool": "rrsp-calculator",          "label": "RRSP Calculator" },
    { "week": 7,  "date": "2025-03-17", "tool": "rent-vs-buy-calculator",   "label": "Rent vs Buy Calculator" },
    { "week": 8,  "date": "2025-03-24", "tool": "cpp-calculator",           "label": "CPP & OAS Estimator" }
  ]
}
```

Update the dates to your actual desired release Monday dates.

---

**`.github/workflows/weekly-release.yml`** — the GitHub Actions workflow:

```yaml
name: Weekly NorthCalc Release

on:
  schedule:
    # Runs every Monday at 9:00 AM Eastern (14:00 UTC)
    - cron: '0 14 * * 1'
  workflow_dispatch:  # also allows manual trigger from GitHub UI

jobs:
  trigger-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Check release schedule and trigger Netlify deploy
        run: |
          TODAY=$(date -u +%Y-%m-%d)
          echo "Today is $TODAY"
          
          # Read the schedule and check if today matches a release date
          SCHEDULE=$(cat release-schedule.json)
          MATCH=$(echo $SCHEDULE | python3 -c "
          import json, sys
          data = json.load(sys.stdin)
          today = '$TODAY'
          for r in data['releases']:
              if r['date'] == today:
                  print(r['label'])
                  break
          ")
          
          if [ -n "$MATCH" ]; then
            echo "Releasing: $MATCH"
            curl -X POST -d '{}' ${{ secrets.NETLIFY_DEPLOY_HOOK }}
            echo "Deploy triggered successfully!"
          else
            echo "No release scheduled for today ($TODAY). Skipping."
          fi
```

Commit and push both files:

```bash
git add .github/workflows/weekly-release.yml release-schedule.json
git commit -m "Add weekly release automation"
git push
```

---

## Step 5 — The "Hidden Until Live" Trick (Optional but Recommended)

If you don't want Google to index future tools before they're announced, add a `noindex` meta tag to upcoming pages, then remove it when they go live.

Add this to any page you want hidden from search engines for now:

```html
<meta name="robots" content="noindex, nofollow">
```

Remove it when that week's workflow runs. You can automate this removal too — but for 8 tools, doing it manually each Monday is easiest: edit the file, commit, and push. Netlify redeploys in seconds.

---

## Step 6 — Test the Whole Pipeline Right Now

Before waiting for Monday, trigger a manual deploy to confirm everything works:

1. Go to your GitHub repo → **Actions** tab
2. Click **Weekly NorthCalc Release** in the left sidebar
3. Click **Run workflow** → **Run workflow** (green button)
4. Watch the logs — you should see "Deploy triggered successfully!"
5. Check your Netlify dashboard — a new deploy should appear within 30 seconds

---

## Full Timeline Example

| Week | Monday Date | Tool Goes Live |
|------|-------------|----------------|
| 1 | Feb 3 | 🏠 Mortgage Calculator |
| 2 | Feb 10 | 🧾 HST / GST Calculator |
| 3 | Feb 17 | 💰 TFSA Room Calculator |
| 4 | Feb 24 | 🏛 Land Transfer Tax |
| 5 | Mar 3 | 📊 Income Tax Calculator |
| 6 | Mar 10 | 📈 RRSP Calculator |
| 7 | Mar 17 | 🔑 Rent vs Buy Calculator |
| 8 | Mar 24 | 👴 CPP & OAS Estimator |

---

## Summary

| Step | What you do | Time |
|------|-------------|------|
| 1 | Push to GitHub | 5 min |
| 2 | Connect Netlify | 5 min |
| 3 | Create deploy hook + GitHub secret | 3 min |
| 4 | Add workflow + schedule files | 5 min |
| 5 | Add noindex to future pages (optional) | 5 min |
| 6 | Test manual trigger | 2 min |

**Total setup time: ~25 minutes.** After that, every Monday at 9 AM Eastern, your next calculator goes live automatically — no action needed from you.

---

*NorthCalc — Made for Canadians 🍁*
