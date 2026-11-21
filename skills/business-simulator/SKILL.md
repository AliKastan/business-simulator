---
name: "business-simulator"
description: "Realistically simulate a business idea's money: what it costs, monthly and yearly earnings, break-even, chance of losing money, with a critic board, a 100-buyer panel and a computed go / test first / no-go. Use for 'bu işi yaparsam ne kazanırım', 'simulate this business', 'is this profitable'."
---

# Business Simulator (realistic Monte Carlo)

Goal: tell the user what would REALLY happen to their money if they start this business: how much it costs, what it earns month by month and year by year, how likely it is to fail, how much money they need, and what they personally take home per hour compared with a normal job. The result must be close enough to reality that a banker or an experienced owner in that industry would nod at it.

Founders are systematically too optimistic (Cassar 2010, Strategic Management Journal: nascent entrepreneurs overestimate sales, and writing financial projections makes it worse). This skill exists to correct that with research, base rates, a tested simulation engine and calibration. Taking a long time is fine. Skipping steps is not.

Version 3 works with two views. The inside view is the simulation built from researched inputs. The outside view is what really happens to businesses of this kind: how many survive, what revenue they reach, how many make a profit. When the two disagree and the evidence is weak, the outside view wins. Around the simulation sit a board of four critics (Step 3B), a panel of 100 simulated buyers read against a real competitor (Step 3C, the `customer-panel` skill), a computed call (GO / TEST FIRST / NO-GO) and a real-world test whose go and stop lines are written before the test.

## Non-negotiable rules

1. Never do business math in your head or in ad hoc code. Every number in the answer comes from the engine `bizsim.py` (appendix at the end of this file), which is tested (cash and P&L identities, tax hand checks, loan amortization).
2. Every uncertain input is a range (low / most likely / high), never a single guess. Single numbers only for hard facts (a published fee, a legal tax rate, a signed quote).
3. Every input carries a `source` in the config: report or URL plus date, or "user's own number". The engine flags anything unsourced.
4. Research before modeling. Use the research protocol below. Never invent a benchmark.
5. Always apply the optimism correction (`demand.reality_factor`) unless demand comes from hard data (existing paying customers, a paid pilot, signed letters of intent).
6. Calibrate against reality (base rates, real comparable businesses, industry ratios). Fix or explicitly justify every warning the engine prints.
7. Report the full distribution honestly: bad (P10) / typical (P50) / good (P90), odds of losing money, money needed, real hourly pay. Never present the good case as the plan. Never let a business sale value carry the verdict.
8. Write the final answer in the user's language, plain words, short sentences, no em dashes or en dashes, no hedging. Arithmetic is written in words, one operation per line (see Step 8).
9. Simulated people are not evidence of demand. Board memos and panel answers shape ranges, risks and the offer. They never lift `meta.evidence_level` above 1. Only real behaviour does: conversations and waitlists 2, costly signals 3, money 4, operating history 5.
10. Hold the model against the outside view. Every run carries `base_rates` for its kind of business. Where the engine says "too rosy", the run with `demand.outside_view_factor` applied is the main result and the unadjusted one is "if the research holds fully".
11. The call (GO / TEST FIRST / NO-GO) is computed by the engine from fixed rules. Report it as computed. If you disagree, say so separately and say why. Never quietly upgrade it.
12. Never write a board memo or a buyer's answer yourself and never edit one. Each is its own sub-agent. Running this skill is the user's request to use sub-agents for the board, the panel and the audit.

## Workflow (make a task list with these steps)

1. Intake and idea file
2. Pick the model shape
3. Research (market, costs, competitors, real-world rates)
3B. Board: four critics
3C. Customer panel: 100 simulated buyers
4. Build the config
5. Run the engine
6. Calibrate: warnings, outside view, independent audit
7. Stress tests, price ladder, what must be true, real-world test plan
8. Report to the user

Quick mode (the user says quick, hızlı, kabaca): skip 3B and the audit, run 3C with 20 buyers or skip it, use `--sims 3000`. Say in the report what was skipped.

---

## Step 1: Intake (ask at most once)

Needed:
- What exactly is sold, to whom, at what price, through which channel (walk-in, online ads, outbound sales, marketplace, app store).
- Where (city and country). Physical location or online.
- Money: cash they will put in now, extra savings they could add later, whether they would borrow (SBA loan, credit cards), any co-founder money.
- Time: hours per week, and what they would otherwise earn per hour (their realistic job alternative).
- Other income this year (a W-2 job changes taxes: business losses offset it, profits stack on top of it).
- Legal form (LLC, sole prop, corp) and state.
- Any hard data: existing customers, pilots, waitlist, letters of intent, quotes from suppliers or landlords.
- Evidence so far, as `meta.evidence_level`: 0 idea and desk research only; 1 a synthetic panel, or proof that people pay competitors for this; 2 real conversations with target buyers, search demand or a waitlist (words and clicks); 3 costly signals short of money (letters of intent, refundable deposits, agreed pilots); 4 money changed hands (pre-sales or paid pilots from strangers); 5 operating history (3 or more months of real revenue).

Write `idea.md` (five lines: what is sold; to whom exactly; the price and what the buyer gets; where and how they meet it; money and hours going in). The board and the panel read this file.

If something is missing, ask for ALL missing items in ONE message. Anything the user cannot answer, research it and state the assumption. Never ask twice. If the user is not there to answer, take the most reasonable reading, say which one at the top, and continue.

## Step 2: Pick the model shape

| Business type | demand.model | Main inputs |
|---|---|---|
| Location or capacity driven: cafe, restaurant, food truck, retail shop, gym, salon chairs, car wash, laundromat, event venue | `traffic` | `mature_units_per_month` (visits or units at full ramp), `price_per_unit`, `capacity_units_per_month` |
| Acquired by marketing then retained or repeat buying: SaaS, subscriptions, ecommerce, apps, agencies, home services, B2B services, coaching | `funnel` | `channels` (budget plus CAC or CPC and conversion), `organic_new_per_month`, `referral_rate`, churn, `first_purchase_value`, repeat orders and value |

Special cases:
- Fixed monthly payment: `funnel` with `subscription: true`, first and repeat value = monthly price (use `{"ref": name}` so both use the same draw).
- One time purchases with little repeat (furniture, weddings): `funnel` with high `churn_monthly` (0.6 to 1.0) and low repeat orders.
- Marketplace or platform: model the paying side; `first_purchase_value` = GMV per order times take rate; supply side acquisition as fixed marketing cost.
- B2B with long sales cycles: put the cycle into `launch_month` and the ramp, put sales salaries or founder time into CAC, use higher `monthly_volatility` (0.25 to 0.4) because deals are lumpy.
- Physical products: `cogs_pct` or `cost_per_order`, `working_capital.inventory_days`, `min_inventory` (supplier MOQ), shipping in `cost_per_order`.
- Service businesses limited by people: `capacity_active_customers` or `capacity_units_per_month` as `{"from_staff": "role", "per_staff": X, "base": owner_capacity}` and a scaling staff role.

## Step 3: Research protocol

Run searches in parallel. Cover every item below. Write each finding into a research log (input, low, most likely, high, source, date). Prefer primary sources and the newest data.

1. Reference class and base rates
   - Survival by industry: BLS Business Employment Dynamics "establishment age and survival" (NAICS), SBA Office of Advocacy FAQ. National average: about 20% of new establishments close in year 1, about 49% survive 5 years.
   - What similar businesses really make: BizBuySell listings and Insight Report for that industry and city (revenue, cash flow, asking price), franchise FDD Item 19 financial performance representations for the same category (average unit volume, sometimes costs), public competitors, trade association reports.
2. Price: at least 5 real competitor prices in that market.
3. Demand: number of target customers in the area (Census County Business Patterns for B2B counts, population and income for B2C), foot traffic or search volume, comparable volumes (tickets per day, covers per seat, units per month).
4. Customer acquisition: CPC and conversion by industry (WordStream / LocaliQ latest benchmarks for Google and Meta), lead to paying customer rate, sales cycle length. CAC = cost to get a PAYING customer, not a lead.
5. Retention: churn or repeat purchase benchmarks for that exact segment and price point.
6. Variable costs: supplier and wholesale prices, food cost percent, 3PL and shipping quotes, payment fees, platform fees, API or AI usage cost per customer.
7. Fixed costs: rent per sq ft in that neighborhood (LoopNet, Crexi listings), utilities, insurance for that industry (Insureon medians), software, bookkeeping, licenses and permits (city website), local business taxes (city business license or gross receipts tax).
8. Labor: local minimum wage and market wage (BLS OEWS), employer burden, how many people must be on shift for the opening hours.
9. Startup costs: build-out, equipment, deposits, legal and formation, branding and website, initial inventory.
10. Financing: current SBA 7(a) rates, credit card APR.
11. Taxes: entity and state. The engine has US 2026 federal, self-employment tax, QBI and California built in. Other states or countries: research the combined rate and use `mode: flat` or `state: flat`.
12. Risks: the most common reasons this type of business fails, turned into `shocks`.
13. Competitor map: at least 3 direct competitors (name, price, what buyers praise and what they complain about in reviews, links), indirect ones, the status quo (what people do today instead; this is the real competitor), the graveyard (who tried this and closed, and why) and platform risk (who could add this as a feature). "No competitors" usually means no market.
14. Outside view: at least 3 real-world rates for this kind of business, written as `base_rates` (Step 4 rule 22): survival by sector, the share of similar new businesses that reach a revenue level, the share that make a profit in year 1, typical owner income. Start from the Outside view block in the benchmark library and look for rates closer to this exact business.
15. A control offer for the panel: one real competing offer the target buyers already know, its real price, and one real number about it (the share of the target population that buys it in a month, or its conversion rate).
16. Buyer raw material for the panel: segment shares, ages and incomes (census, industry reports), what people do today with shares, and at least 8 real objections and 6 habits taken from reviews and forums.

If the user's own numbers conflict with research, build the ranges from research, keep the user's number visible in the report and say how far off it is.

## Step 3B: Board (four critics, before any numbers)

Purpose: find what would kill the business and which inputs are too rosy while that is still cheap. Four sub-agents in parallel (one message, four Agent calls, `run_in_background: false`). Each gets the same brief and ONE lens.

Brief for every seat: the text of `idea.md`, the research log, the competitor map, the planned price and budget, then: "You are a board member reviewing this business before the founder spends money. Use only the lens below. Use only facts in this brief and mark anything else as a guess. Score the business, not the founder. Be specific; 'could be challenging' is not a risk."

Lenses (summaries of published frameworks; never speak as, quote or imitate a real person):
- Offer (value-equation thinking, as in "$100M Offers"): is this market in pain, able to pay, easy to reach, growing? How big is the promised result, how believable, how fast, how much effort for the buyer? Is price the only reason to buy? What would make the offer hard to refuse (a costed guarantee, a real limit)?
- Moat and distribution (as in "Zero to One"): is there a small market this can own? What is ten times better, not ten percent? Why now? Exactly how do the first 100 customers arrive and what does each one cost? What stops a copy? Who could add this as a feature?
- Product and operations (focus and simplicity): can it be said in one sentence? What is left out on purpose? What does the buyer live through from first contact to the repeat purchase? What must go right every single day (staff, suppliers, permits, quality), and what is the unglamorous work nobody wants?
- Skeptic (pre-mortem and base rates): "It is 18 months after opening and the business has closed. Write the three most likely stories, each with a rough chance per year and the first warning sign." Then: what usually kills businesses of this kind, why this one would be different, who already tried it and closed.

Memo format (each seat, under 350 words):

```
Score 1-10 | Vote FUND / FUND IF / PASS
Strong: ...
What would kill it (top 3): risk | chance per year | effect on sales or money
Numbers I do not believe: input | my range | why
Questions for the founder: ...
Conditions for a yes: ...
```

Chair summary (you): votes and average score; risks named by two or more seats first; one merged list of conditions. Then turn the memos into inputs:
- Every kill risk becomes a row in a risk register and then a `shocks` entry or a wider range (Step 4 rule 23).
- Every "number I do not believe": research that input again. If it stays unsure, widen its range toward the critic's number.
- The conditions go into "what must be true" in the report.

A PASS from three or four seats does not stop the simulation. It is reported next to the computed call. No Agent tool: write the four memos yourself, one at a time, each finished before the next, and label the board "one mind, weaker".

## Step 3C: Customer panel (100 simulated buyers against a real competitor)

Load the `customer-panel` skill with the Skill tool and follow it (100 buyers; 20 in quick mode). It returns `panel_report.md`: how the offer pulls relative to the real control offer, a ready config snippet (`panel_factor`, `price_elasticity`, `reference_price`), who buys by segment, the reasons for no, what would flip a maybe, and a bottom-up demand cross-check.

Use it like this:
- The snippet goes into `demand` (Step 4 rule 21). The benchmark sets the level of demand; the panel only moves it up or down relative to the control.
- The top reasons for no: one fix to the offer (one re-test at most) and one row each in the risk register.
- Segments that do not buy: shrink the reachable market.
- The bottom-up cross-check against your benchmark-based demand: more than 2 times apart means one of them is wrong. Find out which before running.
- Health flags (too agreeable, flat, price-insensitive): repeat them in the report and lean on the panel less.

The panel is evidence level 1. If the skill is not installed or there is no Agent tool, skip this step, say so in the report and do not set `panel_factor`.

## Step 4: Build the config

Write `config.json` (schema in the Config reference below). Rules that make it realistic. Each has a default; only deviate with evidence and write the reason into `why`.

1. Distributions
   - `[low, mode, high]` = PERT. Low and high are realistic extremes (roughly 5th and 95th percentile), not fantasies.
   - `{"dist": "lognormal", "p10": a, "p90": b}` for things with a long bad tail: CAC, build-out cost, delays.
   - `{"dist": "mixture", ...}` for "it works or it doesn't".
   - `{"dist": "bernoulli", "p": 0.3}` or `choice` for yes/no events (wins the big contract).
   - Give a `name` to inputs used twice and reuse them with `{"ref": "name"}`.
   - Add `correlations` (pairs with a rank correlation) when there is a real link, for example price vs churn (+0.3), reality_factor vs organic demand (+0.5).
2. Launch delay: `launch_month` covers legal, permits, build-out or MVP. Physical locations 4 to 12 months, software MVP 2 to 6.
3. Ramp: `ramp_start_level` 0.1 to 0.3 for online, 0.3 to 0.6 for a good physical location. `ramp_months` 6 to 24. Default shape exponential; use `s_curve` for word-of-mouth products.
4. Optimism correction, default: `"reality_factor": {"dist": "mixture", "weights": [0.3, 0.7], "components": [{"dist": "uniform", "low": 0.35, "high": 0.7}, {"dist": "pert", "low": 0.7, "mode": 0.9, "high": 1.1}]}`. The first weight is the chance of a bad draw: 0.30 at evidence level 0 or 1, 0.25 at level 2, 0.20 at level 3, 0.10 at level 4; at level 5 use the measured numbers instead. It multiplies traffic, organic and paid acquisition.
5. CAC from the funnel: `cpc` and `conversion` (click to PAYING customer), or `cac` directly. `early_cac_multiplier` 1.3 to 1.5 (new accounts and no reviews). Set `ref_budget` and `diminishing_exponent` 0.2 to 0.5 when spend is large relative to the market.
6. Churn: `early_churn_monthly` (first 2 to 3 months) is higher than `churn_monthly`.
7. Staffing by coverage: opening hours times people needed on shift, minus the owner's hours, gives paid hours. Part-time hourly staff: `scaling` with `fractional: true`. Salaried people: whole counts or `trigger`. `burden_pct` 0.20 to 0.30 for employees (FICA 7.65%, FUTA, state unemployment, workers comp, some benefits), 0 for contractors. Never below local minimum wage.
8. Owner: realistic `hours_per_week` (physical businesses 50 to 60). `opportunity_wage_hourly` = what they would really earn at a job now.
9. Costs: walk the checklist below. Always add a `contingency` line (2 to 5% of revenue or a fixed amount).
10. Inflation 3% on costs; price increases only as strong as the market allows.
11. Working capital: inventory days and MOQ for goods, `dso_days` 30 to 60 for B2B invoices.
12. Shocks: at least 2 or 3 typical for the industry (equipment failure, platform or ad account ban, losing a key customer, competitor price war, liability claim, outage).
13. Quit rule: owners differ and do not all quit in the same month. `max_months_without_profit` is the owner's patience as a range (default `[12, 24, 48]`); past it, a business that still loses money closes with `monthly_quit_prob` (0.2) each month. `min_owner_hourly` (about half of the opportunity wage, checked after 18 to 30 months): while pay stays below it the owner gives up with `low_pay_quit_prob` (0.03) each month, because people hang on to poorly paying businesses for years. `exogenous_exit_annual` is closing for outside reasons (life, health, a move, a better offer): 0.03 for a business with staff or a lease, 0.06 for a solo business with nothing tying the owner down. With these defaults a typical owner-operated cafe built from benchmark numbers comes out 81% open after 1 year and 57% after 5; BLS says 82.5% and 55.3% for accommodation and food services.
14. Taxes: LLC or sole prop = `us_passthrough`. Set `other_income` if they have a job. `ca_llc: true` for a California LLC ($800 plus gross receipts fee). Non-US: `mode: flat` with the researched combined rate.
15. Exit value: `sde_multiple` 1.5 to 3 for small local businesses (BizBuySell average about 2.5), `revenue` basis with 1 to 4 for small SaaS. The report always also shows the result WITHOUT selling.
16. Financing: `founder_cash` (put in now), `founder_reserve` (would add later), `credit_line` (cards), `loans`. The business closes when all of that runs out.
17. Capacity: seats times turns times open days, or staff times jobs per month. Demand above capacity is lost.
18. Benchmarks: put the researched ratio ranges and base rates into `benchmarks` so the engine checks them.
19. Evidence. Set `meta.evidence_level` (Step 1) and grade every uncertain input with `ev` (on the input, or on its item or section; it is inherited like `source`): A measured or quoted (own data, a signed quote), B published benchmark for this exact segment and region, C analog, derived, or synthetic panel, D guess. The engine ranks inputs by their effect on the result and marks the big ones with weak grades as "test first".
20. Does demand exist at all. For anything new to its buyers (a new product, app, SaaS, brand or category) add `fit_probability` and `no_fit_factor: [0.03, 0.12, 0.35]`: most new products find almost no demand and a few find the researched level. Chance by evidence level: 0.20 at level 0, 0.25 at 1, 0.35 at 2, 0.50 at 3, 0.70 at 4; at level 5 leave it out. (With 0.25 a typical solo SaaS built from benchmark numbers passes $1,000 a month within a year in 15% of runs; the real rate for subscription apps is 17%.) For proven formats where the question is location and execution (cafe, salon, cleaning, franchise) leave it out and rely on `reality_factor`.
21. Panel and price. Copy `panel_factor`, `price_elasticity` and `reference_price` from Step 3C into `demand`. Without a panel: no `panel_factor`, and `price_elasticity: [-3.0, -1.8, -0.8]` with `reference_price` = the price your demand research assumed (usually the competitor average). With elasticity set, a higher price lowers demand, and `--sweep` on the price input shows the price ladder.
22. Outside view. Add `base_rates` (at least 3) and set `simulation.months` to 84 so that 5 year checks fit. Each item: `name`, `metric`, `rate` (the real-world share), `source`, plus `years` (metric `survival`), `months_after_launch` and `threshold` (`monthly_revenue`), `year` and `threshold` (`annual_revenue`, `owner_income`), `year` (`profitable_year`); add `"good": "low"` when a lower share is the good direction. Compare like with like: BLS survival is about businesses with employees, so for solo software use revenue-level rates instead; hold a full-time founder against firms that actually had sales, not against every registration.
23. Risks from the board. Every kill risk from Step 3B becomes a `shocks` row (`monthly_prob` = chance per year divided by 12) or a wider range on the input it attacks.
24. Accepted warnings. A warning you keep on purpose goes into `meta.accepted_warnings` as `{"match": "<start of the warning text>", "why": "<evidence>"}`. Only then does it stop blocking a GO. "It looks fine to me" is not a reason; a source is.

Cost checklist (use these `category` names; set `required_categories` to the ones that must exist for this business):
- Recurring: rent, utilities, insurance, accounting, legal (licenses, permits, registered agent, compliance), software, phone_internet, marketing (fixed part), maintenance, vehicle, professional_services, bank_fees, contingency.
- One-time (`one_time_costs`): formation and legal, permits, design and branding, website, opening marketing, training, deposits (`expense: false`).
- Equipment and build-out: `capex` with life in months and salvage percent.
- Variable: COGS, packaging, shipping, payment fees, platform fees, sales commissions, refunds and returns, discounts, spoilage or shrinkage, bad debt.
- Taxes: handled by the engine; add local fixed taxes in `taxes.other_annual_fixed_taxes` and local gross receipts taxes in `taxes.gross_receipts_tax_pct`.

## Step 5: Run the engine

Get the engine. The code is in the appendix of this file between the lines `# === BIZSIM START ===` and `# === BIZSIM END ===`. If the skill file is on disk, extract it instead of retyping:

```bash
F=$(find / -path /proc -prune -o -name SKILL.md -path "*business-simulator*" -print 2>/dev/null | head -1)
python3 - "$F" <<'EOF'
import sys
t = open(sys.argv[1]).read()
code = t.rsplit("# === BIZSIM START ===", 1)[1].split("# === BIZSIM END ===", 1)[0]
open("bizsim.py", "w").write(code)
print("bizsim.py written", len(code), "chars")
EOF
```

