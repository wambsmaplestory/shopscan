# shop_scan

## Using it in a script

```lua
local shop_scan = require("wambs.shop_scan")
```

---

## Forward four hooks

A library can't receive game events itself, so your script passes them on. Without this, a scan never moves.

```lua
function on_tick(stage)       shop_scan.on_tick(stage) end
function on_packet_recv(p)    shop_scan.on_packet(p); return true end
function on_map_load()        shop_scan.on_map_load() end
function on_unload(reason)    shop_scan.stop(); shop_scan.flush() end
```

- **`on_tick`** runs the scan and sends anything waiting to be shared.
- **`on_packet_recv`** lets it see whether a shop let it in.
- **`on_map_load`** lets it notice a channel change.
- **`on_unload`** stops a scan that's under way and saves anything not yet sent.

Forward these from **one** script only. The library is shared by every script that loads it, and forwarding from two scripts would run it twice per tick.

---

## Start a scan

| Call | What it scans | Ends when |
|---|---|---|
| `shop_scan.scan_map(opts)` | the map you're standing on | every shop there has been tried |
| `shop_scan.scan_fm_channel(opts)` | Free Market rooms 1–11 on your current channel | every room has been scanned |
| `shop_scan.scan_fm_all(opts)` | rooms 1–11 on channels 1–20 | every room has been scanned on every channel |

Each returns `true` if the scan started, or `false` plus a reason:

- `"a scan is already running"`: only one scan runs at a time, across every script that uses the library;
- `"not in a field"`;
- `"core.player_shop isn't available in this core"`.

A scan takes many ticks, so results don't come back as a return value. They arrive through the callbacks in `opts`.

### Options

All are optional.

