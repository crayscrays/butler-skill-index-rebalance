# index-rebalance

Holds a basket of 2–15 tokens at fixed target weights. On each fire it values the
basket from the owner's own holdings; when any leg's share is more than `BAND_PCT`
percentage points away from its target, it sells the legs that have run ahead into
USDC and buys the legs that have lagged with those proceeds.

## What it does not do

- **No separate ledger.** The index is the owner's WHOLE holding of each leg's exact
  contract on its chain. Tokens they already held, or buy by hand later, are part of
  the index and get sold or topped up with it.
- **No new money after the seed.** `SEED_USD` is spent once, on the first fire.
  Every rebalance after that is funded only by its own sells — a refused or
  unsettled sell shrinks the matching buys rather than drawing on other cash.
- **Nothing is re-picked.** Constituents and weights stay as filed until the owner
  changes them. No market-cap ranking, no momentum, no cash leg, no exit on a
  drawdown.
- **One fire at a time.** While any trade this duty filed is still executing or
  waiting on the owner's approval, the duty files nothing new and the drift stands.
  A trade in flight for 24 hours is logged and dropped.
- **Sells settle before buys.** Within one fire if they settle inside ~150 seconds,
  otherwise the buys are queued to the next fire.
- **Spot only.** No tokenized stocks, no perps.
- Legs below `MIN_TRADE_USD`, or under the $2 route minimum, are skipped silently
  that fire. A seed slice under $2 is skipped and that leg starts empty.

## Settings

- `LEGS` — one `{SYMBOL, ADDRESS, CHAIN_ID, WEIGHT}` per token. `WEIGHT` is a
  percent of the whole basket and the weights must sum to 100 (±0.5); they are then
  normalised. `ADDRESS` is the exact contract or mint, omitted only for ETH, BNB or
  SOL as a chain's own coin — a bare ticker with no address for anything else is
  rejected. Bad settings are not a crash: the duty notifies once and then does
  nothing every fire until they are fixed.
- `SEED_USD` — dollars, spent once on the first fire. `0` adopts current holdings.
- `BAND_PCT` — percentage POINTS of drift, not percent of the weight.
- `MIN_TRADE_USD` — dollars; smaller corrections are skipped.
- `MAX_REBALANCE_USD` — dollars of sells per rebalance, which caps the buys too.

Trigger is a `timer`: a daily time, or an interval for a tighter index.

## Keys

Every trade is keyed: `index:<duty>:seed:<leg>` for the starting basket, and
`index:<duty>:<slot>:sell:<leg>` / `:buy:<leg>` per scheduled slot. A restart or a
replayed fire never buys or sells twice. An `acp trade` whose outcome could not be
read is never retried under a new key.
