# Write-up

The full flow is in `diagram/process-flow.png`. In short: AI reads each proposal and extracts facts with proof quotes, code checks those facts and does the scoring, and a person reviews flags and makes the final decision.

## 1. AI vs. rules

**AI is used for one step only: reading the proposal and extracting facts.** Proposals are written in free text and every bidder words things differently ("14 installs since 2021", "nine projects of 100 kW or more"). Reading messy text is what AI is good at.

**Everything else is plain code:**
- Turning PDFs into text
- Checking the AI's quotes and missing fields
- Mandatory pass/fail checks
- Scoring and ranking

I drew the line here because once the facts are in a table, the rest is simple maths and yes/no checks. Code does that the same way every time, it is cheap, and it is easy to explain. If AI did the scoring, the same proposal could get a slightly different score each run, and it would be hard to say why.

Scoring rules I used (0–100 for each criterion, then weighted):
- **Experience (30%)**: qualifying installs ÷ highest count among eligible bids.
- **Price (25%)**: lowest price ÷ this price. Must be within the budget ceiling.
- **Timeline (15%)**: shortest timeline ÷ this timeline. Must be within the limit.
- **Warranty (15%)**: half panel, half workmanship, each compared to the best offer.
- **Team (15%)**: 25 points each for a named lead, a named crew, a safety plan and a monitoring plan.

## 2. Consistency and explainability

- Scoring is code, so the same facts always give the same score.
- The AI runs at temperature 0 with a fixed prompt and a fixed output format. The model name and prompt version are saved with each run.
- Facts are extracted and saved once. If someone re-ranks later, the system reuses the saved facts instead of asking the AI again, so the ranking stays the same.
- **Yes, ranking the same proposals twice gives the same result.** The only way it changes is if the extracted facts change, and that would show up in the record.

To see **why** a proposal ranked where it did, the reviewer sees for each proposal:
- The score for each criterion and the weight used
- The fact behind each score, with the quote from the proposal
- A one-line reason, e.g. "Rank 1: most experience (14 installs), full team plan, 12 weeks"

## 3. Human in the loop

A person is involved at three points:
1. **Admin** sets the criteria and weights before anything is scored.
2. **Reviewer** checks every flagged proposal. A proposal is flagged when a required fact is missing, a quote cannot be found in the text, or the proposal contains text trying to instruct the system. Flagged proposals are not ranked until a person decides.
3. **Reviewer** sees the final list and can override any result (status, rank or a fact), but must write a reason. Ineligible proposals are still shown so the reviewer can confirm them.

The tool never makes the award. A person does.

**Logged for audit, for every run:**
- Who uploaded, when, which category and which criteria version
- The extracted text and the AI output (facts + quotes)
- The model name and prompt version
- Each mandatory check result and each score part
- Every override: who, when, old value, new value, and the reason

## 4. Changing criteria

Criteria are versioned, never edited in place. When an admin changes a weight or a rule, a new version is created.

- Runs that were already scored keep their old version. Their results do not change silently.
- The reviewer sees a notice: "Criteria changed since this was scored (v3 → v4). Re-score?"
- Re-scoring with new **weights** is just a recalculation in code using the saved facts. No new AI call is needed.
- If the admin **adds a new criterion** that needs a new fact, the AI extraction runs again for that fact only.
- Both versions of the results are kept, so you can compare them.

## 5. Risks and safeguards

| Risk | Safeguard |
|---|---|
| **Wrong extraction** (AI reads a number wrong) | Every fact has a quote from the proposal. The reviewer can see it next to the value. |
| **Made-up facts** | Code checks that each quote really exists in the proposal text. If not, the fact is flagged. Missing facts must be `null`, never guessed. |
| **Manipulation** (text telling the AI what to do, like Proposal C's appendix) | The prompt treats the proposal as data only. Any such text is copied to `suspicious_text` and the proposal is flagged. Scoring is done by code, so this text can't change the score anyway. |
| **Unclear or missing info** (like Proposal D's timeline) | Missing required facts are flagged for a human instead of scored as zero or guessed. |
| **Bias** (AI favouring polished or confident writing) | AI only extracts facts, it does not judge quality. Scores come from the same formula for everyone. Company names are not used in scoring. |
| **Cost** | One AI call per proposal, and results are cached. Re-ranking and weight changes reuse saved facts. |
| **Over-trusting the tool** | The tool only ranks and explains. A person reviews flags and makes the award decision. |

## Sample set

A ranks 1st and E 2nd (see `ranking.json`). C is flagged for its appendix text, and its "since 2020" install count may include work older than five years. D is flagged because it has no timeline. B is ineligible because the licence belongs to an unnamed subcontractor, not the bidder.

**Assumptions:** "Last 5 years" means 2021 onwards. Price and timeline are only scored if they are within the budget and the 16-week limit; if not, the proposal is flagged.
