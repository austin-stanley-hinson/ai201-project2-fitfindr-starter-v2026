# Acceptance criteria — FitFindr

Five criteria that say what "working" means for this agent, written in unit 3
**before** any results existed.

An acceptance criterion names a target: a number, a count, a rate, or something
a person could plainly observe. *"The agent handles errors"* is an opinion.
*"When search returns nothing, the agent stops before calling the second tool,
in 5 of 5 tries"* is a criterion.

Under each one, write a sentence or two on **why that target** and not a
stricter one. A reason that says something about your tools, your loop, or the
data earns credit; *"80% seemed reasonable"* does not.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

**Two are written for you. You write three.**

---

## 1. A matching query completes all three tools

Given a query that matches at least one listing, the agent completes all three
tool calls and returns a fit card — in at least 4 of 5 tries.

**Why this target:**
4 of 5 allows for occasional parsing failures or wording that plain keyword matching misses. For example, “small” might not match “S” under the exact-token size rule, and my regex parser only picks up a size written right after the word “size.” Two of the three tools also call the model, and a failed model call would end the run before the fit card.

---

## 2. An impossible query stops before the second tool

Given a query that matches no listings, the agent stops before calling
`suggest_outfit` and returns a message naming what to change — 5 of 5 tries.

**Why this target:**
5 of 5 is fair because the empty-results branch is deterministic. No model is called before the [] check, so the same empty results should trigger the same behavior every time.

---

## 3. State consistency

For a query that returns at least one item, outfit_suggestion names the exact title stored in session["selected_item"]["title"] — in 4 of 5 tries.

**Why this target:**
Naming the selected item gives me an observable check that the outfit suggestion refers to it. I expect this consistently, but allow one miss because the model may omit or paraphrase the title.

---

## 4. Fit card

For a successful search with a selected item, fit_card contains 2–4 sentences, mentions the item's price and platform exactly once each, and includes its brand when that brand is not None — in 4 of 5 tries.

**Why this target:**
These details make the card useful for a purchase decision, and the sentence limit keeps it concise. Since a model writes the card, one formatting or repetition mistake across five tries is reasonable.

---

## 5. Exact size matching

For the query “classic streetwear size S,” whose keywords match both size S items and the size US 9 sneakers in data/listings.json, search_results includes at least one size S item and excludes every size US 9 item — in 5 of 5 tries.

**Why this target:**
Size filtering uses a deterministic exact-token rule. Returning US 9 for S would violate that rule, so I expect it to pass every time.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 4 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath it, like this:

         ## 4. Something about the fit card

         The fit card is different every time.

         **Why this target:** ...

         > **Revised in unit 4:** For 5 different items, the 5 fit cards share
         > no opening sentence.
         >
         > **Why revised:** "different" wasn't checkable — two cards that
         > differed by one word still counted. The new version is something I
         > can actually score.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "I said the empty search stops it 5 of 5 times, but I got 3 of 5,
            so 3 of 5 is more realistic."

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.
     ───────────────────────────────────────────────────────────────────────── -->