| Option | Default | What it does |
|---|---|---|
| `on_shop` | none | `function(shop)`, called after each shop is read (see [The shop table](#the-shop-table)) |
| `on_done` | none | `function(summary)`, called once when the scan ends or is stopped |
| `fast` | `true` | Enters shops by sending the enter packet, about 3 shops a second. `false` uses `core.player_shop.enter()`, which allows one shop a second. |
| `channels_first` | `true` | `scan_fm_all` only. `true`: every channel in a room, then the next room. `false`: every room on a channel, then the next channel. |

### Where Free Market scans start and go

- **Inside a Free Market room:** it starts in that room.
- **Anywhere else, including the Free Market entrance:** it travels to **room 6** first.

From the starting room it goes:
- toward room 11, then back down from the room below the start to room 1;
- from room 6 or room 11 it uses its own fixed orders: 6 → 1 then 7 → 11, and 11 → 7 then 1 → 6.

Room 7 is reached by a fixed walking route, because the rusher can't find a way there.

`scan_fm_all` changes channel without leaving the room: 1 → 2 → … → 20, wrapping to 1.

`scan_map` works on any map that has player shops, not only the Free Market.

### Stopping

`shop_scan.stop()` ends the scan, leaves any open shop, and calls `on_done` with `reason = "stopped"`. A scan also stops by itself if:

- you leave the field (a loading screen during a channel change is fine);
- the rusher can't reach a room after 3 tries, or takes longer than 2 minutes;
- a channel change hasn't gone through after 60 seconds;
- an error occurs inside the library.

`shop_scan.is_running()` tells you whether a scan is under way.

---

## The shop table

`on_shop` receives one table per shop:

```lua
{
    shop_id = 12345,           -- the shop's id on this map
    title   = "cheap ores",
    owner   = "SomePlayer",
    kind    = "personal",      -- the shop kind as core.player_shop reports it
    map_id  = 80002006,
    channel = 4,
    items   = {
        { slot = 0, item_id = 4010001, price = 1500, bundles = 3, per_bundle = 10 },
        { slot = 1, item_id = 1302000, price = 250000, bundles = 1, per_bundle = 1,
          stats = { ["16"] = 17, ["12"] = 3 }, upgrade_slots = 7 },
        ...
    },
}
```

| Item field | Meaning |
|---|---|
| `slot` | the row in the shop window, from 0. Items are in slot order. |
| `item_id` | the game's item ID |
| `price` | mesos **per bundle** |
| `per_bundle` | items in one bundle |
| `bundles` | bundles left. **0 means that row has sold out.** |
| `stats` | equipment only: stat number → value, leaving out stats that are 0 (see below) |
| `upgrade_slots` | equipment only: upgrade slots left |

Price per single item is `price / per_bundle`. Items left in the row is `bundles * per_bundle`.

`stats` keys are strings holding these stat numbers:

| Key | Stat | Key | Stat |
|---|---|---|---|
| `"0"` | STR | `"9"` | Magic attack |
| `"1"` | DEX | `"10"` | Weapon defense |
| `"2"` | INT | `"11"` | Magic defense |
| `"3"` | LUK | `"12"` | Accuracy |
| `"4"` | HP | `"13"` | Avoidability |
| `"5"` | MP | `"14"` | Critical rate |
| `"6"` | Speed | `"15"` | Critical damage |
| `"7"` | Jump | `"16"` | Weapon attack |

### What doesn't reach `on_shop`

- **A shop the library couldn't get into.** It tries every other shop on the map first, then tries again, up to 3 rounds per map.
- **A shop that read wrong 3 times in a row**, for example a missing owner name or an impossible price.

### The summary

`on_done` receives:

```lua
{ shops = 42, maps = 11, reason = "done" }   -- reason: "done" or "stopped"
```

`shops` counts the shops passed to `on_shop`, and `maps` counts the maps finished. `"stopped"` covers both `shop_scan.stop()` and the problems listed under [Stopping](#stopping). Turn on `log` to see which one it was.

---

## Settings: `shop_scan.configure{ ... }`

Call it whenever you like. Options you leave out keep their current value.

| Option | Default | What it does |
|---|---|---|
| `log` | `false` | prints scan and travel progress to the console |
| `share` | `false` | sends what is scanned to moople.markets (see [Sharing](#sharing-with-mooplemarkets)) |
| `log_sends` | `false` | while sharing is on, prints a line for each send |

```lua
shop_scan.configure{ log = true }
```

`shop_scan.is_sharing()` tells you whether sharing is on.

---

## Sharing with moople.markets

Sharing is **off** unless your script turns it on with `shop_scan.configure{ share = true }`. While it's on, the library sends to https://moople.markets, the Maple Market price site, for its market prices:

- each shop it reads: owner, title, map, channel and items (item ID, price, quantity, stats);
- after each map, the owners of the shops that were on it, so the site can tell which shops have closed.

**Nothing about the player running the scan is sent:** no character name, account, mesos, inventory or position. The data is not anonymized, though. Shop owners' character names are part of the shop information.

**What the console shows:**
- one line whenever a script turns sharing on or off;
- one line at the start of every scan while sharing is on.

Both always appear, whatever `log` is set to. If your script turns sharing on, tell your users so.

With `log_sends = true`, you also get a line per send:

```
shop_scan: sent 1 shop(s), 0 map pass(es) to https://moople.markets (0 waiting)
shop_scan: couldn't reach https://moople.markets to send 3 shop(s), 1 map pass(es); trying again
shop_scan: https://moople.markets answered 401 to 1 shop(s), 0 map pass(es)
```

**How sending works:**
- Data waits in a queue, and nothing is lost if the site can't be reached.
- The queue is saved, so a script reload or game restart keeps it, as long as your script calls `shop_scan.flush()` from `on_unload`.
- Turning sharing off throws away anything still waiting.

---

## Sample script

`shop_scan_example.lua` adds three buttons and a box to type an item ID into. It scans and then lists the 5 cheapest listings of that item, per single item, with where to find them.

```lua
local shop_scan = require("wambs.shop_scan")

plugin = {
    name = "Cheapest Finder",
    version = "1.0.0",
    author = "Wambs",
    description = "Scans shops with the shop_scan library and lists the cheapest listings of one item",
    load = true,
}

local PANEL = "Cheapest Finder"
local item_box = core.ui.input_text{ id = "cf_item", label = "Item ID", panel = PANEL, default = "4010001" }
local btn_map = core.ui.button{ id = "cf_map", label = "Scan this map", panel = PANEL }
local btn_channel = core.ui.button{ id = "cf_channel", label = "Scan every room on this channel", panel = PANEL }
local btn_stop = core.ui.button{ id = "cf_stop", label = "Stop", panel = PANEL }

local found = {} -- listings of the item being looked for: { each, left, owner, map_id, channel }

local function on_shop(shop)
    local wanted = tonumber(item_box:get())
    for _, it in ipairs(shop.items) do
        if it.item_id == wanted and it.bundles > 0 then
            table.insert(found, { each = it.price / it.per_bundle, left = it.bundles * it.per_bundle,
                owner = shop.owner, map_id = shop.map_id, channel = shop.channel })
        end
    end
end

local function on_done(summary)
    table.sort(found, function(a, b) return a.each < b.each end)
    core.log(string.format("Scan %s: %d shop(s) on %d map(s), %d listing(s) of item %s", summary.reason,
        summary.shops, summary.maps, #found, item_box:get()))
    for i = 1, math.min(5, #found) do
        local f = found[i]
        core.log(string.format("  %d. %.0f mesos each, %d left - %s, ch %d, room %d", i, f.each, f.left, f.owner,
            f.channel, f.map_id - 80002000))
    end
end

local function start(scan)
    if not tonumber(item_box:get()) then
        core.log_warning("Type an item ID first")
        return
    end
    found = {}
    local ok, why = scan{ on_shop = on_shop, on_done = on_done }
    if not ok then core.log_warning("Can't scan: " .. tostring(why)) end
end

function on_tick(stage)
    if btn_map:pressed() then start(shop_scan.scan_map) end
    if btn_channel:pressed() then start(shop_scan.scan_fm_channel) end
    if btn_stop:pressed() then shop_scan.stop() end
    shop_scan.on_tick(stage)
end

function on_packet_recv(p)
    shop_scan.on_packet(p)
    return true
end

function on_map_load()
    shop_scan.on_map_load()
end

function on_unload(reason)
    shop_scan.stop()
    shop_scan.flush()
end
```
