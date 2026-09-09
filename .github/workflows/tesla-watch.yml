#!/usr/bin/env python3
"""
tesla_watch.py
--------------
Polls Tesla's order-status API and inventory API, diffs the results against a
local state file, and notifies only when something meaningful changes.

Run it on a schedule (cron / launchd / GitHub Actions). Each run is stateless
apart from STATE_FILE.

Endpoints used (all unofficial, i.e. the same ones the Tesla app and the
tesla.com inventory page call). They can change without notice; the extractors
below are written defensively so a schema change degrades instead of crashing.
"""

import hashlib
import json
import os
import smtplib
import sys
import time
import urllib.parse
from datetime import datetime, timezone
from email.message import EmailMessage

import requests

# ============================================================================
# MANUAL DATA ENTRY
# ============================================================================

CONFIG = {
    # --- Auth -------------------------------------------------------------
    # One-time setup: obtain an SSO refresh token (see SETUP notes at bottom),
    # then export it as TESLA_REFRESH_TOKEN. The script exchanges it for a
    # short-lived access token on every run.
    "refresh_token": os.environ.get("TESLA_REFRESH_TOKEN", ""),

    # --- Order tracking ---------------------------------------------------
    # Reference number from your order page, format "RN######...".
    # Leave empty to auto-select the first order on the account.
    "reference_number": os.environ.get("TESLA_REFERENCE_NUMBER", ""),
    "device_country": "US",
    "device_language": "en",
    "app_version": "4.45.5",

    # --- Inventory matching -----------------------------------------------
    # EARLY-PICKUP QUERY (recommended, see README).
    # Tesla does not publish a separate "early delivery" endpoint. That view in
    # your account is the same inventory API called with your order's own
    # parameters. Open it once, copy the "query" parameter out of DevTools >
    # Network, and paste the raw JSON string here. Everything below is then
    # ignored and you are polling exactly the list your account shows you.
    "inventory_query_raw": "",

    # Fallback used only when inventory_query_raw is empty. Deliberately broad:
    # Tesla's internal TRIM/PAINT/INTERIOR option codes churn with every model
    # refresh, so we pull wide and narrow down in Python against the
    # human-readable fields in the response.
    "inventory": {
        "model": "my",              # my | m3 | ms | mx | ct
        "condition": "new",         # new | used
        "options": {},              # left empty on purpose; see note above
        "market": "US",
        "language": "en",
        "super_region": "north america",
        "zip": "36695",             # Alabama registration address
        "lat": 30.6510,
        "lng": -88.2200,
        "range": 0,                 # 0 = nationwide; results are annotated, not filtered
        "count": 50,                # page size
        "max_pages": 6,             # hard stop on pagination
    },

    # What actually triggers an alert. Paint is intentionally unfiltered.
    "inventory_filters": {
        "max_price": 75000,                          # None to disable
        "require_trim_contains": "Performance",      # None to disable
        "require_interior_contains": "White",        # None to disable
    },

    # --- Notifications ----------------------------------------------------
    # "ntfy" is the least-effort option: install the ntfy app, subscribe to a
    # private topic name, and push arrives on your phone with no credentials.
    # Multiple channels can run at once. SMS carries a short headline; ntfy or
    # email carries the full detail. Carrier email-to-SMS gateways (@txt.att.net,
    # @tmomail.net, @vtext.com) are shut down or shutting down, so real SMS goes
    # through Twilio.
    "notify": {
        "methods": ["ntfy", "sms"],   # any of: ntfy, sms, email, stdout

        "ntfy_topic": os.environ.get("NTFY_TOPIC", "jet-tesla-CHANGEME"),
        "ntfy_server": "https://ntfy.sh",

        "twilio_sid": os.environ.get("TWILIO_ACCOUNT_SID", ""),
        "twilio_token": os.environ.get("TWILIO_AUTH_TOKEN", ""),
        "twilio_from": os.environ.get("TWILIO_FROM", ""),   # +1XXXXXXXXXX
        "twilio_to": os.environ.get("TWILIO_TO", ""),       # your cell, +1XXXXXXXXXX
        "sms_max_chars": 300,          # ~2 segments; keeps per-alert cost predictable

        "email_to": "",
        "email_from": "",
        "smtp_host": "smtp.gmail.com",
        "smtp_port": 587,
        "smtp_user": "",
        "smtp_pass": os.environ.get("SMTP_APP_PASSWORD", ""),
    },

    # --- Runtime ----------------------------------------------------------
    "state_file": "tesla_state.json",
    "timeout": 20,
    # Set by the "debug" checkbox on a manual run. Prints everything Tesla
    # returned, then exits without saving state or sending notifications.
    "dry_run": os.environ.get("DRY_RUN", "").lower() == "true",
    "notify_on_first_run": True,    # sends a snapshot on run #1 so you can confirm
                                    # setup worked. Set to False after that.
}

