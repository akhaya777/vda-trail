# VDA Trail

Track one batch of crypto from a private wallet to rupees in the bank, and get the exact figures that go into **Schedule VDA** of an Indian ITR.

India-focused · FY 2025–26 rules · runs entirely in the browser

---

## Run it

Double-click `index.html`. That's the whole install.

No Node, no npm, no build step, no server, no account. Everything — the trails, the tax engine, the exports — runs client-side. Data is saved in this browser's `localStorage` on this machine and is never uploaded.

---

## What it does

You paste the transaction IDs for each hop of a journey. The app records the timestamp and the INR value at each hop, works out the gain on each taxable leg, and returns a CA-ready statement.

| # | Hop | You paste | Taxable? |
|---|-----|-----------|----------|
| 1 | Crypto arrives in your private wallet | On-chain tx hash | No — this sets your **cost of acquisition** |
| 2 | You move it from your wallet to Binance | On-chain tx hash | No — moving your own coins is not a sale |
| 3 | On Binance you convert the crypto to USDT | Binance trade / convert ID | **Yes — taxable event #1** |
| 4 | You sell the USDT for INR over P2P | Binance P2P order ID + INR received | **Yes — taxable event #2** |

Colour in the interface encodes **which asset you are holding** — indigo for your crypto, green for USDT, brass for rupees. Hops 1 and 2 share a colour because it is the same batch simply moving. Which means a taxable transfer is exactly a point where the trail changes colour, and there are precisely two.

---

## The tax rules it encodes

The whole tool exists to get this right.

1. **Two taxable events, not one.** Crypto-to-crypto is itself a transfer under Indian law. The convert to USDT is taxed, and the P2P sale is taxed again — separately.
2. **Flat 30% + 4% cess = 31.2%.** No short-term/long-term split, no holding-period benefit.
3. **No loss set-off, no carry-forward.** A loss on one leg can never reduce a gain on another, and cannot be carried into next year. Totals only ever *add*; there is deliberately no code path that nets one leg against another.
4. **Only cost of acquisition is deductible.** Gas, trading fees and P2P spread are recorded so you can see your true economics, but never enter the tax maths.
5. **1% TDS under s.194S on the P2P leg**, flagged as *payable by you* — Binance P2P does not deduct it. It is an advance tax credited against the 30% liability, not an extra charge on top.

### The trap this avoids

The worked example from the brief: a ₹20,000 gain on event #1 and a ₹1,000 loss on event #2.

```
Event #1  ETH → USDT   cost ₹1,00,000  sale ₹1,20,000  gain  ₹20,000   tax ₹6,240   TDS —
Event #2  USDT → INR   cost ₹1,20,000  sale ₹1,19,000  loss  (₹1,000)  tax ₹0       TDS ₹1,190
                                                                        ────────
                                                          total tax     ₹6,240
```

A calculator that nets the two legs into a ₹19,000 gain reports **₹5,928** and under-states the tax by ₹312. VDA Trail reports ₹6,240 and says so explicitly on screen, naming the amount of loss it discarded and why.

---

## Screens

- **Overview** — the landing page: the journey, the rules, the worked example, where each number comes from, and the design note on why trails are not stored on-chain.
- **New Trail** — the four-hop form, with on-chain lookup, historical price lookup, and Binance CSV import.
- **Trails** — every saved journey with its headline gain, tax and TDS.
- **Trail view** — the journey as a vertical rail, with the two taxable legs flagged and their arithmetic shown inline.
- **Tax Summary** — the Schedule VDA table for a chosen financial year, with totals, CSV export, print/PDF, and an automatic Schedule FA flag past ₹20 lakh.
- **Ask** — plain-English questions over your own saved data, answered offline.
- **Settings** — API keys, backup, restore, erase.

---

## Automatic lookup — no API key, no signup

Paste the tx hash into hop 1 and stop typing. The app works out the rest on its own:

- **which token it was** — USDT, USDC, ETH or anything else, read from the transfer log
- **how much arrived** — at the token's real decimal precision
- **the exact minute it landed** — from the block timestamp
- **what it was worth in rupees at that minute** — and fills in your cost of acquisition

| Step | Source | Key |
|---|---|---|
| Token, amount, timestamp | Public RPC nodes — Ethereum, Polygon, BNB Chain, Arbitrum, Base, Optimism | none |
| Crypto price at the hop's minute | Binance public candle feed | none |
| USD → INR for that day | ECB reference rate via Frankfurter | none |
| Fallback for assets Binance doesn't list | CoinGecko daily average in INR | optional |

