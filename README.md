# CrossplayMarket

An auction house for **cross-play servers** (Paper or Folia with Geyser/Floodgate) that Bedrock players can actually
use: **native Bedrock menus** to sell, search, buy and collect, a chest menu for Java players, and no way to
duplicate an item or lose one.

On a cross-play server, Bedrock players (on a phone above all) barely use the auction house: chest menus show
unreadable lores, a tap buys the wrong item, the anvil search does not open, a price typed in chat fails ("1k",
commas). They trade in chat instead. With CrossplayMarket, a player on a phone sells the item they hold, searches,
buys and gets back their unsold items **from menus only**, without typing a price in chat.

> **Status:** version 1.0.0 is finished and tested, not released yet. Free edition: SpigotMC, CurseForge and
> [Releases](../../releases/latest); full edition: Payhip (links when published).

## Editions

Bedrock menus and duplication safety are never limited in the free edition.

| | Free | Full (12.99 USD) |
|---|---|---|
| Native Bedrock menus, Java chest menu | yes | yes |
| Duplication safety, collection box, crash recovery | yes | yes |
| Fixed price, search, sort, expiry, sale notifications | yes | yes |
| Admin commands (remove a listing, ban from the auction house, pending operations) | yes | yes |
| Active listings | 50 on the whole server | unlimited, limit per rank (`crossplaymarket.limit.<n>`) |
| Categories | no (one list and the search) | yes |
| Listing tax, sale tax | no | yes |
| Several servers sharing one market (MySQL) | no (SQLite only) | yes |
| Log | text file | SQL table, `/ah admin logs`, refunds |
| Shulker box content preview | no | yes |
| PlaceholderAPI | no | yes |

## How it looks

- **Bedrock players** (detected by Floodgate) get native Bedrock forms: a home screen (Browse, Search, My listings,
  Collection box, Categories in the full edition), lists of 10 to 15 listings per page (name, amount, price,
  seller), a sheet for each listing (enchantments, durability, lore, shulker content, time left) with Buy and Back.
  After a purchase you are back on the same list, same page, same search.
- **Selling on Bedrock**: `/ah sell` with the item in hand opens one form: the item, a price field, a duration list
  and the fees, then a confirmation with the total after tax. Prices like `1500`, `1 500`, `1k`, `1,5k`, `1.5m`
  work; `1,500` and `1.500` are refused as ambiguous, with a suggestion.
- **Item names**: a Bedrock form only shows text. Items show their custom name, or their vanilla name in the
  player's language, read from the language files that Geyser downloads itself (nothing is redistributed).
- **Java players** get a 6-row chest menu: 45 listings (the real item with price, seller and time left in the lore),
  previous and next page, sort (lowest price, highest price, newest, ending soon), search, categories, my listings,
  collection box. Clicking a listing opens a confirmation menu.

## Duplication safety

The database is the only source of truth. Each duplication technique found in other auction house plugins has a
defense and a bot test that fails when the defense is removed:

| # | Technique | Defense |
|---|---|---|
| 1 | Item left in the menu after confirming | the item leaves the inventory before the listing is written; a menu's content is never trusted |
| 2 | Disconnecting while typing the price | nothing is taken before you confirm; a late answer is ignored |
| 3 | Clicks in the same tick, shift-click, packet-crafted clicks | every click is cancelled, one action at a time per player |
| 4 | Two buyers at the same time, buying while the seller cancels | one conditional database update: only one buyer wins |
| 5 | Race between two servers (MySQL) | the same conditional update; caches are only for display |
| 6 | Balance exactly equal to the price | the buyer is charged first and the result is checked; on failure the listing goes back on sale |
| 7 | Item swapped during `/ah sell` (item frames, armor stands) | the item is read and taken in the same tick, and checked to be the one confirmed |
| 8 | Huge NBT, bundles | saved item size is capped, bundles refused by default |
| 9 | Full inventory when collecting | collection box: items are never dropped on the ground |
| 10 | Server crash between the money and the item | see below |
| 11 | Client mods that take items out of a menu | no slot can be taken, menu items are display copies |

**Server crash in the middle of a purchase.** The player file and the database are not written at the same instant,
and Vault cannot tell whether a payment went through. Each step of a sale, purchase or collection is written to the
database before the next one, in an order where a crash at any step never creates an item or money. At startup an
unfinished operation is **not replayed blindly**: it is listed in the console and in `/ah admin pending`, with what is
certain and what is not, and an admin finishes or reverts it with `/ah admin resolve <ref> <action>`. Tested by killing
the server (`kill -9`) at each step: never a duplicated item or money, and after `resolve` exactly one item and the
original amount of money.

## Commands