# Order fields worth watching. Dotted paths are resolved safely; a missing path
# is simply skipped. Add or remove lines freely.
ORDER_WATCH_PATHS = {
    "Order status":        "order.orderStatus",
    "VIN":                 "order.vin",
    "Delivery window":     "detail.tasks.scheduling.deliveryWindowDisplay",
    "Delivery appointment": "detail.tasks.scheduling.apptDateTimeAddressStr",
    "Delivery center":     "detail.tasks.scheduling.deliveryAddressTitle",
    "Routing status":      "detail.tasks.registration.orderDetails.vehicleRoutingLocation",
    "Reservation date":    "detail.tasks.registration.orderDetails.reservationDate",
    "Order booked date":   "detail.tasks.registration.orderDetails.orderBookedDate",
    "ETA to delivery ctr": "detail.tasks.registration.orderDetails.vehicleETADisplay",
    "Final payment due":   "detail.tasks.finalPayment.data.amountDue",
    "Financing status":    "detail.tasks.financing.status",
    "Insurance status":    "detail.tasks.insurance.status",
    "Trade-in status":     "detail.tasks.tradeIn.status",
}

TESLA_AUTH_URL = "https://auth.tesla.com/oauth2/v3/token"
TESLA_ORDERS_URL = "https://owner-api.teslamotors.com/api/1/users/orders"
TESLA_TASKS_URL = "https://akamai-apigateway-vfx.tesla.com/tasks"
TESLA_INVENTORY_URL = "https://www.tesla.com/inventory/api/v4/inventory-results"

# Registration address, used for the obtainability note. Not a filter.
HOME_ZIP = "36695"
HOME_LAT, HOME_LNG = 30.6510, -88.2200

# Rough travel tiers relative to the Alabama registration address.
STATE_TIERS = {
    "in-state": {"AL"},
    "neighboring": {"FL", "MS", "GA", "LA", "TN"},
    "regional": {"SC", "NC", "AR", "TX", "KY", "MO", "VA", "OK"},
}

USER_AGENT = (
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 "
    "(KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36"
)


# ============================================================================
# Helpers
# ============================================================================

def dig(obj, path, default=None):
    """Resolve a dotted path through nested dicts/lists without raising."""
    cur = obj
    for part in path.split("."):
        if isinstance(cur, dict):
            cur = cur.get(part)
        elif isinstance(cur, list) and part.isdigit():
            idx = int(part)
            cur = cur[idx] if idx < len(cur) else None
        else:
            return default
        if cur is None:
            return default
    return cur


def hash_value(value):
    """Short digest used for change detection without storing the value itself.

    Order details (VIN, delivery appointment and address, amount due) are
    personal, and the state file is committed to a public repo. Storing digests
    means the file can detect that something changed without publishing what.
    """
    return hashlib.sha256(str(value).encode("utf-8")).hexdigest()[:16]


def hash_summary(summary):
    return {label: hash_value(value) for label, value in (summary or {}).items()}


def log(msg):
    stamp = datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M:%SZ")
    print(f"[{stamp}] {msg}", file=sys.stderr)


# ============================================================================
# Auth
# ============================================================================

