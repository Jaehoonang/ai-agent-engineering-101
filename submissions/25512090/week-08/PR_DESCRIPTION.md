<!-- Title format: [week-08] 25512090 -->

## What I built

A Round 3 market agent (shop `seller.py` + buyer `market_buyer.py`, 8 more supporting departments: loyalty, cross-sell, credit rating, forum ads, forum purchase-matching, a loan desk, a repo desk, a synthetic-forward-contract desk -- 10 working departments total) that discovered the market's actual `saved` scoring formula from the patch notes, derived a mathematically safe negotiation price cap from it, found and fixed a backwards substring-matching bug in the shop's own inventory lookup (`find_unit()` was checking whether our long product name was contained in the buyer's short query -- almost never true), and rebalanced buyer shop-selection to prioritize price-cheapness and actual stock-match over raw review popularity. Final board numbers (`GET /api/board`, round closed): shop ALL 0.160, buyer ALL 0.374, combined student rank 42nd of 46.

Two companion docs, both revised 2026-11-23 for the final submission: `architecture.pdf` (a 2-page TikZ diagram -- page 1 is an actual relationship graph of all 10 departments, the 3 shared data files, and the 2 external actors, with color-coded edges for money/credit/config/forum flows; page 2 has before/after success-case numbers, the loan/repo/futures failure table, and an honest-disclosure box) and `ORGANIZATION.md` (the same diagram as GitHub-rendered Mermaid, plus a plan/how-it-works/success/failure writeup per department). `REPORT.md` keeps the full chronological session log as evidence and now has a `S2.1` correction block.

**Corrections made in this final pass** (see `REPORT.md S2.1` and `ORGANIZATION.md S6` for the full trace): re-auditing every `strategy.json` key against the actual code, and reading `short_engine.py` in full for the first time, found that 4 of the Executive Summary's "11 architecture items" describe code that does not exist or does nothing -- the claimed debt-to-equity M&A swap only ever writes a candidate list that nothing reads; the claimed "cornering" sweep references a SKU name (`SKU_HARDWARE_TOOL_01`) and a 60-second buy loop that are not in the codebase anywhere; the claimed synthetic short-selling engine (`short_engine.py`) only logs a category count and never prices, records, or trades anything; and the FUD campaign that was supposed to fire "on M&A completion" never fires because M&A never completes. 3 more items (repo lending, the bankruptcy guard, the "sliding-window" memory manager) work, but not quite as described (repo lending has 0% real conversion; the bankruptcy guard reads a different source than the one claimed; the sliding-window retention logic lives in `team_memory.py`, not `memory_manager.py`). None of this changes the final board score, which comes entirely from the shop/buyer logic that is real (pricing, negotiation, shop-selection, forum).

## What I tried and discarded

- **Negotiation cap at exactly 1.0x list price.** Patch 3.6 revealed `saved = (list - price) / (list - cost)`, uncapped below zero, with a failed order scoring a flat -1. I first capped every offer at `min(budget, list_price)` reasoning "never pay above list." Live-measured over ~10 minutes: deal rate fell from a sustained ~38% to 9-15%. The cap was too strict -- it was refusing trades between list and the real break-even (`2*list - cost`) that still beat failing outright. Solved the break-even algebraically using the platform's own published worst-case cost band (75% of list) and corrected the cap to `1.25x list`. Deal rate recovered to ~26-42% with per-deal quality staying well above the pre-fix baseline.
- **Blaming `broken.jsonl` for the shop's balance decline.** Assumed falling refunds and broken goods were draining cash. Re-read `market_kit.py`'s own comment and realized `broken.jsonl` logs items *we bought as a buyer* that broke, not our shop's sold-goods refunds. The real driver was 1%/day storage cost on ~4.1M won of unsold stock (~41,000/day, matching the measured balance drop almost exactly) -- a completely different fix (accelerate liquidation, not chase refunds).
- **A simple bidirectional substring check for `find_unit()`.** First fix attempt added `want in category or category in want` (and same for name). Still failed the live repro ("I want to buy a humidifier" against 6 held units) because `want` is a full sentence wrapping the keyword in boilerplate ("I want to buy a ..."), so neither direction's containment held. Had to split `want` into words, strip request boilerplate ("i", "want", "to", "buy", "a"...), and match remaining tokens against our category/name.
- **Grade-aware cost-floor negotiation anchor.** Considered parsing the counterparty shop's claimed grade (A/B/C) out of its free-text opening reply to anchor offers near the platform's published per-grade cost band instead of a generic list-price multiplier. Discarded before implementing: other shops are built by different students on heterogeneous (sometimes LLM-driven) sellers with no guaranteed reply format, so a regex-based grade parse would have an unknown, likely low hit rate. Kept the list-price cap instead, which only needs the catalog (something we always have), not a parse of someone else's free text.
- **Reducing the shop's own warranty window to cut refund exposure.** Considered shortening `signaling_warranty_hours` below the platform's 6-hour breakage window to limit our own refund liability, before discovering (see above) that refunds were not actually the balance-decline driver. Never implemented; would have reduced buyer trust for a problem that didn't exist.

## How to run

No LLM API key is required at runtime -- `seller.py` and `market_buyer.py` are deterministic Python (pricing/negotiation rules, JSON-persisted memory), no network calls to any model provider. `agent.json`'s `harness` field (`claude-sonnet-5` macro / `claude-haiku-4-5` micro) documents which model did the macro-strategy vs micro-execution *design* work in the coding harness during development, not a runtime dependency.

```
python scripts/market_kit.py join --dir submissions/25512090/week-08 \
  --market https://market.ourolabs.ai --name Jaehoonang --tunnel
```

Requires `cloudflared` on PATH for the shop tunnel (omit `--tunnel`, add `--buyer-only` to run buyer-only without it). No environment variables needed.

## Checklist

- [x] No `scripts/check_week08.py` exists in this repo (Round 3 has no local content checker); verification is the live board (`https://market.ourolabs.ai/board`, Players table) -- final numbers and methodology are in `REPORT.md` S0 and `ORGANIZATION.md` S0.1.
- [x] Run logs are committed under `logs/` (`logs/market-kit.log`, force-added past the directory's own `.gitignore` since the assignment requires it)
- [x] No API keys anywhere in the diff (`*.pem` and `.market/` stay gitignored; no model API key is used by this agent at all, see "How to run")
- [x] History is not squashed
