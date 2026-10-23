# Ingest API

How a scanner sends what it reads in the Free Market to Moople Markets: the shops it saw, and which shops were still
there when it walked through a room. Everything sent here goes through the site's import queue and appears on the
market, usually within seconds.

- Base URL: `https://moople.markets/api/ingest`
- Method: `POST` with a JSON body (`Content-Type: application/json`), except `GET /ping`.
- Times: ISO-8601, or how long ago in milliseconds (see [Times](#times)). Prices are in mesos.

Contents: [Authentication](#authentication) · [How it fits together](#how-it-fits-together) ·
[Sending a shop](#sending-a-shop) · [The item line](#the-item-line) · [Reporting a pass](#reporting-a-pass) ·
[Sending in batches](#sending-in-batches) · [Times](#times) · [Responses](#responses) ·
[The import queue](#the-import-queue) · [Errors and retrying](#errors-and-retrying) · [Logs](#logs-optional) ·
[Example code](#example-code) · [Checklist](#checklist)

## Authentication

Every request needs an ingest token, issued by the Moople Markets team. It starts with `mmi_`; keep it secret. Send it
as a bearer token:

```
Authorization: Bearer mmi_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Ingest tokens only work here; the read-only external API uses different tokens (`mmr_...`). Check the address and token
with:

```
GET https://moople.markets/api/ingest/ping
```

```json
{"ok": true, "world": "Classic", "batch": 200, "token": "Game PC scanner"}
```

`batch` is the most events one [batch request](#sending-in-batches) may carry.

## How it fits together

A scanner sends two kinds of events:

| Event | When to send it | What the site does with it |
|---|---|---|
| **shop** | Each time it reads a player shop: owner, title, where it is, and every item line | New listings appear, changed prices are recorded, lines that are gone end, sold lines become sales |
| **map_done** (a *pass*) | Each time it finishes walking through one Free Market room on one channel, with the owners of every shop it saw there | Shops last seen in that room and channel that are missing count a missed pass; enough missed passes in a row and the shop is closed, its items no longer for sale |

Send the shops of a pass first, then the pass. Send everything in the order it happened: the site applies events in
order, and a shop scan older than one already stored for the same seller is ignored.

Resending is safe: a shop scan the site already has comes back as `duplicate`, `older` or `unchanged` and changes
nothing, and a rescan showing a shop exactly as before only updates its "last seen" time.

## Sending a shop

```
POST https://moople.markets/api/ingest/shop
```

```json
{
  "owner": "Brickopolis",
  "title": "cheap pots and gloves",
  "map_id": 80002009,
  "channel": 8,
  "age_ms": 1500,
  "items": [
    { "slot": 0, "item_id": 2000006, "price": 9500, "per_bundle": 100, "bundles": 3 },
    { "slot": 1, "item_id": 1082000, "price": 15000, "per_bundle": 1, "bundles": 1,
      "stats": { "1": 3, "10": 4 }, "upgrade_slots": 5 },
    { "slot": 2, "item_id": 1050010, "price": 9999, "per_bundle": 1, "bundles": 0,
      "stats": { "10": 75 }, "upgrade_slots": 8 }
  ]
}
```

| Field | Required | Meaning |
|---|---|---|
| `owner` | yes | The shop owner's character name, 1 to 64 characters. This identifies the shop: one owner has one shop. |
| `title` | | The shop's title as shown in game, up to 200 characters. |
| `map_id` | | The game's map number where the shop is. Free Market rooms are `80002001` (Free Market 1) to `80002022` (Free Market 22). Needed for passes to close the shop later. |
| `channel` | | The channel, 0 to 100. Needed for passes to close the shop later. |
| `items` | | The shop's item lines, in shop order (up to 200). An empty list means the shop shows nothing. See [The item line](#the-item-line). |
| `age_ms` | | How long ago the shop was read, in milliseconds (0 to 7 days). See [Times](#times). |
| `scanned_at` | | Or: when the shop was read, ISO-8601. Use one of `age_ms` and `scanned_at`. Without either, the time the site receives it. |
| `shop_id`, `owner_id` | | The game's IDs for the shop and its owner, if known. Accepted but not used. |

Other fields are ignored.

**Send every line, in order, every time.** Each scan is compared with the previous one of the same shop: a line that
was there and isn't any more has ended. A scan with an unreadable line (a missing or impossible `item_id`) can't tell
which line went missing, so it ends nothing; the other lines are still stored.

## The item line

| Field | Required | Meaning |
|---|---|---|
| `item_id` | yes | The game's item number, 1000000 to 9999999 (e.g. `2000006`, `1082000`). |
| `price` | yes | Price of **one bundle** in mesos, 1 or more. |
| `per_bundle` | yes | Items in one bundle, 1 to 32767. Equipment is always `1`. |
| `bundles` | yes | Bundles still for sale. `0` means the line has sold out: it is recorded as a sale, never shown as a listing. |
| `slot` | | The line's position in the shop, counting from 0. Lets people find the item in the shop; send it if you can. |
| `stats` | equipment | The piece's stats, keyed by **stat code** (below) as strings: `{"1": 3, "10": 4}` is DEX 3, W.DEF 4. Leave out stats the piece doesn't have, and any that are 0. |
| `upgrade_slots` | equipment | Upgrade slots left on the piece, 0 to 255. |

The price per item is `price / per_bundle`. A potion line `price: 9500, per_bundle: 100, bundles: 3` is 300 potions
at 95 mesos each, sold 100 at a time for 9,500.

A line with stats, or with `upgrade_slots` on an equipment item number (1xxxxxx), is treated as equipment. Send both
for every equipment piece: stats tell pieces apart, and fewer upgrade slots than the item normally has means it was
scrolled. Cash equipment with no stats and no slots is recognised from the item number.

Stat codes:

| Code | Stat | Code | Stat | Code | Stat |
|---|---|---|---|---|---|
| `0` | STR | `6` | SPEED | `12` | ACC |
| `1` | DEX | `7` | JUMP | `13` | AVOID |
| `2` | INT | `9` | M.ATK | `14` | CRIT% |
| `3` | LUK | `10` | W.DEF | `15` | CRITDMG |
| `4` | HP | `11` | M.DEF | `16` | W.ATK |
| `5` | MP | | | | |

Code `8` is not used. Unknown codes, values of 0, and values beyond ±32,767 are left out.

**Lines the site can't use:** a line with a `price` or `per_bundle` out of range is stored without a price (it isn't
shown as for sale) and counted in `invalid`. A line that isn't an object, or has no usable `item_id`, is skipped and
counted in `invalid`. Neither stops the rest of the shop.

## Reporting a pass

After walking through one Free Market room on one channel, report every shop owner seen there:

```
POST https://moople.markets/api/ingest/map_done
```

```json
{ "map_id": 80002009, "channel": 8, "owners": ["Brickopolis", "Haste", "Willnie"], "age_ms": 400 }
```

| Field | Meaning |
|---|---|
| `map_id` | The room's map number, as sent with its shops. |
| `channel` | The channel, as sent with its shops. |
| `owners` | Owner names of every shop seen in the room on this pass (up to 500), as sent in `owner`. Case doesn't matter. |
| `age_ms` / `at` | When the pass finished, as for shops (see [Times](#times)). |

Every shop last scanned in that room and channel whose owner is missing from `owners` counts a missed pass. After a
set number of missed passes in a row, the shop is closed: its items are no longer for sale. A closed shop that shows up
on a later pass counts as open again, and its items return with its next scan.

Some passes are safe to send even when they look odd:

- A pass with an empty `owners` list, in a room where shops are known, is ignored. It usually means the room didn't
  load, not that every shop closed at once.
- A shop scanned after the pass's time (because the pass waited in your queue) doesn't count as missed.

Only report a pass when the scanner really saw the whole room: a pass that missed shops still standing there counts
them as missing.

## Sending in batches

When events pile up (the scanner was offline, or the site asked it to wait), send them together, in order:

```
POST https://moople.markets/api/ingest/batch
```

```json
{
  "events": [
    { "type": "shop", "owner": "Brickopolis", "map_id": 80002009, "channel": 8, "age_ms": 61200, "items": [ ... ] },
    { "type": "shop", "owner": "Haste", "map_id": 80002009, "channel": 8, "age_ms": 60800, "items": [ ... ] },
    { "type": "map_done", "map_id": 80002009, "channel": 8, "owners": ["Brickopolis", "Haste"], "age_ms": 60500 }
  ]
}
```

Each event is a shop or pass body with a `type` of `"shop"` or `"map_done"`. A batch carries 1 to 200 events (the
`batch` value from `/ping`). The events are applied in the order given. The answer has one result per event, in the
same order:

```json
{ "results": [ { "status": "stored", "...": "..." }, { "status": "unchanged", "...": "..." }, { "seen": 2, "...": "..." } ] }
```

An event that is invalid (an unknown `type`, a missing `owner`) gets `{"error": "<why>"}` and the others are still
stored. Don't resend an event that got an error: it will fail the same way.

Batches are also the most efficient way to send when things are busy: one request, one transaction.

## Times

The site needs to know when each shop was read and when each pass finished. There are two ways to say it:

- **`age_ms`** (recommended): how many milliseconds ago it happened, **measured when the request is sent**. A shop read
  at 12:00:00.000 and sent at 12:00:01.500 has `age_ms: 1500`. If it waits in your queue and is sent again at
  12:05:00.000, send `age_ms: 300000`. Compute it again on every attempt. The site dates the event "now minus
  `age_ms`", so neither clock needs to be set correctly. 0 to 604,800,000 (7 days).
- **`scanned_at`** (shops) / **`at`** (passes): an ISO-8601 time. Include the offset (`2026-10-23T18:50:14.918Z` or
  `...-04:00`); a time without one is read in the site's scanner time zone. Times more than 5 minutes in the future
  are treated as now.

With neither, the event is dated when the site receives it, which is wrong for anything that waited in a queue.

Get the order right: within one seller, a scan older than the last one stored is answered `older` and ignored.

## Responses

**A shop** (`/shop`, and a shop's entry in a batch):

```json
{ "status": "stored", "owner": "Brickopolis", "items": 3, "sold": 1, "listings_created": 2, "sales": 1, "invalid": 0 }
```

| `status` | Meaning |
|---|---|
| `stored` | The shop changed (or is new): listings, prices and sales were updated. |
| `unchanged` | The shop looks exactly as last time: only its "last seen" time moved. |
| `duplicate` | A scan of this shop with exactly this time was already stored. Nothing changed. |
| `older` | A newer scan of this shop is already stored. Nothing changed. |
| `queued` | Accepted into the import queue; it will be imported shortly (see below). |

| Field | Meaning |
|---|---|
| `items` | Item lines received. |
| `sold` | Lines with `bundles: 0`. |
| `listings_created` | New listings this scan created. |
| `sales` | Sales this scan recorded. |
| `invalid` | Lines that couldn't be used (see [The item line](#the-item-line)). |

**A pass** (`/map_done`, and a pass's entry in a batch):

```json
{ "shops_on_map": 14, "seen": 12, "closed": ["Somebody"], "reopened": [], "listings_closed": 6 }
```

| Field | Meaning |
|---|---|
| `shops_on_map` | Shops the site knows in that room and channel (`"?"` when queued). |
| `seen` | Owners in the pass. |
| `closed` | Shops this pass closed. |
| `reopened` | Closed shops this pass saw again. |
| `listings_closed` | Listings that ended because their shop closed. |
| `ignored` | Present when the pass was ignored, with the reason (e.g. `"no shops seen"`). |
| `queued` | `true` when accepted into the import queue. |

## The import queue

When the site is busy (statistics being recalculated, a backup, or older events still waiting), it doesn't make the
scanner wait. It accepts the events into its import queue straight away and answers `200` with `"queued": true` on each
result. A queued shop reports `"status": "queued"` and zeros for `listings_created` and `sales`; a queued pass reports
`"shops_on_map": "?"`. The site imports the queue in arrival order, usually within seconds.

**A queued event has been received: don't send it again.** The times it was sent with are kept, so it is dated
correctly when it is imported.

While anything is waiting in the queue, new events are queued behind it, so order is always kept.

## Errors and retrying

| Status | Meaning | What to do |
|---|---|---|
| `200` | Accepted: stored, unchanged, duplicate, older or queued (see each result). | Remove the events from your queue. |
| `401` | Unknown token, or one that is switched off. | Stop and check the token; retrying won't help. |
| `404` | Live import isn't enabled on the site. | Stop and contact the Moople Markets team. |
| `422` | The request itself is malformed: not JSON, a top-level field of the wrong type, `owner` missing on `/shop`, more than 200 events. `detail` says what. | Fix the request; don't resend it unchanged. |
| `503` | The site couldn't accept anything right now. Nothing was stored. | Keep the events, wait the `Retry-After` seconds (5), and send them again with `age_ms` recalculated. |
| Network error, timeout or other `5xx` | Unknown whether it arrived. | Keep the events and send them again later. Resending is safe (see [How it fits together](#how-it-fits-together)). |

Errors are JSON: `{"detail": ...}`.

A scanner should keep a local queue: add each event as it happens, send from the front (one event, or a batch when
several are waiting), and only remove events once the site answered `200`. That way nothing is lost when the site or
the network is down, and the order is kept.

## Logs (optional)

A scanner can send its own log lines, which the site keeps in its scanner log for troubleshooting:

```
POST https://moople.markets/api/ingest/log
```

```json
{ "lines": [ { "level": "info", "message": "Scanning Free Market 9, channel 8", "age_ms": 250 } ] }
```

1 to 1000 lines per request; `level` is a short word (`info`, `warning`, `error`). Answers `{"ok": true, "lines": 1}`.

## Example code

### Python

A small client with a local queue: `record_shop` and `record_pass` add events as they happen, and `flush` sends what
is waiting in batches. Uses the `requests` package.

```python
import time
from collections import deque

import requests

BASE = "https://moople.markets/api/ingest"
TOKEN = "mmi_..."  # keep it out of shared code: read it from a file or environment variable
BATCH = 200

session = requests.Session()
session.headers["Authorization"] = f"Bearer {TOKEN}"
waiting = deque()  # (monotonic time it happened, event without age_ms)


def record_shop(owner, title, map_id, channel, lines):
    """lines: [{"slot", "item_id", "price", "per_bundle", "bundles", "stats"?, "upgrade_slots"?}, ...] in shop order"""
    waiting.append((time.monotonic(), {"type": "shop", "owner": owner, "title": title, "map_id": map_id,
                                       "channel": channel, "items": lines}))


def record_pass(map_id, channel, owners):
    waiting.append((time.monotonic(), {"type": "map_done", "map_id": map_id, "channel": channel,
                                       "owners": list(owners)}))


def flush():
    """Send everything waiting, oldest first. Returns False to try again later (the events stay queued)."""
    while waiting:
        chunk = list(waiting)[:BATCH]
        now = time.monotonic()
        events = [{**event, "age_ms": int((now - at) * 1000)} for at, event in chunk]  # recalculated every attempt
        try:
            answer = session.post(f"{BASE}/batch", json={"events": events}, timeout=30)
        except requests.RequestException:
            return False  # network trouble: try again later
        if answer.status_code == 503 or answer.status_code >= 500:
            time.sleep(int(answer.headers.get("Retry-After", 5)))
            return False
        if answer.status_code in (401, 404):
            raise RuntimeError(f"ingest refused ({answer.status_code}): {answer.text}")
        if answer.status_code == 422:
            raise RuntimeError(f"malformed batch: {answer.text}")  # a bug in the client: fix it
        for result, (_, event) in zip(answer.json()["results"], chunk):
            if "error" in result:
                print("rejected:", event.get("owner") or event["type"], result["error"])
            waiting.popleft()
    return True


# Example: one shop, then the pass through its room
record_shop("Brickopolis", "cheap pots", 80002009, 8, [
    {"slot": 0, "item_id": 2000006, "price": 9500, "per_bundle": 100, "bundles": 3},
    {"slot": 1, "item_id": 1082000, "price": 15000, "per_bundle": 1, "bundles": 1,
     "stats": {"1": 3, "10": 4}, "upgrade_slots": 5},
])
record_pass(80002009, 8, ["Brickopolis"])
flush()
```

### curl

```
curl -X POST https://moople.markets/api/ingest/shop \
  -H "Authorization: Bearer mmi_xxxxxxxx..." -H "Content-Type: application/json" \
  -d '{"owner":"Brickopolis","map_id":80002009,"channel":8,"age_ms":0,
       "items":[{"slot":0,"item_id":2000006,"price":9500,"per_bundle":100,"bundles":3}]}'
```

## Checklist

- Send every item line of a shop, in shop order, with `slot`, every time it is read.
- `price` is per **bundle**; `per_bundle` is items per bundle; `bundles: 0` means sold out.
- Equipment: `stats` keyed by stat code, plus `upgrade_slots`.
- Send `map_id` and `channel` with every shop, and a pass after each room, only when the whole room was seen.
- Date everything with `age_ms`, recalculated on each attempt.
- Keep events in order, keep them until the site answers `200`, and treat `queued` as received.
- On `503`, network errors or `5xx`: wait and resend. On `401`, `404` or `422`: stop and fix.
