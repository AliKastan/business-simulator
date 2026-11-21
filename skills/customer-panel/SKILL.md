---
name: "customer-panel"
description: "Tests an offer on a panel of simulated buyers (100 default, 20 quick) against a real competing offer: calibrated demand pull, price sensitivity, who buys, why not. Use for '100 müşteride dene', 'would people buy this', 'test demand'."
---

# Customer panel (simulated buyers, read against a real competitor)

Goal: before money is spent, put the offer in front of a panel of simulated buyers (100 by default, 20 for a quick look) and get out of it what such a panel can honestly give: how the offer pulls compared with a real offer these buyers already know, which way price moves demand, who buys and who does not, the reasons for no, and what would flip a maybe.

What it cannot give: the true number of buyers. Simulated people say yes too easily and too much alike. Published tests show the pattern: asked for a 1 to 5 rating directly, language models pile up in the middle; asked for words first and boxed afterwards, with age and income on the card, they rank concepts almost as reliably as human panels (Maier et al. 2025, 57 real concept surveys); their willingness to pay is sometimes right, often off and occasionally backwards (Brand, Israeli and Ngwe, Harvard Business School); and stated intent overstates purchase even for real people (Jamieson and Bass 1989). So this skill never uses a raw level. Every answer is words first, the box is weighted down to a purchase rate, the offer is read RELATIVE to a real control offer, and the result is pulled halfway back to "same as the control". Real buyers always beat the panel.

## Rules

1. Never answer for a buyer or a rater and never edit an answer. Each buyer is its own sub-agent that gets its brief unchanged. Running this skill is the user's request to use sub-agents for this.
2. All dealing and all arithmetic is done by `panel.py` (appendix). Code deals the cards so the panel is not tilted toward agreeable people. Never tally by eye.
3. Segments, their shares, incomes, today's habits and objections come from research with sources (census or industry data, real reviews, forums), not from imagination.
4. Always give a control: one real competing offer with its real price and, if it can be found, one real number about it. Without a control the result is words only.
5. The panel is weak evidence (evidence level 1 in `business-simulator`). It moves a researched number up or down; it never replaces it and it never proves demand.
6. Report in the user's language, plain words, short sentences, no em dashes, no hedging.

## Workflow

1. Research the inputs
2. Write `panel.json`
3. Get the tool and deal the cards
4. Run the buyers
5. Run the raters
6. Tally and read the result
7. Report (and hand the snippet to `business-simulator` if that is where you came from)

### Step 1: Research the inputs

- Who actually meets the offer (the reachable people, not everybody): 2 to 5 segments with shares that add up to 1, an age range and an income range each (for businesses as buyers: size and revenue ranges, and the buyer is the person who decides). Source every share.
- What they do today about this need (the status quo is the real competitor), with shares: buys from X, does it themselves, nothing.
- At least 8 real objections and 6 real habits, taken from reviews and forum threads about the competing offers.
- The control: one real offer these buyers already know. Its pitch as buyers see it, its real price, and if possible one real number: the share of this population that buys it in a typical month, or its conversion rate (`known.value`, with source).
- Other real alternatives with prices (1 to 3).
- The offer exactly as a buyer would meet it: 3 to 6 plain lines, no sales talk the real thing would not have, the price, and how they come across it (walk past, ad, cold email, a friend).

### Step 2: Write `panel.json`

```json
{
 "business": "Coffee kiosk next to a commuter rail station", "currency": "$", "n": 100, "seed": 11,
 "context": "You walk past it on a weekday morning on your way to the train. There is a short line.",
 "population": {
  "age": [18, 68],
  "segments": [
   {"name": "weekday commuter", "share": 0.5, "about": "Takes the train to an office job 3 to 5 days a week.", "ranges": {"household income per year": [55000, 160000]}},
   {"name": "college student", "share": 0.3, "about": "Lives nearby, has classes most days.", "age": [18, 26], "ranges": {"household income per year": [8000, 30000]}},
   {"name": "neighborhood resident", "share": 0.2, "about": "Lives within a ten minute walk.", "ranges": {"household income per year": [40000, 190000]}}],
  "today": [{"name": "buys coffee from a chain most weekdays", "share": 0.3}, {"name": "brews coffee at home and carries it", "share": 0.4},
            {"name": "regular at an independent cafe", "share": 0.15}, {"name": "rarely drinks coffee", "share": 0.15}],
  "traits": [{"name": "How you treat small daily spending", "levels": [{"name": "you track every dollar", "share": 0.35}, {"name": "you notice prices but treat yourself", "share": 0.45}, {"name": "you do not think about small purchases", "share": 0.2}]}],
  "objections": ["The line will make me miss my train", "Coffee from a kiosk is usually burnt", "Five dollars a day adds up fast"],
  "habits": ["Leaves home at the last possible minute", "Orders ahead on an app when possible", "Carries a reusable cup"]},
 "offer": {"name": "Platform Coffee kiosk", "pitch": "PLATFORM COFFEE. Espresso drinks made to order in under 90 seconds.\nLocal roaster beans. Order ahead by text.\n12 oz latte.",
           "price": 5.5, "price_label": "per latte", "price_cells": [4.75, 5.5, 6.5], "period": "the next month"},
 "control": {"name": "The chain cafe across the street", "pitch": "The national chain cafe across the street. Mobile order ahead. Usually a 4 to 6 minute wait at rush hour.\n12 oz latte.",
             "price": 5.25, "price_label": "per latte", "known": {"value": [0.2, 0.28, 0.36], "source": "..."}},
 "alternatives": [{"name": "Gas station coffee two blocks away", "price": "$2.25", "one_line": "self serve drip coffee"}],
 "atar": {"population": [2500, 3500, 5000], "awareness": [0.5, 0.7, 0.9], "availability": 1.0, "uses_per_month": [3, 6, 10]}
}
```

Fields: `population.segments[]` (`name`, `share`, `about`, optional `age`, `ranges` of numeric facts drawn per buyer, optional own `today` list), `population.today`, `traits` (any extra dealt dimension), `objections`, `habits` (strings or `{name, share}`). `offer` (`name`, `pitch`, `price`, `price_label`, optional `price_cells` and `one_line`, `period` = the time window of the buying question; for a rare purchase use "the next 3 months"). `control` (same fields plus `known.value` as a number or `[low, likely, high]`). `alternatives`. Optional: `frequency_question`, `intent_weights` (5 numbers), `reliability` (default 0.5), `elasticity_prior` (default -1.8), `atar` (reachable people, share who will know it within a year, share who can get it, purchases per month per buyer: a bottom-up cross-check). Every buyer also gets an openness-to-new-things card dealt 16 / 34 / 34 / 16 (early, early majority, late majority, laggard).

