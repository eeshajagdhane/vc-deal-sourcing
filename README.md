# VC Project — Free-Data Deal Sourcing Pipeline

> **Original repository:** [eeshajagdhane/vc-deal-sourcing](https://github.com/eeshajagdhane/vc-deal-sourcing)  
> **Author:** Eesha Jagdhane · © 2026 · All rights reserved.  
> If you found this as a copy or fork, that GitHub URL is the original.

## What is this?

This is a tool that helps a VC (venture capital) investor find promising startups to look
at, using only **free, publicly available data** — no paid subscriptions (no PitchBook, no
Harmonic, no Crunchbase Pro).

It's a rebuild of an earlier project that did the same job using paid data. This version
proves the same idea works using only free sources, and documents exactly where every
piece of data comes from so it's always clear what you're looking at.

Everything lives in **one Jupyter notebook**, `vc_deal_sourcing.ipynb`, written to be
readable by a non-technical person: every step has a plain-English explanation before the
code that does it.

This README is a living document — it's updated after every milestone (build step) so it
always reflects what has actually been built so far.

---

## How to run it

1. Open the notebook:
   ```bash
   cd "/Users/eeshajagdhane/Desktop/VC Project"
   jupyter notebook vc_deal_sourcing.ipynb
   ```
   (or open the folder in VS Code / JupyterLab and open the `.ipynb` file).
2. In the menu, click **Kernel → Restart & Run All**, or run each cell top-to-bottom with
   **Shift + Enter**.
3. **First run takes a few minutes** (it downloads a small AI model once, and does
   rate-limited GitHub lookups). Everything is cached, so **later runs are fast**.
   - *Optional speed-up:* set a free `GITHUB_TOKEN` environment variable (from
     github.com/settings/tokens) before launching, and the GitHub lookups go much faster.
4. The last section starts a small **API** in the background and calls it — you'll see live
   JSON responses, and you can also `curl http://127.0.0.1:8000/top?k=10` from a terminal
   while the notebook is running.

**Python note:** use a Python that has the data packages installed. On this machine that's
the framework Python 3.9 (`/Library/Frameworks/Python.framework/Versions/3.9/bin/python3.9`),
which already has `pandas`, `requests`, `sentence-transformers`, `flask`, and the rest. If you
set up a fresh environment, `pip install -r requirements.txt` covers it.

---

## Where does the data come from?

| Source | What it is, in plain terms | Official API? |
|---|---|---|
| **SEC EDGAR (Form D filings)** | Every US company raising private investment above a certain size is legally required to file a "Form D" with the SEC (the US financial regulator). This is public record. It tells us the company's name, location, rough founding year, industry, and **how much money it's raising right now**. | Yes — official US government API. |
| **Y Combinator directory** | Y Combinator is a famous startup accelerator. It publishes a public list of every company it has funded — with each one's website, description, and sector. We read a free, public copy of that list hosted as a plain JSON file on GitHub (`yc-oss.github.io`). | Community-maintained public mirror (no login/key). Stable and low-risk. |
| **GitHub** *(Milestone 3, not built yet)* | For software startups, shows how active their public engineering work is (a "traction" signal). | Yes — official public API. |
| **Hacker News** *(Milestone 3, not built yet)* | A popular tech discussion site; being talked about there is a small buzz signal. | Yes — official public search API. |
| **Job boards (Greenhouse/Lever)** *(Milestone 3, not built yet)* | Counting a startup's open jobs (especially engineering) is a rough growth signal. | Public pages, used respectfully with caching. |

**On data-source safety (a question that came up):** SEC EDGAR, GitHub, and Hacker News are
official public APIs built for exactly this kind of use — no risk. The YC list is public
data too; we read a static community-maintained copy and cache it locally rather than
hammering any website. If it ever became unavailable, SEC EDGAR alone can still drive the
pipeline.

---

## What the pipeline does (the big picture)

1. **Find companies** *(built — Milestone 1)* — pull real startups from SEC EDGAR (companies
   actively raising money) and the YC directory.
2. **Clean & de-duplicate** *(built — Milestone 2)* — merge duplicate records into one clean
   record per company, keeping the best fields from each source.
3. **Enrich** *(built — Milestone 3)* — add GitHub activity, Hacker News buzz, and open-job
   signals to each company.
4. **Score & rank** *(built — Milestone 4)* — give each company a clear 0–100 "fit score"
   against your thesis, with a transparent, adjustable breakdown of *why*.
5. **Find similar companies** *(built — Milestone 5)* — for any company, surface its closest
   comparables ("comps").
6. **Summarize** *(built — Milestone 6)* — write a short plain-English blurb for each top
   company.
7. **Evaluate** *(built — Milestone 7)* — check that the score behaves correctly.
8. **Serve via an API** *(built — Milestone 8)* — expose the results so other tools can use them.

**The whole pipeline is now complete, end-to-end, on 100% free data.**

---

## Progress so far

### ✅ Milestone 1 — Discovery (built)

**What it does:** builds a first real list of companies to consider and saves it to
`output/companies_raw.csv`.

**What's in that file:** one row per company, with a standard set of fields (name, website,
description, industry, location, funding stage & amount, and which source it came from). On
the most recent run it contained **438 companies** — 38 pulled from SEC EDGAR (companies
that just filed to raise money) and 400 from the YC directory.