def get_access_token(cfg):
    """Exchange the long-lived SSO refresh token for a short-lived access token."""
    if not cfg["refresh_token"]:
        return None
    payload = {
        "grant_type": "refresh_token",
        "client_id": "ownerapi",
        "refresh_token": cfg["refresh_token"],
        "scope": "openid email offline_access",
    }
    resp = requests.post(
        TESLA_AUTH_URL,
        json=payload,
        headers={"User-Agent": USER_AGENT},
        timeout=cfg["timeout"],
    )
    resp.raise_for_status()
    return resp.json().get("access_token")


# ============================================================================
# Order tracking
# ============================================================================

def fetch_orders(token, cfg):
    resp = requests.get(
        TESLA_ORDERS_URL,
        headers={"Authorization": f"Bearer {token}", "User-Agent": USER_AGENT},
        timeout=cfg["timeout"],
    )
    resp.raise_for_status()
    return resp.json().get("response", [])


def fetch_order_detail(token, reference_number, cfg):
    params = {
        "deviceLanguage": cfg["device_language"],
        "deviceCountry": cfg["device_country"],
        "referenceNumber": reference_number,
        "appVersion": cfg["app_version"],
    }
    resp = requests.get(
        TESLA_TASKS_URL,
        params=params,
        headers={"Authorization": f"Bearer {token}", "User-Agent": USER_AGENT},
        timeout=cfg["timeout"],
    )
    resp.raise_for_status()
    return resp.json()


def summarize_order(order, detail):
    """Flatten the watched order fields into a comparable dict."""
    combined = {"order": order, "detail": detail}
    summary = {}
    for label, path in ORDER_WATCH_PATHS.items():
        value = dig(combined, path)
        if value is not None:
            summary[label] = str(value)
    return summary


def collect_order(token, cfg):
    orders = fetch_orders(token, cfg)
    if not orders:
        log("No orders found on this account.")
        return None

    rn = cfg["reference_number"]
    if rn:
        match = next((o for o in orders if o.get("referenceNumber") == rn), None)
        if match is None:
            log(f"Reference number {rn} not found; falling back to first order.")
            match = orders[0]
    else:
        match = orders[0]

    detail = fetch_order_detail(token, match.get("referenceNumber"), cfg)
    return summarize_order(match, detail)


# ============================================================================
# Inventory matching
# ============================================================================

def build_inventory_query(cfg):
    if cfg["inventory_query_raw"]:
        return cfg["inventory_query_raw"]

    inv = cfg["inventory"]
    query = {
        "query": {
            "model": inv["model"],
            "condition": inv["condition"],
            "options": inv["options"],
            "arrangeby": "Price",
            "order": "asc",
            "market": inv["market"],
            "language": inv["language"],
            "super_region": inv["super_region"],
            "lng": inv["lng"],
            "lat": inv["lat"],
            "zip": inv["zip"],
            "range": inv["range"],
        },
        "offset": 0,
        "count": inv["count"],
        "outsideOffset": 0,
        "outsideSearch": False,
    }
    return json.dumps(query, separators=(",", ":"))


def fetch_inventory(cfg, token=None):
    """Page through inventory results. Sends the bearer token when available so
    an account-scoped early-pickup query still resolves."""
    base_query = json.loads(build_inventory_query(cfg))
    page_size = base_query.get("count", 50)
    max_pages = cfg["inventory"].get("max_pages", 6)

    headers = {
        "User-Agent": USER_AGENT,
        "Referer": "https://www.tesla.com/inventory/new/my",
    }
    if token:
        headers["Authorization"] = f"Bearer {token}"

    collected = []
    for page in range(max_pages):
        base_query["offset"] = page * page_size
        encoded = urllib.parse.quote(json.dumps(base_query, separators=(",", ":")))
        resp = requests.get(
            f"{TESLA_INVENTORY_URL}?query={encoded}",
            headers=headers,
            timeout=cfg["timeout"],
        )
        resp.raise_for_status()
        data = resp.json()

        results = data.get("results", data)
        if isinstance(results, dict):
            results = (results.get("exact") or []) + (results.get("approximate") or [])
        results = results or []
        collected.extend(results)

        if len(results) < page_size:
            break
        time.sleep(1)  # be polite between pages

    return collected