Who sees what (set by the tool): with 100 buyers, 35 see the offer at the planned price, about 17 each at a lower and a higher price (default 15% lower and 20% higher, or `price_cells`), and 30 see the control. Each buyer sees one price only, so price answers are not anchored. Under 40 buyers there are no price cells: 60% see the offer, 40% the control. All buyers finish with the same real choice: the offer, the control, the alternatives, keep today's way, or nothing.

### Step 3: Get the tool and deal

```bash
F=$(find / -path /proc -prune -o -name SKILL.md -path "*customer-panel*" -print 2>/dev/null | head -1)
python3 - "$F" <<'EOF'
import sys
t = open(sys.argv[1]).read()
code = t.rsplit("# === PANEL START ===", 1)[1].split("# === PANEL END ===", 1)[0]
open("panel.py", "w").write(code)
print("panel.py written", len(code), "chars")
EOF
python3 panel.py selftest                      # must end with SELFTEST OK
python3 panel.py deal panel.json --out panel   # add --n 20 for a quick look
```

If no file is found, write `panel.py` from the appendix exactly. Needs numpy. `deal` writes `panel/buyers.json` and one brief per buyer in `panel/briefs/`. Read two briefs to check that they sound like real people meeting the real thing.

### Step 4: Run the buyers

One sub-agent per buyer, in waves of 10 (ten Agent calls in one message, `run_in_background: false`). The prompt of each call is the full text of that buyer's brief file and nothing else: no summary of the business, no hint of the answer you hope for. Each buyer saves its answer into `panel/inbox/` and also replies with it. After every wave:

```bash
python3 panel.py save panel        # files what is in the inbox, lists rejects and who is still missing
```

If a buyer could not write its file, paste its raw reply instead: `python3 panel.py save panel - <<'EOF' ... EOF`. A rejected answer means that buyer is asked again with the same brief; never repair an answer by hand. No Agent tool: stop here and say that the panel cannot be run honestly in this session; one mind playing 100 buyers gives 100 copies of the same opinion.

### Step 5: Run the raters

```bash
python3 panel.py rate panel        # writes panel/rater_briefs/R01.txt ... (10 answers each)
```

One sub-agent per rater brief, again in one message, prompt = the brief file. Raters see only each buyer's own words and box them. This words first, box second step is what makes simulated answers spread like real ones. Then `python3 panel.py save panel` again. For a 20 buyer quick look the raters can be skipped (the tally says so).

### Step 6: Tally and read the result

```bash
python3 panel.py tally panel       # prints panel_report.md, writes panel_summary.json
```

How the numbers are made, so you can explain them:
- Calibrated intent: each box counts as a purchase chance. Definitely 0.75, probably 0.25, maybe 0.10, probably not 0.05, definitely not 0.02. For expensive or rarely bought things real conversion is lower still; the comparison below is not affected.
- Appeal vs control: calibrated intent for the offer at the planned price divided by calibrated intent for the control, with a 90% interval from resampling the buyers. 1.00 means the same pull as the control.
- `panel_factor`: that ratio pulled halfway to 1 on a log scale (reliability 0.5) and limited to 0.5 to 1.5. This is the only number that goes into the business model, as a multiplier on demand that was researched from benchmarks.
- Anchor-calibrated buyer share: the control's real share times the factor. This is the honest absolute number, and it still assumes people know the offer and can get it.
- Price slope: how calibrated intent falls across the three price cells, pulled halfway to a prior of -1.8 (a single small firm usually sits between -0.8 and -3). A slope of -2 means 10% more price loses about 20% of buyers.
- Price feel (Van Westendorp): the range between "too cheap to trust" and "too expensive", from the buyers who saw the offer.
- Bottom-up cross-check (`atar`): reachable people times share who know it times share who can get it times buyer share times purchases per month.

Reading it:
- Appeal under 0.7: the offer loses to what people already use. Fix the offer or choose the segment where it wins before spending.
- 0.7 to 1.3: about the same pull. The business then lives or dies on location, awareness and execution, not on the idea.
- Over 1.3: stronger on paper. A new name still has no awareness, no reviews and no habit on its side.
- Reasons for no and "what would change my mind": the top one or two are the cheapest improvements. Change the offer once, deal again with the same seed into a new folder, copy the control buyers' files from the first run's `answers/` (they saw the same thing), run only the offer buyers, tally again and compare the appeal. One round only; a second round fits the offer to the panel instead of to people.
- Health flags: TOO AGREEABLE (levels inflated; only the comparison counts), FLAT PANEL or SEGMENTS DO NOT MATTER (cards too vague: sharpen and deal again), PRICE-INSENSITIVE PANEL (ignore its price answers), COPIED ANSWERS (buyers were not separate agents), NO CONTROL (words only), SELF-RATED ONLY.

### Step 7: Report

Short, in the user's language:
1. One line: how the offer pulls against the control and how sure that is (the interval).
2. Who buys and who does not (segments, what they do today).
3. The top 3 reasons for no, each with one real quote from the panel.
4. What would flip the maybes.
5. Price: the slope in plain words and the acceptable range.
6. What this means for demand (the factor, and the anchor-calibrated share if there is one).
7. What it does not prove, and the cheapest real test that would (a pre-sale, a pop-up day, a landing page with real ad spend), with the pass line written before the test. `business-simulator` computes that line (`bizsim.py testplan`).

When called from `business-simulator`: hand back the config snippet from the report (`panel_factor`, `price_elasticity`, `reference_price`), the top reasons for no, the weak segments, the bottom-up cross-check and the health flags.

---

## Appendix: tool `panel.py`

Copy everything between the two marker lines into `panel.py` (or use the extraction script in Step 3). Do not edit it for a single case. After writing it, `python3 panel.py selftest` must end with `SELFTEST OK`.

