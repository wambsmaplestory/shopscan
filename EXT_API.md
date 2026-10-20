# External API

A read-only API for other applications (Violet scripts, bots, tools) to look up what is for sale in the Free Market:
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

URL-encode the value: `:` becomes `%3A`, `%` in `CRIT%` becomes `%25`, a space `%20`. The `url_encode` helper in
[Example code](#example-code) does this.

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

### Lua (Violet)

A Violet script that looks up the cheapest listings of an item, optionally with stat minimums, and logs where to find
each one. It uses `core.http.get` and `core.ui` from the Violet Lua SDK (see its documentation). Requests are
asynchronous: the answer arrives in a callback on a later tick.

Violet has no JSON decoder, so the script includes a small one (`from_json`, the same parser `npc_auto_buyer.lua` uses).
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
    default = "https://your-site" }
local token = core.ui.input_text{ id = "token", label = "API token", panel = PANEL, default = "",
    flags = { "password" }, tooltip = "From the site's Admin > API tokens (starts with mmr_)" }
local item = core.ui.input_text{ id = "item", label = "Item number", panel = PANEL, default = "1082000" }
local stats = core.ui.input_text{ id = "stats", label = "Stat minimums (optional)", panel = PANEL, default = "",
    tooltip = "e.g. DEX:3,W.ATK:2" }
local find = core.ui.button{ id = "find", label = "Find cheapest", panel = PANEL }

---------------------------------------------------------------------------------------------------------------------
-- JSON (Violet has no decoder)
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