def extract_interior(car):
    """Interior name lives in different places depending on the response shape."""
    candidates = []
    for key in ("INTERIOR", "INTERIOR_COMBO", "PREMIUM_INTERIOR"):
        value = car.get(key)
        if isinstance(value, list):
            candidates.extend(str(v) for v in value)
        elif value:
            candidates.append(str(value))
    for key in ("InteriorCombo", "InteriorName"):
        if car.get(key):
            candidates.append(str(car[key]))
    specs = car.get("OptionCodeData") or []
    if isinstance(specs, list):
        for spec in specs:
            if isinstance(spec, dict) and str(spec.get("group", "")).upper() == "INTERIOR":
                for field in ("name", "long_name", "description"):
                    if spec.get(field):
                        candidates.append(str(spec[field]))
    return " ".join(candidates)


def haversine_miles(lat1, lng1, lat2, lng2):
    from math import radians, sin, cos, asin, sqrt
    lat1, lng1, lat2, lng2 = map(radians, (lat1, lng1, lat2, lng2))
    a = sin((lat2 - lat1) / 2) ** 2 + cos(lat1) * cos(lat2) * sin((lng2 - lng1) / 2) ** 2
    return 2 * 3958.8 * asin(sqrt(a))


def obtainability_note(vehicle):
    """Advisory only. Never filters anything out."""
    state = (vehicle.get("state") or "").upper()

    if vehicle.get("distance_mi") is not None:
        distance = f"~{vehicle['distance_mi']:,.0f} mi from {HOME_ZIP}"
    else:
        distance = f"distance from {HOME_ZIP} unknown"

    if state in STATE_TIERS["in-state"]:
        verdict = "in Alabama, simplest path"
    elif state in STATE_TIERS["neighboring"]:
        verdict = "neighboring state, routine for an AL registration"
    elif state in STATE_TIERS["regional"]:
        verdict = "long haul, weigh transport fee against a pickup trip"
    elif state:
        verdict = "cross-country, Tesla can move it but expect added time and cost"
    else:
        verdict = "location not reported"

    return f"{distance} ({verdict})"


def summarize_vehicle(car):
    price = car.get("InventoryPrice") or car.get("Price")
    lat = car.get("Latitude") or car.get("latitude")
    lng = car.get("Longitude") or car.get("longitude")
    distance = None
    if lat and lng:
        try:
            distance = haversine_miles(HOME_LAT, HOME_LNG, float(lat), float(lng))
        except (TypeError, ValueError):
            distance = None

    vehicle = {
        "vin": car.get("VIN"),
        "trim": car.get("TrimName") or " ".join(car.get("TRIM", []) or []),
        "paint": " ".join(car.get("PAINT", []) or []),
        "interior": extract_interior(car),
        "wheels": " ".join(car.get("WHEELS", []) or []),
        "year": car.get("Year"),
        "odometer": car.get("Odometer"),
        "price": price,
        "city": car.get("City"),
        "state": car.get("StateProvince"),
        "distance_mi": distance,
        "url": f"https://www.tesla.com/my/order/{car.get('VIN')}" if car.get("VIN") else None,
    }
    vehicle["note"] = obtainability_note(vehicle)
    return vehicle


def passes_filters(vehicle, filters):
    if filters.get("max_price") and vehicle["price"]:
        if float(vehicle["price"]) > filters["max_price"]:
            return False
    need_trim = filters.get("require_trim_contains")
    if need_trim and need_trim.lower() not in (vehicle["trim"] or "").lower():
        return False
    need_interior = filters.get("require_interior_contains")
    if need_interior and need_interior.lower() not in (vehicle["interior"] or "").lower():
        return False
    return True


