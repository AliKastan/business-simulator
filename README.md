# Business Simulator

Two Claude skills that tell you what would really happen to your money if you started a business. You don't get a hopeful spreadsheet. You get a tested Monte Carlo simulation checked against how real businesses of that kind actually do.

Ask Claude something like *"simulate this business: a coffee cart at the farmers market, $8k budget"* and you get back:

- **A computed call:** GO / TEST FIRST / NO-GO, from fixed rules
- **Years 1 to 5** of revenue and after-tax profit, in bad (P10), typical (P50) and good (P90) cases
- **Odds:** chance it's still open after 1, 3 and 5 years, chance of losing money, chance it beats a normal job
- **Money:** cash needed, break-even month, payback month, real hourly pay for the owner
- **What must be true:** the minimum sales needed to not lose money, next to what research says is realistic
- **A real-world test** with GO and STOP lines written before you run it
- Charts: cash fan, outcome histogram, survival curve

## Why it exists

Founders are systematically too optimistic, and writing financial projections makes it worse (Cassar 2010, *Strategic Management Journal*). This skill corrects for that:

- Every uncertain input is a range, never a single guess, and every input has a source.
- An optimism correction is applied unless demand comes from hard data (paying customers, a paid pilot).
- The model is held against the **outside view**: base rates for survival, revenue and profit in that industry. When the two disagree and the evidence is weak, the outside view wins.
- Simulated people never count as evidence of demand. Only real buyer behavior raises the evidence level.
- Claude never does the math in its head. Every number comes from the bundled engine, which has its own tests.

## What's inside

| Skill | What it does |
|---|---|
| `skills/business-simulator` | The main workflow: intake, research, a board of four critics, customer panel, config, simulation, calibration, stress tests, report. Includes the engine `bizsim.py` (Monte Carlo, US/CA tax schedules, loans, survival, sanity checks, test planner, selftest). |
| `skills/customer-panel` | 100 simulated buyers (20 in quick mode), each one its own sub-agent, tested against a **real competing offer**. Their answers get calibrated down, because simulated buyers say yes too easily. Includes `panel.py`. |

Both skills are self-contained. The Python tools are embedded in each `SKILL.md` and extracted at run time. The only dependency is `numpy`; `matplotlib` is optional and only needed for the charts.

## Install

**Claude (claude.ai / desktop):** zip each folder in `skills/` and upload it under Settings → Capabilities → Skills.

**Claude Code:**

```bash
git clone https://github.com/AliKastan/business-simulator
cp -r business-simulator/skills/* ~/.claude/skills/
```

Install both. `business-simulator` calls `customer-panel` in step 3C.

## Usage

- "Simulate this business: ..." / "Is this profitable?" / "bu işi yaparsam ne kazanırım"
- Say **quick** for a fast run. It skips the board and the audit, uses 20 buyers and 3,000 simulations.
- Answers come back in the language you write in, in plain words, with each calculation written out step by step.

Check that the engines work:

```bash
awk '/^# === BIZSIM START ===/{f=1;next}/^# === BIZSIM END ===/{f=0}f' skills/business-simulator/SKILL.md > bizsim.py
python3 bizsim.py selftest   # ends with SELFTEST OK
```

## Limits

This is a decision aid, not financial, tax or legal advice. The benchmark library was collected in November 2026 and the numbers age, so the skill re-checks them for each case. Tax schedules are US federal and California. The board and the panel are simulations. Real buyers overrule both.

## Credits

The board and buyer panel ideas were partly inspired by [founder-skill](https://github.com/Jakeschincariol/founder-skill) (MIT). The board lenses summarize published business frameworks and don't quote or imitate any real person.

## License

[MIT](LICENSE) © 2026 Ali Kastan
