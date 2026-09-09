# Setup, step by step

Budget about 25 minutes. Steps 1 to 4 are quick. Step 5 (Twilio) is the longest
because of phone number verification.

---

## Step 1 — Install prerequisites

You need `git`, `python3`, and the GitHub CLI.

```bash
# macOS
brew install git python gh

# Windows (PowerShell as admin)
winget install Git.Git Python.Python.3.12 GitHub.cli
```

Verify:

```bash
python3 --version   # want 3.10 or newer
gh --version
```

---

## Step 2 — Push to GitHub

Unzip the bundle, then from inside the folder:

```bash
cd tesla-watch-repo
git init
git add .
git commit -m "initial commit"
gh auth login          # follow the browser prompt
gh repo create tesla-watch --public --source=. --push
```

**Why public:** at 15-minute polling this runs ~2,880 billed minutes/month.
Free accounts get 2,000 minutes on private repos, so a private repo would hard
stop around day 21. Public repos get unmetered minutes. Nothing sensitive is in
the repo: secrets live in GitHub's encrypted secret store, and state lives in
the Actions cache, not in a commit.

**One caveat:** public repos have scheduled workflows auto-disabled after 60
days with no repository activity. You get a warning email first, and any commit
or a click on Enable in the Actions tab resets it.

---

## Step 3 — Get your Tesla refresh token

1. Download the latest release of `tesla_auth` for your OS:
   https://github.com/adriankumpf/tesla_auth/releases
2. Run it. A window opens with Tesla's real login page.
3. Sign in with your Tesla credentials and complete MFA.
4. It prints an **access token** and a **refresh token**. Copy the refresh
   token, the long string starting with `eyJ`.

This runs entirely on your machine and talks only to Tesla. Nobody else, myself
included, should ever handle your Tesla login.

On macOS you will likely get a Gatekeeper warning since the binary is unsigned.
Right-click the app and choose Open to bypass it.

**Also grab your reference number** while you are in your Tesla account. It
looks like `RN########` and appears on your order page. Optional, but it removes
ambiguity if you ever have more than one order.

---

## Step 4 — Set up ntfy (full-detail push)

1. Generate a topic name that nobody will guess:
   ```bash
   openssl rand -hex 8
   ```
   Prefix it, e.g. `tesla-a3f9c21b8e4d5607`.
2. Install the **ntfy** app from the App Store or Play Store.
3. Open it, tap **+**, and enter that exact topic name.
4. Leave the server as the default `ntfy.sh`.

Topics are unauthenticated on the public server. Anyone who knows the string can
read your notifications, which is why it needs to be random rather than
memorable.

---

## Step 5 — Set up Twilio (SMS)

The free carrier email-to-SMS gateways are gone. T-Mobile's `@tmomail.net`
stopped delivering in late 2024, AT&T shut down `@txt.att.net` on 17 June 2025,
and Verizon's `@vtext.com` is degraded with a hard cutoff of 31 March 2027. Real
SMS has to go through a messaging provider.

1. Sign up at https://www.twilio.com/try-twilio. Trial credit covers this
   easily.
2. Verify your own cell number when prompted. On a trial account Twilio only
   sends to verified numbers, so this step is required.
3. From the Console dashboard, copy your **Account SID** and **Auth Token**.
4. Buy a phone number: **Phone Numbers → Manage → Buy a number**. Pick any US
   local number with SMS capability.
5. Note the number in E.164 format, e.g. `+12055550147`.

**Cost:** roughly $1.15/month for the number plus about $0.008 per SMS. At a
realistic few alerts per week, expect well under $2/month.

Messages are capped at 300 characters in the config, so a burst of matches
cannot silently turn into a twenty-segment text.

---

## Step 6 — Add repository secrets

In your repo: **Settings → Secrets and variables → Actions → New repository
secret**. Add each of these:

| Secret name | Value | Required |
| --- | --- | --- |
| `TESLA_REFRESH_TOKEN` | the `eyJ...` string from Step 3 | yes |
| `NTFY_TOPIC` | your random topic from Step 4 | yes |
| `TESLA_REFERENCE_NUMBER` | your `RN########` | optional |
| `TWILIO_ACCOUNT_SID` | starts with `AC` | for SMS |
| `TWILIO_AUTH_TOKEN` | from the Twilio console | for SMS |
| `TWILIO_FROM` | the number you bought, `+1...` | for SMS |
| `TWILIO_TO` | your cell, `+1...` | for SMS |

If you skip the Twilio ones, edit `notify.methods` in `tesla_watch.py` to just
`["ntfy"]`. Otherwise every run logs a skip message.

---

## Step 7 — Capture your early-pickup query

This is what makes the inventory half actually useful.

1. Sign in to tesla.com and open your order.
2. Find the option to take delivery sooner / view matching inventory.
3. Open DevTools (F12), go to the **Network** tab, and filter for
   `inventory-results`.
4. Reload the page. Click the request that appears.
5. In **Headers → Query String Parameters**, copy the entire value of `query`.
6. Paste it into `inventory_query_raw` in `tesla_watch.py`, between the quotes.
7. Commit and push:
   ```bash
   git add tesla_watch.py && git commit -m "add early-pickup query" && git push
   ```

**Why bother:** the fallback query searches all public inventory nationwide and
filters to Performance trims with a white interior. The early-pickup list is
narrower and more useful, because it is what Tesla will actually let you swap
into without losing your order position or pricing. Different sets.

If you skip this, everything still works on the fallback.

---

## Step 8 — Test

1. Go to the **Actions** tab → **Tesla Watch** → **Run workflow**.
2. Watch the run. Expected output in the log is `Baseline captured on first
   run; notification suppressed.`
3. Run it a second time. If nothing changed, you get `No changes.` That means
   auth worked and polling is live.

To confirm notifications actually reach you, force one:

```bash
# in tesla_watch.py, set:
"notify_on_first_run": True,
```

Push that, then in the Actions tab delete the cache (**Actions → Caches**) so
the next run is treated as a first run. You should get both a push and a text.
Set it back to `False` afterward.

---

## Reading the alerts

SMS gets a headline: `Tesla update: 2 new matches, closest Mobile, AL (detail
in ntfy)`.

ntfy gets the full record, including for each car the paint, interior, price,
location, direct order link, and an obtainability note like `~11 mi from 36695
(in Alabama, simplest path)`. That note is advisory only. Nothing is ever
filtered out on distance.

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `401` on order checks | Refresh token expired or revoked | Rerun `tesla_auth`, update the secret |
| `SMS skipped: Twilio settings incomplete` | A Twilio secret is missing or misnamed | Check all four exist and are spelled exactly as in the table |
| SMS never arrives, no error | Trial account, unverified destination | Verify `TWILIO_TO` in the Twilio console |
| No inventory matches ever | Tesla renamed trim or interior strings | Set `require_interior_contains` to `None`, run, inspect raw values in the log |
| Order fields stop updating | Tesla changed the response schema | Inspect the raw JSON, update `ORDER_WATCH_PATHS` |
| Runs are 5 to 20 min late | GitHub queues scheduled workflows globally | Normal, not a bug |
| Workflow stopped after ~2 months | Public repo inactivity auto-disable | Click Enable in the Actions tab |
