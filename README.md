# Reply Radar

Signal-based LinkedIn outbound scoring with HeyReach + Jev (TypeSafe System One).
Scores every lead, every message variant, and the probability of a reply, then
sorts the result back into HeyReach as lists and draft campaigns.

Scoring reads HeyReach through a client that refuses any endpoint outside a
read-only allowlist (`heyreach.READ_ONLY`). Only `hr setup` and the giveaway
write, through their own allowlist (create a list, add leads to a list, create a
DRAFT campaign). Nothing here starts a campaign or sends a message.

## Setup

```bash
git clone https://github.com/thoughtpilot99/reply-radar.git && cd reply-radar
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env    # TYPESAFE_API_KEY, plus HEYREACH_API_KEY for the hr commands
python -m reply_radar demo
```

Python 3.9+. Outputs go to `data/` (git-ignored). The demo scores its example signals as of 2026-09-22 (`as_of` in `reply_radar/demo/config.json`), so its numbers stay the same whenever you run it.

## What Jev judges

Each lead gets one request and each message variant gets another. The questions
in each request run in parallel. Everything numeric (signal age, word counts,
weights, thresholds) stays in code.

| Lead (state: offer, ICP, profile, signal) | Message (state: offer, recipient, signal, text) |
| --- | --- |
| function fit vs target roles (score) | personalization depth (score) |
| seniority (score) | about them vs about us (score) |
| company fit vs ICP (score) | pitches the product (noul) |
| signal relevance to the problem (score) | what it asks for: question / resource / chat / meeting (choice) |
| signal type: wrote about problem, competitor engagement, hiring... (choice) | stock outreach phrasing (noul) |
| sells something similar / open to work / matches exclusion (nouls) | reveals private tracking (noul) |
| | answerable in one line / clear why-you / mentions the signal (nouls) |

`scoring.py` turns those into:

- **Lead score (0-100)** = fit (function, seniority, company) + intent (signal relevance × signal-type weight × freshness decay). Disqualifiers force tier `Skip`.
- **Message score (0-100)** = weighted message judgments, with penalties for stock phrasing, tracking reveals, unfilled `{MERGE_TAGS}` and length.
- **Reply probability**: a logistic prior around your base reply rate until you calibrate, then a model learned from your own HeyReach inbox.
- **Best variant per lead**: with A/B variants in a campaign step, each lead gets the variant with the highest predicted reply.

## Run

```bash
python -m reply_radar demo
python -m reply_radar score leads.csv --config my.json --messages variants.txt
```

With HeyReach (`HEYREACH_API_KEY` in `.env`), `hr start` looks at the workspace and picks a mode:

```bash
python -m reply_radar hr start --config my.json [--messages variants.txt] [--enrich]
```

- **Campaign mode** (30+ past conversations): scores every past first touch, learns what earned replies in this account (`model.json`, `history.json`), then ranks the leads waiting in live and draft campaigns and picks a variant per lead.
- **Cold start** (no history, a LinkedIn sender and/or lead lists): pulls the sender's 1st-degree network (`--enrich` fills thin profiles via `GetLead`) and every lead list, ranks them against the ICP, and checks draft messages.

Single steps: `hr check | campaigns | lists | accounts`, `hr calibrate`, `hr score --campaign-id N | --list-id N`, `hr network`. `hr calibrate` needs at least 5 past conversations that got a reply and 5 that didn't; with fewer it says so and exits.

`--messages` takes variants separated by a line containing `---`; HeyReach merge tags (`{FIRST_NAME}`, `{COMPANY}`, custom fields like `{SIGNAL_HOOK}`) are filled per lead, and a variant with an unfilled tag is never picked. Signals come from a lead custom field named `signal` (plus `signal_date`); company size from `employee_count` is checked in code against `icp.employee_range`.

### Writing back to HeyReach (drafts only)

```bash
python -m reply_radar hr setup --from RUN/raw.json --messages variants.txt --dry-run
python -m reply_radar hr setup --from RUN/raw.json --messages variants.txt
python -m reply_radar hr setup --campaigns-from RUN/setup.json   # once a sender is connected
```

`hr_setup.py` is the only code that writes, through its own allowlist: create a lead list, add leads to a list, create a campaign. For every (tier, chosen variant) pair it makes a list, with the verdict as custom fields (`rr_tier`, `rr_lead_score`, `rr_reply_p`, `rr_best_variant`, `rr_why`), and a single-message campaign, so each lead gets the variant Jev chose. The campaign is a connection request with no note, then that message 3 hours after the accept. Leads from a sender's 1st-degree network (`hr network`, cold start) get their own list and a campaign that starts with the message, with no connection request. HeyReach creates campaigns in DRAFT and requires a connected LinkedIn sender; nothing here can start a campaign or send a message.

## The giveaway: run it on someone else's HeyReach

```bash
python -m reply_radar.giveaway new HANDLE             # intake.json, filled from their DM
python -m reply_radar.giveaway run HANDLE --dry-run   # score only, nothing written
python -m reply_radar.giveaway run HANDLE             # score + push [RR] lists and drafts
python -m reply_radar.giveaway status
```

Ask them for: what they sell and the problem it solves, target roles, company type and size, who to exclude, optionally their draft messages, and a HeyReach API key made for this (they delete it after). `run` asks for the key at a hidden prompt and keeps it in memory for that run only; it is never written to disk.

`run` learns from their inbox when they have 30+ past conversations, scores the leads not yet being contacted (draft campaigns' lists, unused lists, and the network when asked or when there is nothing else), and pushes one `[RR]` list and one single-message DRAFT campaign per (tier, variant Jev picked) back into their HeyReach. Nothing is started. It writes `report.html` and `deliver.md` (the DM to send) to `data/reply_radar/giveaway/HANDLE/`.

## Test without a real account

```bash
python -m reply_radar.demo.fake_eval
```

Builds two synthetic HeyReach workspaces (a team with 400 past conversations and a 60-lead draft campaign with 3 variants; a cold start with a 150-person network and an imported list), serves them from a local fake HeyReach, runs the real CLI against both, and grades every judgment and prediction against the hidden truth the fake world was built from. Writes `data/reply_radar/fake/test_report.html`.

Outputs land in `data/reply_radar/<run>/` as `report.html`, `scores.json` and `raw.json`.