def report_inventory(cars, matches, cfg):
    """Always log enough to tell 'nothing new' apart from 'matching nothing'."""
    mode = "early-pickup query" if cfg["inventory_query_raw"] else "fallback query"
    log(f"Inventory: {mode} returned {len(cars)} vehicles, {len(matches)} passed filters.")
    log(f"Filters: {cfg['inventory_filters']}")

    if not cars:
        log("Tesla returned nothing at all. The query itself may be malformed.")
        return

    sample = cars if cfg["dry_run"] else cars[:5]
    label = "ALL VEHICLES RETURNED" if cfg["dry_run"] else "Sample of raw values"
    log(f"--- {label} ---")
    for car in sample:
        vehicle = summarize_vehicle(car)
        verdict = "MATCH" if passes_filters(vehicle, cfg["inventory_filters"]) else "no"
        log(
            f"  [{verdict}] {vehicle['vin']} | trim={vehicle['trim']!r} | "
            f"interior={vehicle['interior']!r} | paint={vehicle['paint']!r} | "
            f"price={vehicle['price']!r} | {vehicle['city']}, {vehicle['state']}"
        )
    if not cfg["dry_run"] and len(cars) > 5:
        log(f"  ... {len(cars) - 5} more. Re-run with the debug box ticked to see all.")

    if cars and not matches:
        log("Nothing passed. Compare the trim/interior values above against your "
            "filters; if Tesla renamed them, loosen the filter and re-run.")


def collect_inventory(cfg, token=None):
    cars = fetch_inventory(cfg, token=token)
    matches = {}
    for car in cars:
        vehicle = summarize_vehicle(car)
        if vehicle["vin"] and passes_filters(vehicle, cfg["inventory_filters"]):
            matches[vehicle["vin"]] = vehicle
    report_inventory(cars, matches, cfg)
    return matches


# ============================================================================
# State and diffing
# ============================================================================

def load_state(path):
    if not os.path.exists(path):
        return {"order": {}, "inventory": {}}
    with open(path, "r") as fh:
        return json.load(fh)


def save_state(path, state):
    with open(path, "w") as fh:
        json.dump(state, fh, indent=2, sort_keys=True)


def diff_order(old_hashes, new_summary):
    """old_hashes maps label -> digest. new_summary maps label -> live value.

    Only the new value is shown, since the previous one was never stored.
    """
    changes = []
    for label, value in (new_summary or {}).items():
        previous = (old_hashes or {}).get(label)
        current = hash_value(value)
        if previous is None:
            changes.append(f"{label}: {value}")
        elif previous != current:
            changes.append(f"{label}: now {value}")
    return changes


def diff_inventory(old, new):
    added, price_changes = [], []
    for vin, vehicle in (new or {}).items():
        previous = (old or {}).get(vin)
        if previous is None:
            added.append(vehicle)
        elif str(previous.get("price")) != str(vehicle.get("price")):
            price_changes.append((previous, vehicle))
    return added, price_changes


# ============================================================================
# Notification
# ============================================================================

def send_ntfy(subject, body, settings, timeout):
    requests.post(
        f"{settings['ntfy_server']}/{settings['ntfy_topic']}",
        data=body.encode("utf-8"),
        headers={"Title": subject, "Priority": "default", "Tags": "car"},
        timeout=timeout,
    )


def send_sms(body, settings, timeout):
    """Twilio REST API directly. No SDK needed for a single POST."""
    sid = settings["twilio_sid"]
    if not (sid and settings["twilio_token"] and settings["twilio_from"] and settings["twilio_to"]):
        log("SMS skipped: Twilio settings incomplete.")
        return
    body = body[: settings.get("sms_max_chars", 300)]
    resp = requests.post(
        f"https://api.twilio.com/2010-04-01/Accounts/{sid}/Messages.json",
        auth=(sid, settings["twilio_token"]),
        data={"From": settings["twilio_from"], "To": settings["twilio_to"], "Body": body},
        timeout=timeout,
    )
    resp.raise_for_status()


def send_email(subject, body, settings):
    msg = EmailMessage()
    msg["Subject"] = subject
    msg["From"] = settings["email_from"]
    msg["To"] = settings["email_to"]
    msg.set_content(body)
    with smtplib.SMTP(settings["smtp_host"], settings["smtp_port"]) as server:
        server.starttls()
        server.login(settings["smtp_user"], settings["smtp_pass"])
        server.send_message(msg)