```python
# === PANEL START ===
#!/usr/bin/env python3
"""panel: a panel of simulated buyers for one offer, read against a real competing offer.

  python3 panel.py deal SPEC.json --out DIR [--n 100] [--seed 11]   deal buyer cards and write one brief per buyer
  python3 panel.py save DIR [FILE|-]    file the replies: reads DIR/inbox/ (where sub-agents save), plus FILE or stdin (-) with pasted replies
  python3 panel.py check DIR            who is still missing
  python3 panel.py rate DIR             write briefs for independent raters (they box each buyer's own words)
  python3 panel.py tally DIR            calibrated result, price response, segments, reasons, config snippet
  python3 panel.py selftest             must end with SELFTEST OK

Needs numpy. Simulated buyers say yes too easily, so levels are never used raw: intent boxes are weighted down to
purchase rates, the offer is read RELATIVE to a real control offer, and the result is pulled halfway back to 1.
"""
import argparse, json, math, os, sys
import numpy as np

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

def pert(v, n, rng):
    """v is a number or [low, likely, high]."""
    if not isinstance(v, list):
        return np.full(n, float(v))
    a, m, b = (float(x) for x in v)
    if b - a < 1e-12:
        return np.full(n, m)
    return a + (b - a) * rng.beta(1 + 4 * (m - a) / (b - a), 1 + 4 * (b - m) / (b - a), n)

INTENTS = ("definitely", "probably", "maybe", "probably_not", "definitely_not")
INTENT_W = (0.75, 0.25, 0.10, 0.05, 0.02)  # stated intent -> real purchase (Jamieson and Bass 1989 style weights)
REASONS = ("price", "habit", "need", "trust", "quality", "convenience", "timing", "values", "other")
ADOPTION = (("tries new things before most people do", 0.16), ("tries new things once a few people vouch for them", 0.34),
            ("switches only after most people already have", 0.34), ("keeps what works and changes only when forced", 0.16))

def quota(shares, n):
    """Whole-number head counts that add up to n and follow the shares (largest remainder)."""
    s = np.asarray(shares, float)
    raw = s / s.sum() * n
    base = np.floor(raw + 1e-9).astype(int)
    for j in np.argsort(-(raw - base), kind="stable")[: n - int(base.sum())]:
        base[j] += 1
    return [int(x) for x in base]

def _num(x):
    try:
        v = float(str(x).replace("$", "").replace(",", "").strip())
        return v if math.isfinite(v) else None
    except (TypeError, ValueError):
        return None

def _money(spec, o, price):
    if price is None:
        return o.get("price_text", "")
    return f"{spec.get('currency', '$')}{price:,.2f}".replace(".00", "") + (" " + o["price_label"] if o.get("price_label") else "")

def panel_cells(spec, n):
    """Who sees what: the offer at 3 prices (planned price double weight) and, if given, a real control offer (30%)."""
    off, ctl = spec["offer"], spec.get("control")
    p = _num(off.get("price"))
    prices = [None] if p is None else sorted({float(x) for x in (off.get("price_cells") or ([round(p * 0.85, 2), p, round(p * 1.2, 2)] if n >= 40 else [p]))} | {p})
    share_c = (0.3 if n >= 40 else 0.4) if ctl else 0.0
    w = [2.0 if (x is not None and abs(x - p) < 1e-9) else 1.0 for x in prices]
    cells = [("offer", x, (1 - share_c) * wi / sum(w)) for x, wi in zip(prices, w)]
    if ctl:
        cells.append(("control", _num(ctl.get("price")), share_c))
    return cells, quota([c[2] for c in cells], n)

def panel_deal(spec, n=None, seed=None):
    """Deal buyer cards by quota from researched shares. Code deals, not the model, so the panel is not biased toward agreeable people."""
    import random
    n = int(n or spec.get("n", 100))
    rnd = random.Random(int(seed if seed is not None else spec.get("seed", 11)))
    pop = spec["population"]
    segs = pop["segments"]
    nm = lambda it: it["name"] if isinstance(it, dict) else str(it)
    def deal(items, k):
        out = []
        for it, q in zip(items, quota([it.get("share", 1) if isinstance(it, dict) else 1 for it in items], k)):
            out += [it] * q
        rnd.shuffle(out)
        return out
    dims = {"today": pop.get("today") or [], "adoption": [dict(name=a, share=s) for a, s in ADOPTION]}
    for tr in pop.get("traits") or []:
        dims["trait:" + tr["name"]] = tr["levels"]
    glob = {k: deal(v, n) for k, v in dims.items() if v}
    obj, hab = deal(pop.get("objections") or ["none in particular"], n), deal(pop.get("habits") or ["none in particular"], n)
    cells, counts = panel_cells(spec, n)
    order = [j for _, j in sorted(((i + 0.5) / c, j) for j, c in enumerate(counts) for i in range(c))]  # spread cells evenly over segments
    cards, i = [], 0
    for sg, c in zip(segs, quota([s.get("share", 1) for s in segs], n)):
        loc = deal(sg["today"], c) if sg.get("today") else None
        for j in range(c):
            card = dict(id=f"P{i + 1:03d}", segment=sg["name"], about=sg.get("about", ""))
            a = sg.get("age") or pop.get("age")
            if a:
                card["age"] = rnd.randint(int(a[0]), int(a[1]))
            card["facts"] = {lab: float(f"{rnd.uniform(float(rg[0]), float(rg[1])):.2g}") for lab, rg in (sg.get("ranges") or {}).items()}
            card["today"] = nm(loc[j]) if loc else (nm(glob["today"][i]) if "today" in glob else "")
            card["adoption"] = glob["adoption"][i]["name"]
            card["traits"] = {k[6:]: nm(v[i]) for k, v in glob.items() if k.startswith("trait:")}
            card["objection"], card["habit"] = nm(obj[i]), nm(hab[i])
            card["shown"], card["price"] = cells[order[i]][0], cells[order[i]][1]
            cards.append(card)
            i += 1
    return cards

def panel_options(spec, card):
    """The real choice set: the offer, the control, other real alternatives, staying with today's way, or nothing."""
    off, ctl = spec["offer"], spec.get("control")
    op = card["price"] if card["shown"] == "offer" else _num(off.get("price"))
    opts = [("offer", f"{off['name']} at {_money(spec, off, op)}" + ("" if card["shown"] == "offer" else f": {off.get('one_line', off['pitch'].strip().splitlines()[0])}"))]
    if ctl:
        opts.append(("control", f"{ctl['name']} at {_money(spec, ctl, _num(ctl.get('price')))}" + ("" if card["shown"] == "control" else f": {ctl.get('one_line', ctl['pitch'].strip().splitlines()[0])}")))
    for j, al in enumerate(spec.get("alternatives") or []):
        opts.append((f"alt{j + 1}", f"{al['name']}" + (f" at {al['price']}" if al.get("price") else "") + (f": {al['one_line']}" if al.get("one_line") else "")))
    return opts + [("current", "keep doing exactly what I do today"), ("none", "none of these, and I stop spending on this need")]

def panel_brief(card, spec):
    off, ctl = spec["offer"], spec.get("control")
    shown = off if card["shown"] == "offer" else ctl
    period = off.get("period", "the next month")
    L = ["You are one real person answering a short market survey. Stay in character and answer only as this person. This is role play: do not research anything and do not use tools except to save your answer.", "", "WHO YOU ARE",
         f"- {card['segment']}" + (f", {card['age']} years old" if card.get("age") else "") + (f". {card['about']}" if card.get("about") else "")]
    L += [f"- {k}: about {v:,.0f}" for k, v in (card.get("facts") or {}).items()]
    if card.get("today"):
        L.append(f"- What you do today about this: {card['today']}")
    L.append(f"- With new things you are someone who {card['adoption']}")
    L += [f"- {k}: {v}" for k, v in card["traits"].items()]
    L += [f"- A doubt you tend to have about offers like this: {card['objection']}", f"- A habit of yours: {card['habit']}", "",
          "HOW YOU COME ACROSS IT", spec.get("context", "You come across it during a normal week."), "",
          "WHAT YOU SEE", shown["pitch"].strip(), "Price: " + _money(spec, shown, card["price"]), "",
          "BE HONEST", "Most people do not buy most things. A polite yes is worse than a clear no. Money spent here is money you cannot spend on something else. "
          "If what you do today works, switching costs you effort. Do not be agreeable. Do not invent facts that are not on this page.", "",
          "WHAT ELSE EXISTS (for the last question)"]
    L += [f'- "{k}": {txt}' for k, txt in panel_options(spec, card)]
    L += ["", "ANSWER", "First write what you would really do, in your own words. Then fill in the rest so that it matches what you wrote.",
          "Reply with ONLY this JSON object and nothing else:", "{", f' "id": "{card["id"]}",',
          ' "in_my_words": "2 to 4 sentences in the first person: what I would really do when I come across this, and why",',
          f' "intent": "definitely | probably | maybe | probably_not | definitely_not (would I pay for what I was shown within {period}? definitely = yes without anyone pushing me, probably = more likely than not, maybe = could go either way)",',
          f' "uses_per_month": "a number: {spec.get("frequency_question", "how many times I would buy in a typical month if I became a customer, 0 if never")}",',
          ' "reason_code": "price | habit | need | trust | quality | convenience | timing | values | other (the main reason for my answer)",',
          ' "objection": "the main thing holding me back, in my words, or none",', ' "would_change_my_mind": "the one change that would move me, or nothing",',
          ' "choice": "one key from WHAT ELSE EXISTS: what I really do with all of these available"' + ("," if card["shown"] == "offer" and card["price"] is not None else "")]
    if card["shown"] == "offer" and card["price"] is not None:
        L += [' "price_too_cheap": "number: below this price I would doubt the quality",', ' "price_cheap": "number: a bargain, a great buy",',
              ' "price_expensive": "number: getting expensive, but I would still consider it",', ' "price_too_expensive": "number: too expensive, I would not consider it"']
    L += ["}"]
    if spec.get("_inbox"):
        L += ["", f"Also save that same JSON object as the file {os.path.join(spec['_inbox'], card['id'] + '.json')} (write the file, then reply with the JSON)."]
    return "\n".join(L) + "\n"

def _json_objects(text):
    """Every JSON object found in free text (sub-agents sometimes wrap their JSON in prose or code fences)."""
    out, depth, start, ins, esc = [], 0, 0, False, False
    for i, ch in enumerate(text):
        if ins:
            if esc:
                esc = False
            elif ch == "\\":
                esc = True
            elif ch == '"':
                ins = False
        elif ch == '"' and depth > 0:
            ins = True
        elif ch == "{":
            if depth == 0:
                start = i
            depth += 1
        elif ch == "}" and depth > 0:
            depth -= 1
            if depth == 0:
                try:
                    out.append(json.loads(text[start:i + 1]))
                except ValueError:
                    pass
    return out

def _pload(d):
    spec = json.load(open(os.path.join(d, "panel.json")))
    cards = json.load(open(os.path.join(d, "buyers.json")))
    ans = {}
    for c in cards:
        fp = os.path.join(d, "answers", c["id"] + ".json")
        if os.path.exists(fp):
            ans[c["id"]] = json.load(open(fp))
    rp = os.path.join(d, "ratings.json")
    return spec, cards, ans, (json.load(open(rp)) if os.path.exists(rp) else {})

def panel_save(d, text):
    """File raw sub-agent replies: buyer answers into answers/, rater boxes into ratings.json. Invalid ones are rejected with the reason."""
    spec, cards, ans, rat = _pload(d)
    ib = os.path.join(d, "inbox")
    for fn in sorted(os.listdir(ib)) if os.path.isdir(ib) else []:
        try:
            text += "\n" + open(os.path.join(ib, fn), encoding="utf-8", errors="replace").read()
        except OSError:
            pass
    by = {c["id"]: c for c in cards}
    os.makedirs(os.path.join(d, "answers"), exist_ok=True)
    saved, rated, bad = [], [], []
    norm = lambda x: str(x).strip().lower().replace("-", "_").replace(" ", "_")
    for a in _json_objects(text):
        c = by.get(str(a.get("id", "")).strip().upper())
        if c is None:
            bad.append(f"{a.get('id')}: unknown id")
            continue
        a["id"], a["intent"] = c["id"], norm(a.get("intent"))
        if "choice" in a:
            a["choice"] = norm(a["choice"])
        if "in_my_words" not in a:  # a rater's box
            if a.get("intent") in INTENTS:
                rat[c["id"]] = a["intent"]; rated.append(c["id"])
            else:
                bad.append(f"{c['id']}: rating intent must be one of {INTENTS}")
            continue
        errs = []
        if a.get("intent") not in INTENTS:
            errs.append("intent must be exactly one of " + " / ".join(INTENTS))
        if not str(a.get("in_my_words", "")).strip():
            errs.append("in_my_words is empty")
        keys = [k for k, _ in panel_options(spec, c)]
        if a.get("choice") not in keys:
            errs.append(f"choice must be one of {keys}")
        if _num(a.get("uses_per_month")) is None:
            errs.append("uses_per_month must be a number")
        if a.get("reason_code") not in REASONS:
            a["reason_code"] = "other"
        if errs:
            bad.append(f"{c['id']}: " + "; ".join(errs))
            continue
        json.dump(a, open(os.path.join(d, "answers", c["id"] + ".json"), "w"), indent=1)
        ans[c["id"]] = a; saved.append(c["id"])
    json.dump(rat, open(os.path.join(d, "ratings.json"), "w"), indent=1)
    left = [c["id"] for c in cards if c["id"] not in ans]
    print(f"saved {len(saved)} answers, {len(rated)} ratings. rejected {len(bad)}.")
    for b in bad:
        print("  REJECTED", b, "(ask that same buyer again with the same brief; never fix an answer yourself)")
    print(f"answers on file {len(ans)}/{len(cards)}; ratings on file {len(rat)}. still waiting for: {', '.join(left[:40]) or 'nobody'}")

def panel_rate(d, size=10):
    """Briefs for independent raters: they read each buyer's own words and pick the box. Text first, box second, beats asking for a number."""
    spec, cards, ans, rat = _pload(d)
    todo = [c for c in cards if c["id"] in ans and c["id"] not in rat]
    os.makedirs(os.path.join(d, "rater_briefs"), exist_ok=True)
    period = spec["offer"].get("period", "the next month")
    files = []
    for w in range(0, len(todo), size):
        L = ["You rate survey answers. For each line, read what the person said they would do about the thing they were shown and pick the ONE box that matches their words.",
             "Judge only the words. Do not guess what a typical person would do. Hedged or polite interest without a clear plan to pay is 'maybe' at most.",
             f"Boxes (would they pay for it within {period}?): definitely = clear yes without pushing; probably = more likely than not; maybe = could go either way; probably_not = unlikely; definitely_not = clear no.",
             'Reply with ONLY a JSON list, one item per line below, like [{"id": "P001", "intent": "maybe"}].'
             + (f" Also save that list as the file {os.path.join(spec['_inbox'], f'R{w // size + 1:02d}.json')}." if spec.get("_inbox") else ""), ""]
        for c in todo[w:w + size]:
            o = spec["offer"] if c["shown"] == "offer" else spec["control"]
            L.append(f'{c["id"]} (was shown {o["name"]} at {_money(spec, o, c["price"])}): "{str(ans[c["id"]]["in_my_words"]).strip()}"')
        fp = os.path.join(d, "rater_briefs", f"R{w // size + 1:02d}.txt")
        open(fp, "w").write("\n".join(L) + "\n")
        files.append(fp)
    print(f"{len(todo)} answers need a rating. Rater briefs ({len(files)}): " + ", ".join(files))

def van_westendorp(rows):
    """rows of (too_cheap, cheap, expensive, too_expensive). Classic points: PMC, OPP, IPP, PME. Out-of-order answers are dropped."""
    a = np.array([r for r in rows if all(v is not None for v in r) and r[0] <= r[1] <= r[2] <= r[3]], float)
    if len(a) < 8:
        return None
    grid = np.linspace(a.min(), a.max(), 2001)[:, None]
    tc, ch = (a[:, 0][None, :] >= grid).mean(1), (a[:, 1][None, :] >= grid).mean(1)   # falling curves
    ex, te = (a[:, 2][None, :] <= grid).mean(1), (a[:, 3][None, :] <= grid).mean(1)   # rising curves
    def cross(fall, rise):  # step curves can be equal over a stretch: take its middle
        i, k = np.where(fall - rise <= 0)[0], np.where(fall - rise >= 0)[0]
        if not i.size:
            return None
        return float((grid[i[0], 0] + grid[max(k[-1], i[0]), 0]) / 2) if k.size else float(grid[i[0], 0])
    return dict(n=int(len(a)), dropped=int(len(rows) - len(a)), too_cheap_below=cross(tc, 1 - ch), optimal=cross(tc, te), indifference=cross(ch, ex), too_expensive_above=cross(1 - ex, te))

def _price_fit(groups, plan):
    """groups: [(price, weights array)]. Log-log slope of calibrated intent on price (elasticity) and the fitted level at the planned price."""
    pts = [(p, float(g.mean()), len(g)) for p, g in groups if p is not None and len(g) and g.mean() > 0]
    if plan is not None and len({p for p, _, _ in pts}) >= 2:
        e, a0 = np.polyfit(np.log([p for p, _, _ in pts]), np.log([m for _, m, _ in pts]), 1, w=np.sqrt([k for _, _, k in pts]))
        return float(e), float(math.exp(a0 + e * math.log(plan)))
    allw = np.concatenate([g for _, g in groups]) if groups else np.array([])
    return None, (float(allw.mean()) if allw.size else float("nan"))

def panel_tally(d, boot=2000):
    spec, cards, ans, rat = _pload(d)
    W = dict(zip(INTENTS, spec.get("intent_weights", INTENT_W)))
    rel = float(spec.get("reliability", 0.5))   # how much of the panel's signal to believe; the rest is pulled back to the benchmark
    off, ctl = spec["offer"], spec.get("control")
    plan = _num(off.get("price"))
    rows = []
    for c in cards:
        a = ans.get(c["id"])
        if a:
            box = rat.get(c["id"], a["intent"])
            rows.append(dict(c, self_box=a["intent"], box=box, w=W[box], uses=_num(a.get("uses_per_month")) or 0.0, reason=a.get("reason_code", "other"), a=a))
    if len(rows) < 10:
        raise SystemExit(f"only {len(rows)} answers on file; need at least 10 to tally")
    flags = []
    def stat(rs):
        b = [r["box"] for r in rs]
        return dict(n=len(rs), calibrated=float(np.mean([r["w"] for r in rs])) if rs else None, top2=float(np.mean([x in INTENTS[:2] for x in b])) if rs else None,
                    boxes={k: b.count(k) for k in INTENTS})
    orow, crow = [r for r in rows if r["shown"] == "offer"], [r for r in rows if r["shown"] == "control"]
    prices = sorted({r["price"] for r in orow if r["price"] is not None})
    cells = [dict(shown="offer", price=p, **stat([r for r in orow if r["price"] == p])) for p in (prices or [None])]
    if crow:
        cells.append(dict(shown="control", price=_num(ctl.get("price")), **stat(crow)))
    G = [(p, np.array([r["w"] for r in orow if r["price"] == p])) for p in (prices or [None])]
    C = np.array([r["w"] for r in crow])
    e_hat, at_plan = _price_fit(G, plan)
    rai = at_plan / C.mean() if C.size and C.mean() > 0 else None
    rng = np.random.default_rng(int(spec.get("seed", 11)))
    be, br, ba = [], [], []
    for _ in range(boot):
        e2, a2 = _price_fit([(p, rng.choice(g, len(g))) for p, g in G if len(g)], plan)
        ba.append(a2)
        if e2 is not None:
            be.append(e2)
        if C.size:
            c2 = rng.choice(C, len(C)).mean()
            if c2 > 0:
                br.append(a2 / c2)
    ci = lambda x: [float(v) for v in np.percentile(x, [5, 95])] if len(x) > 20 else None
    res = dict(n_buyers=len(cards), n_answers=len(rows), rated_by_independent_raters=sum(1 for r in rows if r["id"] in rat), cells=cells,
               calibrated_intent_at_planned_price=at_plan, calibrated_intent_interval=ci(ba), intent_weights=W, reliability=rel)
    suggest = {}
    if rai is not None:
        lo, hi = ci(br) or [rai / 1.5, rai * 1.5]
        sh = lambda x: math.exp(rel * math.log(max(x, 1e-6)))
        pf = [float(np.clip(sh(lo) / 1.15, 0.4, 1.6)), float(np.clip(sh(rai), 0.5, 1.5)), float(np.clip(sh(hi) * 1.15, 0.4, 1.6))]
        pf = [round(min(pf[0], pf[1]), 2), round(pf[1], 2), round(max(pf[2], pf[1]), 2)]
        res.update(relative_appeal_vs_control=float(rai), relative_appeal_interval=[lo, hi], control_calibrated_intent=float(C.mean()))
        suggest["panel_factor"] = dict(dist="pert", low=pf[0], mode=pf[1], high=pf[2], ev="C",
                                       source=f"synthetic panel n={len(rows)}: appeal vs {ctl['name']} = {rai:.2f} (90% interval {lo:.2f} to {hi:.2f}), pulled toward 1 with reliability {rel}")
        known = (ctl.get("known") or {}).get("value")
        if known is not None:
            kv = known if isinstance(known, list) else [known, known, known]
            res["anchor_calibrated_buyer_share"] = dict(low=kv[0] * pf[0], mode=kv[1] * pf[1], high=kv[2] * pf[2],
                                                        meaning="real share for the control (researched) x relative appeal: share of people who know the offer and can reach it who buy in the period")
    else:
        res["absolute_only"] = dict(upper_bound=at_plan, suggested=at_plan * 0.5,
                                    note="no control offer: the level is an upper bound; halve it and treat the panel as weak evidence")
        flags.append("NO CONTROL: without a real offer with known results the panel cannot be calibrated. Use it for reasons and price direction only.")
    if e_hat is not None:
        prior = float(spec.get("elasticity_prior", -1.8))
        if e_hat >= 0:
            flags.append(f"PRICE-INSENSITIVE PANEL: intent did not fall as price rose (slope {e_hat:+.2f}). Ignore the panel's price answers; the prior elasticity is used.")
        es = prior + rel * (min(e_hat, 0.0) - prior) if e_hat < 0 else prior
        es = float(np.clip(es, -4.0, -0.3))
        res.update(price_elasticity_raw=e_hat, price_elasticity_interval=ci(be), price_elasticity_prior=prior)
        suggest["price_elasticity"] = dict(dist="pert", low=round(max(-4.5, es * 1.6), 2), mode=round(es, 2), high=round(min(-0.2, es * 0.5), 2), ev="C",
                                           source=f"panel price cells {prices}: raw slope {e_hat:+.2f}, pulled toward prior {prior} with reliability {rel}")
        suggest["reference_price"] = plan
    res["suggested_config"] = suggest
    # who buys, why, why not
    def table(key):
        out = []
        for name in sorted({str(r[key]) for r in orow}):
            rs = [r for r in orow if str(r[key]) == name]
            out.append(dict(group=name, **{k: v for k, v in stat(rs).items() if k != "boxes"}))
        return sorted(out, key=lambda x: -(x["calibrated"] or 0))
    res["by_segment"], res["by_today"], res["by_adoption"] = table("segment"), table("today"), table("adoption")
    yes = [r for r in orow if r["box"] in INTENTS[:2]]
    no = [r for r in orow if r["box"] not in INTENTS[:2]]
    cnt = lambda rs: sorted(((k, sum(1 for r in rs if r["reason"] == k)) for k in REASONS), key=lambda x: -x[1])
    res["why_not"] = [dict(reason=k, count=v, share=v / max(len(no), 1), quotes=[str(r["a"].get("objection", ""))[:200] for r in no if r["reason"] == k][:3]) for k, v in cnt(no) if v]
    res["why_yes"] = [dict(reason=k, count=v, quotes=[str(r["a"].get("in_my_words", ""))[:200] for r in yes if r["reason"] == k][:2]) for k, v in cnt(yes) if v]
    res["would_flip"] = [dict(id=r["id"], segment=r["segment"], change=str(r["a"].get("would_change_my_mind", ""))[:200]) for r in orow if r["box"] in ("maybe", "probably_not")]
    keys = sorted({r["a"].get("choice") for r in rows})
    res["choice_share_all"] = {k: float(np.mean([r["a"].get("choice") == k for r in rows])) for k in keys}
    if crow:
        res["choice_share_offer_among_control_cell"] = float(np.mean([r["a"].get("choice") == "offer" for r in crow]))
    if yes:
        u = np.array([r["uses"] for r in yes])
        res["uses_per_month_likely_buyers"] = dict(median=float(np.median(u)), p25=float(np.percentile(u, 25)), p75=float(np.percentile(u, 75)),
                                                   note="people overstate frequency: use the lower half of this range or the category benchmark, whichever is lower")
    res["van_westendorp"] = van_westendorp([tuple(_num(r["a"].get("price_" + k)) for k in ("too_cheap", "cheap", "expensive", "too_expensive")) for r in orow if r["price"] is not None])
    if rat:
        both = [(INTENTS.index(r["self_box"]), INTENTS.index(r["box"])) for r in rows if r["id"] in rat]
        res["self_vs_rater"] = dict(same_box=float(np.mean([a == b for a, b in both])), within_one_box=float(np.mean([abs(a - b) <= 1 for a, b in both])),
                                    self_mean_weight=float(np.mean([INTENT_W[a] for a, _ in both])), rater_mean_weight=float(np.mean([INTENT_W[b] for _, b in both])))
    # is the panel itself healthy?
    allb = [r["box"] for r in rows]
    top = max(allb.count(k) for k in INTENTS) / len(allb)
    t2 = float(np.mean([x in INTENTS[:2] for x in [r["box"] for r in orow]])) if orow else 0.0
    if top > 0.6:
        flags.append(f"FLAT PANEL: {top:.0%} of answers sit in one box. The buyers are not different enough; sharpen segments and habits and deal again.")
    if t2 > 0.6:
        flags.append(f"TOO AGREEABLE: {t2:.0%} say definitely or probably. Real concept tests rarely pass 60%. Treat levels as inflated; only the comparison with the control is usable.")
    segc = [x["calibrated"] for x in res["by_segment"] if x["n"] >= 5]
    if len(segc) >= 2 and max(segc) - min(segc) < 0.15 * max(max(segc), 1e-9):
        flags.append("SEGMENTS DO NOT MATTER: every segment answered about the same. The persona cards are not driving answers.")
    txt = [str(r["a"].get("in_my_words", "")).strip().lower()[:80] for r in rows]
    if len(set(txt)) < 0.9 * len(txt):
        flags.append("COPIED ANSWERS: many answers start with the same words. Were the buyers run as separate sub-agents?")
    if len(rows) < len(cards):
        flags.append(f"INCOMPLETE: {len(cards) - len(rows)} buyers have no answer on file.")
    if not rat:
        flags.append("SELF-RATED ONLY: no independent raters were used (run 'panel rate'); self-chosen boxes cluster more.")
    res["flags"] = flags
    at = spec.get("atar")
    if at:  # bottom-up cross-check: people x aware x can get it x try it x how often
        r2 = np.random.default_rng(5)
        dr = lambda v: pert(v, 20000, r2)
        share = res.get("anchor_calibrated_buyer_share")
        trial = dr([share["low"], share["mode"], share["high"]]) if share else dr([at_plan * 0.25, at_plan * 0.5, at_plan])
        buyers = dr(at["population"]) * dr(at["awareness"]) * dr(at.get("availability", 1.0)) * trial
        uses = dr(at["uses_per_month"]) if at.get("uses_per_month") is not None else np.full(20000, float(res.get("uses_per_month_likely_buyers", {}).get("p25", 1.0)))
        res["atar_cross_check"] = dict(buyers_per_month={f"p{q}": float(np.percentile(buyers, q)) for q in (10, 50, 90)},
                                       sales_per_month={f"p{q}": float(np.percentile(buyers * uses, q)) for q in (10, 50, 90)},
                                       note="compare with the benchmark-based demand input; if they differ by more than 2x, find out why before trusting either")
    json.dump(res, open(os.path.join(d, "panel_summary.json"), "w"), indent=2)
    L = [f"# Customer panel: {spec.get('business', off['name'])} ({len(rows)} of {len(cards)} buyers answered; {res['rated_by_independent_raters']} rated independently)", "",
         "Synthetic buyers. Levels are inflated; the comparison with the control, the price direction, the segments and the reasons are the usable parts.", "",
         "## Who saw what", "| shown | price | n | calibrated intent | definitely or probably | boxes (def / prob / maybe / prob not / def not) |", "|---|---|---|---|---|---|"]
    for c in [c for c in cells if c["n"]]:
        o = off if c["shown"] == "offer" else ctl
        L.append(f"| {o['name']} | {_money(spec, o, c['price'])} | {c['n']} | {c['calibrated']:.1%} | {c['top2']:.0%} | {' / '.join(str(c['boxes'][k]) for k in INTENTS)} |")
    L += ["", "## Calibrated result"]
    if rai is not None:
        L.append(f"- Appeal vs {ctl['name']}: {rai:.2f} (90% interval {res['relative_appeal_interval'][0]:.2f} to {res['relative_appeal_interval'][1]:.2f}). 1.00 = same pull as the control.")
        if "anchor_calibrated_buyer_share" in res:
            s_ = res["anchor_calibrated_buyer_share"]
            L.append(f"- Anchor-calibrated buyer share (control's real share x relative appeal): low {s_['low']:.1%}, likely {s_['mode']:.1%}, high {s_['high']:.1%}")
    else:
        L.append(f"- No control. Calibrated intent at the planned price {at_plan:.1%} is an upper bound; use about {at_plan * 0.5:.1%}.")
    if e_hat is not None:
        iv = res["price_elasticity_interval"]
        L.append(f"- Price slope in the panel: {e_hat:+.2f} (1% higher price changes buyers by about {e_hat:+.2f}%)" + (f", 90% interval {iv[0]:+.2f} to {iv[1]:+.2f}" if iv else ""))
    vw = res["van_westendorp"]
    if vw:
        L.append(f"- Price feel (Van Westendorp, n={vw['n']}): too cheap below {fmt(vw['too_cheap_below'])}, optimal {fmt(vw['optimal'])}, indifference {fmt(vw['indifference'])}, too expensive above {fmt(vw['too_expensive_above'])}")
    if "uses_per_month_likely_buyers" in res:
        u_ = res["uses_per_month_likely_buyers"]
        L.append(f"- Stated frequency among likely buyers (the frequency question): median {fmt(u_['median'])} (P25 {fmt(u_['p25'])}, P75 {fmt(u_['p75'])}). Overstated; use the low end.")
    L.append("- Choice with everything on the table: " + ", ".join(f"{k} {v:.0%}" for k, v in sorted(res["choice_share_all"].items(), key=lambda x: -x[1])))
    if "atar_cross_check" in res:
        x = res["atar_cross_check"]
        L.append(f"- Bottom-up cross-check: buyers per month P10 {fmt(x['buyers_per_month']['p10'])} / P50 {fmt(x['buyers_per_month']['p50'])} / P90 {fmt(x['buyers_per_month']['p90'])}; "
                 f"sales per month P10 {fmt(x['sales_per_month']['p10'])} / P50 {fmt(x['sales_per_month']['p50'])} / P90 {fmt(x['sales_per_month']['p90'])}")
    L += ["", "## Put this into config.json (demand section)", "```json", json.dumps(suggest, indent=1), "```"]
    for title, key in (("By segment", "by_segment"), ("By what they do today", "by_today"), ("By openness to new things", "by_adoption")):
        L += ["", f"## {title} (offer cells)", "| group | n | calibrated intent | definitely or probably |", "|---|---|---|---|"]
        L += [f"| {x['group']} | {x['n']} | {x['calibrated']:.1%} | {x['top2']:.0%} |" for x in res[key]]
    L += ["", "## Why not (buyers who would not buy)"]
    for x in res["why_not"]:
        L.append(f"- {x['reason']}: {x['count']} ({x['share']:.0%}). " + " | ".join(f'"{q}"' for q in x["quotes"] if q))
    L += ["", "## Why yes"]
    L += [f"- {x['reason']}: {x['count']}. " + " | ".join(f'"{q}"' for q in x["quotes"] if q) for x in res["why_yes"]]
    fl = res["would_flip"]
    L += ["", "## What would flip a maybe or a probably not"] + [f"- {x['id']} ({x['segment']}): {x['change']}" for x in fl[::max(1, len(fl) // 12)][:12]]
    if "self_vs_rater" in res:
        v = res["self_vs_rater"]
        L += ["", f"## Raters vs self: same box {v['same_box']:.0%}, within one box {v['within_one_box']:.0%}; mean weight self {v['self_mean_weight']:.3f} vs raters {v['rater_mean_weight']:.3f}"]
    L += ["", "## Panel health"] + [f"- {f}" for f in (flags or ["no flags"])]
    open(os.path.join(d, "panel_report.md"), "w").write("\n".join(L) + "\n")
    print("\n".join(L))
    return res

def selftest():
    import tempfile
    res = []
    def chk(name, got, want, tol=1e-6):
        good = got is not None and abs(float(got) - want) <= tol
        res.append(good)
        print("PASS" if good else "FAIL", "|", name, "| got", None if got is None else round(float(got), 4), "| want", want)
    chk("quota: 7 people over 50/30/20 -> 4, 2, 1", float(quota([0.5, 0.3, 0.2], 7) == [4, 2, 1]), 1)
    vw = van_westendorp([(i, i + 2, i + 4, i + 6) for i in range(1, 21)])
    chk("price feel: optimal point of a symmetric panel", vw["optimal"], 13.5, 0.05)
    chk("price feel: indifference point of a symmetric panel", vw["indifference"], 13.5, 0.05)
    spec = {"business": "test", "n": 20, "seed": 1, "population": {"age": [20, 60], "segments": [{"name": "a", "share": 0.6, "ranges": {"income": [30000, 90000]}}, {"name": "b", "share": 0.4}],
            "today": [{"name": "x", "share": 0.5}, {"name": "y", "share": 0.5}]}, "offer": {"name": "New", "pitch": "A new thing.", "price": 10},
            "control": {"name": "Old", "pitch": "The old thing.", "price": 9, "known": {"value": 0.2}}}
    cards = panel_deal(spec)
    chk("deal: 20 buyers, 60/40 segments -> 12 in the first", float(sum(1 for c in cards if c["segment"] == "a")), 12)
    chk("deal: 40% of a small panel sees the control", float(sum(1 for c in cards if c["shown"] == "control")), 8)
    big = panel_deal(dict(spec, n=100))
    chk("deal: 100 buyers -> 35 see the planned price, 30 the control", float(sum(1 for c in big if c["shown"] == "offer" and c["price"] == 10) * 100 + sum(1 for c in big if c["shown"] == "control")), 3530)
    with tempfile.TemporaryDirectory() as d:
        json.dump(spec, open(os.path.join(d, "panel.json"), "w")); json.dump(cards, open(os.path.join(d, "buyers.json"), "w"))
        offer_ids = [c["id"] for c in cards if c["shown"] == "offer"]
        raw = []
        for c in cards:
            box = "Probably" if c["shown"] == "control" else ("definitely" if offer_ids.index(c["id"]) % 2 == 0 else "definitely not")
            raw.append(json.dumps({"id": c["id"], "in_my_words": f"words {c['id']}", "intent": box, "uses_per_month": 2, "reason_code": "price", "choice": "offer"}))
        out, sys.stdout = sys.stdout, open(os.devnull, "w")
        try:
            panel_save(d, "noise before\n```json\n" + "\n".join(raw) + "\n```\n trailing words")
            t = panel_tally(d, boot=200)
        finally:
            sys.stdout = out
    chk("save: all 20 replies filed despite code fences and loose spelling", t["n_answers"], 20)
    chk("tally: calibrated intent (6 x 0.75 + 6 x 0.02) / 12", t["calibrated_intent_at_planned_price"], 0.385, 1e-9)
    chk("tally: appeal vs control 0.385 / 0.25", t["relative_appeal_vs_control"], 1.54, 1e-9)
    chk("tally: factor pulled halfway to 1 on a log scale: sqrt(1.54)", t["suggested_config"]["panel_factor"]["mode"], 1.24, 0.005)
    chk("tally: anchor share 0.2 x 1.24", t["anchor_calibrated_buyer_share"]["mode"], 0.248, 1e-6)
    print("SELFTEST OK" if all(res) else f"SELFTEST FAILED: {res.count(False)} of {len(res)} checks")
    if not all(res):
        sys.exit(1)

def main():
    ap = argparse.ArgumentParser(prog="panel.py")
    ap.add_argument("cmd", choices=["deal", "save", "check", "rate", "tally", "selftest"])
    ap.add_argument("path", nargs="?", help="deal: the panel spec json. others: the panel folder")
    ap.add_argument("file", nargs="?", help="save: file with raw replies (default: stdin)")
    ap.add_argument("--out", default="panel")
    ap.add_argument("--n", type=int)
    ap.add_argument("--seed", type=int)
    ap.add_argument("--size", type=int, default=10)
    a = ap.parse_args()
    if a.cmd == "selftest":
        return selftest()
    if a.cmd == "deal":
        spec = json.load(open(a.path))
        if a.n:
            spec["n"] = a.n
        if a.seed is not None:
            spec["seed"] = a.seed
        cards = panel_deal(spec)
        for sub in ("briefs", "answers", "inbox"):
            os.makedirs(os.path.join(a.out, sub), exist_ok=True)
        spec["_inbox"] = os.path.abspath(os.path.join(a.out, "inbox"))
        json.dump(spec, open(os.path.join(a.out, "panel.json"), "w"), indent=1)
        json.dump(cards, open(os.path.join(a.out, "buyers.json"), "w"), indent=1)
        for c in cards:
            open(os.path.join(a.out, "briefs", c["id"] + ".txt"), "w").write(panel_brief(c, spec))
        cells, counts = panel_cells(spec, len(cards))
        print(f"dealt {len(cards)} buyers into {a.out}/ (briefs/P001.txt ...). Cells: " + ", ".join(f"{k} at {p}: {c}" for (k, p, _), c in zip(cells, counts)))
        print("segments: " + ", ".join(f"{s}: {sum(1 for c in cards if c['segment'] == s)}" for s in sorted({c['segment'] for c in cards})))
        print("next: give each brief, unchanged, to its own sub-agent (waves of 10). They save into inbox/. Then: panel.py save " + a.out)
    elif a.cmd in ("save", "check"):
        panel_save(a.path, "" if not a.file else (sys.stdin.read() if a.file == "-" else open(a.file).read()))
    elif a.cmd == "rate":
        panel_rate(a.path, a.size)
    else:
        panel_tally(a.path)


if __name__ == "__main__":
    main()
# === PANEL END ===
```