Several nodes are tried per chain, so one being down or rate-limiting doesn't stop the lookup. Settings lets you point at your own node instead, but nothing there is required.

**Why the minute and not the day.** The brief is emphatic that a few hours can change the price and therefore the tax, so the app prices the exact minute of your hop. That isn't theoretical — on 10 May 2025, ETH was ₹2,00,107 at 04:00 and ₹2,08,858 at 22:00. Nearly ₹8,750 apart on the same date.

Every price shown names its own source on screen, e.g. *"Binance ETH/USDT at that exact minute, converted at the USD/INR rate for 2025-05-09."* Where a weekend or holiday means no FX rate was published, it says which day it fell back to rather than hiding it.

Solana and Bitcoin aren't covered by automatic lookup yet — enter those hops by hand.

> **Nothing here needs a key that can move money.** A Binance API secret must never go into this app. Public nodes and public price feeds are enough to do the whole job.

> **Never paste a Binance API secret into this app, or any app that computes your taxes.** A tax tool needs public addresses, public tx hashes and read-only exported history. Nothing that can move money. If a field would hold a secret that moves funds, it does not belong in this tool.

Hops 3 and 4 are internal to Binance and invisible on the public chain, so they come from your account CSV export or manual entry. The importer matches column names loosely — Binance's export headers drift between versions — and shows you every parsed row so you choose which one is which hop before anything is filled in.

---

## Verification

The tax engine is tested, not asserted. On every page load the app re-runs the brief's worked example and shows a **"Engine verified · 7 checks pass"** chip in the hero; if the maths ever broke, that chip turns red.

A fuller suite runs under Node against the same code, extracted live from `index.html` — so the tests can never drift from what ships:

```
cd tests
node engine-test.js    # 24 checks — the worked example, the no-set-off rule,
                       # both-legs-lose, both-legs-gain, paise precision, fee inertness
node decode-test.js    # 42 checks — ERC-20 log decoding, symbol()/decimals() returns,
                       # CSV parsing, Binance date formats, price-lookup date handling
node ask-routing.js    # every suggested question reaches a specific answer
node html-check.js     # tag balance, and that no colour is defined only in dark mode
node live-test.js      # LIVE: finds a real USDT and USDC transfer on Ethereum and
                       # decodes token, amount and timestamp with no API key
node price-test.js     # LIVE: minute-accurate pricing, weekend FX fallback,
                       # stablecoin shortcut, and readable failure messages
```

Each exits non-zero on failure, so they drop straight into a pre-commit hook or CI step.

Money is handled as **integer paise** throughout. No floats touch a currency value at any point.

---

## Data model

```
trails          id · label · financial_year · asset · quantity · created_at
hops            id · trail_id · hop_no(1-4) · kind(arrival|transfer|convert|p2p_sale)
                source(onchain|binance) · reference_id · chain · timestamp
                asset_in · asset_out · inr_price · inr_value · fee_inr
taxable_events  derived, two per trail — event_type · date_acquisition · date_transfer
                cost_inr · sale_inr · gain_inr · tax_inr · tds_inr
```

`taxable_events` is derived on read rather than stored, so editing a hop can never leave a stale tax figure behind.

Take a backup before filing season. Storing everything locally is good for privacy and bad for accident recovery — clearing site data wipes your trails. **Settings → Download backup** writes a JSON file; restore merges by trail id and never silently overwrites what is already there.

---

## Why the trails are not stored on-chain

The reflection question from the brief, answered in full on the Overview screen. In short:

- **Privacy** — a public chain publishes your finances permanently and to everyone. Writing your income and your bank-facing P2P sales on-chain would tie your identity to every rupee you have ever moved, irreversibly.
- **Immutability cuts the wrong way** — tax records get corrected. A local record is edited; an on-chain record can only be appended to, leaving the mistake permanently visible beside the fix.
- **Cost** — gas per write, per correction, to store data only you will ever read.
- **Wrong tool** — a blockchain's value is trustless shared state between strangers who cannot agree on a referee. A personal ledger has one user, who already trusts themselves. There is no disagreement for consensus to resolve, so all that is left is the overhead.

---

## Limits

This produces an **estimate** to help you and your CA. It is not tax advice and it is not a filing service.

- Tax law changes.
- Gifts, airdrops landing in the wallet, staking rewards and mining income are outside the core flow.
- Schedule FA (foreign assets past ₹20 lakh) is flagged but not computed.
- The importer covers the trade and P2P history exports; other Binance report formats may need manual entry.
- The Ask screen answers from a local rule-based engine, so it handles the common questions rather than arbitrary ones.

Have a CA review the numbers before you file. The tool exists to make you well-informed, not to replace expert judgement.
