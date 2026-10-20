# External API

A read-only API for other applications (bots, spreadsheets, tools) to look up what is for sale in the Free Market:
items, prices, which room and channel a shop is in, and which slot in the shop an item is in.

- Base URL: `https://test.moople.markets/api/external/v1`
- Method: `GET` only. The API never changes anything on the site.
- Format: JSON. Times are ISO-8601 in UTC. Prices are in mesos.

Contents: [Authentication](#authentication) · [Endpoints](#endpoints) · [Recipes](#recipes) ·
[The listing object](#the-listing-object) · [Paging](#paging) · [Errors and limits](#errors-and-limits) ·
[Example code](#example-code)

## Authentication

Every request needs an API token. An administrator makes one in **Admin › API tokens**; it starts with `mmr_` and is
shown only once, so store it somewhere safe. Send it in either header:

```
Authorization: Bearer mmr_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
X-API-Key: mmr_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Ingest tokens (`mmi_...`) and being signed in to the site do not work here. A token that is switched off or deleted is
refused straight away. Check a token with:

```
GET /api/external/v1/ping
```

```json
{"ok": true, "token": "Price bot", "world": "Classic"}
```

## Endpoints

| Endpoint | Use it for |
|---|---|
| [`GET /items`](#get-items) | Find items by name or number, with how many are for sale, the cheapest price and recent average sales |
| [`GET /listings`](#get-listings) | Every listing that is for sale now, with price, room, channel, seller and slot, filtered however you need |
| `GET /ping` | Check that a token works |

What's included: listings that are for sale now, with a price the scanner could read, that aren't a price outlier
(listed at many times the item's cheapest price), of items that are listed publicly on the site (items still waiting in
Admin › Item review are left out). This is the same data as the market page.

### GET /items

Search items. Each result is one item with a summary of its market.

| Parameter | Default | Meaning |
|---|---|---|
| `q` | | Name contains this text (any case), **or**, if it is only digits, the game item number. |
| `category` | | Only this category (any case): `Accessory`, `Cap`, `Cape`, `Cash`, `Coat`, `Consume`, `Etc`, `Glove`, `Longcoat`, `Pants`, `Ring`, `Setup`, `Shield`, `Shoes`, `Weapon` |
| `listed_only` | `true` | `true`: only items with at least one listing now. `false`: also items nobody is selling now. |
| `limit` | `50` | Results per page, 1 to 500. |
| `offset` | `0` | Skip this many results (see [Paging](#paging)). |

Results are sorted by name.

```
GET /api/external/v1/items?q=sauna robe
```

```json
{
  "total": 2, "limit": 50, "offset": 0,
  "items": [
    {
      "id": 41,
      "game_id": 1050010,
      "name": "Blue Sauna Robe",
      "category": "Longcoat",
      "is_equipment": true,
      "is_cash": false,
      "npc_price": 5400.0,
      "listings": 330,
      "cheapest_unit_price": 9999.0,
      "average_sale_24h": 30831.9,
      "average_sale_7d": 31800.2
    },
    { "id": 108, "game_id": 1051004, "name": "Red Sauna Robe", "...": "..." }
  ]
}
```

| Field | Meaning |
|---|---|
| `id` | This site's item ID (use with `/listings?item_id=`). |
| `game_id` | The game's item number (use with `/listings?game_id=`). |
| `npc_price` | What an NPC shop pays for one, or `null` if unknown. |
| `listings` | How many listings it has now (`0` with `listed_only=false`). |
| `cheapest_unit_price` | The cheapest price per item now, or `null` when nothing is listed. |
| `average_sale_24h`, `average_sale_7d` | Average price per item it sold for in the last 24 hours / 7 days, or `null` with no sales. |

### GET /listings

Every listing for sale now, with every filter optional. With no filters you get the whole market, cheapest first.

| Parameter | Meaning |
|---|---|
| `q` | Item name contains this text (any case), or, if it is only digits, the game item number. |
| `game_id` | Only this item, by the game's item number. |
| `item_id` | Only this item, by this site's item ID. |
| `room` | Only shops in this Free Market room (the `n` in "Free Market n"). |
| `channel` | Only shops on this channel. |
| `seller` | Only this seller's shop (exact name, any case). |
| `min_price`, `max_price` | Price **per item** at least / at most, in mesos. |
| `stats` | Equipment only: stat minimums, see [Filtering by stats](#filtering-by-stats). |
| `min_slots` | Equipment only: at least this many upgrade slots left. |
| `sort` | `price` (default): cheapest per item first. `recent`: most recently seen first. |
| `limit` | Results per page, 1 to 500. Default `100`. |
| `offset` | Skip this many results (see [Paging](#paging)). Default `0`. |

Filters combine: a listing must match all of them.

```
GET /api/external/v1/listings?game_id=1050010&room=9&limit=1
```

```json
{
  "total": 12, "limit": 1, "offset": 0,
  "listings": [ { "...": "a listing, see below" } ]
}
```

#### Filtering by stats

`stats` is a comma-separated list of `STAT:minimum`. A piece is included only if it meets **every** minimum. A stat the
piece doesn't have counts as 0.

```
stats=W.ATK:50
stats=W.ATK:50,LUK:4
stats=dex:3, acc:2          (any case, spaces are fine)
```

Stat names: `STR`, `DEX`, `INT`, `LUK`, `HP`, `MP`, `W.ATK`, `M.ATK`, `W.DEF`, `M.DEF`, `ACC`, `AVOID`, `SPEED`,
`JUMP`, `CRIT%`, `CRITDMG`.

URL-encode the value when building URLs by hand: `%` in `CRIT%` becomes `%25`, and a space becomes `%20`. Most HTTP
libraries do this for you when you pass parameters separately (see [Example code](#example-code)).

`min_slots` keeps pieces with at least that many upgrade slots left. A piece with fewer slots than the item normally has
has been scrolled, so `min_slots` set to the item's normal slot count means "clean pieces only".

The format is the same as the stat filter on the site's market page.

## Recipes

| I want... | Request |
|---|---|
| An item's details and cheapest price | `/items?q=1050010` (digits match the game item number) |
| Every listing of one item | `/listings?game_id=1050010&limit=500`, then page (see [Paging](#paging)) |
| The 5 cheapest of an item | `/listings?game_id=1050010&limit=5` |
| An item under a price | `/listings?game_id=2000006&max_price=1000` |
| Work Gloves with specific stats | `/listings?game_id=1082000&stats=DEX:3,W.ATK:2` |
| The cheapest clean (unscrolled) Work Gloves (5 slots when new) | `/listings?game_id=1082000&min_slots=5&limit=1` |
| Search by name | `/listings?q=work gloves` |
| Everything in one room and channel | `/listings?room=8&channel=11&limit=500` |
| Everything one seller has | `/listings?seller=Brickopolis&limit=500` |
| What appeared most recently | `/listings?sort=recent&limit=50` |
| All weapons for sale | `/items?category=Weapon&limit=500` |

### Finding a listing in game

Each listing tells you where to go and what to click:

1. `location.channel`: change to this channel.
2. `location.room`: go to Free Market room `room`.
3. `shop.seller` / `shop.title`: find this player's shop.
4. `shop.slot`: the item's position in the shop, **counting from 0**. `slot: 0` is the first item, `slot: 8` the ninth.

## The listing object

```json
{
  "id": 394835,
  "item": {
    "id": 41,
    "game_id": 1050010,
    "name": "Blue Sauna Robe",
    "category": "Longcoat",
    "is_equipment": true,
    "is_cash": false,
    "npc_price": 5400.0
  },
  "price": { "unit": 9999.0, "total": 9999, "quantity": 1, "bundles_left": 1 },
  "location": { "world": "Classic", "map": "Free Market 9", "room": 9, "channel": 8, "x": null, "y": null },
  "shop": { "seller": "Brickopolis", "title": "hi", "slot": 8 },
  "equipment": {
    "stats": { "STR": 1, "DEX": 1, "INT": 1, "LUK": 1, "HP": 10, "MP": 10, "W.DEF": 75 },
    "upgrade_slots": 8
  },
  "first_seen_at": "2026-10-19T19:19:02.623677+00:00",
  "last_seen_at": "2026-10-20T00:07:46.606476+00:00"
}
```

| Field | Meaning |
|---|---|
| `id` | The listing's ID. It stays the same while the listing exists. |
| `item` | The item, repeated on every listing (fields as in [`/items`](#get-items)). |
| `price.unit` | Price per single item. Use this to compare listings. |
| `price.total` | Price of one bundle (what you pay per purchase). |
| `price.quantity` | Items in one bundle. |
| `price.bundles_left` | How many bundles of this size are still for sale. A potion listing of `quantity: 100`, `bundles_left: 3` has 300 potions. |
| `location.map` | The map name, e.g. `Free Market 9`. |
| `location.room` | The Free Market room number, or `null` outside the Free Market rooms. |
| `location.channel` | The channel. |
| `location.x`, `location.y` | The shop's position on the map. Usually `null`: the scanner doesn't send it. |
| `shop.seller` | The shop owner's character name. |
| `shop.title` | The shop's title, as typed by the seller. |
| `shop.slot` | The item's position in the shop, counting from 0. |
| `equipment` | Equipment only (`null` for everything else). |
| `equipment.stats` | This piece's stats, e.g. `{"W.ATK": 52, "LUK": 4}`. Stats it doesn't have are left out. |
| `equipment.upgrade_slots` | Upgrade slots left on this piece. |
| `first_seen_at` | When this listing was first seen. |
| `last_seen_at` | When a scan last confirmed it was still for sale. |

## Paging

Both lists return `total` (how many match), `limit` and `offset`. To get everything, start at `offset=0` and add
`limit` until `offset` reaches `total`:

```
/listings?game_id=1050010&limit=500&offset=0      -> total: 1240, listings 1-500
/listings?game_id=1050010&limit=500&offset=500    -> listings 501-1000
/listings?game_id=1050010&limit=500&offset=1000   -> listings 1001-1240
```

The market is live. If a scan arrives between two pages, a listing can move between pages and be skipped or returned
twice. Remove duplicates by `id`. Most items have fewer than 500 listings and fit in a single request.

## Errors and limits

Errors are JSON: `{"detail": "<what went wrong>"}`.

| Status | When |
|---|---|
| `200` | OK. A search with no matches is still `200`, with `total: 0` and an empty list. |
| `401` | No token, an unknown token, or one that is switched off or deleted. |
| `404` | The path doesn't exist (e.g. `/items/1050010`: look items up with `/items?q=1050010`). |
| `405` | Anything but `GET`. |
| `422` | A bad parameter: an unknown stat name, `limit` over 500, a word where a number belongs. The message says which. |
| `429` | Over the rate limit. Wait the number of seconds in the `Retry-After` header. |

**Rate limit:** 120 requests a minute per token. With `limit=500` that is up to 60,000 listings a minute.

**How fresh is it:** listings update as the scanner sends shops, usually within seconds. `last_seen_at` says when each
listing was last confirmed.

## Example code

### curl

```bash
curl -H "Authorization: Bearer $MMR_TOKEN" \
  "https://<your site>/api/external/v1/listings?game_id=1082000&stats=DEX:3&limit=5"
```

### Python

```python
import time
import requests

BASE = "https://<your site>/api/external/v1"
session = requests.Session()
session.headers["Authorization"] = "Bearer mmr_..."


def get(path, **params):
    while True:
        r = session.get(f"{BASE}{path}", params=params, timeout=30)
        if r.status_code == 429:  # over the rate limit: wait and try again
            time.sleep(int(r.headers.get("Retry-After", "5")))
            continue
        r.raise_for_status()
        return r.json()


def all_listings(**filters):
    """Every listing matching the filters, following the pages."""
    seen, offset = {}, 0
    while True:
        page = get("/listings", **filters, limit=500, offset=offset)
        for listing in page["listings"]:
            seen[listing["id"]] = listing  # by id: drops a listing that moved between pages
        offset += page["limit"]
        if offset >= page["total"]:
            return sorted(seen.values(), key=lambda l: l["price"]["unit"])


# The cheapest Work Gloves with at least 3 DEX that haven't been scrolled
for l in all_listings(q="work gloves", stats="DEX:3", min_slots=5)[:5]:
    print(f'{l["price"]["unit"]:>10,.0f}  ch{l["location"]["channel"]} FM{l["location"]["room"]}'
          f'  {l["shop"]["seller"]} slot {l["shop"]["slot"]}  {l["equipment"]["stats"]}')
```

### JavaScript

```js
const BASE = "https://<your site>/api/external/v1";
const headers = { Authorization: "Bearer mmr_..." };

async function listings(params) {
  const url = `${BASE}/listings?${new URLSearchParams(params)}`;
  const r = await fetch(url, { headers });
  if (!r.ok) throw new Error(`${r.status}: ${(await r.json()).detail}`);
  return r.json();
}

const { total, listings: cheapest } = await listings({ game_id: 2000006, max_price: 1000, limit: 10 });
console.log(`${total} Mana Elixir listings at 1,000 or less`);
for (const l of cheapest) {
  console.log(l.price.unit, `ch${l.location.channel} FM${l.location.room}`, l.shop.seller, `slot ${l.shop.slot}`);
}
```

Keep the token on a server. Don't put it in a web page or app that other people can open, since anyone who has it can
use it until it is switched off in Admin › API tokens.