def notify(subject, long_body, short_body, cfg):
    """Fan out to every configured channel. One failing channel must not stop
    the others, or a Twilio outage would cost you an order update."""
    settings = cfg["notify"]
    for method in settings.get("methods", ["stdout"]):
        try:
            if method == "stdout":
                print(f"{subject}\n\n{long_body}")
            elif method == "ntfy":
                send_ntfy(subject, long_body, settings, cfg["timeout"])
            elif method == "sms":
                send_sms(f"{subject}: {short_body}", settings, cfg["timeout"])
            elif method == "email":
                send_email(subject, long_body, settings)
            else:
                log(f"Unknown notify method: {method}")
        except Exception as exc:  # noqa: BLE001 - never let one channel kill the rest
            log(f"Notify via {method} failed: {exc}")


def format_vehicle(vehicle):
    price = f"${int(float(vehicle['price'])):,}" if vehicle.get("price") else "price n/a"
    location = ", ".join(p for p in [vehicle.get("city"), vehicle.get("state")] if p)
    return (
        f"- {vehicle.get('year')} {vehicle.get('trim')} | {vehicle.get('paint')}\n"
        f"  {vehicle.get('interior')} | {price} | {location}\n"
        f"  {vehicle.get('note')}\n"
        f"  {vehicle.get('url')}"
    )


# ============================================================================
# Main
# ============================================================================

def main():
    cfg = CONFIG
    state = load_state(cfg["state_file"])
    first_run = not state.get("order_hashes") and not state.get("inventory")
    sections = []      # full detail, for ntfy/email
    headlines = []     # terse, for SMS
    token = None

    # Order status
    try:
        token = get_access_token(cfg)
        if token:
            order = collect_order(token, cfg)
            if order:
                changes = diff_order(state.get("order_hashes"), order)
                if changes:
                    sections.append("ORDER UPDATE\n" + "\n".join(changes))
                    headlines.append(
                        "Order: " + ("; ".join(changes) if len(changes) <= 2
                                     else f"{len(changes)} updates")
                    )
                state["order_hashes"] = hash_summary(order)
        else:
            log("No refresh token set; skipping order tracking.")
    except requests.HTTPError as exc:
        log(f"Order check failed: {exc}")
    except requests.RequestException as exc:
        log(f"Order check network error: {exc}")

    # Inventory matches
    try:
        inventory = collect_inventory(cfg, token=token)
        added, price_changes = diff_inventory(state.get("inventory"), inventory)
        if added:
            sections.append(
                f"NEW INVENTORY MATCHES ({len(added)})\n"
                + "\n".join(format_vehicle(v) for v in added)
            )
            closest = min(
                added,
                key=lambda v: v["distance_mi"] if v.get("distance_mi") is not None else 1e9,
            )
            where = ", ".join(p for p in [closest.get("city"), closest.get("state")] if p)
            headlines.append(
                f"{len(added)} new match{'es' if len(added) > 1 else ''}"
                + (f", closest {where}" if where else "")
            )
        if price_changes:
            lines = [
                f"- {new['trim']} {new['vin'][-6:]}: ${float(old['price']):,.0f} -> ${float(new['price']):,.0f}"
                for old, new in price_changes
                if old.get("price") and new.get("price")
            ]
            if lines:
                sections.append("PRICE CHANGES\n" + "\n".join(lines))
                headlines.append(f"{len(lines)} price change{'s' if len(lines) > 1 else ''}")
        state["inventory"] = inventory
    except requests.RequestException as exc:
        log(f"Inventory check failed: {exc}")

    if cfg["dry_run"]:
        log("DRY RUN: state not saved, no notifications sent.")
        log(f"Would have reported {len(sections)} section(s).")
        return

    state["last_run"] = datetime.now(timezone.utc).isoformat()
    save_state(cfg["state_file"], state)

    if not sections:
        log("No changes.")
        return

    if first_run and not cfg["notify_on_first_run"]:
        log("Baseline captured on first run; notification suppressed.")
        return

    long_body = "\n\n".join(sections)
    short_body = " | ".join(headlines) + " (detail in ntfy)"
    notify("Tesla update", long_body, short_body, cfg)
    log(f"Notified: {len(sections)} section(s).")


if __name__ == "__main__":
    main()