**How the two sources complement each other:**
- **SEC EDGAR** gives us *funding reality* — real dollar amounts and dates, which we use to
  guess a funding stage (Pre-Seed / Seed / Early / Later). It does **not** give websites or
  descriptions.
- **YC** gives us *websites and descriptions and sector tags*, which EDGAR lacks, but no
  funding amounts.

**A note on funding stages:** SEC filings don't say "Seed" or "Series A" — they just give a
dollar amount. So we *infer* a stage from the amount using rough US venture ranges (see the
"stage buckets" in the notebook). These are educated guesses, not official labels.

**Known rough edges (to be cleaned up in later milestones):**
- The same company can appear twice (e.g. in both sources, or under a slightly different
  legal name). → Fixed in Milestone 2.
- A few EDGAR companies have foreign addresses but are currently all labeled US. → Handled in
  Milestone 2's normalization step.

**What was actually built in Milestone 0 + 1:**
- `vc_deal_sourcing.ipynb` — the notebook, with a Setup section (the editable `CONFIG`
  "control panel"), a standard-fields section, and the full Milestone 1 discovery code.
- `requirements.txt` — the Python packages needed.
- Cached raw data lands in `data/raw/`; the final list lands in `output/`.

### ✅ Milestone 2 — Clean & de-duplicate (built)

**What it does:** turns the messy raw list into a clean list with **one row per real
company** (`output/companies_clean.csv`). It does two jobs:
- **Normalize** — tidies values so they're comparable: strips legal endings from names
  ("Wayy Inc." → the matching key `wayy`, while keeping the pretty name for display), and
  fixes the country label using the SEC's state code (so a Canadian "A2" filer isn't
  mislabeled US).
- **Match & merge** — treats two rows as the same company if they share a website, share a
  cleaned-up name, or have near-identical names in the same US state; then merges them,
  keeping funding info from the SEC filing and website/description from YC.

**An honest finding from the real data:** in a given 90-day window, the companies filing
with the SEC and the YC startups barely overlap (on our runs, **zero** cross-source
matches). This is expected, not a bug — YC startups often raise via SAFEs that don't trigger
a public Form D, and YC is a tiny slice of all fundraising startups. The merging logic is
correct and still runs; it simply has little to merge in a short snapshot, and would merge
more as you widen the SEC date window or run it over time. The lasting value of this step is
the cleaning/normalizing (which every later step depends on) and collapsing the SEC's own
repeat filings.

### ✅ Milestone 3 — Enrichment (built)

**What it does:** adds three free "momentum" signals to each company and saves
`output/companies_enriched.csv`:
- **GitHub activity** — for software companies, how many stars their code has and how
  recently they've shipped. *(A company that's actively building in public.)*
- **Hacker News buzz** — how often the company is mentioned on Hacker News, and the score of
  its top mention. *(A proxy for attention in the tech community.)*
