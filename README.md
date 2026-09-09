# tesla-watch

Polls Tesla's order-status and inventory APIs every 30 minutes on GitHub's
runners, diffs against `tesla_state.json`, and pushes a phone notification only
when something meaningful changes.

Pre-configured for any new Model Y Performance with a white interior,
nationwide, at any paint color. Each match is annotated with how far it sits
from the Alabama registration address (36695) and what that means practically.
That annotation is advisory only and never filters anything out.

## What is already done

- Watcher script with order tracking, inventory matching, and diffing
- Scheduled workflow, offset from the top of the hour to reduce queue delay
- Concurrency guard so two runs cannot clobber the state file
- State committed back to the repo each run, giving a timestamped history of
  every status transition the order goes through
- Secrets wired through environment variables, nothing sensitive in the code

## Setup

Full walkthrough with commands and screenshots-level detail is in
[SETUP.md](SETUP.md). Summary below.

### 1. Push this to a repo

Public is recommended at this cadence. See the Cost section: 15-minute polling
exceeds the free allowance on a private repo, and public repos get unmetered
minutes. State is kept in the Actions cache rather than committed, so nothing
sensitive (your VIN, delivery details) is published either way. The code itself
contains no secrets.

```bash
cd tesla-watch
git init
git add .
git commit -m "initial commit"
gh repo create tesla-watch --public --source=. --push
```

One caveat with public: scheduled workflows are auto-disabled after 60 days of
repository inactivity. You will get a warning email first. Any commit resets the
clock, or just click Enable in the Actions tab.

### 2. Get a Tesla refresh token

Download `tesla_auth` from https://github.com/adriankumpf/tesla_auth/releases
and run it. It opens Tesla's real login page in a local window, completes the
OAuth PKCE flow, and prints a refresh token. Nothing is transmitted anywhere
except to Tesla.

This step has to be done by you. It requires your Tesla account credentials,
which should not pass through anyone else's hands, including mine.

### 3. Notifications

SMS via Twilio (headline only) plus ntfy (full detail). Carrier email-to-SMS
gateways are shut down, so Twilio is the practical path for real texts. Both
channels are independent: one failing does not suppress the other.

### 4. Capture your early-pickup query

Tesla publishes no separate endpoint for the early-delivery inventory in your
account. That view calls the same `inventory-results` API, with parameters
derived from your order. To poll exactly the list your account shows you:

1. Sign in and open the early-pickup / take-delivery-sooner view on your order.
2. Open DevTools, Network tab, filter for `inventory-results`.
3. Copy the full value of the `query` parameter.
4. Paste it into `inventory_query_raw` in `tesla_watch.py`.

The script sends your bearer token on inventory requests, so an account-scoped
query resolves correctly.

If you skip this, the fallback query still works. It searches new Model Y
nationwide and filters to Performance trims with a white interior. The
difference is that the fallback shows all public inventory, whereas the
early-pickup list shows what Tesla will actually let you swap into without
losing your order position or pricing. Those are not the same set, which is why
the raw query is worth the two minutes.

### 5. Add repository secrets

Settings → Secrets and variables → Actions → New repository secret:

| Name | Value |
| --- | --- |
| `TESLA_REFRESH_TOKEN` | the `eyJ...` string from step 2 |
| `NTFY_TOPIC` | a long random string, e.g. `tesla-4f9c2a71bd3e` |
| `TESLA_REFERENCE_NUMBER` | optional; your `RN########`. Omit to use the first order on the account |

Anyone who guesses your ntfy topic name can read your notifications, so make it
random rather than memorable. Generate one with `openssl rand -hex 6`.

### 6. Subscribe on your phone

Install ntfy (iOS or Android), tap Subscribe, enter the same topic string.

### 7. Test it

Actions tab → Tesla Watch → Run workflow. The first run captures a baseline
silently. Check the run log: `No changes.` means auth worked and there was
nothing new. A 401 means the refresh token needs regenerating.

To confirm notifications reach your phone, set `notify_on_first_run` to `True`
in `tesla_watch.py`, delete `tesla_state.json`'s contents back to `{}`, and run
again.

## Cost

96 runs per day at one billed minute each is roughly 2,880 minutes per month.
That is well past the 2,000 minutes included with a free GitHub account on a
private repo, and would hard-stop around day 21 of each month. Public
repositories get unmetered minutes on standard runners, which is why the setup
above uses public plus cache-based state rather than private plus committed
state.

If you would rather keep the repo private, the options are a 30-minute cron
(~1,440 min/month, comfortably inside the allowance) or a plan with a
3,000-minute allowance, which leaves only ~120 minutes of headroom at this
cadence.

## Failure modes

- **Order fields stop updating.** Tesla changed a response schema. The `dig()`
  helper skips missing paths rather than crashing, so check the raw JSON and
  update `ORDER_WATCH_PATHS`.
- **401 on order checks.** Refresh tokens are long-lived but not permanent.
  Rerun `tesla_auth` and update the secret.
- **No inventory matches ever.** Tesla may have renamed the trim or interior
  strings, or moved interior data to a field `extract_interior()` does not
  check. Set `require_interior_contains` to `None` temporarily and inspect the
  raw `interior` values coming back.
- **Distance shows as unknown.** Tesla does not return coordinates on every
  result. The note falls back to a state-based tier, which is coarser but still
  useful.
- **Runs are late.** GitHub queues scheduled workflows globally. 5 to 20 minutes
  of drift is normal and not a bug.
