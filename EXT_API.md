# External API

A read-only API for other applications (scripts, tools, other sites) to look up what is for sale in the Free Market
(items, prices, which room and channel a shop is in, and which slot in the shop an item is in) and what things sell
for: sale counts and quantities, price percentiles, time to sell, price history, equipment comparables and valuations,
and opportunities.

- Base URL: `https://moople.markets/api/external/v1`
- Method: `GET` only. The API never changes anything on the site.
- Format: JSON. Times are ISO-8601 in UTC. Prices are in mesos.

Contents: [Authentication](#authentication) · [Endpoints](#endpoints) · [Recipes](#recipes) ·
[The listing object](#the-listing-object) · [Paging](#paging) · [Errors and limits](#errors-and-limits) ·
[Example code](#example-code)

## Authentication

Every request needs an API token, issued by the Moople Markets team. It starts with `mmr_`; keep it somewhere safe and
don't share it. Send it in either header:

```
Authorization: Bearer mmr_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
X-API-Key: mmr_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Ingest tokens (`mmi_...`) and being signed in to the site do not work here. A token that is switched off or deleted is
refused straight away. Check a token with:

```
GET https://moople.markets/api/external/v1/ping
```

From a terminal:

```
curl -H "Authorization: Bearer mmr_xxxxxxxx..." https://moople.markets/api/external/v1/ping
```

```json
{"ok": true, "token": "Price lookup", "world": "Classic"}
```

## Endpoints

| Endpoint | Use it for |
|---|---|
| [`GET /items`](#get-items) | Find items by name or number, with how many are for sale, the cheapest price and recent average sales |
| [`GET /listings`](#get-listings) | Every listing that is for sale now, with price, room, channel, seller and slot, filtered however you need |
| [`GET /items/{id}`](#get-itemsid) | One item's market: listings now, sales and units sold over 24 hours / 7 / 30 days, average and median sale prices, how often and fast it sells, trend, confidence |
| [`GET /items/{id}/sales`](#get-itemsidsales) | An item's sales over a window: price percentiles, median time to sell, day by day, and the sales themselves |
| [`GET /items/{id}/history`](#get-itemsidhistory) | Price history per hour or day: asking prices and sales |
| [`GET /equipment/{id}/comparables`](#get-equipmentidcomparables) | Equipment sales with every stat (Crit. Rate and Crit. Damage too) and slots left, grouped by kind with sample sizes; the item's base stats and normal slots |
| [`GET /equipment/{id}/value`](#get-equipmentidvalue) | What a piece with given stats should sell for, what that's based on, and how confident it is |
| [`GET /opportunities`](#get-opportunities) | Listings priced under what they sell for, with their flip score and the valuation and sales figures behind it |
| `GET /ping` | Check that a token works |

`{id}` is this site's item ID: the `id` from [`/items`](#get-items) (find an item by its game number with
`/items?q=1050010`).

Paths below are relative to the base URL: `/items` means `https://moople.markets/api/external/v1/items`.

What's included: listings that are for sale now, with a price the scanner could read, that aren't a price outlier
(listed at many times the item's cheapest price), of items that are listed publicly on the site. This is the same data as the market page, always live: the API never uses the
delayed 15-minute copy some of the site's visitors see.

A listing stops being "for sale now" when:

- a scan of its shop no longer shows it (sold, or taken down), or
- its shop is closed: the scanner passed through the shop's room and channel and didn't see the shop, or
- its shop hasn't been rescanned for 12 hours of scanning. The clock only runs while scans are arriving, so a scanner
  outage never empties the market.

A shop that is scanned again later brings back whatever it still has for sale.

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
GET https://moople.markets/api/external/v1/items?q=sauna%20robe
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
      "average_sale_7d": 31800.2,
      "average_sale_30d": 32410.5,
      "median_sale_30d": 29999.0,
      "sales": {
        "24h": { "count": 41, "quantity": 41 },
        "7d": { "count": 266, "quantity": 266 },
        "30d": { "count": 1029, "quantity": 1029 }
      },
      "sales_per_day_7d": 38.0,
      "median_hours_to_sell_7d": 3.4
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
| `average_sale_24h`, `average_sale_7d`, `average_sale_30d` | Average price per item it sold for in the last 24 hours / 7 / 30 days, or `null` with no sales. Updated by the site's statistics refresh, usually within a minute of a sale. |
| `median_sale_30d` | The median price per item of the last 30 days' sales. |
| `sales` | Sales in the last 24 hours, 7 days and 30 days: `count` (purchases) and `quantity` (items bought: a bundle of 100 potions is one sale of quantity 100). |
| `sales_per_day_7d` | Sales a day over the last 7 days. |
| `median_hours_to_sell_7d` | The median hours from a listing first being seen to it selling, over the last 7 days' sales. |

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
GET https://moople.markets/api/external/v1/listings?game_id=1050010&room=9&limit=1
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

URL-encode the value: `:` becomes `%3A`, `%` in `CRIT%` becomes `%25`, a space `%20`. The `url_encode` helper in
[Example code](#example-code) does this.

`min_slots` keeps pieces with at least that many upgrade slots left. A piece with fewer slots than the item normally has
has been scrolled, so `min_slots` set to the item's normal slot count means "clean pieces only".

The format is the same as the stat filter on the site's market page.

### GET /items/{id}

One item's market in one request: its listings now, its sales, and how much to trust the figures.

```
GET https://moople.markets/api/external/v1/items/1
```

```json
{
  "id": 1, "game_id": 1032007, "name": "Emerald Earrings", "category": "Accessory",
  "is_equipment": true, "is_cash": false, "npc_price": 4500.0,
  "equipment": { "base_stats": { "M.DEF": 27, "CRITDMG": 2 }, "base_upgrade_slots": 5, "jobs": "All" },
  "listings": { "count": 687, "cheapest_unit_price": 5999.0, "highest_unit_price": 7000000.0,
                "mean_unit_price": 130015.6, "median_unit_price": 24999.0, "value_listed": 89320694,
                "newest_listed_at": "2026-10-23T12:30:51+00:00" },
  "sales": { "24h": { "count": 12, "quantity": 12 }, "7d": { "count": 2173, "quantity": 2173 },
             "30d": { "count": 2681, "quantity": 2681 } },
  "sale_prices": { "average_24h": 48211.0, "average_7d": 49684.4, "average_30d": 50805.8, "median_30d": 29999.0,
                   "last": 31000.0, "last_sold_at": "2026-10-24T09:12:40+00:00" },
  "sales_per_day_7d": 310.4,
  "median_hours_to_sell_7d": 2.5,
  "trend": { "pct_7d": -4.2, "basis_7d": "sales", "pct_24h": 1.3, "basis_24h": "sales" },
  "confidence": { "level": "high", "score": 92, "reasons": [], "strengths": ["2681 sales in 30 days"] },
  "updated_at": "2026-10-24T09:13:02+00:00"
}
```

| Field | Meaning |
|---|---|
| `equipment` | Equipment only (`null` otherwise): the item's `base_stats` (as it drops, before any scrolls), `base_upgrade_slots` (its normal slot count) and the `jobs` it's for. `null` values: not known for this item. |
| `listings` | Its listings now: how many, the cheapest and highest price per item, the mean and median asking price, and `value_listed` (the asking prices of all its listings added up, in mesos). |
| `sales` | As on [`/items`](#get-items): count and quantity over 24 hours, 7 and 30 days. |
| `sale_prices` | Average sale price per item over each window, the 30-day median, and the last sale. |
| `trend` | The % change of its price over the last 7 days against the 7 before (`pct_7d`), and over the last 24 hours against the 24 before (`pct_24h`). `basis`: `sales` (sold prices) or `asking` (asking prices, when it sold too little). `null`: not enough history. |
| `confidence` | How far its price figures can be trusted: `level` (`high`, `medium`, `low`, `insufficient`), `score` (0-100), and the `reasons` and `strengths` behind it (how many sales, how recent, how much prices vary). |

Equipment pieces sell at very different prices by their stats: for prices of pieces like a particular one, use
[`/equipment/{id}/comparables`](#get-equipmentidcomparables) or [`/equipment/{id}/value`](#get-equipmentidvalue).

### GET /items/{id}/sales

An item's sales over the last `days` days: how many, the spread of their prices, how fast they sold, and each sale.

| Parameter | Default | Meaning |
|---|---|---|
| `days` | `30` | Sales of the last this many days, 1 to 90. |
| `stats` | | Equipment: only pieces with at least these stats (as in [Filtering by stats](#filtering-by-stats)). |
| `min_slots` | | Equipment: only pieces with at least this many upgrade slots left. |
| `limit` | `100` | Sales listed, newest first, 0 to 500. `0` gives the figures only. |
| `offset` | `0` | Skip this many sales. |

```
GET https://moople.markets/api/external/v1/items/16/sales?days=30&limit=1
```

```json
{
  "item": { "id": 16, "game_id": 4010001, "name": "Iron Ore", "...": "..." },
  "equipment": null,
  "days": 30,
  "sales": 5976,
  "quantity": 31119,
  "prices": { "min": 1.0, "p10": 375.0, "p25": 400.0, "median": 449.0, "p75": 499.0, "p90": 500.0, "max": 799.0,
              "mean": 441.2 },
  "median_hours_to_sell": 4.95,
  "timed_sales": 5976,
  "sales_per_day": 199.2,
  "discarded_prices": 0,
  "daily": [ { "day": "2026-10-16", "sales": 434, "quantity": 434, "median": 488.0 }, "..." ],
  "timezone": "America/New_York",
  "total": 5976, "limit": 1, "offset": 0,
  "items": [
    { "id": 913442, "unit_price": 450.0, "quantity": 100, "total_price": 45000,
      "sold_at": "2026-10-24T09:01:13+00:00", "listed_at": "2026-10-24T06:40:02+00:00", "hours_to_sell": 2.35,
      "location": { "map": "Free Market 4", "room": 4, "channel": 7 }, "equipment": null }
  ]
}
```

| Field | Meaning |
|---|---|
| `sales`, `quantity` | Sales (purchases) and items bought in the window. |
| `prices` | Price per item: lowest, 10th, 25th, 50th (`median`), 75th and 90th percentile, highest, and mean. |
| `median_hours_to_sell` | The median hours from a listing first being seen to it selling. `timed_sales`: how many sales it is measured from (a sale whose listing wasn't seen before it sold has no time). |
| `sales_per_day` | `sales` divided by `days`. |
| `discarded_prices` | Sales left out of every figure: priced at many times the item's cheapest (misread or joke prices). |
| `daily` | Per day (in `timezone`): sales, quantity and median price. Days without sales are left out. |
| `items` | The sales, newest first. `equipment` is as in [`/equipment/{id}/comparables`](#get-equipmentidcomparables). |

### GET /items/{id}/history

Price history, as charted on the site: per hour for `24h` and `7d`, per day for longer ranges.

| Parameter | Default | Meaning |
|---|---|---|
| `range` | `30d` | `24h`, `7d` (hourly), `30d`, `90d`, `1y`, `all` (daily, days in the site's time zone). |

```json
{
  "item": { "id": 16, "...": "..." },
  "range": "30d", "granularity": "day", "start": "...", "end": "...", "timezone": "America/New_York",
  "points": [
    { "t": "2026-10-23T04:00:00+00:00", "day": "2026-10-23", "has_data": true,
      "asking": { "mean": 471.3, "median": 450.0, "min": 380.0, "max": 650.0, "listings": 212, "sellers": 141,
                  "value_listed": 4825100 },
      "sales": { "count": 221, "quantity": 1830, "median": 449.0, "min": 400.0, "max": 500.0 } }
  ]
}
```

`asking` describes what was listed in the period (each listing counted once, at its latest price there;
`value_listed` is their asking prices added up, in mesos); `sales` what sold (`quantity`: items bought). A period with nothing listed has `has_data: false` and `null` asking figures; nothing is filled in.

### GET /equipment/{id}/comparables

The item's sales as comparables: every sold piece with all its stats and slots, grouped into kinds (pieces with
exactly the same stats and slots) with each kind's sample size and prices. `422` for an item that isn't equipment.

| Parameter | Default | Meaning |
|---|---|---|
| `days` | `30` | Sales of the last this many days, 1 to 90. |
| `stats`, `min_slots` | | Only pieces with at least these stats / slots left (as in [Filtering by stats](#filtering-by-stats)). |
| `include_wasted` | `false` | Include pieces whose upgrade slots were spent for little or nothing (failed scrolls): they sell for less than their stats suggest, so they're left out by default. |
| `limit`, `offset` | `100`, `0` | The sales listed, newest first (0 to 500). |

```json
{
  "item": { "id": 1, "game_id": 1032007, "name": "Emerald Earrings", "...": "..." },
  "equipment": { "base_stats": { "M.DEF": 27, "CRITDMG": 2 }, "base_upgrade_slots": 5, "jobs": "All" },
  "stat_keys": ["STR", "DEX", "INT", "LUK", "HP", "MP", "W.ATK", "M.ATK", "W.DEF", "M.DEF", "ACC", "AVOID",
                "SPEED", "JUMP", "CRIT%", "CRITDMG"],
  "days": 30,
  "sales": 2411, "quantity": 2411,
  "prices": { "min": 2000.0, "p10": 18888.0, "p25": 24444.0, "median": 29999.0, "p75": 37999.5, "p90": 50000.0,
              "max": 2500000.0, "mean": 50847.4 },
  "median_hours_to_sell": 2.47, "timed_sales": 2302,
  "discarded_prices": 0, "wasted_scroll_sales": 270,
  "kinds": [
    { "stats": { "M.DEF": 27, "CRITDMG": 2 }, "crit_rate": null, "crit_damage": 2, "upgrade_slots": 5,
      "stat_diff": {}, "scrolled": false, "wasted_scrolls": false, "kind": "7d170ced9d64",
      "sales": 996, "quantity": 996,
      "prices": { "min": 9000.0, "p10": 15000.0, "p25": 17999.0, "median": 19999.0, "p75": 24000.0, "p90": 28000.0,
                  "max": 60000.0, "mean": 21104.2 },
      "median_hours_to_sell": 1.9, "timed_sales": 951 }
  ],
  "total": 2411, "limit": 100, "offset": 0,
  "items": [
    { "id": 912201, "unit_price": 42000.0, "quantity": 1, "total_price": 42000,
      "sold_at": "2026-10-24T08:44:10+00:00", "listed_at": "2026-10-24T07:02:51+00:00", "hours_to_sell": 1.69,
      "location": { "map": "Free Market 2", "room": 2, "channel": 5 },
      "equipment": { "stats": { "M.DEF": 30, "CRITDMG": 4 }, "crit_rate": null, "crit_damage": 4,
                     "upgrade_slots": 3, "stat_diff": { "M.DEF": 3, "CRITDMG": 2 }, "scrolled": true,
                     "wasted_scrolls": false, "kind": "a1f0c3e8b2d4" } }
  ]
}
```

| Field | Meaning |
|---|---|
| `equipment` | The item's base stats, normal upgrade slots and jobs (as on [`/items/{id}`](#get-itemsid)). |
| `sales`, `quantity`, `prices`, `median_hours_to_sell` | Over every piece sold in the window (wasted-scroll pieces left out unless `include_wasted`). |
| `wasted_scroll_sales` | Sales of pieces with wasted scrolls in the window. |
| `kinds` | Kinds of piece that sold (exactly the same stats and slots), most sold first. `sales` is the kind's sample size; `prices` and `median_hours_to_sell` are its own. |
| `items[].equipment` | The sold piece. |

Every piece (a sale, a kind, a listing or the valued piece) is described the same way:

| Field | Meaning |
|---|---|
| `stats` | All its stats, e.g. `{"W.ATK": 52, "LUK": 4, "CRIT%": 2}`. Stats it doesn't have are left out. |
| `crit_rate`, `crit_damage` | Its Crit. Rate (`CRIT%`) and Crit. Damage (`CRITDMG`), or `null` without them. |
| `upgrade_slots` | Upgrade slots left. |
| `stat_diff` | Each stat against the item's base: `{"W.ATK": 3}` is 3 more than base, `{"DEX": -1}` 1 less. `null` when the base isn't known. |
| `scrolled` | `true` when it has fewer slots than the item normally has. |
| `wasted_scrolls` | `true` when slots were spent for little or nothing. |
| `kind` | An ID shared by pieces with exactly the same stats and slots. |

### GET /equipment/{id}/value

What a piece with these stats should sell for, the same valuation as the site's price checker. `422` for an item that
isn't equipment.

| Parameter | Default | Meaning |
|---|---|---|
| `stats` | (required) | The piece's stats, e.g. `W.ATK:20,DEX:3,CRIT%:2`, in the [stat format](#filtering-by-stats). |
| `upgrade_slots` | | Upgrade slots left on the piece. |

```
GET https://moople.markets/api/external/v1/equipment/1/value?stats=M.DEF%3A27%2CCRITDMG%3A2&upgrade_slots=5
```

```json
{
  "item": { "id": 1, "...": "..." },
  "equipment": { "base_stats": { "M.DEF": 27, "CRITDMG": 2 }, "base_upgrade_slots": 5, "jobs": "All" },
  "piece": { "stats": { "M.DEF": 27, "CRITDMG": 2 }, "crit_rate": null, "crit_damage": 2, "upgrade_slots": 5,
             "stat_diff": {}, "scrolled": false, "wasted_scrolls": false, "kind": "7d170ced9d64" },
  "value": { "estimate": 19999.0, "quick_sale": 16999.8, "hold_out": 24249.7, "suggested": 19999.0,
             "capped_by_listing": false, "basis": "same_stats" },
  "samples": { "days": 30, "sales": 2651, "identical_sales": 996,
               "nearest_sales": { "same": 12, "better": 0, "worse": 0 },
               "listings_shown": { "same": 9, "better": 3, "worse": 0 },
               "sales_per_day_7d": 310.4, "median_hours_to_sell_7d": 2.5 },
  "confidence": { "level": "high", "score": 100, "reasons": [],
                  "strengths": ["996 sale(s) of pieces with exactly these stats", "Sale prices are consistent"] },
  "cheapest_as_good": { "listing_id": 537820, "unit_price": 19500.0, "quantity": 1, "relation": "same",
                        "seller": "Brickopolis", "room": 3, "channel": 3, "stats": { "...": "..." } },
  "comparable_sales": [ { "unit_price": 20000.0, "quantity": 1, "relation": "same",
                          "sold_at": "2026-10-24T08:01:00+00:00", "stats": { "...": "..." } } ],
  "competing_listings": [ { "listing_id": 537820, "unit_price": 19500.0, "relation": "same", "...": "..." } ]
}
```

| Field | Meaning |
|---|---|
| `value.estimate` | What it's worth: from the item's sales of the last 30 days, each adjusted for how much better or worse the sold piece was for the job it suits. |
| `value.quick_sale`, `value.hold_out` | A price it should sell at quickly, and the most worth holding out for (`null` with fewer than 3 sales). |
| `value.suggested` | What to list it for: the estimate, unless a listing at least as good is cheaper (`capped_by_listing`), since buyers take that one first. |
| `value.basis` | `same_stats` (pieces with exactly these stats sold), `better_stats` (only pieces it beats sold), `estimate` (only other pieces sold: a rough guide), or `null` (no sales). |
| `samples` | The sample sizes: the item's sales in the window, how many were `identical` to this piece, and the nearest sales and competing listings shown, by how they compare. |
| `confidence` | How far the valuation can be trusted (`high`, `medium`, `low`, `insufficient`, and a 0-100 `score`): highest when many identical pieces sold at consistent prices, lower when it rests on better pieces or an estimate, when prices vary widely, or with few sales. `reasons` and `strengths` say why. |
| `cheapest_as_good` | The cheapest listing now that's at least as good as the piece. |
| `comparable_sales`, `competing_listings` | Up to 12 of each: the sales nearest to the piece, and the listings it competes with. `relation`: `same`, `better` or `worse` than the piece, for the job it suits. |

### GET /opportunities

Listings priced under what their item has sold for in the last 7 days, best first: the site's Opportunities page.

| Parameter | Default | Meaning |
|---|---|---|
| `tab` | `all` | `all`: equipment, scrolls, set-up and cash items, and anything an NPC pays more for. Or one inventory tab: `equip`, `use`, `setup`, `etc`, `cash`. |
| `q` | | Item name contains this, or the game item number. |
| `min_profit` | | Profit per item at least this many mesos. |
| `min_discount` | | At least this % under what it sells for. |
| `min_listings` | the site's setting | Only items with at least this many listings now (NPC flips don't need any). |
| `min_confidence` | | `low`, `medium` or `high`. |
| `min_per_day` | | Only items selling at least this many a day. |
| `max_hours` | | Only items whose listings usually sell within this many hours. |
| `cheapest_only` | `false` | Only each item's cheapest listing. |
| `sort` | `flip` | `flip` (score), `often`, `fast`, `discount`, `profit`, `price`, `confidence`. |
| `page`, `limit` | `1`, `50` | Page number, and results per page (1 to 200). |

```json
{
  "total": 11177, "page": 1, "limit": 50, "tab": "all", "sort": "flip",
  "tabs": { "all": 11177, "equip": 4543, "use": 4010, "setup": 38, "etc": 5375, "cash": 3056 },
  "items": [
    {
      "item": { "id": 1, "game_id": 1032007, "name": "Emerald Earrings", "...": "..." },
      "listing": { "id": 537820, "price": { "unit": 5999.0, "total": 5999, "quantity": 1 },
                   "location": { "map": "Free Market 3", "room": 3, "channel": 3, "x": null, "y": null },
                   "shop": { "seller": "Brickopolis", "title": "Cheap stuff" },
                   "equipment": { "stats": { "M.DEF": 27 }, "crit_rate": null, "crit_damage": null, "upgrade_slots": 5,
                                  "...": "..." },
                   "first_seen_at": "...", "last_seen_at": "...", "is_cheapest": true },
      "score": 81,
      "score_parts": { "discount": 34.1, "often": 30.0, "fast": 17.2 },
      "valuation": { "reference": 19999.0, "basis": "same_stats", "average_sale_30d": 50805.8, "median_sale_30d": 29999.0,
                     "mean_asking": 130015.6, "median_asking": 24999.0, "npc_price": 4500.0 },
      "sales": { "sales_7d": 2173, "sales_30d": 2681, "per_day_7d": 310.4, "median_hours_to_sell_7d": 2.5,
                 "last_sale_price": 31000.0, "last_sold_at": "...", "trend_pct_7d": -4.2 },
      "profit": { "best": "sale", "gross": 14000.0, "discount_pct": 70.0, "gross_vs_sale": 14000.0,
                  "discount_vs_sale_pct": 70.0, "gross_vs_npc": null },
      "confidence": { "level": "high", "score": 92, "reasons": [], "strengths": ["..."] }
    }
  ]
}
```

| Field | Meaning |
|---|---|
| `score` | The flip score, 0-100: how good a buy-and-resell it is. `null` for a listing that's only an NPC flip. |
| `score_parts` | The score's three parts: `discount` (how far under it is listed), `often` (how often the item sells), `fast` (how fast its listings sell). They add up to `score`. |
| `valuation.reference` | What it's measured against: what pieces like it sold for in the last 7 days. `basis`: `same_stats` (pieces with exactly these stats), `better_stats` (pieces it beats), or `item` (not equipment: the item's sales). |
| `valuation` (the rest) | The item's average and median sale price over 30 days, its mean and median asking price now, and the NPC price. |
| `sales` | The item's sales figures behind the score. |
| `profit` | Profit per item: `best` says whether reselling (`sale`) or selling to an NPC (`npc`) pays more; `gross` and `discount_pct` are for that one. |
| `confidence` | The item's confidence, capped at `low` for an extreme discount (possibly a misread price). |
| `tabs` | How many opportunities each tab has with the same filters. |

Paging here is by `page` (not `offset`): `page=2&limit=50` is results 51-100.

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
| How many of an item sold, and for how much | `/items/41` (counts and averages), `/items/41/sales?days=7&limit=0` (percentiles) |
| A price chart | `/items/41/history?range=30d` |
| What W.ATK 20 / 4-slot Hand Axes sold for | `/equipment/593/comparables?stats=W.ATK:20&min_slots=4` |
| What my piece is worth | `/equipment/593/value?stats=W.ATK:20,DEX:2&upgrade_slots=4` |
| The best flips right now | `/opportunities?limit=20` |
| Flips that sell fast | `/opportunities?max_hours=6&min_confidence=medium` |

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
| `404` | The path doesn't exist, or no item has that ID (`/items/{id}` takes this site's item ID, not the game's item number: look an item up with `/items?q=1050010`). |
| `405` | Anything but `GET`. |
| `422` | A bad parameter: an unknown stat name, `limit` over 500, a word where a number belongs, or an `/equipment` path for an item that isn't equipment. The message says which. |
| `429` | Over the rate limit. Wait the number of seconds in the `Retry-After` header. |

**Rate limit:** 120 requests a minute per token. With `limit=500` that is up to 60,000 listings a minute.

**How fresh is it:**

- Listings, `listings` and `cheapest_unit_price` change as the scanner sends shops, usually within seconds. When the
  site is busy, new scans wait briefly in an import queue first, so they can occasionally be a minute or two behind.
- Sale averages, medians, sales per day, time to sell, trend and confidence come from the site's statistics refresh,
  usually within a minute of a sale. Sale counts and quantities, `/sales`, `/history`, `/comparables` and `/value` are
  worked out at the time of the request.
- `last_seen_at` says when each listing was last confirmed. It is the time the scanner read the shop, not the time the
  site stored it.

## Example code

### Lua

A Lua script that looks up the cheapest listings of an item, optionally with stat minimums, and logs where to find
each one. It is written for a scripting host that provides an asynchronous HTTP client (`core.http.get`), a small UI
(`core.ui`) and a per-tick callback (`on_tick`); adapt those calls to your environment. Requests are asynchronous: the
answer arrives in a callback on a later tick.

The script includes a small JSON decoder (`from_json`) for environments without one.
JSON `null` comes back as `nil`, so a missing key and a `null` value look the same: for example `l.equipment` is `nil`
for items that aren't equipment.

```lua
plugin = {
    name = "Market lookup",
    version = "1.0.0",
    author = "",
    description = "Finds the cheapest listings of an item on Moople Markets and where they are",
    load = true,
}

local PANEL = "Market lookup"
local site = core.ui.input_text{ id = "site", label = "Moople Markets address", panel = PANEL,
    default = "https://moople.markets" }
local token = core.ui.input_text{ id = "token", label = "API token", panel = PANEL, default = "",
    flags = { "password" }, tooltip = "Your Moople Markets API token (starts with mmr_)" }
local item = core.ui.input_text{ id = "item", label = "Item number", panel = PANEL, default = "1082000" }
local stats = core.ui.input_text{ id = "stats", label = "Stat minimums (optional)", panel = PANEL, default = "",
    tooltip = "e.g. DEX:3,W.ATK:2" }
local find = core.ui.button{ id = "find", label = "Find cheapest", panel = PANEL }

---------------------------------------------------------------------------------------------------------------------
-- JSON decoding
---------------------------------------------------------------------------------------------------------------------

local function from_json(text)
    if type(text) ~= "string" then return nil end
    local pos = 1
    local function skip() pos = text:find("[^ \t\r\n]", pos) or #text + 1 end
    local value
    local function utf8_char(code)
        if code < 0x80 then return string.char(code) end
        if code < 0x800 then return string.char(0xC0 + code // 64, 0x80 + code % 64) end
        return string.char(0xE0 + code // 4096, 0x80 + (code // 64) % 64, 0x80 + code % 64)
    end
    local function str()
        local out = {}
        pos = pos + 1
        while true do
            local c = text:sub(pos, pos)
            if c == "" then error("unterminated string") end
            if c == '"' then pos = pos + 1; return table.concat(out) end
            if c == "\\" then
                local e = text:sub(pos + 1, pos + 1)
                if e == "u" then
                    table.insert(out, utf8_char(tonumber(text:sub(pos + 2, pos + 5), 16) or 63))
                    pos = pos + 6
                else
                    table.insert(out, ({ b = "\b", f = "\f", n = "\n", r = "\r", t = "\t" })[e] or e)
                    pos = pos + 2
                end
            else
                local stop = text:find('["\\]', pos) or #text + 1
                table.insert(out, text:sub(pos, stop - 1))
                pos = stop
            end
        end
    end
    value = function()
        skip()
        local c = text:sub(pos, pos)
        if c == "{" then
            local obj = {}
            pos = pos + 1
            skip()
            if text:sub(pos, pos) == "}" then pos = pos + 1; return obj end
            while true do
                skip()
                if text:sub(pos, pos) ~= '"' then error("expected a key") end
                local k = str()
                skip()
                if text:sub(pos, pos) ~= ":" then error("expected :") end
                pos = pos + 1
                obj[k] = value()
                skip()
                local sep = text:sub(pos, pos)
                pos = pos + 1
                if sep == "}" then return obj end
                if sep ~= "," then error("expected , or }") end
            end
        elseif c == "[" then
            local arr = {}
            pos = pos + 1
            skip()
            if text:sub(pos, pos) == "]" then pos = pos + 1; return arr end
            while true do
                table.insert(arr, value())
                skip()
                local sep = text:sub(pos, pos)
                pos = pos + 1
                if sep == "]" then return arr end
                if sep ~= "," then error("expected , or ]") end
            end
        elseif c == '"' then
            return str()
        end
        if text:sub(pos, pos + 3) == "true" then pos = pos + 4; return true end
        if text:sub(pos, pos + 4) == "false" then pos = pos + 5; return false end
        if text:sub(pos, pos + 3) == "null" then pos = pos + 4; return nil end
        local num = text:match("^-?%d+%.?%d*[eE]?[-+]?%d*", pos)
        if not num or num == "" then error("unexpected " .. c) end
        pos = pos + #num
        return tonumber(num)
    end
    local ok, result = pcall(value)
    return ok and result or nil
end

---------------------------------------------------------------------------------------------------------------------
-- Calling the API
---------------------------------------------------------------------------------------------------------------------

local function url_encode(s)
    return (tostring(s):gsub("[^%w%-_%.~]", function(c) return string.format("%%%02X", string.byte(c)) end))
end

-- {game_id = 1082000, stats = "DEX:3"} -> "game_id=1082000&stats=DEX%3A3" (empty values are left out)
local function query_string(params)
    local parts = {}
    for key, value in pairs(params) do
        if value ~= nil and value ~= "" then table.insert(parts, key .. "=" .. url_encode(value)) end
    end
    return table.concat(parts, "&")
end

local retry_at, retry = 0, nil  -- after a 429: when to try again, and what

local function header(response, name)
    for key, value in pairs(response.headers or {}) do
        if string.lower(key) == string.lower(name) then return value end
    end
    return nil
end

-- GET /api/external/v1<path>?<params>; callback(data) with the decoded answer, or callback(nil) on an error
local function api_get(path, params, callback)
    local base = site:get():match("^%s*(.-)/*%s*$")
    local url = base .. "/api/external/v1" .. path .. "?" .. query_string(params)
    local headers = { ["Authorization"] = "Bearer " .. token:get():match("^%s*(.-)%s*$") }
    core.http.get(url, headers, function(response)
        if response.status == 429 then  -- over the rate limit: on_tick tries again after Retry-After seconds
            local wait = tonumber(header(response, "Retry-After") or "") or 5
            retry_at = core.get_update_time() + wait * 1000
            retry = function() api_get(path, params, callback) end
            return
        end
        local data = from_json(response.body)
        if response.status ~= 200 or data == nil then
            local reason = (data and data.detail) or response.body or "no answer"
            core.log_warning("[Market] " .. tostring(response.status) .. ": " .. tostring(reason))
            return callback(nil)
        end
        callback(data)
    end)
end

-- Every listing matching the filters, 500 per request, following the pages. done(list) gets them cheapest first,
-- or done(nil) on an error.
local function all_listings(filters, done)
    local found, offset = {}, 0
    local function next_page()
        local params = { limit = 500, offset = offset }
        for key, value in pairs(filters) do params[key] = value end
        api_get("/listings", params, function(page)
            if page == nil then return done(nil) end
            for _, listing in ipairs(page.listings) do
                found[listing.id] = listing  -- by id: a listing that moved between pages is kept once
            end
            offset = offset + page.limit
            if offset < page.total then return next_page() end
            local list = {}
            for _, listing in pairs(found) do table.insert(list, listing) end
            table.sort(list, function(a, b) return a.price.unit < b.price.unit end)
            done(list)
        end)
    end
    next_page()
end

local function stat_text(equipment)
    if equipment == nil or equipment.stats == nil then return "" end
    local parts = {}
    for stat, value in pairs(equipment.stats) do table.insert(parts, stat .. " " .. tostring(value)) end
    table.sort(parts)
    return "  [" .. table.concat(parts, ", ") .. "; " .. tostring(equipment.upgrade_slots) .. " slots]"
end

---------------------------------------------------------------------------------------------------------------------
-- The button
---------------------------------------------------------------------------------------------------------------------

function on_tick(stage)
    if retry ~= nil and core.get_update_time() >= retry_at then
        local again = retry
        retry = nil
        again()
    end

    if find:pressed() then
        all_listings({ game_id = item:get(), stats = stats:get() }, function(list)
            if list == nil then return end
            if #list == 0 then return core.log("[Market] Nobody is selling that right now") end
            core.log("[Market] " .. #list .. " listing(s) of " .. list[1].item.name .. ", cheapest first:")
            for i = 1, math.min(5, #list) do
                local l = list[i]
                core.log(string.format("[Market] %s mesos each  ch %s, FM %s  %s's shop, slot %d (item %d)%s",
                    string.format("%.0f", l.price.unit), tostring(l.location.channel), tostring(l.location.room),
                    l.shop.seller, l.shop.slot or -1, (l.shop.slot or -1) + 1, stat_text(l.equipment)))
            end
        end)
    end
end
```

Pressing **Find cheapest** with item `1082000` and stats `DEX:3` logs something like:

```
[Market] 4 listing(s) of Work Gloves, cheapest first:
[Market] 15000 mesos each  ch 4, FM 3  Haste's shop, slot 2 (item 3)  [DEX 3; 5 slots]
[Market] 22000 mesos each  ch 11, FM 8  Willnie's shop, slot 0 (item 1)  [DEX 4, W.DEF 3; 5 slots]
...
```

To use other filters, change the table passed to `all_listings`, e.g. `{ q = "mana elixir", max_price = 1000 }` or
`{ room = 8, channel = 11 }`. Any parameter from [GET /listings](#get-listings) works. For `/items`, call
`api_get("/items", { q = "sauna robe" }, function(data) ... end)` and read `data.items`.

Keep the token in the password field, not in the script file, so it isn't shared when the script is.