If no file is found, write `bizsim.py` from the appendix exactly. Needs numpy (and matplotlib for charts): `pip install numpy matplotlib --break-system-packages` if missing. Then `python3 bizsim.py selftest` must end with `SELFTEST OK` (golden numbers worked out by hand). If it does not, the file was copied wrong: extract it again.

Then:

```bash
python3 bizsim.py config.json --validate-only          # typos and missing keys, lists uncertain input names
python3 bizsim.py config.json --out run1 --solve <main demand input name> --solve <price input name> --sweep <price input name>=<5 prices around the plan>
```

Outputs in the out folder:
- `report.md`: computed call, odds, the split by demand found or not, P10/P50/P90 tables, key numbers, unit economics, stress tests, outside view, drivers, assumption map, sanity ratios, what must be true, sweep, warnings.
- `summary.json`: everything in machine form.
- `explain_median_path.json`: year by year line items of the typical run (use for the plain-words calculation).
- `annual_percentiles.csv`, `monthly_percentiles.csv`, `scenario_paths.csv` (full monthly P&L and cash for the P10, P50, P90 runs and the median-input path), `assumptions.csv`, `tornado.csv`.
- `charts/`: cash fan, revenue fan, operating profit fan, outcome histogram, survival curve, tornado.

Definitions the user must hear correctly:
- Month 0 = today. Launch = first month of sales.
- Break-even month = first of 3 months in a row with positive operating profit (EBITDA).
- Money needed = the deepest point of cumulative cash flow (including owner draws). Recommend having the P80 to P90 amount available.
- Founder net gain = everything the founder took out + what they own at the end (cash minus debts plus sale value if still open) minus everything they put in. All after tax.
- Owner hourly = after-tax profit plus any salary, divided by hours worked, only for runs still open that year.
- Computed call. NO-GO: the business fails even when demand is found (it loses money in 60% or more of those runs, or beats the job alternative in under 15%, or half never break even, or a sale loses money before fixed costs). GO: over all runs it loses money in at most 35%, beats the job alternative in at least half, runs out of money in at most 20%, evidence is level 3 or higher and no warning is open. Everything else is TEST FIRST. Half of any sale value is counted. The lines can be moved in `verdict` only with a stated reason.

## Step 6: Calibrate and clear warnings

Read every warning in `report.md` and act:
- `UNKNOWN KEY` or `UNKNOWN DIST FIELD`: a typo. Fix it. Typos silently fall back to defaults.
- `UNSOURCED INPUT`: research it, or mark the source as "user's own number".
- `MISSING COST`: add the category or write why it is zero.
- `NO OPTIMISM CORRECTION`: add `reality_factor`.
- `BENCHMARK MISS`: find the input that causes it. Fix it if it is wrong. If this business truly differs, keep it and explain why in the report.
- `OUTSIDE VIEW` (the model is rosier than real businesses of this kind allow at this evidence level): first check that the rate fits this business and re-check the demand inputs. If it stays, the report gives an outside-view calibration: the demand multiplier at which the model sits closest to the real rates. Set `demand.outside_view_factor` to the suggested value (source: the base rates), rerun, and use that run as the main result. A gap that demand cannot close (for example the survival of a solo business with almost no fixed costs, where closing is a choice) is handled in the quit rule or accepted with a written reason.
- `PESSIMISM CHECK`: the model is far below the real rate. Look for missing revenue or too little starting money. If the inputs are right, this plan is worse than the typical business of its kind; say so.
- `UNGRADED KEY INPUT` / `GUESS ON A KEY INPUT`: grade the input (`ev`) or research it. A top driver that is only a guess is the first thing to test for real.
- `CONFIG ERROR`: fix it.
- `RED FLAG` (very high margins, break-even in under 4 months, LTV/CAC above 8, churn under 1%, huge market share, owner under 20 hours with no staff): fix the inputs or justify.

Comparables check: find at least 2 real comparable businesses with known numbers (BizBuySell listings for the same type and region, FDD Item 19, public data). Put their mature yearly revenue range into `benchmarks.mature_annual_revenue` and their margin range into `benchmarks.net_margin` or `ebitda_margin`, rerun, and make sure the typical surviving run lands inside or explain why not.

Independent audit (skip in quick mode): give one sub-agent `config.json`, `assumptions.csv`, `report.md` and the research log with this task: "You are an experienced owner in this industry and a banker reviewing this model before lending. List every missing cost, every input more optimistic than its source, every double count, every unit error (monthly vs yearly, per order vs per unit) and every timeline that cannot happen. For each: what is wrong, the evidence, the corrected value or range." Apply what holds up, rerun, and note what you changed.

Repeat until clean or until every remaining warning has a written reason (usually 2 to 5 runs). Keep a short changelog of what you changed and why. Never tune inputs just to get a nicer answer; tune only toward evidence.

## Step 7: Stress tests and what must be true

1. `--solve` the 2 or 3 inputs that matter most (main demand input, price, CAC or rent). The report then says: "total profit turns positive at X; owner earns a market wage at Y". Compare X and Y with the realistic P50.
2. Stress tests are already in the report (price -10%, demand -20%, variable costs +15%, fixed costs +20%, acquisition cost +50%, launch 3 months late, everything a bit worse). Add one "smarter plan" variant the founder could actually choose (cheaper location, smaller build-out, pre-sales first, part-time start while keeping the job): copy the config, change it, rerun with `--sims 3000 --no-charts`, and put it next to the base case.
3. Price ladder: `--sweep <price input>=<5 prices around the plan>` (needs `price_elasticity`). Pick the price with the best owner earnings that is still inside the panel's acceptable range.
4. Real-world test plan: the cheapest test that lifts the evidence level one step. From level 0 or 1: ten conversations with strangers in the target group, or a landing page with real ad spend (100 visitors or more). From level 2: a pre-sale, deposits or letters of intent. From level 3: a paid pilot or a pop-up. Take the rate the business needs from `--solve` (break-even conversion, or break-even sales divided by people exposed) and fix the lines before the test: `python3 bizsim.py testplan --need <needed rate> --n <people exposed> --expect <realistic rate>` prints the GO line, the STOP line and how many people a fair test needs. Rough guides when no rate can be derived: a well-targeted landing page under 5% signups is weak, 5 to 15% is worth one more test, above 15% is strong; a pre-sale passes at 10 or more strangers paying; friends and family never count.

## Step 8: Report to the user

Write in the user's language, simple words, no em dashes, no hedging. Order:

1. The computed call (GO / TEST FIRST / NO-GO), the engine's reasons in plain words, and the one number that decides it. If demand may not exist (`fit_probability`): the chance it is found, what happens if it is, what is lost if it is not.
2. Table: year 1 to 5 revenue and after-tax profit, bad (P10) / typical (P50) / good (P90).
3. Odds: chance still open 1, 3 and 5 years after opening (`still_open_Ny_after_opening`); chance of losing money overall; chance it beats a normal job. Then the outside view: the model next to the real-world rate for each check.
4. Money: money needed (P80 to P90 including owner draws), break-even month, payback month.
5. Owner pay: real hourly earnings per year vs their job alternative.
6. How the typical year is calculated, in words, one operation per line, using `explain_median_path.json` (first mature year). No formula symbols. Turkish example:
   - Ayda ortalama 3.900 satış var.
   - Her satış ortalama 11,50 dolar.
   - 3.900 çarpı 11,50 eder 44.850 dolar aylık ciro.
   - Malzeme maliyeti cironun yüzde 31'i: 44.850 çarpı 0,31 eder 13.904 dolar.
   - 44.850 eksi 13.904 eder 30.946 dolar brüt kâr.
   - (continue: staff, rent, other costs, operating profit, loan payments, tax, what is left, divided by hours)
7. Unit economics in one breath: what one sale leaves after its own costs, how many sales a day break even, how far the typical case sits above or below that.
8. What moves the result most (top 3 to 5 drivers in plain words, from the Spearman and tornado lists) and which of them rest on weak evidence (assumption map).
9. What the board said (votes, the risks two or more seats named) and what the panel said (pull against the control, top reasons for no, who buys). Label both as simulated.
10. What must be true (solve results): "Para kaybetmemek için ayda en az X satış lazım; araştırmaya göre gerçekçi beklenti Y." Then the stress table in one or two lines and the price ladder.
11. The real-world test: what to do this week, what it costs, the GO line and the STOP line, fixed before the test. Real buyers overrule the panel and the model.
12. Key assumptions with sources (short list). Mention that the full assumption table exists.