- **Open job postings** — how many jobs they're advertising on public boards
  (Greenhouse/Lever). *(A proxy for how fast they're growing.)*

**How we keep these accurate:** matching by company *name* alone is unreliable (searching
"Mount" would match thousands of unrelated things). So we match on the company's **website
domain** wherever possible — e.g. we search Hacker News for `mount.insure`, not "Mount", and
we only accept a GitHub repo if it's clearly the company's own (its linked homepage is the
company website, or the GitHub account matches the company's brand).

**Honest limitations (by design):**
- Free APIs are **rate-limited** (GitHub allows ~10 lookups/minute without a token), so we
  enrich a **capped number** of companies per run (default 40, website-having companies
  first). Set a free `GITHUB_TOKEN` environment variable to go faster.
- Not every startup uses GitHub or a public job board, so **many companies legitimately come
  back with empty signals** — especially very early or non-technical ones. The notebook
  reports exactly how many it found for each signal, so coverage is never overstated.

### ✅ Milestone 4 — Score & rank (built)

**What it does:** gives every company a single **0–100 fit score** against your thesis and
saves the ranked list to `output/companies_ranked.csv`.

**How the score is built** — six ingredients, each 0-to-1, combined with editable weights:
- **semantic_sim** (weight 0.40) — how closely the company's description matches your thesis
  *in meaning*, measured with a small local AI model (free, no API). This is the biggest driver.
- **stage_fit** (0.20) — is it at a funding stage you target?
- **geo_fit** (0.10) — is it in a region you target?
- **traction** (0.15) — momentum from the GitHub/HN/jobs signals added in Milestone 3.
- **investor_signal** (0.10) — repeat SEC filings + a real offering amount.
- **hygiene** (0.05) — is the record complete (website, description, year, industry)?

**Why it's "explainable":** because the score is just a weighted sum, we store **each
feature's points**, and they add up *exactly* to the score (the notebook checks this). So for
any company you can see a bar-chart-style breakdown of what earned its score — and you can
change the weights and instantly see the ranking shift (the notebook demonstrates this).

**Honest note on the two sources:** because YC and SEC companies carry different fields, the
breakdown will show YC companies scoring mostly on *semantic match* and SEC companies on
*stage/geo*. Unknown fields are treated as **neutral (0.5)**, not zero, so a company isn't
penalized for a source not providing a field. The transparency is the point — you can see
what's driving each score rather than trusting a black box.

### ✅ Milestone 5 — Find similar companies / comps (built)

**What it does:** for any company, finds the others most similar *in meaning* — useful for
spotting competitors or comparable companies ("comps"). It reuses the AI fingerprints
("embeddings") already computed in Milestone 4, so it's just a fast math comparison — no
extra downloads, no paid vector database (a full one isn't needed at this scale). A worked
example is saved to `output/similar_example.json`.

### ✅ Milestone 6 — Summaries (built)

**What it does:** writes a short, readable paragraph for each of the top-ranked companies
(`output/companies_summaries.csv`) — what it does, how it's funded, its momentum, and why it
scored the way it did. Example: *"InsForge (B2B) — the agent-native AWS: cloud infrastructure
that AI coding agents can understand… Momentum signals: 12,521 GitHub stars, a top Hacker
News post at 62 points. It scores 59/100, driven mainly by thesis match and traction."*

By default the summaries are built from a **template** (free, instant, no AI service). If you
install **Ollama** (a free tool that runs an AI model locally), the notebook will
automatically use it for more natural wording — but nothing is required, and no paid AI
service is ever used.

### ✅ Milestone 7 — Evaluation (built)

**What it does:** checks whether the score actually behaves correctly, and saves the results
to `output/eval_report.json`.

**The honest problem:** the ideal way to grade a deal-sourcing tool is against **real investor
decisions** (which companies a partner chased or funded). We don't have those — free public
data has no such answer key, and brand-new startups have no known outcome yet. So instead of
faking it, we validate two things we *can* verify:
1. **Does the score respond to the thesis?** When we swap the thesis from "AI dev tools" to
   "healthcare", healthcare companies jump from **0% to 43%** of the top 30 — proof the score
   genuinely follows the thesis rather than returning a fixed list.
2. **Are the "similar companies" coherent?** A company's top-5 comps share its industry
   **77%** of the time, vs **39%** by random chance — about **2× better than random**.

*(We're explicit in the notebook that this validates the system's behavior, not real
investment returns — that would require real investor decisions to measure.)*

### ✅ Milestone 8 — Serving API (built)

**What it does:** starts a small web service (API) **inside the notebook** so other tools — a
dashboard, a Slack bot, an n8n workflow — can request results on demand. Three endpoints:
- `GET /top?k=20` — the top companies by fit score
- `GET /memo/<company_id>` — a full "deal memo": facts, summary, score breakdown, and comps
- `GET /similar/<company_id>?k=5` — a company's closest comparables

Because it runs in the background, the notebook calls its own API in the next cell to prove
it works. (You can also copy that cell's code into an `app.py` and run it as a standalone
always-on service if you prefer.)

### 🎉 The pipeline is complete
Discover (SEC + YC) → clean & de-duplicate → enrich (GitHub / HN / jobs) → score & explain →
find comps → summarize → evaluate → serve — all on free data, no paid APIs.

**Ideas for later:** widen the SEC window and enrich more companies for richer traction; look
up websites for SEC companies so the two sources unify; wire the API into Slack/Notion or an
n8n workflow (like the original project); and, when real investor decisions become available,
swap the behavior-based evaluation for real precision/recall.

---

## Project layout
```
VC Project/
├── README.md                 <- you are here (updated every milestone)
├── requirements.txt            <- Python packages needed
├── vc_deal_sourcing.ipynb       <- THE notebook — all the code + explanations
├── data/
│   └── raw/                        <- cached downloads from the free APIs
│       ├── edgar_formd.json           (cached SEC filings)
│       ├── yc_companies.json           (cached YC directory)
│       ├── enrichment_cache.json        (cached GitHub/HN/jobs lookups)
│       └── embeddings.npz                (cached AI fingerprints for scoring/comps)
└── output/
    ├── companies_raw.csv            <- Milestone 1: the raw combined list
    ├── companies_clean.csv           <- Milestone 2: de-duplicated, one row per company
    ├── companies_enriched.csv         <- Milestone 3: clean list + traction signals
    ├── companies_ranked.csv            <- Milestone 4: + fit score & per-feature breakdown
    ├── similar_example.json             <- Milestone 5: a worked "comps" example
    ├── companies_summaries.csv           <- Milestone 6: short summaries of the top companies
    └── eval_report.json                   <- Milestone 7: evaluation results
```
