# Megabrain (Jev) & Six Grok Bots — Setup Guide (archived copy)

> **Source:** X long-form article by [@savipww](https://x.com/savipww), published
> 23 September 2026.
> Original post: `https://x.com/savipww/status/2102720919185617314`
> **Archived here for review only. This code has never been run.**

---

## ⚠️ Read this before you run anything

This archive exists so the code could be read before being trusted. It was read.
Five findings, worst first.

The author's own closing disclaimer ("this is my setup, not your strategy", "copy
the shape, not the constants", "NFA and Always DYOR") is in section 3 and is worth
reading. The findings below are what reading the code adds to it.

### 1. The author's referral link is hardcoded into the execution path

In the `FILLS` prompt:

> *"Fills go through fomo.family/r/savipww and nowhere else. One venue, one path."*

`fomo.family/r/savipww` is the author's **referral link**. Every trade you place
through this desk pays him a cut. The guide states the mechanism itself in
section 6: *"the referral discount applies to every trade including the ones your
bots place."*

This is not a neutral architecture decision presented as one. "One venue, one
execution path" is given a sound engineering justification — traceability — and
the venue chosen happens to be the one the author earns from. Replace that URL
with a non-referral endpoint before running anything.

### 2. A $100 starting bank cannot place a single trade — by the guide's own arithmetic

This is the finding that matters most, because the author's daily posts advertise
exactly that starting figure.

The sizing rules:

- `SIZE` step 1: ticket is clamped at **6% of the book**
- `FILLS` step 1: `effective_fee = max(0.0045 * ticket, 0.95) / ticket`
- `FILLS` step 2: over the max → **do not send, return `FEE_FLOOR`**

The $0.95 flat floor dominates small tickets. Working it through:

| bank | max ticket (6%) | effective fee |
|---|---|---|
| $100 | $6.00 | **15.83%** |
| $333 | $19.98 | 4.75% |
| $500 | $30.00 | 3.17% |
| $1,000 | $60.00 | 1.58% |
| $3,518 | $211.08 | 0.45% |

The guide explicitly rejects the 4.75% case: *"A $20 entry against a $0.95 floor
is 4.75% round trip and no meme edge covers that."*

A $100 bank yields a $6 ticket at **15.83%** — more than three times the cost the
author himself calls unviable. `FILLS` would return `FEE_FLOOR` on every order and
the desk would never trade.

The flat fee stops dominating at a ticket of ~$211, which at 6% of book requires a
bank of roughly **$3,500**.

Meanwhile the same author's posts of 9–11 October narrate a running
"$100 → $100,000" series at $52,437, $60,472 and $68,316. Those posts and this
guide cannot both be describing the same system.

### 3. The one file that handles your credentials is the one file not included

`collect.py` and `main.py` both open with:

```python
from fomo_api import Fomo      # Privy bearer out of Chrome over CDP
```

`fomo_api.py` is **never shown anywhere in the guide**. That single comment
describes extracting a live auth token out of your logged-in Chrome session over
the Chrome DevTools Protocol, for a platform that holds trading funds.

So the component that touches your credentials is the component you cannot read,
and you'd be writing it yourself from a one-line hint. The code as published also
**cannot run** without it — this is not a subjective judgement, it's a missing
import.

### 4. There is no price stop-loss

`RISK` has exactly one rule:

```
avg_6h = volume.h24 / 4
ratio  = volume.h6 / avg_6h
ratio < 0.20 -> CLOSE, fully, inside 60 seconds
```

That closes on **volume decay only**. A position that falls 90% in price while
volume holds up will not trigger a close. For a desk trading minutes-old
low-liquidity tokens, that is a large hole in the seat whose entire job is
closing.

`avg_6h` is also unguarded against zero: a token with no 24h volume divides by
zero on the first poll.

### 5. Undefined pieces you would have to invent

- `kelly(edge)` — `SIZE` step 1 depends on it; `edge` is never defined, computed,
  or sourced anywhere in the guide. The sizing formula has no input.
- `desk.bank()`, `desk.read_x()`, `desk.log_shadow()`, `desk.report()`,
  `desk.send_to_seats()` — all five are listed as things "your Grok Bot side has
  to provide". None are written.
- "The six Grok Bot seats **from the original guide**" — this article depends on a
  prior article not included here.

### Smaller code issues

- **Internal contradiction on `worth_trading_at_all`.** The `PICK` rules say
  *"NEVER skip `worth_trading_at_all`"*, but `run_once` routes a single survivor
  around `pick()` entirely, so the absolute gate never runs in that path. The prose
  acknowledges this ("One survivor goes to the SOFT gates and straight to the
  bot"), which means the rule and the code disagree on the page.
- **Chain id mismatch.** `collect.GT_NET` maps `8453` → `"base"`; `main.CHAIN_SET`
  maps `8453` → `"bsc"`. `universe()` defaults to Solana/BSC/Robinhood only, so
  Base never enters the funnel and the inconsistency stays dormant rather than
  being caught.
- **Field-shape assumption.** `normalise()` reads `m["change"].get(300)`,
  `.get(3600)`, `.get(14400)`, `.get(86400)` — integer second keys. The documented
  FOMO response names those fields `change5m / change1 / change4 / change12 /
  change24`. One of the two is wrong, and it fails silently as `None` rather than
  raising.
- **`judge.py` is exposed publicly.** It binds `0.0.0.0` and is published via
  `cloudflared tunnel`, guarded only by a shared `DESK_SECRET` with no rate
  limiting. If that secret leaks, a stranger can spend your TypeSafe credits.

### What checks out

Not everything here is vapour, and it's worth being precise about that:

- **TypeSafe and Jev are real.** Out of stealth 15 September 2026, $40M seed led
  by DCVC; a founder co-authored the InstructGPT work. The quoted economics
  ($0.042/Mtok input, free output), the 70–500ms latency, and the
  `noul`/`choice`/`score` primitives all match the real product.
- **`typesafe-sdk` is the genuine package** — PyPI v0.7.4, `requires_python
  >=3.10`, project URLs pointing at `typesafe.ai` and `github.com/typesafe-ai`.
- **The funnel design is sound engineering.** Ordering the passes cheapest-first,
  killing on plain facts before spending a model call, benching a rejection for a
  duration keyed to *which* check failed, and refusing to let a held position
  trigger a new scan — these are real, sensible ideas, well explained.

**The infrastructure is real. The profit claim is not evidenced, and the money
path routes through the author's referral link.** Those are separate statements
and both are true.

---

## 1. What you need first

Six things. Each one says what it is, what you do with it, and why the desk needs
it.

### 1.1 A Jev key

**What it is.** TypeSafe came out of stealth September 15, $40M led by DCVC,
founder co-wrote the InstructGPT paper. Jev does not generate text at all. You
send it state and typed questions, it sends back typed answers with probabilities
in 70 to 500ms.

**What you do.** Sign in at `console.typesafe.ai`, open Keys, create one, copy it
the moment it appears because you will not see it again. Put it in your
environment:

```bash
export TYPESAFE_API_KEY="ts-..."          # mac / linux
setx TYPESAFE_API_KEY "ts-..."            # windows, open a new terminal after
```

> Never commit a real key. `ts-...` is a placeholder here and should stay one.

**Why.** One service on the desk holds this key and calls this endpoint. No bot
ever sees it.

```http
POST https://api.typesafe.ai/v1/systemone
Authorization: Bearer $TYPESAFE_API_KEY

model:   jev-latest
input:   $0.042 / Mtok
output:  FREE
limits:  250k tok/sec, 1200 req/min
context: 64k, 32k for state + longest question
```

Before anything else, prove the key works. One curl, and you will never again
confuse a bad key with a bad question:

```bash
curl -X POST https://api.typesafe.ai/v1/systemone \
  -H "Authorization: Bearer $TYPESAFE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"state":"payouts have been failing for 3 days","model":"jev-latest",
       "questions":{"urgent":{"type":"noul","instructions":"This conveys urgency"}}}'
```

```json
{"model":"jev-1.13.0",
 "answers":{"urgent":{"type":"noul","noul":0.95}},
 "usage":{"input_tokens":296,"output_tokens":20}}
```

### 1.2 The three question types

**What they are.** The entire API. Every question the desk asks is one of these
three, and you will see them in every file below.

| type | asks | returns |
|---|---|---|
| `noul` | is this true | noul 0 to 1 |
| `choice` | which one, up to 255 options | choice, probabilities, confidence |
| `score` | rate on my rubric, up to 10 levels | score, probabilities, legend, confidence |

**What you do.** Nothing yet, and nothing by hand once it runs.

The playground is a one time thing while you are writing questions: drop any
token in as the state, ask your question, watch the probabilities move as you
reword it. A question is a prompt and it has the same failure modes as a prompt.
Once the wording is right it goes into `questions.py` and you never open the
playground again.

In production nothing is pasted anywhere. `collect.py` pulls fresh pools from
GeckoTerminal, metrics from FOMO twenty at a time, trade counts from DexScreener
and the contract data from the chain, then builds the state and sends it. You log
into FOMO in Chrome once, and that is the only manual step in the entire desk.

**Why.** `noul` is an if, `choice` is a switch, `score` is a sort. Your code reads
the number, the threshold decides, nobody reads a paragraph.

**The split you have to hold onto.** Jev takes the judgements. Code takes the
arithmetic. Every number on the desk is computed before the call and passed in as
a field.

### 1.3 The SDK

**What it is.** The Python client for that endpoint.

**What you do.** One command:

```bash
pip install typesafe-sdk          # needs python 3.10 or newer
```

**Why.** `judge.py` imports `AsyncTypeSafeClient` from it and `questions.py`
imports `Choice`, `Noul` and `Score`. Without it both files fail on their first
line.

### 1.4 A FOMO session

**What it is.** Where the token universe comes from. FOMO has no public API, so
the collector reads your own logged in session.

**What you do.** Log into FOMO in Chrome and leave that profile alone. The bearer
lives about an hour and the client refreshes it from the browser on its own.

> See finding 3. This is the riskiest step in the guide and the code that does it
> is not published.

**Why.** This one call returns every metric the free pass needs, twenty tokens at
a time, so hundreds of candidates cost you one request instead of hundreds.

```http
POST prod-api.fomo.family/proxy/filterTokens
body: ["<address>:<netId>", ...]      20 at a time

-> marketCap, liquidity, volume24, holders, priceUSD,
   change5m / change1 / change4 / change12 / change24, createdAt

netIds: Solana 1399811149 · Robinhood 4663 · BSC 56 · Base 8453 · ETH 1 · Monad 143
```

### 1.5 Chain data

**What it is.** GeckoTerminal, free, no signup, no key.

**What you do.** Nothing to set up. Just hold onto one number: 10 calls per minute.

**Why.** That limit is the shape of the entire funnel. Six calls list fresh pools
across three chains, which leaves three dossiers per cycle, which is why
everything above the dossier has to kill hard.

```http
GET api.geckoterminal.com/api/v2/networks/{net}/new_pools
GET api.geckoterminal.com/api/v2/networks/{net}/tokens/{addr}/info
```

```python
GT_NET = {1399811149: "solana", 4663: "robinhood", 56: "bsc", 8453: "base", 1: "eth"}
```

Read this table before you write a single question. The desk trades Solana, BSC
and Robinhood, and they do not hand back the same fields. The author reports
calling all three on September 23:

| field | Solana | BSC | Robinhood |
|---|---|---|---|
| `holders.count` + distribution | yes | yes | null on fresh |
| `mint_authority` / `freeze_authority` | yes, decides | null | null |
| `is_honeypot` | n/a | true / false | `"unknown"` |
| `twitter_handle` | clean | clean | sometimes a post URL |
| exact top wallet | Solana RPC, free | Etherscan PRO | no free path |
| holder count fallback | not needed | not needed | FOMO `filterTokens` |

So each chain gets its own question set, routed in code. Solana gets
`getTokenLargestAccounts` and `getTokenSupply` off the public RPC for free.
Robinhood's own RPC is `rpc.mainnet.chain.robinhood.com` and answers 403 without
a browser User-Agent. Etherscan V2 is one key for 60+ chains via `?chainid=`, but
`topholders`, `tokenholdercount` and `tokeninfo` are all PRO and none of them
cover Solana or Robinhood. Worth buying if you trade mostly BSC, and it drops into
the dossier step without changing a single question.

### 1.6 The desk itself

**What it is.** The six Grok Bot seats from the original guide, plus the venue
they fill on.

**What you do.** Nothing new. The seats already exist, they just get their token
from a different place now. The project's X account is read by Grok Bot's own X
plugin, and the handle comes off chain rather than from a search, so no bot goes
hunting for "the project's Twitter" and finds a fan account.

**Why.** Fills go through FOMO and nowhere else. One venue, one execution path, so
a bad fill is always traceable to one place, and the referral discount applies to
every trade including the ones your bots place.

> See finding 1. That referral tie is the author's, not yours.

```
0. UNIVERSE  GeckoTerminal new_pools, 3 chains          -> fresh launches
1. LIST      FOMO filterTokens, 20 per call             -> hundreds, one batch
2. FREE CUT  age, liquidity, volume, mcap. No network   -> tens
3. TRADE CUT DexScreener buys and sells, one per token  -> a handful
4. DOSSIER   GeckoTerminal info + chain RPC + X         -> three per cycle
5. JUDGE     market + chain + social per token          -> scored shortlist
6. PICK      one choice over the shortlist              -> one token, or none
```

Each pass is more expensive than the one above it, so each one has to kill harder
than the one below. The free cut touches no network at all: everything it reads
arrived with the FOMO batch. Only what survives it is worth a DexScreener call,
and only what survives that is worth one of your three dossier slots.

---

## 2. The files

Seven files and four prompts. Build them in this order, top to bottom.

### 2.0 JUDGE — `judge.py`

It runs on one machine, holds the only Jev key, and makes no trading decision of
any kind. Bots get a desk secret and never the key itself.

```python
import os
from fastapi import FastAPI, Header, HTTPException
from pydantic import BaseModel
from typesafe_sdk import AsyncTypeSafeClient
from questions import SETS

DESK_SECRET = os.environ["DESK_SECRET"]     # for the bots. NOT the TypeSafe key.
client = AsyncTypeSafeClient()              # reads TYPESAFE_API_KEY itself
app = FastAPI()


class Ask(BaseModel):
    question_set: str
    state: dict


@app.post("/judge")
async def judge(ask: Ask, authorization: str = Header("")):
    if authorization != f"Bearer {DESK_SECRET}":
        raise HTTPException(401, "bad desk secret")
    if ask.question_set not in SETS:
        raise HTTPException(422, f"unknown question set {ask.question_set}")

    qs = SETS[ask.question_set]
    qs = qs(ask.state) if callable(qs) else qs        # pick builds options at call time

    r = await client.system_one(state=ask.state, questions=qs)

    # raw answers out. never flattened, never thresholded here.
    return {"model": r.model,
            "answers": {k: v.model_dump() for k, v in r.answers.items()},
            "usage": r.usage.model_dump()}
```

```bash
pip install fastapi uvicorn
export DESK_SECRET="$(openssl rand -hex 24)"
uvicorn judge:app --host 0.0.0.0 --port 8080
cloudflared tunnel --url http://localhost:8080     # bots run in xAI's cloud, not on your box
```

Rules baked into those forty lines:

```
- ONE CALL PER TOKEN, never one per question. State is tokenized once. Splitting a set
  into five calls multiplies the bill by five for identical answers.
- STATE IS A NAMED OBJECT, never a blob. Send only fields the questions read. Accuracy
  falls as the state fills with material unrelated to the decision.
- ANSWERS GO BACK RAW. The caller owns the threshold, the judge does not.
- THRESHOLDS DO NOT TRANSFER between primitives. A choice is relative and settles which.
  A noul is absolute and can be low for all of them.
- LOG THE MODEL ID from the response. Aliases move when a release ships.
- ARITHMETIC STAYS IN CODE. Compute it, pass the result in as a field.
- 401 bad key, 422 bad question, 429 rate limit, 529 overloaded. Back off on 429 and 529.
  Never retry a 422, the question is wrong and will stay wrong.
```

### 2.1 SCAN and VET — `collect.py`

Jev fetches nothing, it is a function, so this is the code that goes and gets
everything. SCAN runs universe and shortlist, VET runs dossier.

```python
import time, requests
from fomo_api import Fomo                      # Privy bearer out of Chrome over CDP

GT  = "https://api.geckoterminal.com/api/v2"
DEX = "https://api.dexscreener.com/latest/dex/tokens"

# the three chains the desk trades, plus Base which shares the BSC question set
GT_NET   = {1399811149: "solana", 4663: "robinhood", 56: "bsc", 8453: "base"}
FOMO_NET = {v: k for k, v in GT_NET.items()}


def age_minutes(created) -> float:
    """createdAt comes back as epoch seconds or milliseconds depending on the row."""
    if not created:
        return 0.0
    c = float(created)
    if c > 1e11:                                # milliseconds
        c /= 1000
    return max(0.0, (time.time() - c) / 60)


def universe(nets=("solana", "bsc", "robinhood"), pages=2) -> list[str]:
    """Where the whole thing starts. Fresh pools per chain -> ['<addr>:<netId>', ...].
       Costs one GeckoTerminal slot per chain per page, so keep pages small."""
    ids, seen = [], set()
    for net in nets:
        for page in range(1, pages + 1):
            try:
                r = requests.get(f"{GT}/networks/{net}/new_pools",
                                 params={"page": page}, timeout=20).json()
            except Exception:
                break
            for pool in r.get("data", []):
                base = ((pool.get("relationships") or {}).get("base_token") or {})
                gid  = (base.get("data") or {}).get("id")      # 'solana_<addr>'
                if not gid:
                    continue
                addr = gid.split("_", 1)[1]
                tid  = f"{addr}:{FOMO_NET[net]}"
                if tid not in seen:
                    seen.add(tid)
                    ids.append(tid)
    return ids


def normalise(tid: str, m: dict) -> dict:
    """FOMO's field names become the desk's field names, once, here.
       Every file downstream reads these names and only these."""
    addr, net = tid.split(":")
    return {"addr": addr, "net": int(net), "tid": tid, "ticker": m["symbol"],
            "mcap_usd": m["mcap"], "liquidity_usd": m["liq"],
            "volume_h24": m["vol24"], "price_usd": m["price"],
            "holder_count": m["holders"] or None,
            "change": {"5m": m["change"].get(300), "1h": m["change"].get(3600),
                       "4h": m["change"].get(14400), "24h": m["change"].get(86400)},
            "age_minutes": age_minutes(m["created"])}


def shortlist(fomo: Fomo, ids: list[str]) -> list[dict]:
    """Pass one over everything FOMO knows. No network beyond FOMO itself:
       one call per twenty tokens, and not a single request per token."""
    out = []
    for tid, m in fomo.tokens(ids).items():             # 20 per call
        t = normalise(tid, m)
        if t["net"] in GT_NET:
            out.append(t)
    # turnover ranks the queue. It orders work, it does not decide anything
    out.sort(key=lambda t: t["volume_h24"] / max(t["mcap_usd"], 1), reverse=True)
    return out


def trade_counts(t: dict) -> dict:
    """buys and sells per window. FOMO does not return them, DexScreener does.
       Called ONLY for tokens that already cleared the free checks. One per token,
       so this runs on tens, never on the whole universe."""
    try:
        pairs = requests.get(f"{DEX}/{t['addr']}", timeout=20).json().get("pairs") or []
    except Exception:
        return {"buys_h1": None, "sells_h1": None, "trades_h24": None}
    if not pairs:
        return {"buys_h1": None, "sells_h1": None, "trades_h24": None}
    x = max(pairs, key=lambda p: (p.get("liquidity") or {}).get("usd") or 0)["txns"]
    return {"buys_h1": x["h1"]["buys"], "sells_h1": x["h1"]["sells"],
            "buys_h6": x["h6"]["buys"], "sells_h6": x["h6"]["sells"],
            "trades_h24": x["h24"]["buys"] + x["h24"]["sells"]}


def dossier(t: dict) -> dict:
    """One GT call per token. Fills what the chain actually has, null where it does not."""
    net = GT_NET[t["net"]]
    a = requests.get(f"{GT}/networks/{net}/tokens/{t['addr']}/info",
                     timeout=20).json()["data"]["attributes"]

    d = {**t, "chain": net,
         # GT first, FOMO as the fallback. On Robinhood GT is null and FOMO is all you get.
         "holder_count": (a.get("holders") or {}).get("count") or t["holder_count"],
         "top_10_percent": ((a.get("holders") or {}).get("distribution_percentage")
                            or {}).get("top_10"),
         "developer_holding_percentage": a.get("developer_holding_percentage"),
         "gt_score_details": a.get("gt_score_details"),
         "is_honeypot": a.get("is_honeypot"),
         "mint_authority": a.get("mint_authority"),
         "freeze_authority": a.get("freeze_authority"),
         "description": a.get("description"),
         "x_handle": clean_handle(a.get("twitter_handle"))}

    # Solana only: exact top wallet share, free, off the public RPC
    if t["net"] == 1399811149:
        d["top_wallet_percent"] = sol_top_wallet(t["addr"])

    return d


def clean_handle(h):
    """GT returned 'LuffyX100X/status/2102659581109272876' on a Robinhood token.
       Take the first path segment, or treat the account as missing."""
    if not h:
        return None
    h = h.strip().lstrip("@").split("?")[0].split("/")[0]
    return h if h and h.replace("_", "").isalnum() and len(h) <= 15 else None


def sol_top_wallet(mint: str):
    rpc = "https://api.mainnet-beta.solana.com"
    q = lambda m, p: requests.post(rpc, json={"jsonrpc": "2.0", "id": 1,
                                              "method": m, "params": p},
                                   timeout=20).json()["result"]
    supply = float(q("getTokenSupply", [mint])["value"]["amount"])
    top = q("getTokenLargestAccounts", [mint])["value"]
    return float(top[0]["amount"]) / supply if supply and top else None


def social_state(d: dict) -> dict:
    """What SOCIAL hands the judge. The X block is filled by the bot's X plugin."""
    return {"x_account": d["x_account"],                 # collected by SOCIAL, not here
            "token": {"ticker": d["ticker"], "narrative": d.get("description")}}
```

And the client every seat uses to reach the judge. Six lines, and it is the only
place a bot touches the network for a judgement:

```python
# judge_client.py
import os, requests

URL, SECRET = os.environ["JUDGE_URL"], os.environ["DESK_SECRET"]


def judge(question_set: str, state: dict) -> dict:
    r = requests.post(URL, timeout=30,
                      headers={"Authorization": f"Bearer {SECRET}"},
                      json={"question_set": question_set, "state": state})
    if r.status_code == 422:
        raise RuntimeError(f"malformed question set {question_set}: {r.text}")
    r.raise_for_status()
    return r.json()
```

```
NEVER invent a number. A field that came back null stays null and the question sees it.
NEVER let null mean fine. Missing data gets its own option and its own consequence.
NEVER pull a dossier for a token stage 2 killed. That is a wasted rate limit slot.
```

### 2.2 QUESTIONS — `questions.py`

`judge.py` imports `SETS` from it, and this is the one file you will actually
reread later. Every question the desk can ask lives here and nowhere else.

```python
from typesafe_sdk import Choice, Noul, Score

MARKET = {
    "shape": Choice(
        instructions="Classify the shape of this launch from the fields in `state`.",
        criteria={
            "crowd": "Holders growing faster than price. Buys outnumber sells across both "
                     "recent windows. Volume spread rather than spiking once.",
            "one_buyer": "Price climbing faster than holders. Holder growth flat while "
                         "price rises. One wallet walking the price up.",
            "fading": "Recent volume is a small fraction of the daily average, or sells "
                      "outnumber buys in both recent windows.",
            "too_early": "Too few data points to tell any of the above apart yet.",
        }),
    "liquidity_fits_ticket": Noul(
        instructions="A position of `intended_ticket_usd` could be exited into "
                     "`liquidity_usd` without moving the price more than a few percent."),
    "momentum_already_spent": Noul(
        instructions="The move in `change` has already happened, so entering now means "
                     "buying after the information is public.",
        criteria={"true": "The largest change sits in the older windows.",
                  "false": "The recent windows carry the move."}),
}

CONCENTRATION = Noul(
    instructions="Holding this token means being exit liquidity, based on "
                 "`top_10_percent`, `top_wallet_percent` and `holder_count`.",
    criteria={"true": "A few wallets can end the market by selling.",
              "false": "The float is spread widely enough to absorb a large holder."})

DEV_LOADED = Noul(
    instructions="`developer_holding_percentage` is large enough that the creator "
                 "selling would meaningfully move the price.")

CHAIN_SOLANA = {                      # 1399811149
    "authority_risk": Choice(
        instructions="Judge contract control risk from `mint_authority` and "
                     "`freeze_authority`.",
        criteria={
            "renounced": "Both null. Supply cannot be inflated, balances cannot be frozen.",
            "mint_open": "mint_authority is set. Supply can be inflated at will.",
            "freeze_open": "freeze_authority is set. Balances can be frozen at will.",
            "both_open": "Both are set.",
        }),
    "concentration_is_exit_risk": CONCENTRATION,
    "dev_still_loaded": DEV_LOADED,
}

CHAIN_BSC = {                         # 56, same set works for Base
    "sell_side_risk": Choice(
        instructions="Judge whether a position here can be sold, from `is_honeypot`, "
                     "`gt_score_details` and the buy and sell counts.",
        criteria={
            "clean": "Not flagged, and sells are going through in the data.",
            "flagged": "Explicitly flagged as a honeypot.",
            "suspicious": "Not flagged, but sells are absent or vanishingly rare while "
                          "buys are plentiful.",
            "unknown": "The honeypot field is missing or unknown and trade counts are "
                       "too thin to stand in for it.",
        }),
    "concentration_is_exit_risk": CONCENTRATION,
    "pool_quality": Score(
        instructions="Rate the pool from `gt_score_details` and `liquidity_usd`.",
        criteria=["Thin and new. One withdrawal ends the market.",
                  "Usable, but a large ticket would move it.",
                  "Deep enough that normal desk size is invisible."]),
}

CHAIN_ROBINHOOD = {                   # 4663, the one with holes in the data
    "data_coverage": Choice(
        instructions="Judge how much of this token is visible, from which fields in "
                     "`state` carry values and which are null or unknown.",
        criteria={
            "indexed": "Holder count and distribution present, honeypot field is a real "
                       "answer.",
            "partial": "Holder count present from the venue, but distribution or the "
                       "honeypot field is missing.",
            "dark": "Neither distribution nor honeypot available. Only price, volume and "
                    "pool age are known.",
        }),
    "sellable_by_evidence": Noul(
        instructions="Sells are going through on this token, judged from the buy and sell "
                     "counts rather than from any honeypot flag.",
        criteria={"true": "Sells appear across recent windows in a normal ratio.",
                  "false": "Buys with almost no sells, or no trades at all."}),
    "concentration_is_exit_risk": CONCENTRATION,
    "dev_still_loaded": DEV_LOADED,
}

SOCIAL = {
    "account_is_the_project": Noul(
        instructions="The account in `x_account` is the token's official account, not a "
                     "fan account, an impersonator, or an unrelated similar name.",
        criteria={"true": "Handle matches the one published on chain and the content is "
                          "about this token.",
                  "false": "Similar name, different subject, or no link back."}),
    "audience_is_real": Noul(
        instructions="Engagement in `x_account` is consistent with its follower count, "
                     "rather than a large follower number with almost no replies or "
                     "reposts on recent posts."),
    "recycled_account": Noul(
        instructions="`x_account` shows signs of being repurposed: far older than the "
                     "token, with a handle or content history belonging to a different "
                     "project."),
    "effort": Score(
        instructions="Rate how much work is visibly behind this project from `x_account`.",
        criteria=["One post, one image, nothing else.",
                  "A handful of posts, all promotional.",
                  "Regular posting with substance beyond price.",
                  "A visible team shipping visible things."]),
}


def PICK(state):
    """Options built from the shortlist at call time. Choice takes up to 255."""
    return {
        "best": Choice(
            instructions="Choose the single token in `candidates` that is the best entry "
                         "right now. Weigh crowd shape, contract risk, concentration and "
                         "the project account together. Prefer a clean unspent setup over "
                         "a larger move that already happened.",
            criteria={c["ticker"]: c["summary"] for c in state["candidates"]}),
        "worth_trading_at_all": Noul(
            instructions="At least one token in `candidates` is worth a position today, "
                         "rather than all of them being mediocre.",
            criteria={"true": "At least one is a clean setup.",
                      "false": "Every candidate has a disqualifying weakness."}),
    }


SETS = {"market": MARKET, "social": SOCIAL, "pick": PICK,
        "solana": CHAIN_SOLANA, "bsc": CHAIN_BSC, "robinhood": CHAIN_ROBINHOOD}
```

Four rules that make this file work:

```
ROUTE THE CHAIN SET IN CODE. mint_authority and freeze_authority carry real values on
  Solana and come back null on every EVM chain, so each chain gets the set written for
  its own evidence. CHAIN_SET does the routing.

data_coverage IS A REAL QUESTION. On Robinhood the newest tokens are the ones
  GeckoTerminal has not indexed yet, so how much you can see is itself an input to size.
  dark does not mean skip, it means cut the ticket.

THE PICK NEEDS worth_trading_at_all NEXT TO IT. A choice is relative and settles which
  of these. The noul is the absolute gate that says whether today is a day at all.

THE SUMMARY IN `candidates` IS TWO LINES, assembled in code from answers Jev already gave.
  My first pick call carried all ten full dossiers and the confidence sagged on every run.
  A fat state costs accuracy.
```

### 2.3 THE FILTER — `thresholds.py` and `filter.py`

The first holds every number, the second holds the order they fire in. When you
retune the desk, you edit `thresholds.py` and nothing else.

```python
# thresholds.py
HARD = {                        # stage 2, arithmetic, runs before anything costs money
    "min_age_minutes":   15,    # younger than this and the data is noise
    "max_age_hours":     72,    # older than this and it is not a launch any more
    "min_liquidity_usd": 12_000,
    "min_volume_h24":    40_000,
    "min_mcap_usd":      60_000,
    "max_mcap_usd":      8_000_000,
    "min_trades_h24":    150,
    "max_top_wallet":    0.05,  # solana only, exact, from RPC
    "max_top_10":        0.60,  # where distribution exists
    "min_holders":       80,
}

SOFT = {                        # applied to Jev's answers, per token
    "concentration_is_exit_risk": ("max", 0.55),
    "momentum_already_spent":     ("max", 0.60),
    "liquidity_fits_ticket":      ("min", 0.60),
    "account_is_the_project":     ("min", 0.70),
    "recycled_account":           ("max", 0.50),
    "audience_is_real":           ("min", 0.45),
    "effort":                     ("min", 1.0),
    "dev_still_loaded":           ("max", 0.55),
    "sellable_by_evidence":       ("min", 0.60),   # robinhood
}

SHAPE_MIN_CROWD = 0.55          # probabilities["crowd"], not the winning label
PICK_MIN_WORTH  = 0.60
PICK_MIN_CONF   = 0.55
DARK_TICKET_CUT = 0.40          # robinhood, data_coverage == dark
NO_SOCIAL_CUT   = 0.60          # no usable X handle: trade smaller, do not skip
```

```python
# filter.py
from thresholds import HARD, SOFT, SHAPE_MIN_CROWD


def free_kill(t) -> str | None:
    """Pass one. Runs on the whole universe, costs nothing, touches no network.
       Everything it reads came back with the FOMO batch."""
    if not HARD["min_age_minutes"] <= t["age_minutes"] <= HARD["max_age_hours"] * 60:
        return "age"
    if t["liquidity_usd"] < HARD["min_liquidity_usd"]:      return "liquidity"
    if t["volume_h24"]    < HARD["min_volume_h24"]:         return "volume"
    if not HARD["min_mcap_usd"] <= t["mcap_usd"] <= HARD["max_mcap_usd"]:
        return "mcap"
    return None


def trade_kill(t) -> str | None:
    """Pass two. One DexScreener call already spent on this token. Tens, not hundreds."""
    if t["trades_h24"] is None:                             return "no_pair"
    if t["trades_h24"] < HARD["min_trades_h24"]:            return "trades"
    if t["sells_h1"] == 0 and (t["buys_h1"] or 0) > 20:     return "no_sells"
    return None


def chain_kill(d) -> str | None:
    """After the dossier, still free. Facts, not judgements."""
    if d.get("top_wallet_percent") is not None and \
       d["top_wallet_percent"] > HARD["max_top_wallet"]:
        return "top_wallet"
    if d.get("top_10_percent") is not None and \
       float(d["top_10_percent"]) / 100 > HARD["max_top_10"]:
        return "top_10"
    if d.get("holder_count") is not None and d["holder_count"] < HARD["min_holders"]:
        return "holders"
    if d["chain"] == "solana" and (d["mint_authority"] or d["freeze_authority"]):
        return "authority_open"          # a fact, no model needed
    if d["chain"] == "bsc" and d.get("is_honeypot") is True:
        return "honeypot"                # also a fact
    return None


def soft_kill(ans) -> str | None:
    """Jev's answers against SOFT. First failure wins."""
    for name, (direction, limit) in SOFT.items():
        a = ans.get(name)
        if a is None:
            continue                     # question not asked for this chain
        v = a.get("noul", a.get("score"))
        if v is None:
            continue
        if direction == "max" and v > limit: return name
        if direction == "min" and v < limit: return name

    shape = ans.get("shape")
    if shape:
        if shape["choice"] in ("fading", "one_buyer"):        return "shape"
        if shape["probabilities"]["crowd"] < SHAPE_MIN_CROWD: return "shape_weak"

    chain = ans.get("sell_side_risk")
    if chain and chain["choice"] in ("flagged", "suspicious"): return "sell_side"
    return None
```

The order is the whole point. `free_kill` touches no network and runs on hundreds.
`trade_kill` costs one DexScreener call and runs on tens. `chain_kill` costs one of
your three GeckoTerminal slots. `soft_kill` costs a judgement and runs on a
handful. Facts kill before judgements do, which is why `authority_open` and
`honeypot` are comparisons here and not questions in `questions.py`. You ask Jev
how dangerous the concentration is, and the comparisons handle the plain facts.

Log every rejection with the check that fired. Under ten rejections a day and your
filter is misconfigured, not your market.

### 2.4 SOCIAL — a prompt, not a file

Paste it into the SOCIAL bot. It took BOOK's chair, and it exists because Grok Bot
ships with an X plugin that reads. Grok reads, Jev judges.

The handle comes off chain and is already normalised by the collector. Never let
this seat go searching for "the project's Twitter", it will find a fan account and
be confident about it. No handle means no social read, and the token carries that
as a gap rather than a pass.

```
You are SOCIAL. You read one X account and you report what is there. You never decide
whether the token is good.

INPUT: x_handle from the dossier. If it is null, return {"x_account": null} and stop.

WHAT YOU COLLECT, with your X plugin, into named fields:
{ "handle": ..., "created_at": ..., "followers": ..., "following": ...,
  "post_count": ..., "posts_last_7d": ...,
  "recent": [ {"text": ..., "posted": ..., "replies": ..., "reposts": ...}, ...x10 ],
  "handle_history": [...] | null,
  "bio": ..., "linked_site": ... }

Then POST question_set "social" with that as state.x_account, plus token.ticker and
token.narrative so the model can tell whether the account is about THIS token.

NEVER summarise the posts. Send them. A summary is your opinion and your opinion is not
part of this pipeline.
NEVER count anything yourself beyond what the plugin returns as a number.
NEVER substitute a similar handle when the exact one returns nothing. Missing is missing.
```

`recycled_account` is the highest value question in the whole file. It holds two
timelines side by side, the account's and the token's, and hands you one number for
the gap.

### 2.5 PICK — `pick.py`

CHIEF runs it, and it is the only call that ever sees more than one token at a
time. A `Choice` takes up to 255 options and your shortlist is never longer than
ten, so the whole decision fits in one request.

```python
# pick.py
from thresholds import PICK_MIN_WORTH, PICK_MIN_CONF, DARK_TICKET_CUT, NO_SOCIAL_CUT


def summary(d, ans) -> str:
    """Two lines per candidate, built from answers Jev already gave.
       Never the raw dossier. A fat state costs accuracy."""
    bits = [f"{d['chain']}, {d['age_minutes']}m old, ${d['mcap_usd']:,.0f} mcap, "
            f"${d['liquidity_usd']:,.0f} liq, {d['holder_count'] or '?'} holders",
            f"crowd {ans['shape']['probabilities']['crowd']:.2f}, "
            f"concentration risk {ans['concentration_is_exit_risk']['noul']:.2f}"]

    if "authority_risk" in ans:
        bits.append(f"authority {ans['authority_risk']['choice']}")
    if "sell_side_risk" in ans:
        bits.append(f"sell side {ans['sell_side_risk']['choice']}")
    if "data_coverage" in ans:
        bits.append(f"data {ans['data_coverage']['choice']}")
    if "account_is_the_project" in ans:
        bits.append(f"official account {ans['account_is_the_project']['noul']:.2f}, "
                    f"effort {ans['effort']['score']:.1f}")
    else:
        bits.append("no usable X account")
    return "; ".join(bits)


def pick(judge, survivors) -> dict | None:
    """survivors: [(dossier, answers), ...]. Returns the order, or None."""
    if not survivors:
        return None

    state = {"candidates": [{"ticker": d["ticker"], "summary": summary(d, a)}
                            for d, a in survivors]}
    r = judge("pick", state)
    best, worth = r["answers"]["best"], r["answers"]["worth_trading_at_all"]

    if worth["noul"] < PICK_MIN_WORTH:
        return None                      # every candidate is mediocre. Normal outcome.
    if best["confidence"] < PICK_MIN_CONF:
        return None                      # flat over ten options means no favourite.

    d, ans = next((x for x in survivors if x[0]["ticker"] == best["choice"]), (None, None))
    if d is None:
        return None                      # the schema guarantees the option is in the
                                         # list, so log this one and stand down.

    size_factor = 1.0
    if ans.get("data_coverage", {}).get("choice") == "dark":
        size_factor *= DARK_TICKET_CUT   # less visibility, smaller ticket
    if "account_is_the_project" not in ans:
        size_factor *= NO_SOCIAL_CUT

    return {"model": r["model"],
            "token": {"ticker": d["ticker"], "address": d["addr"],
                      "network_id": d["net"], "chain": d["chain"]},
            "size_factor": round(size_factor, 2),
            "confidence": best["confidence"],
            "runner_up": sorted(best["probabilities"].items(),
                                key=lambda kv: -kv[1])[1:2],
            "why": {k: v for k, v in ans.items()}}
```

```
NEVER re-rank the winner. The choice settled it. If you disagree with the pick, you
  disagree with a threshold, and thresholds live in thresholds.py.
NEVER run pick on one survivor. A choice over a single option returns that option with
  high confidence, because it is the only thing there. One survivor goes to the SOFT
  gates and straight to the bot, or nowhere.
NEVER skip worth_trading_at_all. A choice settles which of these. The noul is what
  settles whether today is a day at all, so the two always ship together.
```

No trade is a result. Log it with the reason, send it to Telegram, and stand down
until the next run. A desk that trades every cycle is a desk with no filter.

### 2.6 THE HANDOFF — a prompt

Paste this above the prompt of every seat that makes a judgement: SCAN, VET,
SOCIAL and CHIEF. It is what removes the bot's own opinion from the loop.

```
YOU DO NOT FORM OPINIONS ABOUT TOKENS. YOU CALL THE JUDGE.

You do not reason about a candidate, you do not weigh it, you do not write a paragraph
about it. You build a state, you call the judge once, you act on the numbers.

THE CALL
  POST $JUDGE_URL
  Authorization: Bearer $DESK_SECRET
  {"question_set": "<market|solana|bsc|robinhood|social|pick>", "state": {...}}

WHAT COMES BACK
  {"model":"jev-1.13.0",
   "answers":{"<name>":{"type":"noul","noul":0.72}, ...},
   "usage":{"input_tokens":1387,"output_tokens":54}}

HOW YOU USE IT
- Compare the numbers against the thresholds in your prompt. That comparison is the
  decision. You do not have a second opinion about it.
- A noul is a probability, not a yes. 0.49 and 0.51 are nearly the same reading and the
  threshold is what makes them different. Never narrate around a number near your line.
- confidence is a separate axis. Low confidence is not a no, it is a do not act alone.
- Log every answer with the model id, exactly as returned.

WHAT YOU NEVER DO
- Never call api.typesafe.ai directly. You do not have that key and will not be given it.
- Never ask for a question set that is not yours. Unknown sets return 422, that is the
  system working.
- Never retry a 422.
- Never substitute your own judgement when the judge is unreachable. A missing answer is
  missing, not neutral, and no token passes on your say so.
- Never put a number in a report that did not come from the judge or from your own
  arithmetic on desk data.
```

Record one judge call by hand in front of Grok Bot and save it as a skill. Skills
are shared across every bot on the desk, so you paste this once instead of six
times.

Prove the link from a bot's own terminal, not from your laptop. Passing on your
machine and failing on theirs is a firewall problem and it is how this setup dies
quietly.

```bash
curl -X POST $JUDGE_URL -H "Authorization: Bearer $DESK_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"question_set":"market","state":{"ticker":"TEST","age_minutes":42,
       "holder_count":310,"change":{"5m":0.04,"1h":0.22,"24h":0.61},
       "buys_h1":540,"sells_h1":120,"liquidity_usd":48000,"mcap_usd":310000,
       "volume_h24":610000,"intended_ticket_usd":900}}'
```

`answers.shape.choice` must be one of your options, `probabilities` must sum to 1,
`model` must be a version string and not an alias. If any of the three is off, stop
here.

What CHIEF drops into the desk channel when the pick comes back. This is the whole
handoff, and the bots work it top to bottom:

```json
{
  "order_id": "2026-09-23T10:15:00Z",
  "token": {"ticker": "...", "address": "...", "network_id": 1399811149,
            "chain": "solana"},
  "size_factor": 1.0,
  "confidence": 0.78,
  "why": {
    "shape": "crowd", "crowd_p": 0.91,
    "concentration_is_exit_risk": 0.14,
    "authority_risk": "renounced",
    "account_is_the_project": 0.97, "recycled_account": 0.06, "effort": 2.4
  },
  "model": "jev-1.13.0"
}
```

```
THE ORDER, WORKED IN THIS SEQUENCE, NOBODY SKIPS AHEAD

1. SIZE   reads token + size_factor. Computes the ticket by the four steps in its
          prompt. Returns dollars, or 0 with a reason. A 0 ends the order here.
2. FILLS  reads the ticket. Checks the fee floor BEFORE sending. Sends one market
          order through FOMO. Reports filled, fill_price, slippage_bps, partial.
3. RISK   starts its timer the moment a fill is reported, not when the order was
          created. Polls every 5 minutes. Fires on its own authority.
4. CHIEF  logs the order id, the model id and every answer that produced it, then
          sends the line to Telegram.

NOBODY RE-READS `why`. It is there for the log and for you, not as an input. SIZE does
not size up because crowd_p was 0.91, and RISK does not hold longer because confidence
was high. The judgement is finished. What is left is arithmetic.

NO ORDER IS ALSO AN ORDER. When pick returns nothing, CHIEF sends one line saying so
with the reason, and the desk stands down until the next run.
```

### 2.7 SIZE, FILLS, RISK — three prompts

One per seat. Nothing to install. These are unchanged from the original guide,
except the ticker now arrives from the pick and carries a `size_factor` with it. No
judge call between the three of them, and no key.

```
SIZE   1. ticket = kelly(edge) * bank, clamped at 6% of the book. Free cash only,
          never the locked bag.
       2. ticket *= size_factor from the pick. dark data cuts to 0.40, a missing X
          account cuts to 0.60, both stack.
       3. ticket = min(ticket, liquidity_usd * 0.02). If you are more than 2% of the
          pool you are the exit, not a participant.
       4. if ticket < fee floor viable size -> return 0 and log it. Never size below
          what pays its own fees.
       Not exitable inside the slippage budget means the size is wrong, whatever the
       pick confidence said.

FILLS  1. effective_fee = max(0.0045 * ticket, 0.95) / ticket
       2. over the max -> do not send, return FEE_FLOOR, let SIZE raise or drop it.
          A $20 entry against a $0.95 floor is 4.75% round trip and no meme edge
          covers that.
       3. one market order through FOMO, no ladder, no waiting for a better price.
       4. slippage over max -> complete and flag loudly, never absorb it silently.
       5. never sell into a distributing whale. Hold and report.
       Fills go through fomo.family/r/savipww and nowhere else. One venue, one path,
       so a bad fill is always traceable to one place.

RISK   One rule, no conversation, final authority, nobody overrules it.
         avg_6h = volume.h24 / 4
         ratio  = volume.h6 / avg_6h
         ratio < 0.20 -> CLOSE, fully, inside 60 seconds.
       Poll every 5 minutes. No answer, retry twice, then CLOSE anyway. A position you
       cannot measure is a position you do not hold.
       The moment the close is filled, call book.release(). Until you do, the desk does
       not scan, so a close you forgot to report is a desk that stopped working.
```

> See findings 1, 2 and 4. This section contains the hardcoded referral URL, the
> fee-floor arithmetic that rules out a small bank, and the volume-only exit with
> no price stop.

The exit is a single comparison on two numbers the desk already holds, so it runs
the instant the data lands and it runs the same way every time. That is what you
want in the seat that closes.

Jev picks what to hold. Grok Bot decides how much and how long. The exit rule
answers to neither of them.

### 2.8 THE BOOK — `book.py`

It creates `desk.db` on first run, nothing to set up. Two things the desk has to
remember between cycles, and skipping either one costs real money.

**One position at a time.** The cycle runs every fifteen minutes. Without a guard
it picks a second token while the first is still open, then a third, and by evening
RISK is trying to manage a portfolio nobody sized. While a position is open, the
scan does not run at all.

**A rejected token stays rejected for a while.** A token that failed at 10:00 is
still the same token at 10:15. Without a bench it burns a GeckoTerminal slot and
three judge calls every cycle until it ages out. And the bench is not one length: a
honeypot flag will never become false, but too early to tell becomes tellable
within the hour.

```python
# book.py
import sqlite3, time

DB = sqlite3.connect("desk.db", check_same_thread=False)
DB.executescript("""
CREATE TABLE IF NOT EXISTS position(
  id INTEGER PRIMARY KEY CHECK (id = 1),
  ticker TEXT, addr TEXT, net INT, opened_at REAL);
CREATE TABLE IF NOT EXISTS bench(
  tid TEXT PRIMARY KEY, reason TEXT, until REAL);
""")

# how long a rejection stands, by what fired it
BENCH_MINUTES = {
    # facts that will not change while this token exists
    "honeypot": 100_000, "authority_open": 100_000,
    "top_wallet": 100_000, "sell_side": 100_000,
    # slow to change
    "recycled_account": 360, "account_is_the_project": 360,
    # can change as the float moves
    "top_10": 90, "holders": 90, "dev_still_loaded": 90,
    "concentration_is_exit_risk": 90,
    # can change inside the hour, keep it short or you miss the token maturing
    "shape": 25, "shape_weak": 25, "momentum_already_spent": 25,
    "liquidity_fits_ticket": 25, "liquidity": 25, "volume": 25,
    "trades": 25, "mcap": 25, "age": 20,
}
DEFAULT_BENCH = 45


def held():
    r = DB.execute("SELECT ticker, opened_at FROM position WHERE id=1").fetchone()
    return {"ticker": r[0], "minutes": (time.time() - r[1]) / 60} if r else None


def take(order):
    t = order["token"]
    DB.execute("INSERT OR REPLACE INTO position VALUES (1,?,?,?,?)",
               (t["ticker"], t["address"], t["network_id"], time.time()))
    DB.commit()


def release():
    """RISK calls this the moment a close is filled. Nothing else calls it."""
    DB.execute("DELETE FROM position")
    DB.commit()


def benched(tid: str) -> bool:
    r = DB.execute("SELECT until FROM bench WHERE tid=?", (tid,)).fetchone()
    return bool(r and r[0] > time.time())


def sit(tid: str, reason: str):
    mins = BENCH_MINUTES.get(reason, DEFAULT_BENCH)
    DB.execute("INSERT OR REPLACE INTO bench VALUES (?,?,?)",
               (tid, reason, time.time() + mins * 60))
    DB.commit()
```

```
RISK CALLS release() AND NOBODY ELSE DOES. Not CHIEF, not SIZE, not you from a console
  because the chart looks fine. The seat that closed the position is the seat that frees
  the book, and that is the only way the two can never disagree.

BENCH ON THE FAILED CHECK, NOT ON THE TOKEN. The reason is what sets the length. Bench
  everything for the same hour and you will keep re-paying for honeypots while missing
  the token that was simply five minutes too young.

A HELD POSITION MEANS NO SCAN AT ALL. Not a scan that ends in no trade, no scan. You
  save the rate limit and the judge calls for the cycle where you can actually act.
```

### 2.9 THE SHIFT — `main.py`

This is the process that never stops: it owns the cycle, calls everything above in
order, and hands finished orders to the seats. Start it with `shadow=True` and
leave it that way for a week.

```python
# main.py
import time, logging
from fomo_api import Fomo
from collect import universe, shortlist, trade_counts, dossier, social_state
from filter import free_kill, trade_kill, chain_kill, soft_kill
from pick import pick
import book

CHAIN_SET     = {1399811149: "solana", 56: "bsc", 8453: "bsc", 4663: "robinhood"}
CYCLE_SECONDS = 900
GT_PER_MINUTE = 10          # free tier
GT_UNIVERSE   = 6           # 3 chains x 2 pages, spent before the funnel starts
GT_DOSSIER    = 3           # what is left for dossiers in the same minute
DEX_BUDGET    = 25          # DexScreener calls per cycle, pass two only
log = logging.getLogger("desk")


def run_once(fomo, judge, desk, bank, shadow=True):
    if (h := book.held()):                       # RISK owns the desk right now
        log.info("holding %s for %.0f min, no scan this cycle",
                 h["ticker"], h["minutes"])
        return None, {"held": h["ticker"], "minutes": round(h["minutes"])}

    stats = {"seen": 0, "benched": 0, "free": {}, "trade": {},
             "chain": {}, "soft": {}}
    survivors = []
    gt_slots, dex_slots = GT_DOSSIER, DEX_BUDGET

    ids = universe()                             # fresh pools, 3 chains, GT_UNIVERSE slots
    for t in shortlist(fomo, ids):               # pass one: free, no per-token requests
        stats["seen"] += 1
        if book.benched(t["tid"]):               # already judged, still serving its time
            stats["benched"] += 1
            continue
        if (k := free_kill(t)):
            book.sit(t["tid"], k)
            stats["free"][k] = stats["free"].get(k, 0) + 1
            continue

        if dex_slots <= 0 or gt_slots <= 0:
            break                                # out of budget, not out of ideas

        t |= trade_counts(t)                     # pass two: one DexScreener call
        dex_slots -= 1
        if (k := trade_kill(t)):
            book.sit(t["tid"], k)
            stats["trade"][k] = stats["trade"].get(k, 0) + 1
            continue

        try:
            d = dossier(t)                       # pass three: one GeckoTerminal slot
            gt_slots -= 1
        except Exception as e:
            log.warning("dossier failed %s: %s", t["ticker"], e)
            gt_slots -= 1                        # a failed call still cost you the slot
            book.sit(t["tid"], "dossier_failed")
            continue                             # missing is missing, not a pass

        if (k := chain_kill(d)):
            book.sit(t["tid"], k)                # facts bench longest
            stats["chain"][k] = stats["chain"].get(k, 0) + 1
            continue

        d["intended_ticket_usd"] = bank * 0.06   # the most SIZE could ever allow
        d["x_account"] = desk.read_x(d["x_handle"]) if d["x_handle"] else None

        ans = {}
        try:
            ans |= judge("market", d)["answers"]                  # pass four
            ans |= judge(CHAIN_SET[d["net"]], d)["answers"]
            if d["x_account"]:
                ans |= judge("social", social_state(d))["answers"]
        except Exception as e:
            log.warning("judge failed %s: %s", d["ticker"], e)
            continue                             # no bench: the token is not at fault

        if (k := soft_kill(ans)):
            book.sit(t["tid"], k)
            stats["soft"][k] = stats["soft"].get(k, 0) + 1
            continue

        survivors.append((d, ans))

    log.info("cycle: %(seen)s seen, %(benched)s benched, free %(free)s, "
             "trade %(trade)s, chain %(chain)s, soft %(soft)s", stats)

    if not survivors:
        return None, stats
    if len(survivors) == 1:                      # a choice over one option proves nothing
        d, ans = survivors[0]
        order = {"model": "single-survivor", "size_factor": 1.0, "confidence": None,
                 "token": {"ticker": d["ticker"], "address": d["addr"],
                           "network_id": d["net"], "chain": d["chain"]},
                 "why": ans}
    else:
        order = pick(judge, survivors)           # pass five

    if order is None:
        return None, stats
    if shadow:
        desk.log_shadow(order, stats)            # written, never sent
        return None, stats

    book.take(order)          # the desk is now held. No scan until RISK calls release().
    return order, stats


def main(fomo, judge, desk, shadow=True):
    """desk is your Grok Bot side. It has to provide five things:
         bank()                 -> float, free cash right now
         read_x(handle)         -> the X block SOCIAL collects with its plugin, or None
         log_shadow(order, st)  -> append a row for the shadow week
         report(order, stats)   -> one line to Telegram, trade or no trade
         send_to_seats(order)   -> hand it to SIZE, then FILLS, then RISK, in that order
    """
    while True:
        try:
            fomo.token()                         # Privy bearer lives ~60 min, refresh it
            order, stats = run_once(fomo, judge, desk, desk.bank(), shadow)
            desk.report(order, stats)            # every cycle, trade or not
            if order:
                desk.send_to_seats(order)
        except Exception as e:
            log.exception("cycle blew up: %s", e)
        time.sleep(CYCLE_SECONDS)
```

The budget is the design. GeckoTerminal gives you ten calls a minute, six of them
go to listing fresh pools across three chains, so three dossiers per cycle is what
is left. That is why the free pass has to kill hard: by the time a token reaches a
dossier it has already survived everything a comparison can do. Want more dossiers,
cut pages to 1 and you get six.

Failure handling, because this runs unattended:

```
429 from GeckoTerminal  -> you exceeded 10/min. Back off a full minute, do not retry in
                           place. If it keeps happening, drop universe(pages=1): six
                           slots freed, three more dossiers per cycle.
429 from DexScreener    -> lower DEX_BUDGET. Pass two is the only place it is called and
                           it is capped per cycle for exactly this reason.
429 or 529 from Jev     -> the SDK retries with backoff on its own. Leave it alone.
422 from Jev            -> your question is malformed. Never retry. Log the field and
                           stop the cycle, because every token will hit the same wall.
dossier throws          -> skip that token. Not a pass, not a retry loop.
judge unreachable       -> skip the cycle entirely. A desk with no judge does not fall
                           back to guessing, it stands down.
FOMO token expired      -> refresh the Privy bearer out of Chrome and continue. It dies
                           roughly hourly and that is normal.
```

Run it with `shadow=True` for a week. It does everything except send the order and
take the book, and `desk.log_shadow` writes one row per would-be trade: the ticker,
every answer with the model id, the rejection counters, and what your old logic did
with the same candidates. Read only the rows where the two disagree. That is twenty
minutes each evening and it is where your thresholds come from. Then flip the flag.

One more thing that bites unattended runs: the FOMO bearer lives about an hour.
`main` calls `fomo.token()` at the top of every cycle so the client can refresh it
out of Chrome before the batch goes out, rather than discovering it expired halfway
through a funnel.

---

## 3. The bill

State is charged once per call, output is free.

```
cost_per_call = (state_tokens + question_tokens) / 1_000_000 * 0.042
```

The rate limit upstream caps this for you. Three dossiers a cycle means at most
three tokens reach the judge, and each one takes three calls (market, chain,
social) plus one pick over the survivors. Ten calls, roughly 1,400 input tokens
each:

```
10 calls x 1,400 tokens = 14,000 tokens
14,000 / 1,000,000 x $0.042 = $0.00059 per cycle
96 cycles a day (every 15 min) = about 5.7 cents a day
```

Under six cents a day for every judgement the desk makes. The DexScreener and
GeckoTerminal calls that feed it are free, and the only thing you actually pay for
besides this is the SuperGrok plan the bots already run on.

Split any set into one call per question and you pay three to four times that for
identical answers, because the state gets charged again each time. That is the
whole reason `questions.py` groups them into sets instead of exposing them one at a
time.

> **This arithmetic checks out.** Recomputed: $0.000588 per cycle, $0.0564/day
> (~5.6 cents), about $1.69/month. The model-inference cost genuinely is trivial.
> Note what that figure excludes, though: trading fees (section 2.7), the SuperGrok
> plan, and any VPS. The cheap number is the cheapest part of the bill.

### The author's own disclaimer

> **This is my setup, not your strategy!!!**
>
> Every threshold above is mine, tuned against the launches I trade, over one week,
> on my bank and my risk tolerance. Run it in shadow mode first: log every answer
> next to what you would have decided yourself, and move the number where you
> disagree. Copy the shape, not the constants! NFA and Always DYOR!
>
> Code fetches, the model judges, code decides. A seat that computes does not get a
> judge call. A seat that judges does not get a calculator.
>
> The full desk this sits on top of: the original setup guide! (six figs was made
> using this guide)

> **Note the last line.** The six-figure result is attributed to the **original**
> guide, not to this one. So the headline on this article — *"Six Grok Bots That
> Already Made Me Six Figures"* — points at a prior article that is not included
> here. This document cannot be assessed as the thing that produced the number, and
> the prior guide was never reviewed.
>
> "Copy the shape, not the constants" is sound advice and worth taking seriously.
> It also sits oddly beside finding 2: the shape itself — a 6%-of-bank clamp against
> a $0.95 flat fee floor — is what rules out a small bank, independently of any
> constant you retune.

---

## Files the guide names

| file | shown? | role |
|---|---|---|
| `judge.py` | yes | holds the Jev key, returns raw answers |
| `collect.py` | yes | universe, shortlist, trade counts, dossier |
| `judge_client.py` | yes | the six-line client each seat uses |
| `questions.py` | yes | every question the desk can ask |
| `thresholds.py` | yes | every tunable number |
| `filter.py` | yes | the order the kills fire in |
| `pick.py` | yes | one choice over the shortlist |
| `book.py` | yes | position guard + rejection bench |
| `main.py` | yes | the cycle |
| **`fomo_api.py`** | **NO** | **reads your Privy bearer out of Chrome over CDP** |

Prompts (not files): SOCIAL, THE HANDOFF, SIZE, FILLS, RISK.

---

*Archived 12 October 2026. Nothing in this repository has been executed.*