Send the charts (cash fan, outcome histogram, survival) with the answer. Offer the full monthly model as a spreadsheet. Say plainly what the model does not cover when it matters (sales tax is passed through and excluded, personal health insurance, the founder's lost study time, big one-off opportunities).

---

## Config reference

All money is in `meta.currency`. Months are relative: `start`, `month`, `end` inside items and `financing.owner_draw_start` count from LAUNCH (negative = before launch; anything earlier than today happens at month 0). Any numeric field can be a number, a `[low, mode, high]` PERT list, a dist object, or `{"ref": "name"}`, unless noted.

Distributions: `{"dist": "pert", "low", "mode", "high"}`, `triangular` (same fields), `uniform` (low, high), `normal` (mean, sd), `lognormal` (p10, p90 or median, p90), `choice` (values, probs), `bernoulli` (p), `mixture` (components, weights), `fixed` (value). Optional on any dist: `min`, `max`, `name`, `source`, `why`, `ev` (evidence grade A to D). Items (staff, costs, channels...) may carry a `source` that their dists inherit.

- `meta`: `name`, `currency`, `software` (true skips the high-margin red flag), `evidence_level` (0 to 5), `accepted_warnings` (list of `{match, why}`).
- `simulation` (fixed numbers): `months` (default 60), `n_sims` (10000), `seed`, `start_calendar_month` (1 to 12, for seasonality), `inflation_annual` (0.03), `discount_rate_annual` (0.15, for NPV).
- `timeline`: `launch_month` (months from today to first sale), `ramp_months`, `ramp_start_level` (share of mature demand at launch), `ramp_shape` (`exponential` default, `s_curve`, `linear`).
- `demand` (common): `model` (`traffic` or `funnel`), `reality_factor`, `seasonality` (12 numbers, normalized), `monthly_volatility` (0.15; lognormal month noise), `volatility_autocorr` (0.5), `annual_price_increase` (= inflation), `discount_pct`, `refund_pct`, `bad_debt_pct`.
- `demand` (v3, all multiply demand): `panel_factor` (from the customer panel), `outside_view_factor` (from the outside-view calibration), `fit_probability` with `no_fit_factor` (chance that demand exists; multiplier when it does not), `price_elasticity` with `reference_price` (demand times (price / reference_price) to the power of the elasticity; price = `price_per_unit` or `first_purchase_value`).
- `demand` traffic: `mature_units_per_month`, `price_per_unit`, `units_per_transaction` (1; units per payment), `capacity_units_per_month` (number or from_staff dict).
- `demand` funnel: `channels` list (`name`, `monthly_budget`, `cac` or `cpc` plus `conversion`, `start`, `ref_budget`, `diminishing_exponent`, `pct_of_revenue` with `min_budget`/`max_budget`, `annual_budget_growth`), `organic_new_per_month` (at full ramp), `referral_rate` (new customers per active customer per month), `early_cac_multiplier` (1.3), `market_size_customers`, `saturation_basis` (`active` or `ever`), `first_purchase_value`, `repeat_orders_per_active_month` (1), `repeat_order_value`, `early_churn_monthly`, `early_period_months` (3), `churn_monthly` (0.05), `capacity_active_customers`, `subscription` (false), `repeat_seasonal`.
- `variable_costs`: `cogs_pct` (of net revenue), `cost_per_unit`, `cost_per_order`, `shrinkage_pct_of_cogs`, `payment_fee_pct` (0.029), `payment_fee_fixed` (0.30), `card_share` (1.0), `platform_fee_pct`, `platform_fee_per_order`, `sales_commission_pct`, `annual_cost_increase`.
- `staff` list: `name`, `monthly_cost` or `hourly_wage` plus `hours_per_month` (173 = full time), `count` (1), `burden_pct` (0.25), `annual_raise`, `start`, `end`, `direct` (true = counts in COGS), `recruiting_cost`, `trigger` (`metric`, `threshold`, `lag_months`), `scaling` (`metric`, `per_staff`, `offset` = volume the owner handles alone, `min`, `max`, `lag_months`, `fractional`). Metrics: `revenue`, `units`, `active`, `transactions`, `demand`, `ebitda`.
- `fixed_costs` list: `name`, `category`, `monthly`, `pct_of_revenue` (royalties, percentage rent, contingency), `start`, `end`, `annual_increase`.
- `one_time_costs` list: `name`, `amount`, `month`, `expense` (true; false = deposit or asset), `refundable_at_close`.
- `capex` list: `name`, `amount`, `month`, `life_months`, `replace` (true = buy again at end of life), `salvage_pct` (0.2 at closure).
- `maintenance_pct_of_capex_annual`.
- `working_capital`: `dso_days`, `dpo_days`, `inventory_days`, `min_inventory`, `fixed_costs_on_terms`.
- `financing`: `founder_cash`, `founder_reserve`, `min_cash_buffer`, `loans` list (`amount`, `annual_rate`, `term_months`, `interest_only_months`, `month`), `credit_line` (`limit`, `annual_rate`), `owner_draw_monthly`, `owner_draw_start`, `owner_draw_only_if_profitable` (true).
- `owner`: `hours_per_week`, `opportunity_wage_hourly`, `salary_monthly` (company-paid salary, S-corp style), `salary_burden_pct`, `job_effective_tax_rate` (0.22).
- `taxes` (fixed values): `mode` (`us_passthrough`, `us_c_corp`, `flat`, `none`), `filing` (`single`, `mfj`), `state` (`CA`, `flat`, `none`), `state_rate`, `ca_llc`, `other_income`, `other_w2_wages`, `losses_offset_other_income` (true), `qbi` (true), `sstb`, `nol_limit_pct` (0.8), `flat_rate`, `state_corp_rate`, `other_annual_fixed_taxes`, `gross_receipts_tax_pct`. Taxes are computed per 12 months from today and paid the month after.
- `shocks` list: `name`, `type` (`cost` uses `amount`; `revenue_hit` uses `pct` and `duration_months`; `customer_loss` uses `pct`), `monthly_prob`.
- `quit_rule` (values may be ranges: owners differ): `max_months_without_profit` (patience: from this many months after launch, while the last 6 months of operating profit add up to zero or less, the owner closes with `monthly_quit_prob`, default 0.2, each month), `min_owner_hourly` (while the last 12 months paid the owner less than this per hour before tax, checked from `min_owner_hourly_after_months` after launch, the owner closes with `low_pay_quit_prob`, default 0.03, each month), `max_total_loss` (closes at once when the founder has lost this much), `exogenous_exit_annual` (default 0.03: closing for outside reasons).
- `exit`: `basis` (`sde` or `revenue`), `sde_multiple`, `revenue_multiple`, `count_exit_value`, `forced_sale_fraction` (0.5: share of the normal sale value received when a working business closes for outside reasons).
- `liquidation`: `ar_collect_pct` (0.9), `inventory_recovery_pct` (0.3).
- `benchmarks` (ranges `[lo, hi]`, or single numbers for survival): `gross_margin`, `ebitda_margin`, `net_margin`, `marketing_pct`, `labor_pct`, `rent_pct`, `prime_cost_pct`, `payment_fee_pct`, `monthly_churn`, `ltv_cac`, `cac_payback_months`, `mature_annual_revenue`, `revenue_per_fte`, `median_breakeven_months_after_launch`, `survival_year_1`, `survival_year_2`, `survival_year_3`, `survival_year_5`.
- `required_categories`: fixed cost categories that must exist (default insurance, accounting, software, contingency).
- `base_rates`: list of real-world rates the model is held against (fields in Step 4 rule 22). `benchmarks.survival_year_N` is read as a base rate too.
- `verdict` (optional, each with a reason in `notes`): `go_loss_prob` (0.35), `go_beats_job` (0.50), `go_cashout_prob` (0.20), `go_min_evidence` (3), `nogo_loss_prob` (0.60), `nogo_beats_job` (0.15), `exit_weight` (0.5).
- `correlations`: list of `[name_or_path_a, name_or_path_b, rank_correlation]`.

Starter template (funnel, subscription). Replace every number with researched ranges and sources:

```json
{
  "meta": {"name": "IDEA", "currency": "USD", "software": true, "evidence_level": 0},
  "simulation": {"months": 84, "n_sims": 10000, "seed": 7, "start_calendar_month": 1, "inflation_annual": 0.03},
  "timeline": {"launch_month": [2, 3, 6], "ramp_months": [9, 15, 24], "ramp_start_level": 0.15},
  "demand": {"model": "funnel", "subscription": true,
    "channels": [{"name": "google", "monthly_budget": 1500, "cpc": {"dist": "pert", "low": 3, "mode": 5, "high": 9, "source": "..."},
                  "conversion": {"dist": "pert", "low": 0.005, "mode": 0.012, "high": 0.025, "source": "..."}, "ref_budget": 1500, "diminishing_exponent": 0.3}],
    "organic_new_per_month": [1, 4, 10], "referral_rate": [0.002, 0.008, 0.02],
    "first_purchase_value": {"dist": "pert", "low": 39, "mode": 49, "high": 59, "name": "price", "source": "...", "ev": "B"},
    "repeat_order_value": {"ref": "price"}, "repeat_orders_per_active_month": 1,
    "early_churn_monthly": [0.08, 0.12, 0.2], "early_period_months": 3, "churn_monthly": [0.03, 0.05, 0.08],
    "refund_pct": 0.02, "monthly_volatility": 0.2,
    "reality_factor": {"dist": "mixture", "weights": [0.3, 0.7], "components": [{"dist": "uniform", "low": 0.35, "high": 0.7}, {"dist": "pert", "low": 0.7, "mode": 0.9, "high": 1.1}], "source": "optimism correction (Cassar 2010)"},
    "fit_probability": 0.2, "no_fit_factor": [0.03, 0.12, 0.35], "price_elasticity": [-3.0, -1.8, -0.8], "reference_price": 49},
  "variable_costs": {"cost_per_order": [2, 4, 8], "payment_fee_pct": 0.029, "payment_fee_fixed": 0.30},
  "staff": [],
  "fixed_costs": [
    {"name": "software", "category": "software", "monthly": [100, 200, 400]},
    {"name": "insurance", "category": "insurance", "monthly": [40, 60, 120]},
    {"name": "bookkeeping", "category": "accounting", "monthly": [100, 200, 300]},
    {"name": "contingency", "category": "contingency", "monthly": 0, "pct_of_revenue": 0.03}],
  "one_time_costs": [{"name": "formation_legal", "amount": [500, 1200, 3000], "month": -1}],
  "capex": [],
  "working_capital": {},
  "financing": {"founder_cash": 10000, "founder_reserve": 5000, "credit_line": {"limit": 5000, "annual_rate": 0.25}, "owner_draw_monthly": 0},
  "owner": {"hours_per_week": 25, "opportunity_wage_hourly": 22},
  "taxes": {"mode": "us_passthrough", "state": "CA", "ca_llc": true, "filing": "single", "other_income": 0},
  "shocks": [{"name": "platform_or_ad_ban", "type": "revenue_hit", "monthly_prob": 0.01, "pct": [0.2, 0.4, 0.7], "duration_months": 2}],
  "quit_rule": {"max_months_without_profit": [12, 24, 48], "monthly_quit_prob": 0.2, "min_owner_hourly": 10, "min_owner_hourly_after_months": 24, "low_pay_quit_prob": 0.03, "exogenous_exit_annual": 0.06},
  "exit": {"basis": "revenue", "revenue_multiple": [1, 2, 3.5]},
  "benchmarks": {"gross_margin": [0.6, 0.9], "monthly_churn": [0.02, 0.08]},
  "base_rates": [
    {"name": "1,000 a month in revenue within 12 months of launch", "metric": "monthly_revenue", "months_after_launch": 12, "threshold": 1000, "rate": 0.17, "source": "..."},
    {"name": "10,000 a month in revenue within 24 months of launch", "metric": "monthly_revenue", "months_after_launch": 24, "threshold": 10000, "rate": 0.05, "source": "..."},
    {"name": "profit in the first year", "metric": "profitable_year", "year": 1, "rate": 0.45, "source": "..."}]
}
```

For a `traffic` business replace the funnel fields with `mature_units_per_month`, `price_per_unit`, `capacity_units_per_month`, add staff with `scaling` on `demand`, rent, utilities, capex for build-out, `working_capital.inventory_days`, and restaurant style benchmarks.

## Benchmark library (collected Nov 2026; verify for each case, numbers age)

Base rates
- New US establishments: about 20.4% close in year 1; 49.2% survive 5 years; 33.9% reach 10 years (SBA Office of Advocacy Feb 2026 using BLS BED). Varies by industry; look up the NAICS sector.
- BizBuySell 2025 (sold small businesses): median revenue $703,000, median cash flow (SDE) $158,950, median sale price $350,000; cash flow multiples around 2.5 to 2.7; sold at about 94% of asking.

Outside view: what really happens (use as `base_rates`, then look for rates closer to the exact business)
- Survival after opening, businesses with employees (BLS Business Employment Dynamics; open after 1 / 5 / 10 years): all private 79.6 / 50.6 / 34.7%; accommodation and food 82.5 / 55.3 / 38.2%; retail 84.2 / 58.3 / 41.7%; other services 82.5 / 56.9 / 39.6%; professional and technical services 77.0 / 46.3 / 30.9%; information 74.9 / 44.3 / 29.1%; construction 76.0 / 53.9 / 40.1%.
- All new US firms (Kauffman Firm Survey, 2004 cohort, 4,928 firms, 59% without employees): 36.7% had no revenue in the first year. First-year revenue above $25,000: 35% of all firms, about 55% of those with any revenue. Above $100,000: 17% of all, about 27% of those with revenue (2004 dollars; multiply by about 1.7 for today). About 45% made a profit in the first year. 49% were still operating after 6 years; of the survivors in year 8, 45% had revenue above $100,000.
- Closing is not always failing: about a third of owners of closed businesses called them successful at closing (Headd 2003). Restaurants: 26% closed or changed hands in year 1 and 59% within 3 years (Parsa et al., Columbus).
- Owner income: sole proprietors (IRS SOI 2023, 31.1 million returns) averaged $66,285 in receipts and $12,119 in net income; 31% had no net income; the profitable ones averaged $24,790. After 10 years the median self-employed person earns about 35% less than in comparable paid work; only the top quarter earns more (Hamilton 2000).
- Cash: the median small business holds 27 days of cash (restaurants 16, retail 19, personal services 21, high-tech services 33) (JPMorgan Chase Institute).
- New software: subscription apps (RevenueCat 2024, about 30,000 apps): 17% reach $1,000 of monthly revenue within a year of launch and 3.5% reach $10,000; within two years about 20 to 25% and about 5%. Indie SaaS listings: about 70% under $1,000 MRR, about 6% above $10,000. "No market need" appears in 35 to 42% of startup post-mortems (CB Insights).
- Repeat custom: most first-time restaurant guests never return; a loyalty program lifts the return rate from about 7% to almost 30% (Toast).
- Early signals: waitlist signup from cold or paid traffic 3 to 10%; waitlist to paid 10 to 20% for software, under 5% for physical products.
- Founders' own forecasts: 62% overestimate first-year sales (Cassar 2010); 81% of new owners put their odds of success at 7 in 10 or better and a third at 10 in 10 (Cooper, Woo and Dunkelberg 1988); revenue plans of venture-backed startups ran several times above what happened (Mokwa and Sievers).

Restaurants and food
- Food cost about 28 to 35% (QSR 28 to 30%, full service 34 to 36%); labor 25 to 35%; prime cost (food plus labor) target 55 to 65%; rent 5 to 10% ideal (Bay Area often higher); net margin full service 3 to 6%, quick service 6 to 10% (Complete Controller, Toast, NRA summaries). Public restaurant companies: 32% gross, 9.4% net (Damodaran Jan 2026; much better than independents).

Ecommerce
- Conversion: Shopify average about 1.4% (Littledata), top 20% above 3.2%, top 10% above 4.7%; global ecommerce about 2 to 3%.
- Returns: Shopify average 17 to 20% of items; apparel 30 to 40% online.
- Fees: Shopify payments 2.9% + 30c plus plan; Etsy $0.20 listing + 6.5% transaction + 3% + $0.25 processing (+12 to 15% offsite ads on attributed sales); Amazon referral about 15% for most categories.

SaaS and subscriptions
- B2B monthly churn median about 3.5% (Recurly 2025); SMB tools under $25/month 5 to 7%; good B2B under 1% monthly.
- Gross margin 70 to 80%; LTV/CAC about 3 is healthy; CAC payback 12 to 18 months acceptable; SMB CAC roughly $1,100 to $1,400 (OpenView 2025 as cited in secondary sources).
- App stores: Apple 30% standard, 15% Small Business Program and renewals after year 1.

Advertising (US, WordStream / LocaliQ)
- Google search 2026 report: average CPC $5.42, conversion rate 8.18% (to lead), cost per lead $66.69; legal and home services the most expensive. 2025 report: CPC $5.26, conversion 7.52%, CPL $70.11.
- Meta 2026: traffic CPC about $0.60, lead campaigns CPC about $1.80 and CPL about $27.39.
- A lead is not a customer: multiply by your lead to paying rate.

Payments and fees
- Stripe US: 2.9% + 30c online, 2.7% + 5c in person, +1.5% international cards, +1% currency conversion, $15 per dispute; ACH 0.8% capped at $5.

Labor (US)
- Real employer cost 1.25 to 1.4 times base pay (taxes, benefits). Employer FICA 7.65%, FUTA 0.6% on first $7,000, plus state unemployment and workers comp.
- California minimum wage $16.90 (2026), $17.40 (Jan 2027). Berkeley $19.61 (from Jul 2026). Many Bay Area cities are higher than the state.

Insurance
- General liability median about $42 to $45 per month; business owner's policy about $83 per month (Insureon customer medians). Food, construction and cleaning cost more.

Taxes built into the engine
- US 2026 federal brackets (single 10% to $12,400, 12% to $50,400, 22% to $105,700, 24% to $201,775, 32% to $256,225, 35% to $640,600, 37% above; MFJ doubled except 35% to $768,700), standard deduction $16,100 single / $32,200 MFJ.
- Self-employment tax 15.3% on 92.35% of profit (Social Security part only up to $184,500 in 2026), half deductible; 0.9% additional Medicare above $200,000 single.
- QBI deduction 20% (permanent), limits start at $201,750 single / $403,500 MFJ (phase-in $75,000 / $150,000).
- California 2025 schedule (latest published), standard deduction $5,706 single, exemption credit $153; CA LLC $800 per year plus fee from $250,000 gross receipts ($900 / $2,500 / $6,000 / $11,790).
- NOLs carry forward and offset 80% of later profit; business losses offset W-2 income when `other_income` is set.

Optimism
- Cassar (2010, SMJ): nascent entrepreneurs overestimate the chance of getting to operation and overestimate sales and employment; preparing projections increases the overestimation. Treat any founder forecast as the optimistic end of the range until proven.

---

## Appendix: engine `bizsim.py`

Copy everything between the two marker lines into `bizsim.py` (or use the extraction script in Step 5). Do not edit the engine for a single case; change the config instead. If you find a real bug, fix it, rerun the self-check below, and tell the user.

After writing the engine, `python3 bizsim.py selftest` must end with `SELFTEST OK`.

```python
# === BIZSIM START ===
#!/usr/bin/env python3
"""bizsim v3: realistic Monte Carlo simulator for small business ideas.

Usage:
  python3 bizsim.py config.json --out OUTDIR [--sims 10000] [--seed 7] [--no-charts] [--solve NAME] [--sweep NAME=v1,v2,..]
  python3 bizsim.py selftest                                       golden-number checks (must end with SELFTEST OK)
  python3 bizsim.py testplan --need 0.04 --n 200 [--expect 0.06]   go and stop lines for a real-world test

Only needs numpy (matplotlib optional for charts). All money in config currency.
Month 0 = the month the founder starts spending (today). Launch = first month of sales.
"""
import argparse, json, math, os, sys
import numpy as np

Z90 = 1.2815515655446004
SKIP_KEYS = {"meta", "benchmarks", "seasonality", "correlations", "notes", "required_categories", "base_rates", "verdict"}

# ---------------------------------------------------------------- distributions
def _is_dist(x):
    return isinstance(x, dict) and "dist" in x

def _is_short(x):
    return isinstance(x, list) and len(x) == 3 and all(isinstance(v, (int, float)) for v in x)

def sample_dist(d, n, rng):
    k = d["dist"]
    if k == "fixed":
        v = np.full(n, float(d["value"]))
    elif k == "pert":
        a, m, b = float(d["low"]), float(d["mode"]), float(d["high"])
        if not a <= m <= b:
            raise ValueError(f"pert needs low<=mode<=high: {d}")
        if b - a < 1e-12:
            v = np.full(n, m)
        else:
            lam = float(d.get("lamb", 4.0))
            al = 1 + lam * (m - a) / (b - a)
            be = 1 + lam * (b - m) / (b - a)
            v = a + (b - a) * rng.beta(al, be, n)
    elif k == "triangular":
        a, m, b = float(d["low"]), float(d["mode"]), float(d["high"])
        v = np.full(n, m) if b - a < 1e-12 else rng.triangular(a, m, b, n)
    elif k == "uniform":
        v = rng.uniform(float(d["low"]), float(d["high"]), n)
    elif k == "normal":
        v = rng.normal(float(d["mean"]), float(d["sd"]), n)
    elif k == "lognormal":
        if "p10" in d and "p90" in d:
            lo, hi = float(d["p10"]), float(d["p90"])
            mu = (math.log(lo) + math.log(hi)) / 2
            sg = (math.log(hi) - math.log(lo)) / (2 * Z90)
        else:
            mu = math.log(float(d["median"]))
            sg = (math.log(float(d["p90"])) - mu) / Z90
        v = np.exp(rng.normal(mu, sg, n))
    elif k == "choice":
        vals = np.asarray(d["values"], float)
        pr = np.asarray(d.get("probs", [1 / len(vals)] * len(vals)), float)
        v = rng.choice(vals, n, p=pr / pr.sum())
    elif k == "bernoulli":
        v = (rng.random(n) < float(d["p"])).astype(float)
    elif k == "mixture":
        comps = d["components"]
        w = np.asarray(d.get("weights", [1] * len(comps)), float)
        pick = rng.choice(len(comps), n, p=w / w.sum())
        v = np.zeros(n)
        for j, c in enumerate(comps):
            v = np.where(pick == j, sample_dist(c, n, rng), v)
    else:
        raise ValueError(f"unknown dist {k}")
    if "min" in d:
        v = np.maximum(v, float(d["min"]))
    if "max" in d:
        v = np.minimum(v, float(d["max"]))
    return v

def extract(node, path, reg, src="", ev=""):
    """Replace every uncertain input with a placeholder and register it. Items pass their 'source' and 'ev' grade down."""
    if _is_dist(node) or _is_short(node):
        spec = dict(node) if _is_dist(node) else {"dist": "pert", "low": node[0], "mode": node[1], "high": node[2]}
        name = spec.get("name") or path
        if name in reg:
            raise ValueError(f"duplicate variable name: {name}")
        spec["path"] = path
        if not spec.get("source") and src:
            spec["source"] = src
        if not spec.get("ev") and ev:
            spec["ev"] = ev
        reg[name] = spec
        return {"__var__": name}
    if isinstance(node, dict) and set(node) == {"ref"}:
        return {"__var__": node["ref"]}  # reuse another uncertain input (same draw), e.g. repeat price = first price
    if isinstance(node, dict):
        src2 = node.get("source", src) if isinstance(node.get("source", ""), str) else src
        ev2 = node.get("ev", ev) if isinstance(node.get("ev", ""), str) else ev
        out = {}
        for k, v in node.items():
            out[k] = v if k in SKIP_KEYS else extract(v, f"{path}.{k}" if path else k, reg, src2, ev2)
        return out
    if isinstance(node, list):
        out = []
        for i, v in enumerate(node):
            tag = v.get("name", i) if isinstance(v, dict) and not _is_dist(v) else i
            out.append(extract(v, f"{path}[{tag}]", reg, src, ev))
        return out
    return node

DIST_FIELDS = {"dist", "low", "mode", "high", "value", "mean", "sd", "p10", "p90", "median", "values", "probs", "p", "min", "max", "lamb", "name", "source", "why", "ev", "components", "weights"}
DOC = {"source", "why", "name", "notes", "ev"}
SCHEMA = {
    "": {"meta", "simulation", "timeline", "demand", "variable_costs", "staff", "fixed_costs", "one_time_costs", "capex",
         "maintenance_pct_of_capex_annual", "working_capital", "financing", "owner", "taxes", "shocks", "quit_rule", "exit",
         "benchmarks", "correlations", "required_categories", "liquidation", "notes", "base_rates", "verdict"},
    "simulation": {"months", "n_sims", "seed", "start_calendar_month", "inflation_annual", "discount_rate_annual"},
    "timeline": {"launch_month", "ramp_months", "ramp_start_level", "ramp_shape"},
    "demand": {"model", "mature_units_per_month", "price_per_unit", "units_per_transaction", "capacity_units_per_month", "channels",
               "organic_new_per_month", "referral_rate", "early_cac_multiplier", "market_size_customers", "saturation_basis",
               "first_purchase_value", "repeat_orders_per_active_month", "repeat_order_value", "early_churn_monthly",
               "early_period_months", "churn_monthly", "capacity_active_customers", "subscription", "repeat_seasonal", "seasonality",
               "monthly_volatility", "volatility_autocorr", "annual_price_increase", "discount_pct", "refund_pct", "bad_debt_pct", "reality_factor",
               "panel_factor", "outside_view_factor", "fit_probability", "no_fit_factor", "price_elasticity", "reference_price"},
    "demand.channels[]": {"monthly_budget", "cac", "cpc", "conversion", "start", "ref_budget", "diminishing_exponent", "pct_of_revenue",
                          "min_budget", "max_budget", "annual_budget_growth"},
    "variable_costs": {"cogs_pct", "cost_per_unit", "cost_per_order", "shrinkage_pct_of_cogs", "payment_fee_pct", "payment_fee_fixed",
                       "card_share", "platform_fee_pct", "platform_fee_per_order", "sales_commission_pct", "annual_cost_increase"},
    "staff[]": {"role", "monthly_cost", "hourly_wage", "hours_per_month", "count", "burden_pct", "annual_raise", "start", "end", "direct",
                "recruiting_cost", "trigger", "scaling"},
    "fixed_costs[]": {"category", "monthly", "pct_of_revenue", "start", "end", "annual_increase"},
    "one_time_costs[]": {"amount", "month", "expense", "refundable_at_close"},
    "capex[]": {"amount", "month", "life_months", "replace", "salvage_pct"},
    "working_capital": {"dso_days", "dpo_days", "inventory_days", "min_inventory", "fixed_costs_on_terms"},
    "financing": {"founder_cash", "founder_reserve", "min_cash_buffer", "loans", "credit_line", "owner_draw_monthly", "owner_draw_start",
                  "owner_draw_only_if_profitable"},
    "financing.loans[]": {"amount", "annual_rate", "term_months", "interest_only_months", "month"},
    "financing.credit_line": {"limit", "annual_rate"},
    "owner": {"hours_per_week", "opportunity_wage_hourly", "salary_monthly", "salary_burden_pct", "job_effective_tax_rate"},
    "taxes": {"mode", "filing", "state", "state_rate", "ca_llc", "other_income", "other_w2_wages", "losses_offset_other_income", "qbi",
              "sstb", "nol_limit_pct", "flat_rate", "state_corp_rate", "other_annual_fixed_taxes", "gross_receipts_tax_pct"},
    "shocks[]": {"type", "monthly_prob", "amount", "pct", "duration_months"},
    "quit_rule": {"max_months_without_profit", "max_total_loss", "min_owner_hourly", "min_owner_hourly_after_months", "monthly_quit_prob",
                  "low_pay_quit_prob", "exogenous_exit_annual"},
    "exit": {"basis", "sde_multiple", "revenue_multiple", "count_exit_value", "forced_sale_fraction"},
    "verdict": {"go_loss_prob", "go_beats_job", "go_min_evidence", "go_cashout_prob", "nogo_loss_prob", "nogo_beats_job", "exit_weight"},
    "base_rates[]": {"metric", "years", "year", "months_after_launch", "threshold", "rate", "good"},
    "liquidation": {"ar_collect_pct", "inventory_recovery_pct"},
}

BASE_METRICS = {"survival", "monthly_revenue", "annual_revenue", "profitable_year", "owner_income"}
EVIDENCE = ("idea and desk research only", "synthetic panel or proof that people pay competitors", "real conversations, search demand or waitlist (words and clicks)",
            "costly signals short of money (letters of intent, deposits, agreed pilots)", "money changed hands (pre-sales or paid pilots from strangers)",
            "operating history (3+ months of real revenue)")
LIFT = (0.15, 0.20, 0.30, 0.45, 0.60, 1.0)  # how far above a real-world base rate the model may sit, by evidence level

def validate(cfg):
    """Return a list of problems: unknown keys (typos are silent killers) and bad values."""
    probs = []
    def chk(d, key, where):
        if not isinstance(d, dict):
            return
        allowed = SCHEMA.get(key, set()) | DOC
        for k, v in d.items():
            if k not in allowed:
                probs.append(f"UNKNOWN KEY '{k}' in {where or 'top level'} (typo? it is ignored)")
            if _is_dist(v):
                bad = set(v) - DIST_FIELDS
                if bad:
                    probs.append(f"UNKNOWN DIST FIELD {sorted(bad)} in {where}.{k}")
            if k == "ev" and v not in ("A", "B", "C", "D"):
                probs.append(f"BAD EVIDENCE GRADE '{v}' in {where or 'top level'}: use A, B, C or D")
    chk(cfg, "", "")
    for sec in ("simulation", "timeline", "demand", "variable_costs", "working_capital", "financing", "owner", "taxes", "quit_rule", "exit", "liquidation", "verdict"):
        chk(cfg.get(sec), sec, sec)
    chk((cfg.get("financing") or {}).get("credit_line"), "financing.credit_line", "financing.credit_line")
    for sec in ("staff", "fixed_costs", "one_time_costs", "capex", "shocks", "base_rates"):
        for i, it in enumerate(cfg.get(sec) or []):
            chk(it, sec + "[]", f"{sec}[{it.get('name', i) if isinstance(it, dict) else i}]")
    for i, it in enumerate((cfg.get("demand") or {}).get("channels") or []):
        chk(it, "demand.channels[]", f"demand.channels[{it.get('name', i)}]")
    for i, it in enumerate((cfg.get("financing") or {}).get("loans") or []):
        chk(it, "financing.loans[]", f"financing.loans[{it.get('name', i)}]")
    dm = (cfg.get("demand") or {}).get("model", "funnel")
    if dm not in ("funnel", "traffic"):
        probs.append(f"CONFIG ERROR: demand.model must be 'funnel' or 'traffic', got {dm}")
    if dm == "traffic" and "mature_units_per_month" not in (cfg.get("demand") or {}):
        probs.append("CONFIG ERROR: traffic model needs demand.mature_units_per_month")
    if dm == "funnel" and "first_purchase_value" not in (cfg.get("demand") or {}):
        probs.append("CONFIG ERROR: funnel model needs demand.first_purchase_value")
    for sh in cfg.get("shocks") or []:
        if sh.get("type", "cost") not in ("cost", "revenue_hit", "customer_loss"):
            probs.append(f"CONFIG ERROR: shock type must be cost, revenue_hit or customer_loss: {sh.get('name')}")
    dmd = cfg.get("demand") or {}
    if dmd.get("price_elasticity") is not None and dmd.get("reference_price") is None:
        probs.append("CONFIG ERROR: demand.price_elasticity needs demand.reference_price (the price at which the demand inputs were researched)")
    for br in cfg.get("base_rates") or []:
        if br.get("metric") not in BASE_METRICS:
            probs.append(f"CONFIG ERROR: base_rates metric must be one of {sorted(BASE_METRICS)}: {br.get('name')}")
        if not isinstance(br.get("rate"), (int, float)):
            probs.append(f"CONFIG ERROR: base_rates item needs a numeric 'rate' (the real-world share): {br.get('name')}")
    lv = (cfg.get("meta") or {}).get("evidence_level", 0)
    if lv not in (0, 1, 2, 3, 4, 5):
        probs.append("CONFIG ERROR: meta.evidence_level must be 0, 1, 2, 3, 4 or 5")
    return probs

def resolve(node, V):
    if isinstance(node, dict):
        if "__var__" in node:
            return V[node["__var__"]]
        return {k: resolve(v, V) for k, v in node.items()}
    if isinstance(node, list):
        return [resolve(v, V) for v in node]
    return node

def sample_all(reg, n, rng, correlations):
    V = {k: sample_dist(s, n, rng) for k, s in reg.items()}
    if correlations:
        names = []
        for a, b, _ in correlations:
            for x in (a, b):
                if x not in V:
                    raise ValueError(f"correlation refers to unknown variable '{x}'. Known: {sorted(V)}")
                if x not in names:
                    names.append(x)
        k = len(names)
        C = np.eye(k)
        for a, b, r in correlations:
            i, j = names.index(a), names.index(b)
            C[i, j] = C[j, i] = float(r)
        w, Q = np.linalg.eigh(C)
        C = Q @ np.diag(np.clip(w, 1e-6, None)) @ Q.T
        d = np.sqrt(np.diag(C))
        C = C / np.outer(d, d)
        Z = rng.standard_normal((n, k)) @ np.linalg.cholesky(C).T
        for j, nm in enumerate(names):  # Iman-Conover rank reordering
            ranks = np.argsort(np.argsort(Z[:, j]))
            V[nm] = np.sort(V[nm])[ranks]
    return V

# ---------------------------------------------------------------- taxes (US 2026 / CA 2025 schedules)
FED = {
    "single": ([0, 12400, 50400, 105700, 201775, 256225, 640600], [.10, .12, .22, .24, .32, .35, .37], 16100, 200000, 201750, 75000),
    "mfj": ([0, 24800, 100800, 211400, 403550, 512450, 768700], [.10, .12, .22, .24, .32, .35, .37], 32200, 250000, 403500, 150000),
}
CA = {
    "single": ([0, 11079, 26264, 41452, 57542, 72724, 371479, 445771, 742953], [.01, .02, .04, .06, .08, .093, .103, .113, .123], 5706, 153),
    "mfj": ([0, 22158, 52528, 82904, 115084, 145448, 742958, 891542, 1485906], [.01, .02, .04, .06, .08, .093, .103, .113, .123], 11412, 306),
}
SS_WAGE_BASE = 184500.0

def bracket_tax(x, lows, rates):
    x = np.maximum(x, 0.0)
    t = np.zeros_like(x)
    for i, lo in enumerate(lows):
        hi = lows[i + 1] if i + 1 < len(lows) else np.inf
        t += rates[i] * np.clip(x - lo, 0, hi - lo)
    return t

def ca_tax(x, fs):
    lows, rates, _, credit = CA[fs]
    t = bracket_tax(x, lows, rates) + 0.01 * np.maximum(x - 1e6, 0)
    return np.maximum(t - credit, 0)

def ca_llc_fee(receipts):
    return np.select([receipts >= 5e6, receipts >= 1e6, receipts >= 5e5, receipts >= 2.5e5], [11790, 6000, 2500, 900], 0.0)

def annual_tax(b, receipts, nol, tx, frac_year):
    """b: business taxable profit for the year (n,). Returns (tax, new_nol). Tax may be negative (loss offsets other income)."""
    mode = tx.get("mode", "us_passthrough")
    z = np.zeros_like(b)
    if mode == "none":
        return z, nol
    limit = float(tx.get("nol_limit_pct", 0.8))
    if mode in ("flat", "us_c_corp"):
        pos = np.maximum(b, 0)
        use = np.minimum(nol, limit * pos)
        taxable = pos - use
        nol = nol - use + np.maximum(-b, 0)
        if mode == "flat":
            tax = float(tx.get("flat_rate", 0.25)) * taxable
        else:
            st = float(tx.get("state_corp_rate", 0.0884 if tx.get("state", "CA") == "CA" else 0.0))
            tax = 0.21 * taxable + st * taxable
            if tx.get("state", "CA") == "CA":
                tax = np.maximum(tax, 800.0 * frac_year)
        return tax, nol
    # us_passthrough: incremental owner-level tax caused by the business
    fs = tx.get("filing", "single")
    lows, rates, std, addl_thr, qbi_thr, qbi_rng = FED[fs]
    other = float(tx.get("other_income", 0.0))
    other_w2 = float(tx.get("other_w2_wages", other))
    se_base = np.maximum(b, 0) * 0.9235
    room = max(0.0, SS_WAGE_BASE - other_w2)
    se = 0.124 * np.minimum(se_base, room) + 0.029 * se_base + 0.009 * np.maximum(se_base + other_w2 - addl_thr, 0) - 0.009 * max(other_w2 - addl_thr, 0)
    use = np.where(b > 0, np.minimum(nol, limit * np.maximum(b, 0)), 0.0)
    if tx.get("losses_offset_other_income", True):
        absorb = np.where(b < 0, np.minimum(-b, other), 0.0)
    else:
        absorb = z
    b_inc = np.where(b > 0, b - use, -absorb)
    nol = nol - use + np.where(b < 0, -b - absorb, 0.0)
    agi_w = other + b_inc - 0.5 * se
    ti_w = np.maximum(agi_w - std, 0)
    qbi = np.maximum(b_inc - 0.5 * se, 0)
    qded = 0.2 * np.minimum(qbi, ti_w) if tx.get("qbi", True) else z
    if tx.get("sstb", False):
        qded = qded * (1 - np.clip((ti_w - qbi_thr) / qbi_rng, 0, 1))
    fed = bracket_tax(ti_w - qded, lows, rates) - bracket_tax(np.array([max(other - std, 0)]), lows, rates)[0]
    st_mode = tx.get("state", "CA")
    if st_mode == "CA":
        ca_std = CA[fs][2]
        state = ca_tax(np.maximum(agi_w - ca_std, 0), fs) - ca_tax(np.array([max(other - ca_std, 0)]), fs)[0]
    elif st_mode == "flat":
        state = float(tx.get("state_rate", 0.0)) * (agi_w - other)
    else:
        state = z
    llc = z
    if tx.get("ca_llc", False):
        llc = 800.0 * frac_year + ca_llc_fee(receipts)
    extra = float(tx.get("other_annual_fixed_taxes", 0.0)) * frac_year + float(tx.get("gross_receipts_tax_pct", 0.0)) * receipts
    return se + fed + state + llc + extra, nol

# ---------------------------------------------------------------- helpers
def g(d, k, default=0.0):
    if not d:
        return default
    v = d.get(k, default)
    return default if v is None else v

class Arr:
    def __init__(self, n):
        self.n = n
    def __call__(self, x):
        return np.broadcast_to(np.asarray(x, float), (self.n,)).astype(float).copy()

# ---------------------------------------------------------------- core model
def simulate(P, n, T, rng, stochastic=True, store_all=True, opt=None):
    """opt: force_fit (ignore the fit lottery), stress (multipliers: price, demand, var_cost, fixed_cost, cac; delay_months)."""
    A = Arr(n)
    opt = opt or {}
    st = opt.get("stress") or {}
    s_price, s_dem, s_vc, s_fc, s_cac = (float(st.get(k, 1.0)) for k in ("price", "demand", "var_cost", "fixed_cost", "cac"))
    S = P.get("simulation", {})
    infl = float(S.get("inflation_annual", 0.03))
    start_cal = int(S.get("start_calendar_month", 1))
    tl = P.get("timeline", {})
    dem = P.get("demand", {})
    vc = P.get("variable_costs", {})
    fin = P.get("financing", {})
    own = P.get("owner", {})
    tx = P.get("taxes", {"mode": "us_passthrough"})
    wcc = P.get("working_capital", {})
    model = dem.get("model", "funnel")

    launch = np.clip(np.rint(A(g(tl, "launch_month", 0)) + float(st.get("delay_months", 0))).astype(int), 0, T - 1)
    ramp_m = np.maximum(A(g(tl, "ramp_months", 12)), 0.5)
    ramp0 = np.clip(A(g(tl, "ramp_start_level", 0.15)), 0, 1)
    shape = tl.get("ramp_shape", "exponential")

    def ramp(rel):
        m = np.maximum(rel, 0).astype(float)
        if shape == "exponential":
            s = 1 - np.exp(-3 * m / ramp_m)
        elif shape == "linear":
            s = np.clip(m / ramp_m, 0, 1)
        else:
            x = 1 / (1 + np.exp(-10 * (m / ramp_m - 0.5)))
            x0 = 1 / (1 + math.exp(5))
            s = np.clip((x - x0) / (1 - x0), 0, 1)
        return np.where(rel >= 0, ramp0 + (1 - ramp0) * s, 0.0)

    seas = np.asarray(dem.get("seasonality") or [1.0] * 12, float)
    seas = seas / seas.mean()
    sig = A(g(dem, "monthly_volatility", 0.15))
    rho = float(g(dem, "volatility_autocorr", 0.5))
    price_inc = A(g(dem, "annual_price_increase", infl))
    disc_pct, ref_pct, bad_pct = A(g(dem, "discount_pct")), A(g(dem, "refund_pct")), A(g(dem, "bad_debt_pct"))
    subscription = bool(dem.get("subscription", False))
    repeat_seasonal = bool(dem.get("repeat_seasonal", not subscription))

    # variable costs
    cogs_pct, cpu, cpo = A(g(vc, "cogs_pct")), A(g(vc, "cost_per_unit")), A(g(vc, "cost_per_order"))
    shrink_pct = A(g(vc, "shrinkage_pct_of_cogs"))
    pay_pct, pay_fix = A(g(vc, "payment_fee_pct", 0.029)), A(g(vc, "payment_fee_fixed", 0.30))
    card_share = A(g(vc, "card_share", 1.0))
    plat_pct, plat_fix = A(g(vc, "platform_fee_pct")), A(g(vc, "platform_fee_per_order"))
    comm_pct = A(g(vc, "sales_commission_pct"))
    vc_infl = A(g(vc, "annual_cost_increase", infl))

    # staff
    roles = []
    for r in P.get("staff", []) or []:
        wage = A(r["monthly_cost"]) if "monthly_cost" in r else A(g(r, "hourly_wage")) * A(g(r, "hours_per_month", 173))
        roles.append(dict(spec=r, name=r.get("name", r.get("role", "staff")), wage=wage, count=A(g(r, "count", 1)),
                          burden=A(g(r, "burden_pct", 0.25)), raise_=A(g(r, "annual_raise", infl)),
                          start=A(g(r, "start", 0)), end=A(g(r, "end", 10 ** 6)), direct=bool(r.get("direct", False)),
                          recruit=A(g(r, "recruiting_cost", 0)), hired=np.zeros(n, bool), prev=np.zeros(n)))
    role_by_name = {r["name"]: r for r in roles}

    fixed = []
    for f in P.get("fixed_costs", []) or []:
        fixed.append(dict(cat=f.get("category", "other"), m=A(g(f, "monthly")), pct=A(g(f, "pct_of_revenue")),
                          start=A(g(f, "start", 0)), end=A(g(f, "end", 10 ** 6)), inc=A(g(f, "annual_increase", infl))))
    onetime = [dict(amt=A(g(o, "amount")), mon=np.rint(A(g(o, "month", -1))).astype(int), expense=bool(o.get("expense", True)),
                    refundable=bool(o.get("refundable_at_close", False))) for o in P.get("one_time_costs", []) or []]
    capex = [dict(amt=A(g(c, "amount")), mon=np.rint(A(g(c, "month", -1))).astype(int), life=np.maximum(np.rint(A(g(c, "life_months", 60))), 1).astype(int),
                  replace=bool(c.get("replace", True)), salvage=A(g(c, "salvage_pct", 0.2))) for c in P.get("capex", []) or []]
    maint_pct = A(g(P, "maintenance_pct_of_capex_annual", 0.0))

    channels = []
    if model == "funnel":
        for c in dem.get("channels", []) or []:
            cac = A(c["cac"]) if "cac" in c else A(g(c, "cpc")) / np.maximum(A(g(c, "conversion", 0.02)), 1e-9)
            channels.append(dict(budget=A(g(c, "monthly_budget")), cac=cac, start=A(g(c, "start", 0)),
                                 ref=A(g(c, "ref_budget", 0)), exp_=A(g(c, "diminishing_exponent", 0.3)),
                                 pct=A(g(c, "pct_of_revenue", 0)), mn=A(g(c, "min_budget", 0)), mx=A(g(c, "max_budget", 1e12)),
                                 grow=A(g(c, "annual_budget_growth", 0))))
    early_cac = A(g(dem, "early_cac_multiplier", 1.3))
    reality = A(g(dem, "reality_factor", 1.0)) * A(g(dem, "panel_factor", 1.0)) * A(g(dem, "outside_view_factor", 1.0)) * s_dem
    fit = np.ones(n, bool)  # does the product find real demand? (new products: usually not)
    if dem.get("fit_probability") is not None:
        fp = A(dem["fit_probability"])
        if not opt.get("force_fit"):
            fit = (rng.random(n) < fp) if stochastic else (fp >= 0.5)
        reality = reality * np.where(fit, 1.0, A(g(dem, "no_fit_factor", 0.12)))
    if dem.get("price_elasticity") is not None:  # demand moves when price differs from the price the research was done at
        p0 = A(g(dem, "price_per_unit" if model == "traffic" else "first_purchase_value"))
        refp = A(dem["reference_price"])
        reality = reality * (np.maximum(p0, 1e-9) / np.maximum(refp, 1e-9)) ** A(dem["price_elasticity"])

    shocks = []
    for s in P.get("shocks", []) or []:
        shocks.append(dict(type=s.get("type", "cost"), p=A(g(s, "monthly_prob", 0)), amt=A(g(s, "amount", 0)), pct=A(g(s, "pct", 0)),
                           dur=A(g(s, "duration_months", 3)), rem=np.zeros(n)))

    def capacity_of(spec, counts):
        if spec is None:
            return None
        if isinstance(spec, dict) and "from_staff" in spec:
            c = counts.get(spec["from_staff"], np.zeros(n))
            return c * A(g(spec, "per_staff")) + A(g(spec, "base", 0))
        return A(spec)

    loans = [dict(amt=A(g(l, "amount")), rate=A(g(l, "annual_rate", 0.1)), term=np.maximum(np.rint(A(g(l, "term_months", 60))), 1),
                  io=np.rint(A(g(l, "interest_only_months", 0))), mon=np.rint(A(g(l, "month", -1))).astype(int),
                  bal=np.zeros(n), pay=np.zeros(n), since=np.full(n, -1.0)) for l in fin.get("loans", []) or []]
    cl_limit, cl_rate = A(g(fin.get("credit_line", {}), "limit", 0)), A(g(fin.get("credit_line", {}), "annual_rate", 0.24))
    buffer = A(g(fin, "min_cash_buffer", 0))
    draw_target = A(g(fin, "owner_draw_monthly", 0))
    draw_start = A(g(fin, "owner_draw_start", 0))
    draw_if_profit = bool(fin.get("owner_draw_only_if_profitable", True))
    owner_salary = A(g(own, "salary_monthly", 0))
    owner_salary_burden = A(g(own, "salary_burden_pct", 0.0765))
    hours = A(g(own, "hours_per_week", 40))
    opp_wage = A(g(own, "opportunity_wage_hourly", 25))
    quit = P.get("quit_rule", {}) or {}
    q_months = A(quit["max_months_without_profit"]) if quit.get("max_months_without_profit") is not None else None
    q_loss = A(quit["max_total_loss"]) if quit.get("max_total_loss") is not None else None
    q_hourly = A(quit["min_owner_hourly"]) if quit.get("min_owner_hourly") is not None else None
    q_hourly_after = A(g(quit, "min_owner_hourly_after_months", 24))
    q_prob = np.clip(A(g(quit, "monthly_quit_prob", 0.2)), 0, 1.0)      # chance per month of closing while the business loses money past the owner's patience
    q_low = np.clip(A(g(quit, "low_pay_quit_prob", 0.03)), 0, 1.0)       # chance per month of closing while it pays the owner too little (people hang on for years)
    exo_m = 1 - (1 - np.clip(A(g(quit, "exogenous_exit_annual", 0.03)), 0, 0.99)) ** (1 / 12)  # life events, moves, better offers
    q_streak = np.zeros(n)
    exi = P.get("exit", {}) or {}
    forced_sale = np.zeros(n)
    ar_coll, inv_rec = float(g(P.get("liquidation", {}), "ar_collect_pct", 0.9)), float(g(P.get("liquidation", {}), "inventory_recovery_pct", 0.3))

    # state
    cash = A(g(fin, "founder_cash", 0))
    equity_in = cash.copy()
    reserve = A(g(fin, "founder_reserve", 0))
    funds_total = equity_in + reserve + cl_limit + sum((l["amt"] for l in loans), np.zeros(n))
    reserve_used = np.zeros(n)
    credit = np.zeros(n)
    alive = np.ones(n, bool)
    end_month = np.full(n, -1)
    end_reason = np.zeros(n, int)  # 0 running, 1 ran out of cash, 2 quit
    young = np.zeros(n); mature = np.zeros(n); ever = np.zeros(n)
    eps = np.zeros(n)
    nol = np.zeros(n)
    wc_prev = np.zeros(n)
    ar_prev = np.zeros(n); inv_prev = np.zeros(n); ap_prev = np.zeros(n)
    tax_due = np.zeros(n)
    ebt_year = np.zeros(n); rec_year = np.zeros(n); alive_year_start = np.ones(n, bool)
    hist = {k: np.zeros((T, n)) for k in ("revenue", "units", "active", "transactions", "ebitda", "profit_pre_tax", "demand", "interest")}
    draws_total = np.zeros(n)

    keys = ["gross_revenue", "discounts", "refunds", "net_revenue", "units", "transactions", "new_customers", "active_customers",
            "lost_demand", "capacity", "cogs_goods", "shrinkage", "payment_fees", "platform_fees", "commissions", "direct_labor",
            "cogs_total", "gross_profit", "marketing", "indirect_labor", "fixed_costs", "rent", "owner_salary", "maintenance",
            "bad_debt", "startup_expenses", "shock_costs", "recruiting", "opex_total", "ebitda", "depreciation", "interest",
            "ebt", "tax_expense", "net_income", "tax_paid", "delta_wc", "capex", "deposits", "fcf", "loan_flow", "owner_draw",
            "injection", "credit_balance", "loan_balance", "cash", "alive", "staff_fte"]
    R = {k: np.zeros((n, T), np.float32) for k in keys}

    for t in range(T):
        rel = t - launch
        live = (rel >= 0) & alive
        yr_abs = t // 12
        relyr = np.maximum(rel, 0) // 12
        pf = (1 + price_inc) ** relyr
        season = seas[(start_cal - 1 + t) % 12]
        if stochastic:
            eps = rho * eps + math.sqrt(1 - rho ** 2) * rng.standard_normal(n)
            noise = np.exp(sig * eps - sig ** 2 / 2)
        else:
            noise = np.ones(n)
        r = ramp(rel)

        # shocks
        rev_mult = np.ones(n); shock_cost = np.zeros(n); cust_loss = np.zeros(n)
        for s in shocks:
            if stochastic:
                hit = (rng.random(n) < s["p"]) & live
                if s["type"] == "cost":
                    shock_cost += hit * s["amt"]
                elif s["type"] == "revenue_hit":
                    s["rem"] = np.where(hit, s["dur"], np.maximum(s["rem"] - 1, 0))
                    rev_mult *= 1 - s["pct"] * (s["rem"] > 0)
                elif s["type"] == "customer_loss":
                    cust_loss = 1 - (1 - cust_loss) * (1 - hit * s["pct"])
            else:
                if s["type"] == "cost":
                    shock_cost += live * s["p"] * s["amt"]
                elif s["type"] == "revenue_hit":
                    rev_mult *= 1 - live * s["pct"] * np.minimum(1, s["p"] * s["dur"])
                elif s["type"] == "customer_loss":
                    cust_loss = 1 - (1 - cust_loss) * (1 - live * s["p"] * s["pct"])

        # staff (based on last month's metrics)
        lagm = {k: (hist[k][t - 1] if t >= 1 else np.zeros(n)) for k in hist}
        counts = {}; direct_lab = np.zeros(n); indirect_lab = np.zeros(n); recruiting = np.zeros(n); fte = np.zeros(n)
        for ro in roles:
            sp = ro["spec"]
            act = (rel >= ro["start"]) & (rel < ro["end"]) & alive
            if "trigger" in sp:
                tr = sp["trigger"]
                lag = int(g(tr, "lag_months", 1))
                m = hist[tr.get("metric", "revenue")][t - lag] if t - lag >= 0 else np.zeros(n)
                ro["hired"] |= (m >= float(tr["threshold"])) & live
                act &= ro["hired"]
            if "scaling" in sp:
                sc = sp["scaling"]
                lag = int(g(sc, "lag_months", 1))
                m = hist[sc.get("metric", "demand")][t - lag] if t - lag >= 0 else np.zeros(n)
                raw = np.maximum(m - float(g(sc, "offset", 0)), 0) / float(sc["per_staff"])  # offset = volume the owner handles alone
                if not sc.get("fractional", False):
                    raw = np.ceil(raw - 1e-9)  # whole people; fractional=true means part-time hours that follow demand
                cnt = np.clip(raw, float(g(sc, "min", 0)), float(g(sc, "max", 1e6)))
            else:
                cnt = ro["count"].copy()
            cnt = cnt * act
            counts[ro["name"]] = cnt
            cost = cnt * ro["wage"] * (1 + ro["burden"]) * (1 + ro["raise_"]) ** yr_abs
            recruiting += np.maximum(cnt - ro["prev"], 0) * ro["recruit"]
            ro["prev"] = cnt
            fte += cnt
            if ro["direct"]:
                direct_lab += cost
            else:
                indirect_lab += cost

        # demand
        new = np.zeros(n); lost = np.zeros(n); active_c = np.zeros(n); mkt = np.zeros(n)
        if model == "traffic":
            lam = A(g(dem, "mature_units_per_month")) * reality * r * season * noise * rev_mult * live
            units_d = rng.poisson(np.maximum(lam, 0)).astype(float) if stochastic else lam
            cap = capacity_of(dem.get("capacity_units_per_month"), counts)
            units = np.minimum(units_d, cap) if cap is not None else units_d
            lost = units_d - units
            transactions = units / np.maximum(A(g(dem, "units_per_transaction", 1)), 1e-9)
            gross = units * A(g(dem, "price_per_unit")) * pf
            capv = cap if cap is not None else np.zeros(n)
            demand_now = units_d
        else:
            paid = np.zeros(n)
            prev_rev = hist["revenue"][t - 1] if t >= 1 else np.zeros(n)
            eff_mult = 1 + (early_cac - 1) * (1 - np.where(rel >= 0, ramp(rel), 0))
            for c in channels:
                on = (rel >= c["start"]) & alive
                bud = np.where(c["pct"] > 0, np.clip(c["pct"] * prev_rev, c["mn"], c["mx"]), c["budget"] * (1 + c["grow"]) ** relyr)
                bud = bud * on
                cac = c["cac"] * eff_mult * (1 + infl) ** yr_abs * s_cac
                cac = np.where((c["ref"] > 0) & (bud > c["ref"]), cac * (bud / np.maximum(c["ref"], 1e-9)) ** c["exp_"], cac)
                paid += np.where(live, bud / np.maximum(cac, 1e-9), 0)
                mkt += bud
            organic = A(g(dem, "organic_new_per_month")) * r
            existing0 = young + mature
            referral = A(g(dem, "referral_rate")) * existing0
            lam = (paid + organic) * reality * season * noise * rev_mult + referral * rev_mult
            msize = dem.get("market_size_customers")
            if msize is not None:
                basis = ever if dem.get("saturation_basis", "active") == "ever" else existing0
                lam = lam * np.clip(1 - basis / np.maximum(A(msize), 1), 0, 1)
            lam = np.maximum(lam, 0) * live
            new = rng.poisson(lam).astype(float) if stochastic else lam
            ec, ch = A(g(dem, "early_churn_monthly", g(dem, "churn_monthly", 0.05))), A(g(dem, "churn_monthly", 0.05))
            k = np.maximum(A(g(dem, "early_period_months", 3)), 1)
            if stochastic:
                ys = rng.binomial(young.astype(np.int64), np.clip(1 - ec, 0, 1)).astype(float)
                ms = rng.binomial(mature.astype(np.int64), np.clip(1 - ch, 0, 1)).astype(float)
                grad = rng.binomial(ys.astype(np.int64), np.clip(1 / k, 0, 1)).astype(float)
                if np.any(cust_loss > 0):
                    ys = rng.binomial(ys.astype(np.int64), 1 - cust_loss).astype(float)
                    grad = np.minimum(grad, ys)
                    ms = rng.binomial(ms.astype(np.int64), 1 - cust_loss).astype(float)
            else:
                ys = young * (1 - ec) * (1 - cust_loss); ms = mature * (1 - ch) * (1 - cust_loss); grad = ys / k
            ys = ys * alive; ms = ms * alive; grad = grad * alive
            existing = ys + ms
            cap = capacity_of(dem.get("capacity_active_customers"), counts)
            if cap is not None:
                room = np.maximum(cap - existing, 0)
                lost = np.maximum(new - room, 0)
                new = np.minimum(new, room)
            young = ys - grad + new
            mature = ms + grad
            ever += new
            active_c = young + mature
            rep_orders = existing * A(g(dem, "repeat_orders_per_active_month", 1.0)) * (season if repeat_seasonal else 1.0)
            if not subscription:
                rep_orders = rep_orders * rev_mult * (noise if stochastic else 1.0)
            transactions = new + rep_orders
            units = transactions
            gross = (new * A(g(dem, "first_purchase_value")) + rep_orders * A(g(dem, "repeat_order_value", g(dem, "first_purchase_value")))) * pf
            capv = cap if cap is not None else np.zeros(n)
            demand_now = active_c + lost

        gross = gross * alive * s_price
        discounts = gross * disc_pct
        after = gross - discounts
        refunds = after * ref_pct
        net = after - refunds

        vcf = (1 + vc_infl) ** yr_abs
        cogs_goods = (net * cogs_pct + (units * cpu + transactions * cpo) * vcf) * s_vc
        shrink = cogs_goods * shrink_pct
        pay_fees = card_share * (after * pay_pct + transactions * pay_fix)
        plat = net * plat_pct + transactions * plat_fix
        comm = net * comm_pct
        cogs_total = cogs_goods + shrink + pay_fees + plat + comm + direct_lab
        gp = net - cogs_total

        fixed_tot = np.zeros(n); rent = np.zeros(n)
        for f in fixed:
            act = (rel >= f["start"]) & (rel < f["end"]) & alive
            c = (f["m"] * (1 + f["inc"]) ** yr_abs + f["pct"] * net) * act * s_fc
            fixed_tot += c
            if f["cat"] == "rent":
                rent += c
        startup = np.zeros(n); deposits = np.zeros(n); capex_cash = np.zeros(n); dep = np.zeros(n); maint_base = np.zeros(n)
        for o in onetime:
            hit = (rel == np.maximum(o["mon"], -launch)) & alive
            if o["expense"]:
                startup += hit * o["amt"]
            else:
                deposits += hit * o["amt"]
        for c in capex:
            since = rel - np.maximum(c["mon"], -launch)
            owned = (since >= 0) & alive & (c["replace"] | (since < c["life"]))
            buy = alive & (since >= 0) & ((since == 0) | (c["replace"] & (since % c["life"] == 0)))
            capex_cash += buy * c["amt"] * (1 + infl) ** yr_abs
            dep += owned * c["amt"] / c["life"]
            maint_base += owned * c["amt"]
        maint = maint_base * maint_pct / 12
        bad = net * bad_pct
        osal = owner_salary * (1 + owner_salary_burden) * live
        opex = mkt + indirect_lab + fixed_tot + osal + maint + bad + startup + shock_cost + recruiting
        ebitda = gp - opex

        # debt
        interest = np.zeros(n); principal = np.zeros(n); disb = np.zeros(n)
        for l in loans:
            hitd = (rel == np.maximum(l["mon"], -launch)) & alive & (l["since"] < 0)
            l["bal"] = np.where(hitd, l["amt"], l["bal"])
            disb += hitd * l["amt"]
            l["since"] = np.where(hitd, 0, l["since"])
            rm = l["rate"] / 12
            nam = np.maximum(l["term"] - l["io"], 1)
            pay = np.where(rm > 0, l["amt"] * rm / (1 - (1 + rm) ** (-nam)), l["amt"] / nam)
            active_l = (l["since"] >= 1) & alive
            it = l["bal"] * rm * active_l
            pr = np.where(l["since"] > l["io"], np.minimum(l["bal"], np.maximum(pay - it, 0)), 0) * active_l
            l["bal"] = l["bal"] - pr
            interest += it; principal += pr
            l["since"] = np.where(l["since"] >= 0, l["since"] + 1, l["since"])
        interest += credit * cl_rate / 12 * alive
        ebt = ebitda - dep - interest

        # working capital
        ar = net * A(g(wcc, "dso_days", 0)) / 30
        inv = np.maximum(cogs_goods * A(g(wcc, "inventory_days", 0)) / 30, A(g(wcc, "min_inventory", 0)) * live)
        ap = (cogs_goods + fixed_tot * float(g(wcc, "fixed_costs_on_terms", 0))) * A(g(wcc, "dpo_days", 0)) / 30
        wc = (ar + inv - ap) * alive
        dwc = (wc - wc_prev) * alive
        wc_prev = np.where(alive, wc, wc_prev)
        ar_prev, inv_prev, ap_prev = np.where(alive, ar, ar_prev), np.where(alive, inv, inv_prev), np.where(alive, ap, ap_prev)

        # taxes (annual, owner level for pass-through)
        tax_paid = tax_due * alive
        tax_due = np.zeros(n)
        ebt_year += ebt * alive
        rec_year += net * alive
        tax_exp = np.zeros(n)
        if t % 12 == 11 or t == T - 1:
            frac = ((t % 12) + 1) / 12
            tx_y, nol = annual_tax(ebt_year, rec_year, nol, tx, frac)
            tx_y = tx_y * alive_year_start
            tax_exp = tx_y
            if t == T - 1:
                tax_paid = tax_paid + tx_y * alive
            else:
                tax_due = tx_y
            ebt_year = np.zeros(n); rec_year = np.zeros(n)

        net_income = ebt - tax_exp
        fcf = (ebitda - interest - tax_paid - dwc - capex_cash - deposits) * alive
        loan_flow = (disb - principal) * alive

        # owner draws
        hist["ebitda"][t] = ebitda
        hist["profit_pre_tax"][t] = ebt
        hist["interest"][t] = interest
        if t >= 2:
            trail = (hist["profit_pre_tax"][t] + hist["profit_pre_tax"][t - 1] + hist["profit_pre_tax"][t - 2]) / 3
        else:
            trail = ebt
        avail = cash + fcf + loan_flow - buffer
        want = draw_target * (live & (rel >= draw_start))
        if draw_if_profit:
            want = np.minimum(want, np.maximum(trail, 0))
        draw = np.clip(np.minimum(want, avail), 0, None) * alive
        draws_total += draw

        cash = cash + fcf + loan_flow - draw
        inj = np.where(alive & (cash < buffer), np.minimum(reserve, buffer - cash), 0)
        inj = np.maximum(inj, 0)
        reserve -= inj; reserve_used += inj; cash += inj
        need = np.where(alive & (cash < 0), -cash, 0)
        cdraw = np.minimum(need, np.maximum(cl_limit - credit, 0))
        credit += cdraw; cash += cdraw
        repay = np.where(alive & (credit > 0) & (cash > buffer), np.minimum(credit, cash - buffer), 0)
        credit -= repay; cash -= repay

        # failure / quitting
        failed = alive & (cash < -1e-6)
        haz = np.zeros(n)  # the owner's monthly chance of giving up; real owners do not all quit in the same month
        if q_hourly is not None and t >= 11:
            earn12 = (hist["ebitda"][t - 11:t + 1].sum(axis=0) - hist["interest"][t - 11:t].sum(axis=0) - interest + 12 * owner_salary)
            hourly_now = earn12 / np.maximum(hours * 52, 1)
            haz = np.where((rel >= q_hourly_after) & (hourly_now < q_hourly), q_low, haz)
        if q_months is not None and t >= 6:
            trail6 = hist["ebitda"][t - 5:t + 1].sum(axis=0)
            haz = np.where((rel >= q_months) & (trail6 <= 0), q_prob, haz)
        haz = haz * alive
        if stochastic:
            quit_now = (rng.random(n) < haz) if haz.max() > 0 else np.zeros(n, bool)
        else:  # no dice in the typical-path runs: close once the monthly chances add up to 1
            q_streak = np.where(haz > 0, q_streak + haz, 0)
            quit_now = q_streak >= 1 - 1e-9
        if q_loss is not None:  # founder's money lost = money in + debts - money out - business assets (cash and working capital)
            debt_now = credit + sum((l["bal"] for l in loans), np.zeros(n))
            lostmoney = equity_in + reserve_used + debt_now - draws_total - cash - wc_prev
            quit_now = quit_now | (alive & (lostmoney >= q_loss))
        quit_now = quit_now & ~failed
        exo = np.zeros(n, bool)
        if stochastic and exo_m.max() > 0:
            exo = live & ~failed & ~quit_now & (rng.random(n) < exo_m)
        closing = failed | quit_now | exo
        if np.any(closing):
            salv = np.zeros(n)
            for c in capex:
                since = rel - np.maximum(c["mon"], -launch)
                owned = (since >= 0) & (c["replace"] | (since < c["life"]))
                age = np.where(c["replace"], since % c["life"], since)
                book = np.clip(c["amt"] * (1 - age / c["life"]), 0, None) * owned
                salv += book * c["salvage"]
            dep_back = np.zeros(n)
            for o in onetime:
                if (not o["expense"]) and o["refundable"]:
                    dep_back += (rel >= np.maximum(o["mon"], -launch)) * o["amt"]
            liq = ar_prev * ar_coll + inv_prev * inv_rec - ap_prev + salv + dep_back
            cash = np.where(closing, cash + liq, cash)
            if exo.any() and exi.get("count_exit_value", True):  # a working business that closes for outside reasons is often sold cheaply
                lo = max(0, t - 11)
                if exi.get("basis", "sde") == "revenue":
                    val = (hist["revenue"][lo:t].sum(axis=0) + net) * A(g(exi, "revenue_multiple", 0.6))
                else:
                    val = np.maximum(hist["ebitda"][lo:t + 1].sum(axis=0) + osal * (t + 1 - lo), 0) * A(g(exi, "sde_multiple", 2.5))
                forced_sale = np.where(exo, val * float(g(exi, "forced_sale_fraction", 0.5)), forced_sale)
            end_month = np.where(closing & (end_month < 0), t, end_month)
            end_reason = np.where(failed, 1, np.where(quit_now, 2, np.where(exo, 3, end_reason)))
            alive = alive & ~closing

        if t % 12 == 11:
            alive_year_start = alive.copy()
        hist["revenue"][t] = net
        hist["demand"][t] = demand_now
        hist["units"][t] = units
        hist["active"][t] = active_c
        hist["transactions"][t] = transactions
        loanbal = sum((l["bal"] for l in loans), np.zeros(n))
        row = dict(gross_revenue=gross, discounts=discounts, refunds=refunds, net_revenue=net, units=units, transactions=transactions,
                   new_customers=new, active_customers=active_c, lost_demand=lost, capacity=capv, cogs_goods=cogs_goods,
                   shrinkage=shrink, payment_fees=pay_fees, platform_fees=plat, commissions=comm, direct_labor=direct_lab,
                   cogs_total=cogs_total, gross_profit=gp, marketing=mkt, indirect_labor=indirect_lab, fixed_costs=fixed_tot,
                   rent=rent, owner_salary=osal, maintenance=maint, bad_debt=bad, startup_expenses=startup, shock_costs=shock_cost,
                   recruiting=recruiting, opex_total=opex, ebitda=ebitda, depreciation=dep, interest=interest, ebt=ebt,
                   tax_expense=tax_exp, net_income=net_income, tax_paid=tax_paid, delta_wc=dwc, capex=capex_cash, deposits=deposits,
                   fcf=fcf, loan_flow=loan_flow, owner_draw=draw, injection=inj, credit_balance=credit, loan_balance=loanbal,
                   cash=cash, alive=alive.astype(float), staff_fte=fte)
        for k_, v_ in row.items():
            R[k_][:, t] = v_

    out = dict(R=R, launch=launch, end_month=end_month, end_reason=end_reason, equity_in=equity_in,
               reserve_used=reserve_used, credit=credit, alive=alive, draws_total=draws_total, hours=hours, opp_wage=opp_wage,
               loans_left=sum((l["bal"] for l in loans), np.zeros(n)), fit=fit, forced_sale=forced_sale, funds_total=funds_total)
    return out

# ---------------------------------------------------------------- metrics
def first_true(mask):
    any_ = mask.any(axis=1)
    idx = mask.argmax(axis=1).astype(float)
    idx[~any_] = np.nan
    return idx

def metrics(out, P):
    R = out["R"]
    n, T = R["cash"].shape
    S = P.get("simulation", {})
    ex = P.get("exit", {}) or {}
    own = P.get("owner", {}) or {}
    ebitda = R["ebitda"].astype(float)
    alive_m = R["alive"] > 0.5
    pos = (ebitda > 0) & alive_m
    run3 = pos[:, :-2] & pos[:, 1:-1] & pos[:, 2:] if T >= 3 else pos
    be = first_true(run3)
    opp_month = out["opp_wage"] * out["hours"] * 52 / 12
    econ = ((ebitda - opp_month[:, None]) > 0) & alive_m
    run3e = econ[:, :-2] & econ[:, 1:-1] & econ[:, 2:] if T >= 3 else econ
    be_econ = first_true(run3e)
    cumf = np.cumsum(R["fcf"].astype(float), axis=1)
    cumfd = np.cumsum((R["fcf"] - R["owner_draw"]).astype(float), axis=1)
    peak_need = np.maximum(-cumf.min(axis=1), 0)
    peak_need_draws = np.maximum(-cumfd.min(axis=1), 0)
    neg_seen = np.maximum.accumulate(cumf < 0, axis=1)
    payback = first_true(neg_seen & (cumf >= 0) & alive_m)
    # terminal value: what the founder owns at the end
    last12_sde = (R["ebitda"][:, -12:] + R["owner_salary"][:, -12:]).sum(axis=1).astype(float)
    basis = ex.get("basis", "sde")
    if basis == "revenue":
        biz_val = R["net_revenue"][:, -12:].sum(axis=1).astype(float) * np.broadcast_to(np.asarray(ex.get("revenue_multiple", 0.6), float), (n,))
    else:
        biz_val = np.maximum(last12_sde, 0) * np.broadcast_to(np.asarray(ex.get("sde_multiple", 2.5), float), (n,))
    if not ex.get("count_exit_value", True):
        biz_val = np.zeros(n)
    alive_end = out["alive"]
    cash_end = R["cash"][:, -1].astype(float)
    debts = out["loans_left"] + out["credit"]
    terminal = np.where(alive_end, biz_val, 0.0) + out["forced_sale"] + cash_end - debts
    money_in = out["equity_in"] + out["reserve_used"]
    founder_net = out["draws_total"] + terminal - money_in
    founder_net_no_exit = out["draws_total"] + cash_end - debts - money_in
    r_ann = float(S.get("discount_rate_annual", 0.15))
    rm = (1 + r_ann) ** (1 / 12) - 1
    disc = (1 + rm) ** -np.arange(T)
    flows = (R["owner_draw"] - R["injection"]).astype(float)
    npv = -out["equity_in"] + (flows * disc).sum(axis=1) + terminal * disc[-1]
    months_active = np.where(out["end_month"] >= 0, out["end_month"] + 1, T)
    job_tax = float(own.get("job_effective_tax_rate", 0.22))
    job_alt = opp_month * months_active * (1 - job_tax)  # what a job would have paid during the months spent on the business
    years = T // 12
    ann = {}
    for k in ("net_revenue", "gross_profit", "ebitda", "net_income", "owner_draw", "fcf", "marketing", "tax_expense"):
        ann[k] = np.stack([R[k][:, 12 * y:12 * (y + 1)].sum(axis=1) for y in range(years)], axis=1).astype(float)
    ann["cash_end"] = np.stack([R["cash"][:, 12 * (y + 1) - 1] for y in range(years)], axis=1).astype(float)
    ann["active_customers_end"] = np.stack([R["active_customers"][:, 12 * (y + 1) - 1] for y in range(years)], axis=1).astype(float)
    ann["alive_end"] = np.stack([R["alive"][:, 12 * (y + 1) - 1] for y in range(years)], axis=1).astype(float)
    owner_income = ann["net_income"] + np.stack([R["owner_salary"][:, 12 * y:12 * (y + 1)].sum(axis=1) for y in range(years)], axis=1)
    hourly = owner_income / np.maximum(out["hours"][:, None] * 52, 1)
    full_year = np.stack([(out["end_month"] < 0) | (out["end_month"] >= 12 * (y + 1)) for y in range(years)], axis=1)
    hourly = np.where(full_year, hourly, np.nan)  # only runs that operated the whole year
    return dict(breakeven=be, breakeven_after_launch=be - out["launch"], breakeven_econ=be_econ, payback=payback,
                peak_need=peak_need, peak_need_draws=peak_need_draws, terminal=terminal, founder_net=founder_net,
                founder_net_no_exit=founder_net_no_exit, npv=npv, money_in=money_in, job_alt=job_alt, ann=ann, hourly=hourly,
                end_month=out["end_month"], end_reason=out["end_reason"], alive_end=alive_end, biz_val=biz_val)

def pct(x, qs=(10, 25, 50, 75, 90)):
    x = np.asarray(x, float)
    x = x[~np.isnan(x)]
    if x.size == 0:
        return {f"p{q}": None for q in qs}
    return {f"p{q}": float(np.percentile(x, q)) for q in qs}

# ---------------------------------------------------------------- sanity checks
def pick_sanity_path(M, T, reps):
    """Mature economics: median survivor if enough survive, else the 12 operating months before the typical run closed."""
    surv = np.where(M["alive_end"])[0]
    if surv.size >= max(20, 0.05 * M["alive_end"].size):
        fn = M["founder_net"][surv]
        i = int(surv[np.argsort(fn)[surv.size // 2]])
        return i, slice(T - 12, T), "median surviving run, last 12 months"
    i = reps["p50"]
    end = int(M["end_month"][i]) + 1 if M["end_month"][i] >= 0 else T
    return i, slice(max(0, end - 12), end), "typical run, last 12 months before it closed (few runs survive)"

def sanity(R, M, P, i, sl):
    """Compare mature-year ratios with researched benchmarks + generic red flags."""
    W = []
    B = P.get("benchmarks", {}) or {}
    def s(k):
        return float(R[k][i, sl].sum())
    rev = s("net_revenue")
    ratios = {}
    if rev > 0:
        labor = s("direct_labor") + s("indirect_labor")
        ratios = dict(gross_margin=s("gross_profit") / rev, ebitda_margin=s("ebitda") / rev, net_margin=s("net_income") / rev,
                      marketing_pct=s("marketing") / rev, labor_pct=labor / rev, rent_pct=s("rent") / rev,
                      prime_cost_pct=(s("cogs_goods") + labor) / rev, payment_fee_pct=s("payment_fees") / rev,
                      mature_annual_revenue=rev, revenue_per_fte=rev / max(float(R["staff_fte"][i, sl].mean()) + 1, 1))
        act = R["active_customers"][i, sl].astype(float)
        newc = R["new_customers"][i, sl].astype(float)
        if act.mean() > 0:
            churned = (act[:-1] + newc[1:] - act[1:]).clip(min=0).sum()
            churn = churned / max(act[:-1].sum(), 1)
            gp_per_cust_month = s("gross_profit") / max(act.sum(), 1)
            cac = s("marketing") / max(newc.sum(), 1)
            ratios.update(monthly_churn=churn, blended_cac=cac, gp_per_customer_month=gp_per_cust_month,
                          ltv_cac=(gp_per_cust_month / max(churn, 1e-4)) / max(cac, 1e-9) if cac > 0 else None,
                          cac_payback_months=cac / max(gp_per_cust_month, 1e-9) if cac > 0 else None)
    be = np.nanmedian(M["breakeven_after_launch"]) if np.any(~np.isnan(M["breakeven_after_launch"])) else None
    ratios["median_breakeven_months_after_launch"] = None if be is None else float(be)
    for k, rng_ in B.items():
        v = ratios.get(k)
        if v is None or not isinstance(rng_, list) or len(rng_) != 2:
            continue
        lo, hi = rng_
        if v < lo or v > hi:
            W.append(f"BENCHMARK MISS: {k} = {v:,.3f} is outside the researched range [{lo}, {hi}]. Re-check inputs.")
    nm = ratios.get("net_margin")
    if nm is not None and nm > 0.35 and not P.get("meta", {}).get("software", False):
        W.append(f"RED FLAG: mature net margin {nm:.0%} is very high for a non-software small business. Costs are probably missing.")
    if be is not None and be < 4:
        W.append(f"RED FLAG: median break-even only {be:.0f} months after launch. Real small businesses rarely do this. Ramp or costs too optimistic?")
    lc = ratios.get("ltv_cac")
    if lc is not None and lc > 8:
        W.append(f"RED FLAG: LTV/CAC = {lc:.1f}. Above about 5 usually means CAC is underestimated or churn is too low.")
    ch = ratios.get("monthly_churn")
    if ch is not None and ch < 0.01 and P.get("demand", {}).get("subscription", False):
        W.append(f"RED FLAG: monthly churn {ch:.2%} is below 1%. Only strong B2B products get this. SMB/consumer is usually 3 to 8%.")
    ms = P.get("demand", {}).get("market_size_customers")
    if ms is not None:
        share = float(R["active_customers"][i, sl.stop - 1]) / max(float(np.median(np.asarray(ms, float))), 1)
        if share > 0.15:
            W.append(f"RED FLAG: ends with {share:.0%} of the whole reachable market. Very rare for a new business.")
    hrs = float(np.median(np.asarray(P.get("owner", {}).get("hours_per_week", 40), float)))
    if hrs < 20 and not P.get("staff"):
        W.append("RED FLAG: owner works under 20 h/week and there is no staff. Who does the work?")
    if "reality_factor" not in (P.get("demand") or {}) and int((P.get("meta") or {}).get("evidence_level", 0)) < 5:
        W.append("NO OPTIMISM CORRECTION: demand.reality_factor is missing. Founders usually overestimate first-year sales; add it unless demand comes from real operating history (evidence level 5).")
    cats = {f.get("category", "other") for f in P.get("fixed_costs", []) or []}
    for need in P.get("required_categories", ["insurance", "accounting", "software", "contingency"]):
        if need not in cats:
            W.append(f"MISSING COST: no fixed cost with category '{need}'. Add it or explain why it is zero.")
    return ratios, W

# ---------------------------------------------------------------- outputs
def fmt(x):
    if x is None or (isinstance(x, float) and math.isnan(x)):
        return "n/a"
    ax = abs(x)
    if ax >= 1e6:
        return f"{x/1e6:,.2f}M"
    if ax >= 1e4:
        return f"{x/1e3:,.0f}k"
    if ax >= 100:
        return f"{x:,.0f}"
    if ax >= 1:
        return f"{x:,.2f}"
    return f"{x:.3g}"

STRESS = [("base case", {}), ("price -10%, same volume", {"price": 0.9}), ("demand -20%", {"demand": 0.8}),
          ("variable costs +15%", {"var_cost": 1.15}), ("fixed costs +20%", {"fixed_cost": 1.2}),
          ("customer acquisition cost +50%", {"cac": 1.5}), ("launch 3 months late", {"delay_months": 3}),
          ("everything a bit worse", {"price": 0.95, "demand": 0.9, "var_cost": 1.07, "fixed_cost": 1.1, "cac": 1.2, "delay_months": 1})]
BLOCKING = ("UNKNOWN KEY", "UNKNOWN DIST FIELD", "BAD EVIDENCE GRADE", "RED FLAG", "OUTSIDE VIEW", "MISSING COST", "UNSOURCED INPUT",
            "BENCHMARK MISS", "NO OPTIMISM CORRECTION", "GUESS ON A KEY INPUT", "UNGRADED KEY INPUT", "CONFIG ERROR")

def no_ruin(tmpl):
    """Copy of the model where money never runs out and nobody quits: measures pure economics."""
    tm2 = json.loads(json.dumps(tmpl))
    tm2.setdefault("financing", {})["founder_reserve"] = 1e12
    tm2["quit_rule"] = {}
    return tm2

def win_sum(arr, start, length):
    """Per-run sum of arr[i, start[i] : start[i] + length]; NaN when the window does not fit in the horizon."""
    n, T = arr.shape
    cs = np.concatenate([np.zeros((n, 1)), np.cumsum(arr.astype(float), axis=1)], axis=1)
    a, b = np.clip(start, 0, T), np.clip(start + length, 0, T)
    r = np.arange(n)
    return np.where((start >= 0) & (start + length <= T), cs[r, b] - cs[r, a], np.nan)

def base_rate_table(cfg, R, M, out, T):
    """Outside view: the model's odds next to what really happens to businesses of this kind."""
    L = int((cfg.get("meta") or {}).get("evidence_level", 0))
    items = list(cfg.get("base_rates") or [])
    for yk in (1, 2, 3, 5):  # old style benchmarks.survival_year_N still works
        br = (cfg.get("benchmarks") or {}).get(f"survival_year_{yk}")
        if br is not None:
            items.append(dict(name=f"still open {yk} year(s) after opening", metric="survival", years=yk, rate=float(br), source="benchmarks"))
    launch, rows, warns = out["launch"], [], []
    for br in items:
        m, thr, nm = br.get("metric"), float(br.get("threshold", 0)), br.get("name", br.get("metric"))
        if m == "survival":
            mo = int(round(12 * float(br.get("years", 1))))
            ok = launch + mo <= T
            val = ~((M["end_month"] >= 0) & (M["end_month"] - launch < mo))
        elif m == "monthly_revenue":
            x = win_sum(R["net_revenue"], launch + int(br.get("months_after_launch", 12)) - 3, 3) / 3
            ok, val = ~np.isnan(x), x >= thr
        elif m in ("annual_revenue", "profitable_year", "owner_income"):
            st0 = launch + 12 * (int(br.get("year", 1)) - 1)
            if m == "annual_revenue":
                x = win_sum(R["net_revenue"], st0, 12); val = x >= thr
            elif m == "profitable_year":
                x = win_sum(R["net_income"], st0, 12); val = x > 0
            else:
                x = win_sum(R["net_income"] + R["owner_salary"], st0, 12); val = x >= thr
            ok = ~np.isnan(x)
        else:
            continue
        if ok.mean() < 0.8:
            rows.append(dict(name=nm, model=None, world=float(br["rate"]), allowed=None, flag="horizon too short: raise simulation.months", source=br.get("source", "")))
            continue
        model, rate, lift = float(val[ok].mean()), float(br["rate"]), LIFT[L]
        hi_good = br.get("good", "high") == "high"
        allowed = rate + (1 - rate) * lift if hi_good else rate * (1 - lift)
        flag = ""
        if (hi_good and model > allowed + 1e-9) or (not hi_good and model < allowed - 1e-9):
            flag = "too rosy"
            warns.append(f"OUTSIDE VIEW: '{nm}': model says {model:.0%}, the real-world rate is {rate:.0%}. With evidence level {L} ({EVIDENCE[L]}) the model "
                         f"may go to about {allowed:.0%}. Lower the demand inputs or fit_probability, or get stronger evidence (a real test).")
        elif (hi_good and model < rate - 0.35) or (not hi_good and model > rate + 0.35):
            flag = "much worse than typical"
            warns.append(f"PESSIMISM CHECK: '{nm}': model says {model:.0%}, the real-world rate is {rate:.0%}. Check for missing revenue or too little starting money. If the inputs are right, this plan is worse than the typical business of its kind.")
        rows.append(dict(name=nm, model=model, world=rate, allowed=float(allowed), flag=flag, source=br.get("source", "")))
    return rows, warns

def outside_view_k(cfg, Ps, ns, T, seed):
    """Reference-class correction: the demand multiplier at which the model sits closest to the real-world rates
    (no rosier than the evidence allows, no gloomier than the real rate). Each kind of check counts once."""
    def miss(k):
        o = simulate(Ps, ns, T, np.random.default_rng(seed + 202), stochastic=True, opt={"stress": {"demand": k}})
        rows, _ = base_rate_table(cfg, o["R"], metrics(o, Ps), o, T)
        grp = {}
        for r, br in zip(rows, [b for b in (cfg.get("base_rates") or [])] + [None] * 9):
            if r["model"] is None:
                continue
            lo, hi = sorted((r["world"], r["allowed"]))
            grp.setdefault(br.get("metric") if br else "survival", []).append(max(0.0, r["model"] - hi, lo - r["model"]) ** 2)
        return sum(float(np.mean(v)) for v in grp.values())
    grid = [1.0, 0.9, 0.8, 0.7, 0.6, 0.5, 0.4, 0.3, 0.2]
    sc = [miss(k) for k in grid]
    j = int(np.argmin(sc))
    if sc[0] < 0.003 or sc[0] - sc[j] < 0.15 * sc[0]:  # already close, or demand is not what separates the model from reality
        j = 0
    return grid[j], sc[0], sc[j]

def unit_econ(o, T, model):
    """Unit economics of the typical mature year (median inputs, money never runs out, last 12 months)."""
    R = o["R"]; sl = slice(max(0, T - 12), T); mths = sl.stop - sl.start
    def s(k):
        return float(R[k][0, sl].sum())
    rev, tr = s("net_revenue"), s("transactions")
    if rev <= 0 or tr <= 0:
        return None, ["RED FLAG: the typical case has no sales in the final year. Check demand inputs."]
    W = []
    var = s("cogs_goods") + s("shrinkage") + s("payment_fees") + s("platform_fees") + s("commissions") + s("direct_labor") + s("bad_debt")
    fixed = (s("marketing") + s("indirect_labor") + s("fixed_costs") + s("owner_salary") + s("maintenance") + s("depreciation") + s("interest")) / mths
    cps = (rev - var) / tr
    wage = float(o["opp_wage"][0] * o["hours"][0] * 52 / 12)
    u = dict(revenue_per_sale=rev / tr, variable_cost_per_sale=var / tr, contribution_per_sale=cps, contribution_margin=(rev - var) / rev,
             fixed_costs_per_month=fixed, owner_market_wage_per_month=wage, sales_per_month_typical=tr / mths)
    if cps <= 0:
        W.append("RED FLAG: every sale loses money before fixed costs (price is below variable cost). Volume cannot fix this.")
        return u, W
    be = fixed / cps
    u.update(breakeven_sales_per_month=be, breakeven_sales_per_day=be / 30.4, breakeven_sales_per_month_incl_owner_wage=(fixed + wage) / cps,
             safety_margin=(tr / mths) / be if be > 0 else None)
    cap = float(R["capacity"][0, sl].mean())
    act = R["active_customers"][0, sl].astype(float)
    if model == "funnel" and act.mean() > 0:
        newc = R["new_customers"][0, sl].astype(float)
        cpcm = (rev - var) / act.sum()
        churn = max((act[:-1] + newc[1:] - act[1:]).clip(min=0).sum() / max(act[:-1].sum(), 1e-9), 1e-4)
        cac = s("marketing") / newc.sum() if newc.sum() > 0 else None
        u.update(active_customers_typical=float(act.mean()), contribution_per_customer_month=cpcm, monthly_churn=churn, customer_lifetime_months=1 / churn,
                 lifetime_value=cpcm / churn, breakeven_active_customers=fixed / cpcm, max_cac_for_12_month_payback=12 * cpcm, max_cac_for_ltv_3x=cpcm / churn / 3)
        if cac:
            u.update(cac=cac, ltv_cac=cpcm / churn / cac, cac_payback_months=cac / cpcm)
        if cap > 0:
            u["breakeven_share_of_capacity"] = fixed / cpcm / cap
    elif cap > 0 and s("units") > 0:
        u["breakeven_share_of_capacity"] = be / (cap * tr / s("units"))
    if u.get("breakeven_share_of_capacity", 0) > 1:
        W.append(f"RED FLAG: break-even needs {u['breakeven_share_of_capacity']:.0%} of capacity. The business cannot break even at this size.")
    return u, W

def verdict(cfg, M, out, warns, amap, ue):
    """Computed call: GO / TEST FIRST / NO-GO. Rules are fixed here so the call cannot drift toward what the founder hopes.
    NO-GO looks at the case where demand IS found (if even that fails, stop). GO looks at all runs (needs evidence)."""
    vd = cfg.get("verdict") or {}
    L = int((cfg.get("meta") or {}).get("evidence_level", 0))
    w = float(vd.get("exit_weight", 0.5))  # share of the sale value that counts: many small businesses never find a buyer
    net = M["founder_net_no_exit"] + w * (M["founder_net"] - M["founder_net_no_exit"])
    fit = out["fit"] if out["fit"].any() else np.ones_like(out["fit"])
    lose, job = net < 0, net > M["job_alt"]
    p_lose, p_job, p_cash = float(lose.mean()), float(job.mean()), float((M["end_reason"] == 1).mean())
    f_lose, f_job, f_never = float(lose[fit].mean()), float(job[fit].mean()), float(np.isnan(M["breakeven"][fit]).mean())
    cond = " even when demand is found" if not fit.all() else ""
    g_ = lambda k, d: float(vd.get(k, d))
    blocking = [x for x in warns if x.startswith(BLOCKING) and "[accepted:" not in x]
    nogo, fail, ok = [], [], []
    if ue and ue.get("contribution_per_sale", 1) <= 0:
        nogo.append("every sale loses money before fixed costs")
    if f_lose >= g_("nogo_loss_prob", 0.60):
        nogo.append(f"the founder loses money in {f_lose:.0%} of runs{cond} (no-go line {g_('nogo_loss_prob', 0.60):.0%})")
    if f_job < g_("nogo_beats_job", 0.15):
        nogo.append(f"it beats the job alternative in only {f_job:.0%} of runs{cond} (no-go line {g_('nogo_beats_job', 0.15):.0%})")
    if f_never >= 0.5:
        nogo.append(f"{f_never:.0%} of runs never reach break-even{cond}")
    def test(bad, msg_bad, msg_ok):
        (fail if bad else ok).append(msg_bad if bad else msg_ok)
    test(p_lose > g_("go_loss_prob", 0.35), f"chance of losing money {p_lose:.0%} is above the go line {g_('go_loss_prob', 0.35):.0%}", f"chance of losing money {p_lose:.0%}")
    test(p_job < g_("go_beats_job", 0.50), f"beats the job alternative in {p_job:.0%} of runs, below the go line {g_('go_beats_job', 0.50):.0%}", f"beats the job alternative in {p_job:.0%} of runs")
    test(p_cash > g_("go_cashout_prob", 0.20), f"runs out of money in {p_cash:.0%} of runs: bring more money or shrink the plan", f"runs out of money in {p_cash:.0%} of runs")
    need = int(vd.get("go_min_evidence", 3))
    test(L < need, f"evidence is level {L} ({EVIDENCE[L]}); a go needs level {need} ({EVIDENCE[need]})", f"evidence level {L}")
    test(len(blocking) > 0, f"{len(blocking)} warning(s) not fixed or accepted", "no open warnings")
    weak = [a for a in amap[:5] if a["ev"] in ("?", "C", "D")]
    res = dict(call="NO-GO" if nogo else ("TEST FIRST" if fail else "GO"), no_go_reasons=nogo, missing_for_go=fail, passed=ok, evidence_level=L,
               exit_weight=w, loss_prob=p_lose, beats_job_prob=p_job, cashout_prob=p_cash, loss_prob_if_demand_found=f_lose, beats_job_prob_if_demand_found=f_job,
               test_first=[f"{a['variable']} (evidence {a['ev']}, swing {fmt(a['swing'])})" for a in weak])
    if not fit.all():
        res["demand_found_share"] = float(fit.mean())
        res["p50_money_lost_if_demand_not_found"] = float(-np.median(M["founder_net_no_exit"][~fit])) if (~fit).any() else None
    return res

def run(cfg, outdir, n=None, seed=None, charts=True, solve=(), sweep=()):
    os.makedirs(outdir, exist_ok=True)
    S = cfg.get("simulation", {})
    n = int(n or S.get("n_sims", 10000))
    T = int(S.get("months", 60))
    seed = int(seed if seed is not None else S.get("seed", 7))
    problems = validate(cfg)
    reg = {}
    tmpl = extract(cfg, "", reg)
    def _refs(node):
        if isinstance(node, dict):
            if "__var__" in node and node["__var__"] not in reg:
                raise ValueError(f"ref '{node['__var__']}' does not match any named input. Known: {sorted(reg)}")
            for v in node.values():
                _refs(v)
        elif isinstance(node, list):
            for v in node:
                _refs(v)
    _refs(tmpl)
    rng = np.random.default_rng(seed)
    V = sample_all(reg, n, rng, cfg.get("correlations"))
    P = resolve(tmpl, V)
    out = simulate(P, n, T, rng, stochastic=True)
    R = out["R"]
    M = metrics(out, P)
    dmodel = (cfg.get("demand") or {}).get("model", "funnel")
    has_fit = (cfg.get("demand") or {}).get("fit_probability") is not None

    # representative paths by founder_net
    order = np.argsort(M["founder_net"])
    reps = {f"p{q}": int(order[min(n - 1, int(n * q / 100))]) for q in (10, 50, 90)}

    # deterministic expected path (median inputs, no noise; assumes the product finds demand)
    Vmed = {k: np.array([float(np.median(v))]) for k, v in V.items()}
    out_med = simulate(resolve(tmpl, Vmed), 1, T, np.random.default_rng(seed), stochastic=False, opt={"force_fit": True})
    M_med = metrics(out_med, resolve(tmpl, Vmed))

    # tornado on total profit (no ruin, demand found); row 0 = all inputs at their median
    varying = [k for k, v in V.items() if np.percentile(v, 90) - np.percentile(v, 10) > 1e-12]
    m = 1 + 2 * len(varying)
    Vt = {k: np.full(m, float(np.median(v))) for k, v in V.items()}
    for j, k in enumerate(varying):
        Vt[k][1 + 2 * j] = np.percentile(V[k], 10)
        Vt[k][2 + 2 * j] = np.percentile(V[k], 90)
    tm2 = no_ruin(tmpl)
    Pt = resolve(tm2, Vt)
    ot = simulate(Pt, m, T, np.random.default_rng(seed), stochastic=False, opt={"force_fit": True})
    Mt = metrics(ot, Pt)
    y3 = min(2, T // 12 - 1)
    cum = Mt["ann"]["net_income"].sum(axis=1)
    tor = []
    for j, k in enumerate(varying):
        lo, hi = cum[1 + 2 * j], cum[2 + 2 * j]
        tor.append(dict(variable=k, input_p10=float(np.percentile(V[k], 10)), input_p90=float(np.percentile(V[k], 90)),
                        outcome_at_p10=float(lo), outcome_at_p90=float(hi), swing=float(abs(hi - lo)),
                        y3_net_income_at_p10=float(Mt["ann"]["net_income"][1 + 2 * j, y3]),
                        y3_net_income_at_p90=float(Mt["ann"]["net_income"][2 + 2 * j, y3])))
    tor.sort(key=lambda d: -d["swing"])
    ue, ue_warn = unit_econ(ot, T, dmodel)
    # spearman
    def rank(x):
        return np.argsort(np.argsort(x)).astype(float)
    sp = []
    fr = rank(M["founder_net"]); pr = rank(M["peak_need_draws"])
    for k in varying:
        rk = rank(V[k])
        sp.append(dict(variable=k, rho_founder_net=float(np.corrcoef(rk, fr)[0, 1]), rho_peak_cash_need=float(np.corrcoef(rk, pr)[0, 1])))
    sp.sort(key=lambda d: -abs(d["rho_founder_net"]))

    si, ssl, slabel = pick_sanity_path(M, T, reps)
    ratios, warns = sanity(R, M, P, si, ssl)
    warns = problems + warns + ue_warn
    ratios["_basis"] = slabel
    br_rows, br_warn = base_rate_table(cfg, R, M, out, T)
    warns += br_warn
    if has_fit and not br_rows:
        warns.append("OUTSIDE VIEW: no base_rates given. A new product needs at least one real-world rate to be checked against (for example the share of similar products that reach a revenue level).")

    # assumption audit and assumption map (impact x evidence)
    assumptions = []
    for k, s in reg.items():
        v = V[k]
        assumptions.append(dict(variable=k, dist=s["dist"], p10=float(np.percentile(v, 10)), p50=float(np.median(v)), p90=float(np.percentile(v, 90)),
                                ev=s.get("ev", "?"), source=s.get("source", ""), note=s.get("why", "")))
        if not s.get("source") and not s.get("path", "").startswith(("demand.no_fit_factor", "demand.fit_probability", "demand.price_elasticity", "quit_rule.")):
            warns.append(f"UNSOURCED INPUT: '{k}' has no source. Research it or mark it as the user's own number.")
    corr = ("demand.reality_factor", "demand.no_fit_factor", "demand.fit_probability", "demand.panel_factor", "demand.outside_view_factor")
    amap = [dict(variable=d["variable"], swing=d["swing"], ev="correction" if reg[d["variable"]].get("path", "").startswith(corr) else reg[d["variable"]].get("ev", "?"),
                 source=reg[d["variable"]].get("source", "")) for d in tor[:12]]
    for a in amap[:5]:
        if a["ev"] == "?":
            warns.append(f"UNGRADED KEY INPUT: '{a['variable']}' is a top driver and has no evidence grade. Add ev: A (measured or quoted), B (published benchmark for this segment), C (analog or synthetic panel) or D (guess).")
        elif a["ev"] == "D":
            warns.append(f"GUESS ON A KEY INPUT: '{a['variable']}' is a top driver and is only a guess. Research it or test it for real before trusting the result.")
    acc = (cfg.get("meta") or {}).get("accepted_warnings") or []
    for j, w in enumerate(warns):
        for a in acc:
            if a.get("match") and a["match"] in w:
                warns[j] = w + f" [accepted: {a.get('why', 'no reason given')}]"
                break

    # stress tests: same inputs, one thing worse
    ns = min(n, 3000)
    Ps = resolve(tmpl, {k: v[:ns] for k, v in V.items()})
    stress = []
    ov = None
    if any(r_["flag"] == "too rosy" for r_ in br_rows):
        k, miss0, miss1 = outside_view_k(cfg, Ps, ns, T, seed)
        cur = float(np.median(np.asarray((cfg.get("demand") or {}).get("outside_view_factor", 1.0), float)))
        ov = dict(demand_multiplier=k, suggested_outside_view_factor=round(k * cur, 2), mismatch_before=miss0, mismatch_after=miss1,
                  note=("The model is rosier than real businesses of this kind. Demand scaled by this factor is where the model sits closest to the real-world rates at this evidence level. "
                        "Set demand.outside_view_factor to the suggested value and rerun; report that run as the main result and this one as 'if the research holds fully'.") if k < 1 else
                       ("The gap to the real-world rates is small; no demand correction needed." if miss0 < 0.003 else
                        "Lowering demand does not bring the model closer to the real-world rates. Check that the base rates fit this kind of business, then look at costs, starting money and the quit rule."))
    for name, sc in STRESS + ([(f"demand scaled to real-world rates (x{ov['demand_multiplier']})", {"demand": ov["demand_multiplier"]})] if ov and ov["demand_multiplier"] < 1 else []):
        if "cac" in sc and len(sc) == 1 and dmodel != "funnel":
            continue
        o2 = simulate(Ps, ns, T, np.random.default_rng(seed + 101), stochastic=True, opt={"stress": sc})
        m2 = metrics(o2, Ps)
        hl = m2["hourly"][:, -1]
        stress.append(dict(scenario=name, loses_money=float((m2["founder_net_no_exit"] < 0).mean()), p50_founder_net_no_sale=float(np.median(m2["founder_net_no_exit"])),
                           money_needed_p80=float(np.percentile(m2["peak_need_draws"], 80)), still_open_at_end=float(m2["alive_end"].mean()),
                           p50_owner_hourly_last_year=None if np.all(np.isnan(hl)) else float(np.nanmedian(hl))))

    years = T // 12
    A_ = M["ann"]
    fit_block = None
    if has_fit:
        def part(mask):
            if not mask.any():
                return None
            x = M["founder_net_no_exit"][mask]
            return dict(share=float(mask.mean()), loses_money=float((x < 0).mean()), p50_founder_net_no_sale=float(np.median(x)),
                        p50_year3_revenue=float(np.median(A_["net_revenue"][mask, min(2, years - 1)])), p90_year3_revenue=float(np.percentile(A_["net_revenue"][mask, min(2, years - 1)], 90)),
                        still_open_at_end=float(M["alive_end"][mask].mean()))
        fit_block = dict(finds_demand=part(out["fit"]), does_not_find_demand=part(~out["fit"]))
    summary = {
        "meta": cfg.get("meta", {}), "n_sims": n, "months": T, "seed": seed,
        "verdict": verdict(cfg, M, out, warns, amap, ue),
        "probabilities": {
            "still_operating_at_end": float(M["alive_end"].mean()),
            "ran_out_of_cash": float((M["end_reason"] == 1).mean()),
            "owner_gave_up": float((M["end_reason"] == 2).mean()),
            "closed_for_outside_reasons": float((M["end_reason"] == 3).mean()),
            **{f"closed_by_year_{y+1}": float(((M["end_month"] >= 0) & (M["end_month"] < 12 * (y + 1))).mean()) for y in range(years)},
            **{f"profitable_year_{y+1}": float((A_["net_income"][:, y] > 0).mean()) for y in range(years)},
            **{f"still_open_{y+1}y_after_opening": float(1 - ((M["end_month"] >= 0) & (M["end_month"] - out["launch"] < 12 * (y + 1))).mean())
               for y in range(years) if 12 * (y + 1) <= T - int(np.median(out["launch"]))},
            "never_breaks_even": float(np.isnan(M["breakeven"]).mean()),
            "founder_loses_money_overall": float((M["founder_net"] < 0).mean()),
            "founder_loses_money_without_exit_value": float((M["founder_net_no_exit"] < 0).mean()),
            "beats_job_alternative": float((M["founder_net"] > M["job_alt"]).mean()),
            "beats_job_alternative_without_exit_value": float((M["founder_net_no_exit"] > M["job_alt"]).mean()),
            "loses_more_than_starting_cash": float((M["founder_net"] < -np.median(M["money_in"])).mean()),
        },
        "by_fit": fit_block,
        "annual": {k: [dict(year=y + 1, mean=float(A_[k][:, y].mean()), **pct(A_[k][:, y])) for y in range(years)] for k in A_},
        "breakeven_month_from_today": pct(M["breakeven"]),
        "breakeven_months_after_launch": pct(M["breakeven_after_launch"]),
        "economic_breakeven_month_from_today": pct(M["breakeven_econ"]),
        "never_beats_market_wage": float(np.isnan(M["breakeven_econ"]).mean()),
        "payback_month_from_today": pct(M["payback"]),
        "never_pays_back": float(np.isnan(M["payback"]).mean()),
        "peak_cash_need": dict(**pct(M["peak_need"], (50, 75, 80, 90, 95))),
        "peak_cash_need_with_owner_draws": dict(**pct(M["peak_need_draws"], (50, 75, 80, 90, 95))),
        "founder_money_available": float(np.median(out["funds_total"])),
        "founder_net_gain_total": dict(mean=float(M["founder_net"].mean()), **pct(M["founder_net"], (5, 10, 25, 50, 75, 90, 95))),
        "founder_net_gain_without_exit_value": dict(mean=float(M["founder_net_no_exit"].mean()), **pct(M["founder_net_no_exit"], (10, 25, 50, 75, 90))),
        "job_alternative_after_tax_total": float(np.median(M["job_alt"])),
        "npv_founder": dict(**pct(M["npv"])),
        "owner_hourly_by_year": [dict(year=y + 1, **pct(M["hourly"][:, y])) for y in range(years)],
        "close_month_distribution": pct(M["end_month"][M["end_month"] >= 0].astype(float)) if (M["end_month"] >= 0).any() else None,
        "representative_paths": reps,
        "expected_path_median_inputs": {
            "assumes_demand_is_found": True,
            "annual_net_revenue": [float(x) for x in M_med["ann"]["net_revenue"][0]],
            "annual_net_income": [float(x) for x in M_med["ann"]["net_income"][0]],
            "breakeven_month": None if np.isnan(M_med["breakeven"][0]) else float(M_med["breakeven"][0]),
            "peak_cash_need": float(M_med["peak_need"][0]),
        },
        "unit_economics_typical_mature_year": ue,
        "stress_tests": stress,
        "outside_view": br_rows,
        "outside_view_calibration": ov,
        "assumption_map": amap,
        "sanity_ratios_median_path_last12m": ratios,
        "warnings": warns,
        "tornado_total_profit_no_ruin": tor[:15],
        "spearman": sp[:15],
    }
    summary["solve"] = [solve_levels(tmpl, V, reg, nm, T, seed) for nm in (solve or [])]
    summary["sweep"] = [sweep_levels(tmpl, V, reg, sw, T, seed) for sw in (sweep or [])]
    with open(os.path.join(outdir, "summary.json"), "w") as f:
        json.dump(summary, f, indent=2, default=float)

    # CSVs
    def wcsv(name, header, rows):
        with open(os.path.join(outdir, name), "w") as f:
            f.write(",".join(header) + "\n")
            for r_ in rows:
                f.write(",".join("" if v is None else (f"{v:.4f}" if isinstance(v, float) else str(v)) for v in r_) + "\n")
    keys_m = ["net_revenue", "gross_profit", "ebitda", "net_income", "fcf", "cash", "active_customers", "units", "owner_draw", "staff_fte"]
    rows = []
    for t in range(T):
        row = [t, float(R["alive"][:, t].mean())]
        for k in keys_m:
            q = np.percentile(R[k][:, t], [10, 25, 50, 75, 90])
            row += [float(x) for x in q]
        rows.append(row)
    wcsv("monthly_percentiles.csv", ["month", "share_operating"] + [f"{k}_p{q}" for k in keys_m for q in (10, 25, 50, 75, 90)], rows)
    rows = []
    for k in A_:
        for y in range(years):
            q = pct(A_[k][:, y])
            rows.append([k, y + 1, float(A_[k][:, y].mean())] + [q[f"p{x}"] for x in (10, 25, 50, 75, 90)])
    wcsv("annual_percentiles.csv", ["metric", "year", "mean", "p10", "p25", "p50", "p75", "p90"], rows)
    allk = list(R.keys())
    rows = []
    for lab, i in reps.items():
        for t in range(T):
            rows.append([lab, t] + [float(R[k][i, t]) for k in allk])
    Rm = out_med["R"]
    for t in range(T):
        rows.append(["expected_median_inputs", t] + [float(Rm[k][0, t]) for k in allk])
    wcsv("scenario_paths.csv", ["scenario", "month"] + allk, rows)
    q_ = lambda x: '"' + str(x).replace('"', "'") + '"'
    wcsv("assumptions.csv", ["variable", "dist", "p10", "p50", "p90", "evidence", "source", "note"],
         [[a["variable"], a["dist"], a["p10"], a["p50"], a["p90"], a["ev"], q_(a["source"]), q_(a["note"])] for a in assumptions])
    wcsv("tornado.csv", ["variable", "input_p10", "input_p90", "total_profit_at_p10", "total_profit_at_p90", "swing", "y3_ni_p10", "y3_ni_p90"],
         [[d["variable"], d["input_p10"], d["input_p90"], d["outcome_at_p10"], d["outcome_at_p90"], d["swing"], d["y3_net_income_at_p10"], d["y3_net_income_at_p90"]] for d in tor])

    # explain: yearly line items of the representative median path
    i = reps["p50"]
    explain = []
    for y in range(years):
        sl = slice(12 * y, 12 * (y + 1))
        e = {k: float(R[k][i, sl].sum()) for k in allk if k not in ("cash", "alive", "active_customers", "credit_balance", "loan_balance", "staff_fte", "capacity")}
        e.update(year=y + 1, cash_end=float(R["cash"][i, 12 * (y + 1) - 1]), active_customers_end=float(R["active_customers"][i, 12 * (y + 1) - 1]),
                 avg_staff=float(R["staff_fte"][i, sl].mean()), operating_months=float(R["alive"][i, sl].sum()))
        e["avg_price_per_transaction"] = e["gross_revenue"] / e["transactions"] if e["transactions"] > 0 else None
        explain.append(e)
    with open(os.path.join(outdir, "explain_median_path.json"), "w") as f:
        json.dump(explain, f, indent=2)

    write_report(summary, outdir, cfg)
    if charts:
        try:
            make_charts(R, M, tor, outdir, T)
        except Exception as e:  # charts are optional
            print("charts skipped:", e)
    return summary

def _typical_grid(tmpl, V, reg, name, grid, T, seed, demand_only=True):
    """Typical case (median inputs, money never runs out, demand found) with one input swept over grid."""
    k = len(grid)
    Vt = {kk: np.full(k, float(np.median(v))) for kk, v in V.items()}
    path = reg[name].get("path", name)
    demand_like = any(x in path for x in ("mature_units_per_month", "organic_new_per_month", "conversion"))
    hair = [kk for kk, sp in reg.items() if sp.get("path") in ("demand.reality_factor", "demand.panel_factor")]
    real_med = float(np.prod([np.median(V[kk]) for kk in hair])) if hair else 1.0
    if demand_only and demand_like:
        for kk in hair:
            Vt[kk] = np.ones(k)  # solve for the ACTUAL demand level needed (no optimism haircut on top)
    Vt[name] = np.asarray(grid, float)
    Pt = resolve(no_ruin(tmpl), Vt)
    return simulate(Pt, k, T, np.random.default_rng(seed), stochastic=False, opt={"force_fit": True}), demand_like, real_med

def solve_levels(tmpl, V, reg, name, T, seed, n_grid=400):
    """What value of one input makes the typical case (median inputs, money never runs out) break even / pay a market wage?"""
    if name not in V:
        raise ValueError(f"--solve: unknown input '{name}'. Known: {sorted(V)}")
    p10, p90 = float(np.percentile(V[name], 10)), float(np.percentile(V[name], 90))
    lo = p10 / 4 if p10 > 0 else p10 - 3 * abs(p90 - p10)
    hi = p90 * 4 if p90 > 0 else p90 + 3 * abs(p90 - p10)
    if 0 <= p10 and p90 <= 1:
        lo, hi = max(lo, 0.0), min(hi, 1.0)
    grid = np.geomspace(max(lo, 1e-9), hi, n_grid) if lo > 0 else np.linspace(lo, hi, n_grid)
    ot, demand_like, real_med = _typical_grid(tmpl, V, reg, name, grid, T, seed)
    R = ot["R"]
    total_profit = R["net_income"].sum(axis=1).astype(float)
    last12 = (R["ebitda"][:, -12:] - R["interest"][:, -12:] + R["owner_salary"][:, -12:]).sum(axis=1).astype(float)
    wage_gap = last12 - ot["opp_wage"] * ot["hours"] * 52
    def cross(y):
        sgn = np.sign(y)
        idx = np.where(sgn[:-1] * sgn[1:] < 0)[0]
        if idx.size == 0:
            return "positive over the whole search range" if y.min() > 0 else "negative over the whole search range"
        i = idx[0]
        return float(grid[i] + (grid[i + 1] - grid[i]) * (0 - y[i]) / (y[i + 1] - y[i]))
    return dict(variable=name, current_p10=p10, current_p50=float(np.median(V[name])), current_p90=p90, search_range=[float(grid[0]), float(grid[-1])],
                realistic_p50_after_optimism_correction=float(np.median(V[name])) * real_med if demand_like else None,
                level_for_total_profit_zero=cross(total_profit), level_for_market_wage_in_final_year=cross(wage_gap))

def sweep_levels(tmpl, V, reg, spec, T, seed):
    """--sweep NAME=v1,v2,...: the typical case at each value (all haircuts kept). With price_elasticity set this is the price ladder."""
    name, _, vals = spec.partition("=")
    if name not in V:
        raise ValueError(f"--sweep: unknown input '{name}'. Known: {sorted(V)}")
    grid = [float(x) for x in vals.split(",") if x.strip()]
    ot, _, _ = _typical_grid(tmpl, V, reg, name, grid, T, seed, demand_only=False)
    R = ot["R"]
    rows = []
    for j, v in enumerate(grid):
        own12 = float((R["ebitda"][j, -12:] - R["interest"][j, -12:] + R["owner_salary"][j, -12:]).sum())
        rows.append(dict(value=v, sales_per_month_final_year=float(R["transactions"][j, -12:].mean()), revenue_final_year=float(R["net_revenue"][j, -12:].sum()),
                         owner_earnings_final_year_before_tax=own12, total_profit_after_tax=float(R["net_income"][j].sum())))
    return dict(variable=name, current_p50=float(np.median(V[name])), rows=rows)

def write_report(s, outdir, cfg):
    L = []
    cur = cfg.get("meta", {}).get("currency", "USD")
    L.append(f"# Simulation report: {cfg.get('meta', {}).get('name', 'business')}  ({s['n_sims']:,} runs, {s['months']} months, {cur})\n")
    v = s["verdict"]
    L.append(f"## Computed call: {v['call']}")
    for x in v["no_go_reasons"]:
        L.append(f"- NO-GO because: {x}")
    for x in v["missing_for_go"]:
        L.append(f"- Missing for a GO: {x}")
    for x in v["passed"]:
        L.append(f"- Passed: {x}")
    L.append(f"- Counted {v['exit_weight']:.0%} of any sale value. All runs: loses money {v['loss_prob']:.0%}, beats the job {v['beats_job_prob']:.0%}."
             + (f" If demand is found ({v['demand_found_share']:.0%} of runs): loses money {v['loss_prob_if_demand_found']:.0%}, beats the job {v['beats_job_prob_if_demand_found']:.0%}. "
                f"If it is not found the typical loss is {fmt(v['p50_money_lost_if_demand_not_found'])}." if "demand_found_share" in v else ""))
    if v["test_first"]:
        L.append("- Test these first (big effect on the result, weak evidence): " + "; ".join(v["test_first"]))
    p = s["probabilities"]
    L.append("\n## Odds")
    for k, x in p.items():
        L.append(f"- {k}: {x:.1%}")
    if s.get("by_fit"):
        L.append("\n## If the product finds demand vs if it does not")
        for k, d in s["by_fit"].items():
            if d:
                L.append(f"- {k}: {d['share']:.0%} of runs | loses money {d['loses_money']:.0%} | P50 founder net gain without sale {fmt(d['p50_founder_net_no_sale'])} | "
                         f"year-3 revenue P50 {fmt(d['p50_year3_revenue'])}, P90 {fmt(d['p90_year3_revenue'])} | still open at end {d['still_open_at_end']:.0%}")
    L.append("\n## Annual results (P10 bad / P50 typical / P90 good)")
    L.append("| metric | year | P10 | P50 | P90 | mean |")
    L.append("|---|---|---|---|---|---|")
    for k in ("net_revenue", "gross_profit", "ebitda", "net_income", "owner_draw", "cash_end", "active_customers_end"):
        for r in s["annual"][k]:
            L.append(f"| {k} | {r['year']} | {fmt(r['p10'])} | {fmt(r['p50'])} | {fmt(r['p90'])} | {fmt(r['mean'])} |")
    L.append("\n## Key numbers")
    b = s["breakeven_month_from_today"]
    L.append(f"- Break-even month (3 months in a row of positive operating profit), from today, among runs that ever get there: P10 {fmt(b['p10'])}, P50 {fmt(b['p50'])}, P90 {fmt(b['p90'])}. Runs that never break even: {s['probabilities']['never_breaks_even']:.0%}")
    e = s["economic_breakeven_month_from_today"]
    L.append(f"- Month operating profit beats a market wage for your hours (among runs that get there): P50 {fmt(e['p50'])}. Never within horizon: {s['never_beats_market_wage']:.0%}")
    pb = s["payback_month_from_today"]
    L.append(f"- Payback month (cumulative cash flow back to zero, among runs that get there): P50 {fmt(pb['p50'])}. Never within horizon: {s['never_pays_back']:.0%}")
    pk = s["peak_cash_need_with_owner_draws"]
    L.append(f"- Money needed to never run out (incl. owner draws): P50 {fmt(pk['p50'])}, P80 {fmt(pk['p80'])}, P90 {fmt(pk['p90'])}. Money the founder can bring (cash, reserve, credit, loans): {fmt(s['founder_money_available'])}")
    fn = s["founder_net_gain_total"]
    L.append(f"- Founder total net gain incl. business value at end: P10 {fmt(fn['p10'])}, P50 {fmt(fn['p50'])}, P90 {fmt(fn['p90'])}, mean {fmt(fn['mean'])}")
    fn2 = s["founder_net_gain_without_exit_value"]
    L.append(f"- Same but WITHOUT selling the business: P10 {fmt(fn2['p10'])}, P50 {fmt(fn2['p50'])}, P90 {fmt(fn2['p90'])}")
    L.append(f"- After-tax pay a normal job would have given over the same months and hours: P50 {fmt(s['job_alternative_after_tax_total'])}")
    for r in s["owner_hourly_by_year"]:
        L.append(f"- Owner real hourly earnings year {r['year']} (after tax, runs still operating): P10 {fmt(r['p10'])}, P50 {fmt(r['p50'])}, P90 {fmt(r['p90'])}")
    ue = s.get("unit_economics_typical_mature_year")
    if ue:
        PC = {"contribution_margin", "monthly_churn", "breakeven_share_of_capacity"}
        L.append("\n## Unit economics (typical mature year: median inputs, demand found, final 12 months)")
        for k, x in ue.items():
            if x is not None:
                L.append(f"- {k}: {x:.1%}" if k in PC else f"- {k}: {fmt(x)}")
    L.append("\n## Stress tests (same runs, one thing worse)")
    L.append("| scenario | loses money | P50 founder net gain, no sale | money needed P80 | still open at end | P50 owner hourly, final year |")
    L.append("|---|---|---|---|---|---|")
    for r in s["stress_tests"]:
        L.append(f"| {r['scenario']} | {r['loses_money']:.0%} | {fmt(r['p50_founder_net_no_sale'])} | {fmt(r['money_needed_p80'])} | {r['still_open_at_end']:.0%} | {fmt(r['p50_owner_hourly_last_year'])} |")
    if s.get("outside_view"):
        L.append("\n## Outside view (model vs what really happens to businesses like this)")
        L.append("| check | model | real world | model may go to | flag | source |")
        L.append("|---|---|---|---|---|---|")
        for r in s["outside_view"]:
            mo = "n/a" if r["model"] is None else f"{r['model']:.0%}"
            al = "n/a" if r["allowed"] is None else f"{r['allowed']:.0%}"
            L.append(f"| {r['name']} | {mo} | {r['world']:.0%} | {al} | {r['flag']} | {r['source']} |")
        ov = s.get("outside_view_calibration")
        if ov:
            L.append(f"- Outside-view calibration: demand multiplier {ov['demand_multiplier']}, suggested demand.outside_view_factor {ov['suggested_outside_view_factor']}. {ov['note']}")
    L.append("\n## Biggest drivers")
    L.append("Rank correlation with founder net gain across all runs (+ means higher input = better outcome):")
    for d in s["spearman"][:8]:
        L.append(f"- {d['variable']}: {d['rho_founder_net']:+.2f} (with money needed: {d['rho_peak_cash_need']:+.2f})")
    L.append("Tornado: total profit over the horizon if money never ran out and demand is found, input at its P10 vs P90:")
    for d in s["tornado_total_profit_no_ruin"][:8]:
        L.append(f"- {d['variable']}: {fmt(d['input_p10'])} -> {fmt(d['outcome_at_p10'])} | {fmt(d['input_p90'])} -> {fmt(d['outcome_at_p90'])} (swing {fmt(d['swing'])})")
    if s.get("assumption_map"):
        L.append("\n## Assumption map (impact x evidence; A measured, B benchmark, C analog or panel, D guess, ? ungraded)")
        for a in s["assumption_map"][:10]:
            L.append(f"- {a['variable']}: swing {fmt(a['swing'])}, evidence {a['ev']}" + ("  <- test first" if a["ev"] in ("?", "C", "D") else ""))
    L.append("\n## Sanity ratios (median path, last 12 months)")
    PCT = {"gross_margin", "ebitda_margin", "net_margin", "marketing_pct", "labor_pct", "rent_pct", "prime_cost_pct", "payment_fee_pct", "monthly_churn"}
    L.append(f"(basis: {s['sanity_ratios_median_path_last12m'].get('_basis', '')})")
    for k, x in s["sanity_ratios_median_path_last12m"].items():
        if x is None or k == "_basis":
            continue
        L.append(f"- {k}: {x:.1%}" if k in PCT else f"- {k}: {round(x, 2) if abs(x) < 100 else fmt(x)}")
    if s.get("solve"):
        L.append("\n## What must be true (typical case, one input changed, money never runs out)")
        for d in s["solve"]:
            adj = f", realistic P50 after optimism correction {fmt(d['realistic_p50_after_optimism_correction'])}" if d.get("realistic_p50_after_optimism_correction") else ""
            f2 = lambda x: x if isinstance(x, str) else "crosses zero at " + fmt(x)
            L.append(f"- {d['variable']} (your estimate P10 {fmt(d['current_p10'])} / P50 {fmt(d['current_p50'])} / P90 {fmt(d['current_p90'])}{adj}): "
                     f"total profit {f2(d['level_for_total_profit_zero'])}; owner earnings minus a market wage, final year: {f2(d['level_for_market_wage_in_final_year'])} "
                     f"(searched {fmt(d['search_range'][0])} to {fmt(d['search_range'][1])})")
    for d in s.get("sweep") or []:
        L.append(f"\n## Sweep of {d['variable']} (typical case; current P50 {fmt(d['current_p50'])})")
        L.append("| value | sales per month, final year | revenue, final year | owner earnings before tax, final year | total profit after tax |")
        L.append("|---|---|---|---|---|")
        for r in d["rows"]:
            L.append(f"| {fmt(r['value'])} | {fmt(r['sales_per_month_final_year'])} | {fmt(r['revenue_final_year'])} | {fmt(r['owner_earnings_final_year_before_tax'])} | {fmt(r['total_profit_after_tax'])} |")
    L.append("\n## Warnings")
    for w in s["warnings"] or ["none"]:
        L.append(f"- {w}")
    with open(os.path.join(outdir, "report.md"), "w") as f:
        f.write("\n".join(L) + "\n")

def make_charts(R, M, tor, outdir, T):
    import matplotlib
    matplotlib.use("Agg")
    import matplotlib.pyplot as plt
    from matplotlib.ticker import FuncFormatter
    money = FuncFormatter(lambda v, _: fmt(v))
    cd = os.path.join(outdir, "charts")
    os.makedirs(cd, exist_ok=True)
    x = np.arange(T)
    for key, title in (("cash", "Cash in the bank (P10 to P90)"), ("net_revenue", "Monthly revenue (P10 to P90)"), ("ebitda", "Monthly operating profit (P10 to P90)")):
        q = np.percentile(R[key], [10, 25, 50, 75, 90], axis=0)
        fig, ax = plt.subplots(figsize=(9, 4.5))
        ax.fill_between(x, q[0], q[4], alpha=0.18, color="#3b6ea5", label="P10 to P90")
        ax.fill_between(x, q[1], q[3], alpha=0.35, color="#3b6ea5", label="P25 to P75")
        ax.plot(x, q[2], color="#1d3f66", lw=2, label="median")
        ax.axhline(0, color="#999", lw=1)
        ax.set_title(title + ("  (closed businesses keep their final cash)" if key == "cash" else "")); ax.set_xlabel("month from today")
        ax.legend(loc="upper left", frameon=False); ax.yaxis.set_major_formatter(money)
        ax.spines[["top", "right"]].set_visible(False)
        fig.tight_layout(); fig.savefig(os.path.join(cd, f"{key}_fan.png"), dpi=130); plt.close(fig)
    fig, ax = plt.subplots(figsize=(9, 4.5))
    v = M["founder_net"]
    lo, hi = np.percentile(v, [1, 99])
    ax.hist(np.clip(v, lo, hi), bins=60, color="#3b6ea5", alpha=0.85)
    ax.axvline(0, color="#c0392b", lw=1.5)
    ax.set_title(f"Founder total net gain: {np.mean(v < 0):.0%} of runs lose money"); ax.xaxis.set_major_formatter(money)
    ax.spines[["top", "right"]].set_visible(False)
    fig.tight_layout(); fig.savefig(os.path.join(cd, "outcome_histogram.png"), dpi=130); plt.close(fig)
    fig, ax = plt.subplots(figsize=(9, 4))
    ax.plot(x, R["alive"].mean(axis=0) * 100, color="#1d3f66", lw=2)
    ax.set_ylim(0, 105); ax.set_title("% of runs still operating"); ax.set_xlabel("month from today")
    ax.spines[["top", "right"]].set_visible(False)
    fig.tight_layout(); fig.savefig(os.path.join(cd, "survival.png"), dpi=130); plt.close(fig)
    if tor:
        top = tor[:10][::-1]
        fig, ax = plt.subplots(figsize=(9, 0.5 * len(top) + 1.5))
        base = None
        for j, d in enumerate(top):
            a, b = d["outcome_at_p10"], d["outcome_at_p90"]
            ax.barh(j, b - a, left=a, color="#3b6ea5" if b > a else "#c0392b", alpha=0.85)
        ax.set_yticks(range(len(top))); ax.set_yticklabels([d["variable"][-40:] for d in top], fontsize=8)
        ax.xaxis.set_major_formatter(money)
        ax.set_title("Total profit over horizon if money never ran out (input P10 vs P90)")
        ax.spines[["top", "right"]].set_visible(False)
        fig.tight_layout(); fig.savefig(os.path.join(cd, "tornado.png"), dpi=130); plt.close(fig)

# ---------------------------------------------------------------- real-world test planner
def binom_cdf(k, n, p):
    """P(X <= k) for X ~ Binomial(n, p), exact."""
    if k < 0:
        return 0.0
    if k >= n:
        return 1.0
    if p <= 0:
        return 1.0
    if p >= 1:
        return 0.0
    lp, lq = math.log(p), math.log(1 - p)
    return min(1.0, sum(math.exp(math.lgamma(n + 1) - math.lgamma(i + 1) - math.lgamma(n - i + 1) + i * lp + (n - i) * lq) for i in range(int(k) + 1)))

def testplan(need, n, expect=None, conf=0.9):
    """Pass and kill lines for a real test with n people exposed, when the business needs a rate of `need`.
    Go line: seeing this many or more would be unlikely (chance <= 1-conf) if the true rate were only `need`.
    Kill line: seeing this many or fewer would be unlikely if the true rate were at least `need`."""
    a = 1 - conf
    go = next((k for k in range(n + 1) if 1 - binom_cdf(k - 1, n, need) <= a), None)
    kill = max((k for k in range(n + 1) if binom_cdf(k, n, need) <= a), default=None)
    out = dict(need=need, n=n, confidence=conf, expected_count_at_need=need * n, go_line=go, kill_line=kill)
    if expect is not None:
        out["expected_rate"] = expect
        out["chance_to_clear_go_line_if_expected_rate_is_true"] = None if go is None else 1 - binom_cdf(go - 1, n, expect)
        if expect > need:
            m = n
            while m < 200000:
                g2 = next((k for k in range(m + 1) if 1 - binom_cdf(k - 1, m, need) <= a), None)
                if g2 is not None and 1 - binom_cdf(g2 - 1, m, expect) >= 0.8:
                    break
                m = int(m * 1.25) + 1
            out["people_needed_for_80pct_chance_to_clear"] = m
    return out

def testplan_main(argv):
    ap = argparse.ArgumentParser(prog="bizsim.py testplan")
    ap.add_argument("--need", type=float, required=True, help="rate the business needs (from --solve), e.g. 0.04")
    ap.add_argument("--n", type=int, required=True, help="people exposed in the test (visitors, calls, passers-by)")
    ap.add_argument("--expect", type=float, help="rate you realistically expect")
    ap.add_argument("--conf", type=float, default=0.9)
    a = ap.parse_args(argv)
    r = testplan(a.need, a.n, a.expect, a.conf)
    print(json.dumps(r, indent=1))
    print(f"\nWrite these down BEFORE the test. {a.n} people exposed, the business needs {a.need:.2%}:")
    print(f"- GO if {r['go_line']} or more buy (the true rate is then very likely above what the business needs)." if r["go_line"] is not None else "- No go line is reachable with this few people.")
    print(f"- STOP if {r['kill_line']} or fewer buy (the true rate is then very likely below what the business needs)." if r["kill_line"] is not None else "- No stop line: with this few people even zero buyers proves nothing. Expose more people.")
    print("- In between: not decided. Expose more people before spending more.")

# ---------------------------------------------------------------- self test (golden numbers worked out by hand)
def selftest():
    import copy
    res = []
    def chk(name, got, want, tol=1e-6):
        good = got is not None and abs(float(got) - want) <= tol
        res.append(good)
        print("PASS" if good else "FAIL", "|", name, "| got", None if got is None else round(float(got), 4), "| want", want)
    def det(cfg, T=24, opt=None):
        reg = {}
        tm = extract(cfg, "", reg)
        V = {k: np.array([np.median(sample_dist(s, 2001, np.random.default_rng(0)))]) for k, s in reg.items()}
        return simulate(resolve(tm, V), 1, T, np.random.default_rng(0), stochastic=False, opt=opt)
    base = {"simulation": {"inflation_annual": 0.0}, "timeline": {"launch_month": 0, "ramp_start_level": 1.0},
            "demand": {"model": "traffic", "mature_units_per_month": 1000, "price_per_unit": 10, "monthly_volatility": 0, "annual_price_increase": 0},
            "variable_costs": {"cogs_pct": 0.3, "payment_fee_pct": 0, "payment_fee_fixed": 0},
            "fixed_costs": [{"name": "rent", "category": "rent", "monthly": 2000}], "financing": {"founder_cash": 10000}, "taxes": {"mode": "none"}}
    R = det(base)["R"]
    chk("traffic: 1000 x 10 x 0.7 - 2000 = 5000 operating profit", R["ebitda"][0, 5], 5000)
    chk("traffic: cash after 24 months = 10000 + 24 x 5000", R["cash"][0, 23], 130000, 1e-2)
    c = copy.deepcopy(base); c["demand"].update(price_per_unit=11, reference_price=10, price_elasticity=-2)
    chk("elasticity: 1000 x (11/10)^-2 units", det(c)["R"]["units"][0, 5], 826.4463, 1e-3)
    chk("stress: price -10% at the same volume = 9000 revenue", det(base, opt={"stress": {"price": 0.9}})["R"]["net_revenue"][0, 5], 9000)
    c = copy.deepcopy(base); c["demand"].update(fit_probability=0.2, no_fit_factor=0.1, panel_factor=0.5)
    chk("no fit and panel factor: 1000 x 0.1 x 0.5 units", det(c)["R"]["units"][0, 5], 50)
    chk("demand found: 1000 x 0.5 units", det(c, opt={"force_fit": True})["R"]["units"][0, 5], 500)
    c = copy.deepcopy(base); c["demand"]["mature_units_per_month"] = 0; c["financing"]["founder_cash"] = 1e6
    c["quit_rule"] = {"max_months_without_profit": 6, "monthly_quit_prob": 0.5}
    chk("quit: reason exists from month 6, closes after 1/0.5 = 2 months of it", det(c)["end_month"][0], 7)
    f = {"simulation": {"inflation_annual": 0.0}, "timeline": {"launch_month": 0, "ramp_start_level": 1.0},
         "demand": {"model": "funnel", "subscription": True, "channels": [{"name": "ads", "monthly_budget": 1000, "cac": 100}], "early_cac_multiplier": 1.0,
                    "churn_monthly": 0.1, "early_churn_monthly": 0.1, "first_purchase_value": 50, "repeat_order_value": 50, "repeat_orders_per_active_month": 1,
                    "monthly_volatility": 0, "annual_price_increase": 0},
         "variable_costs": {"payment_fee_pct": 0, "payment_fee_fixed": 0}, "financing": {"founder_cash": 100000}, "taxes": {"mode": "none"}}
    R = det(f, 120)["R"]
    chk("funnel: 10 new a month, 10% churn -> 100 customers", R["active_customers"][0, -1], 100, 0.01)
    chk("funnel: 100 customers x 50 = 5000 revenue", R["net_revenue"][0, -1], 5000, 0.5)
    chk("stress: CAC +50% -> 1000 / 150 new customers", det(f, 24, {"stress": {"cac": 1.5}})["R"]["new_customers"][0, 3], 6.6667, 1e-3)
    tax, _ = annual_tax(np.array([100000.0]), np.array([300000.0]), np.zeros(1), {"mode": "us_passthrough", "state": "CA", "ca_llc": True}, 1.0)
    chk("tax: 100k profit, single, CA LLC, 300k receipts", tax[0], 28462.5, 2.0)
    tax, nol = annual_tax(np.array([-20000.0]), np.array([10000.0]), np.zeros(1), {"mode": "us_passthrough", "state": "CA", "ca_llc": True}, 1.0)
    chk("tax: a loss year still pays the 800 LLC tax", tax[0], 800)
    chk("tax: the loss carries forward", nol[0], 20000)
    ln = {"simulation": {"inflation_annual": 0.0}, "timeline": {"launch_month": 0},
          "demand": {"model": "traffic", "mature_units_per_month": 0, "price_per_unit": 0, "monthly_volatility": 0},
          "financing": {"founder_cash": 50000, "loans": [{"amount": 10000, "annual_rate": 0.12, "term_months": 12, "month": 0}]}, "taxes": {"mode": "none"}}
    R = det(ln)["R"]
    chk("loan: first month interest on 10000 at 12%", R["interest"][0, 1], 100)
    chk("loan: 12 payments of 888.49 repay it", -R["loan_flow"][0, 1:13].sum(), 10000, 1e-2)
    # everything switched on at once: the accounting identities must hold in every run
    big = {"simulation": {"inflation_annual": 0.03}, "timeline": {"launch_month": [1, 2, 4], "ramp_months": [6, 10, 18], "ramp_start_level": 0.2},
           "demand": {"model": "funnel", "channels": [{"name": "google", "monthly_budget": 1500, "cpc": [3, 4, 6], "conversion": [0.02, 0.03, 0.05]}],
                      "organic_new_per_month": [5, 15, 30], "referral_rate": 0.01, "churn_monthly": [0.04, 0.06, 0.1], "early_churn_monthly": 0.15,
                      "first_purchase_value": [50, 60, 75], "reference_price": 60, "price_elasticity": [-2.5, -1.5, -0.8], "repeat_order_value": 45,
                      "repeat_orders_per_active_month": 0.5, "refund_pct": 0.05, "fit_probability": 0.5, "no_fit_factor": [0.05, 0.15, 0.3]},
           "variable_costs": {"cogs_pct": 0.35, "cost_per_order": 6},
           "fixed_costs": [{"name": "software", "category": "software", "monthly": 150}, {"name": "ins", "category": "insurance", "monthly": 60}],
           "staff": [{"name": "helper", "hourly_wage": 20, "hours_per_month": 80, "start": 6}],
           "capex": [{"name": "equipment", "amount": 4000, "month": -1, "life_months": 36}],
           "one_time_costs": [{"name": "legal", "amount": 1200, "month": -1}, {"name": "deposit", "amount": 2000, "month": -1, "expense": False, "refundable_at_close": True}],
           "working_capital": {"inventory_days": 30, "dpo_days": 15},
           "financing": {"founder_cash": 30000, "founder_reserve": 5000, "credit_line": {"limit": 5000, "annual_rate": 0.25},
                         "loans": [{"amount": 5000, "annual_rate": 0.11, "term_months": 36, "month": 0}], "owner_draw_monthly": 1500},
           "shocks": [{"name": "bigcost", "type": "cost", "monthly_prob": 0.02, "amount": 3000}],
           "quit_rule": {"max_months_without_profit": [9, 15, 30], "monthly_quit_prob": 0.2, "exogenous_exit_annual": 0.04},
           "taxes": {"mode": "us_passthrough", "state": "CA", "ca_llc": True}}
    reg = {}
    tm = extract(big, "", reg)
    rng = np.random.default_rng(3)
    V = sample_all(reg, 3000, rng, [["demand.churn_monthly", "demand.organic_new_per_month", -0.5]])
    o = simulate(resolve(tm, V), 3000, 48, rng)
    R, alive = o["R"], o["alive"]
    rhs = o["equity_in"] + R["fcf"].sum(1) + R["loan_flow"].sum(1) - R["owner_draw"].sum(1) + R["injection"].sum(1) + o["credit"]
    chk("cash identity: largest error among open businesses", np.abs(R["cash"][:, -1] - rhs)[alive].max() if alive.any() else 0.0, 0, 1.0)
    chk("P&L identity: revenue - costs = operating profit", np.abs(R["net_revenue"] - R["cogs_total"] - R["opex_total"] - R["ebitda"]).max(), 0, 1.0)
    chk("P&L identity: profit before tax - tax = net income", np.abs(R["ebt"] - R["tax_expense"] - R["net_income"]).max(), 0, 1.0)
    em = o["end_month"]
    late = max((float(np.abs(R["net_revenue"][i, em[i] + 1:]).sum()) for i in np.where(~alive)[0] if em[i] + 1 < 48), default=0.0)
    chk("closed businesses stay closed (revenue after closing)", late, 0)
    chk("all three ways to close occur (cash, gave up, outside reasons)", float(len(set(o["end_reason"].tolist()) & {1, 2, 3})), 3)
    rk = lambda x: np.argsort(np.argsort(x))
    chk("rank correlation asked for -0.5", np.corrcoef(rk(V["demand.churn_monthly"]), rk(V["demand.organic_new_per_month"]))[0, 1], -0.48, 0.05)
    chk("window sum helper", win_sum(np.arange(12.0).reshape(2, 6), np.array([1, 4]), 2)[0], 3)
    chk("window that does not fit is NaN", float(np.isnan(win_sum(np.arange(12.0).reshape(2, 6), np.array([1, 5]), 2)[1])), 1)
    chk("binomial: P(X <= 2), n 10, p 0.5 = 56/1024", binom_cdf(2, 10, 0.5), 0.0546875, 1e-9)
    tp = testplan(0.05, 200, 0.08)
    chk("test plan: go line is the smallest count unlikely at the needed rate",
        float(1 - binom_cdf(tp["go_line"] - 1, 200, 0.05) <= 0.1 < 1 - binom_cdf(tp["go_line"] - 2, 200, 0.05)), 1)
    chk("test plan: kill line is the largest count unlikely at the needed rate",
        float(binom_cdf(tp["kill_line"], 200, 0.05) <= 0.1 < binom_cdf(tp["kill_line"] + 1, 200, 0.05)), 1)
    print("SELFTEST OK" if all(res) else f"SELFTEST FAILED: {res.count(False)} of {len(res)} checks")
    if not all(res):
        sys.exit(1)

def main():
    if len(sys.argv) > 1 and sys.argv[1] == "testplan":
        return testplan_main(sys.argv[2:])
    if len(sys.argv) > 1 and sys.argv[1] == "selftest":
        return selftest()
    ap = argparse.ArgumentParser()
    ap.add_argument("config")
    ap.add_argument("--out", default="sim_out")
    ap.add_argument("--sims", type=int)
    ap.add_argument("--seed", type=int)
    ap.add_argument("--no-charts", action="store_true")
    ap.add_argument("--solve", action="append", default=[], help="input name to solve for break-even / market-wage level (repeatable)")
    ap.add_argument("--sweep", action="append", default=[], help="NAME=v1,v2,...: typical case at each value (repeatable), e.g. price=39,49,59")
    ap.add_argument("--validate-only", action="store_true")
    a = ap.parse_args()
    with open(a.config) as f:
        cfg = json.load(f)
    if a.validate_only:
        pr = validate(cfg)
        reg = {}
        extract(cfg, "", reg)
        print("\n".join(pr) if pr else "config OK")
        print("uncertain inputs:", ", ".join(sorted(reg)))
        return
    run(cfg, a.out, a.sims, a.seed, not a.no_charts, a.solve, a.sweep)
    with open(os.path.join(a.out, "report.md")) as f:
        print(f.read())

if __name__ == "__main__":
    main()
# === BIZSIM END ===
```