| Command | What it does | Permission (default) |
|---|---|---|
| `/ah` | open the auction house | `crossplaymarket.use` (everyone) |
| `/ah sell [price] [duration]` | sell the item in your hand (Bedrock without a price: form) | `crossplaymarket.sell` (everyone) |
| `/ah search <text>` | search by name, material or enchantment | `crossplaymarket.use` |
| `/ah selling` | your active listings; remove one and get the item back (collection box if your inventory is full) | `crossplaymarket.use` |
| `/ah collect` | collection box: unsold items, items bought with a full inventory | `crossplaymarket.use` |
| `/ah admin remove <id>`, `ban <player> [reason]`, `unban <player>`, `view <player>`, `purge` | moderation (`remove` and `purge` send the items to the seller's box) | `crossplaymarket.admin` (op) |
| `/ah admin pending`, `/ah admin resolve <ref> <action>` | unfinished operations after a crash | `crossplaymarket.admin` (op) |
| `/ah admin logs [player]`, `/ah admin refund <transaction>` | transaction log and refunds (full edition) | `crossplaymarket.admin` (op) |
| `/crossplaymarket reload`, `/crossplaymarket version` | admin | `crossplaymarket.admin` (op) |

`/hdv` is an alias of `/ah` (`commands.aliases` in the config). Other permissions: `crossplaymarket.limit.<n>` and
`crossplaymarket.limit.unlimited` (active listings per player, full edition; the highest wins),
`crossplaymarket.bypass.tax` (op).

A refund gives the buyer back the price and takes back from the seller what they received; it is refused when the
seller no longer has that balance (no negative balance, no money created), the item is not taken back, and a
transaction is refunded only once.

## PlaceholderAPI (full edition)

`%crossplaymarket_active%` (your active listings), `%crossplaymarket_box%` (items in your collection box),
`%crossplaymarket_active_<player>%`.

## Configuration

`plugins/CrossplayMarket/config.yml`, commented. It works without any change (SQLite).

```yaml
language: en              # server log, and players whose game language has no text file
update-check: true
storage:
  type: sqlite            # or mysql (full edition): host, port, database, user, password
server-name: server1      # different on each server of a network
sync-interval-seconds: 3
durations: [6h, 24h, 48h]
default-duration: 24h
min-price: 1
max-price: 1000000000
default-limit: 5          # active listings per player without a crossplaymarket.limit.<n> permission
tax:                      # full edition
  listing-percent: 0
  sale-percent: 0
  refund-on-expire: false
categories: ...           # full edition: name, icon, materials and item tags; everything else goes to "misc"
blacklist:
  materials: []
  name-regex: []
  lore-regex: []
  custom-model-data: []
  allow-bundles: false
allow-damaged: true
max-item-bytes: 65536
item-names-folder: ''     # Geyser on a proxy only: copy its locales folder here
commands:
  aliases: [ah, hdv]
menus:
  java: ...               # slot and item of each button of the chest menu
```

Texts: `plugins/CrossplayMarket/lang/en.yml` and `fr.yml` (MiniMessage). Each player gets the language of their game
when a file exists; add a language by copying a file (`de.yml`, `es.yml`...).

## Several servers (full edition)

Set `storage.type: mysql` with the same MySQL or MariaDB database on every server, and a different `server-name` on
each. A listing put on sale on one server can be bought on another; the seller is told within a few seconds on the
server they are on, or gets a summary when they join. The MySQL driver comes with Paper.

**Every server must use the same economy data** (an economy plugin that shares its balances between servers, or a
shared database). Otherwise a seller is paid on the buyer's server, not on their own. CrossplayMarket cannot detect
this.

Mixed Minecraft versions: a listing saved by a newer server is hidden on older servers (it stays visible on the newer
ones), and the console says so at startup.

## Requirements

- Paper or Folia 1.21.4 to 26.x, Java 21+ (tested on Paper 1.21.11, Paper 26.3 and Folia 26.3)
- [Vault](https://www.spigotmc.org/resources/vault.34315/) (on Folia: VaultUnlocked, the maintained fork of Vault)
  **and an economy plugin**. CrossplayMarket works with any economy that supports Vault (tested with EssentialsX on Paper and EternalEconomy on Folia). On Folia, the economy
  plugin must support Folia too.
- Optional: [Floodgate](https://geysermc.org/download), for the Bedrock menus. Without it everyone gets the Java
  menu.
- Optional (full edition): [PlaceholderAPI](https://www.spigotmc.org/resources/placeholderapi.6245/).

## Data

`plugins/CrossplayMarket/market.db` (SQLite) or your MySQL database. Items are saved with Paper's versioned item
format, so they survive Minecraft updates with their components and custom data. Free edition log:
`plugins/CrossplayMarket/logs/transactions.log`.

## Support

Bug reports and questions: [issues](https://github.com/TheMax98000/CrossplayMarket/issues).

## Also by the author

- [CrossplayHomes](https://www.spigotmc.org/resources/crossplayhomes.139801/): homes and TPA with native Bedrock
  menus.
- [CrossplayShield](https://www.spigotmc.org/resources/crossplayshield.139762/): anti-exploit for cross-play servers
  that does not kick Bedrock players.
- [BedrockGuard](https://www.spigotmc.org/resources/bedrockguard-free.139718/): anti-cheat built for Bedrock players.
- [BedrockAssist](https://www.spigotmc.org/resources/bedrockassist.139746/) (free): native Bedrock menus for what
  Geyser cannot translate.
