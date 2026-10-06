# FitFindr

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first** — setup, every
> command, and what to do when something breaks.
>
> Once `python test.py` passes:
>
> ```bash
> python app.py listings --full -n 6      # read the data (Milestone 1)
> python app.py fields                    # what you can filter on
> python app.py ask 'vintage graphic tee under $30'
> ```
>
> All three tools are stubs, so that last command will do nothing useful yet.
> That's the starting position.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     HOW TO USE THIS FILE

     This is your submission. Fill each section in as you finish the milestone
     it belongs to — don't leave it all to the end.

     Unit 3 asks for the first five sections. Unit 4 adds the five below them.
     Leave the unit 4 sections alone until then; they're here so you know
     what's coming.

     Everything is pasted as TEXT. No screenshots, no images, no video links.
     A typed block of output gets full credit; a picture of the same output
     gets none.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 3 — THE BUILD ═══════════════════════ -->

## What This Does

<!-- Three or four sentences: what a user asks for, and what they get back. -->

FitFindr is a shopping agent for secondhand clothes. You type what you're
looking for in plain language, like `'vintage graphic tee under $30, size M'`,
and it searches 40 thrift listings by keyword, size and price. If something
matches, it takes the best match, suggests one or two outfits built from the
pieces already in your wardrobe, and writes a short caption you could post
about the find. If nothing matches, it stops and tells you which part of your
search to loosen.

---

## Data Notes

Notes from reading `data/listings.json` and `data/wardrobe_schema.json`
(Milestone 1), before writing any tools.

**A listing has:** `id` (str), `title` (str), `description` (str), `category`
(str), `style_tags` (list of str), `size` (str), `condition` (str), `price`
(float), `colors` (list of str), `brand` (str or null), `platform` (str).

**A wardrobe item has:** `id`, `name`, `category`, `colors` (list),
`style_tags` (list), `notes` (str or null). A wardrobe is `{"items": [...]}`,
so an empty wardrobe is `{"items": []}`.

**Things that will matter for `search_listings`:**

- 40 listings across five categories: tops, bottoms, outerwear, shoes,
  accessories.
- Sizes aren't consistent — 22 different values, like `M`, `M/L`, `S/M`,
  `XL (oversized)`, `One Size / Oversized`, `W30 L30`, `US 8.5`. An exact
  match on `"M"` would miss `M/L` and `S/M`.
- Prices are floats from $12 to $75, so `max_price` is a plain `<=` check.
- `brand` is null on 32 of 40 listings, so nothing should depend on it being
  there — including the fit card.
- Words people search with ("graphic tee", "vintage", "grunge") mostly show up
  in `style_tags` and `title`, so those are the fields to match against.

---

## Tool Inventory

<!-- Four lines per tool. This is worth 2 points and it's the single most
     common place students lose them.

     "Returns a list" earns NOTHING. The description has to say what is IN
     the list.

     The empty case isn't optional either — it's the thing your loop branches
     on, and if you don't decide it here you'll discover it as a crash in
     Milestone 5. -->

### `search_listings`

- **What it does:** Filters the 40 listings in `data/listings.json` by price
  and size, scores what's left by how many of the description's keywords
  appear in the listing's `title`, `style_tags`, `category`, `colors` and
  `description`, and returns the best matches. No model call.
- **Inputs:** `description` (str) — keywords like `"vintage graphic tee"`;
  `size` (str or None) — e.g. `"M"`, `None` skips the size filter;
  `max_price` (float or None) — inclusive ceiling, `None` skips the price
  filter. **Size match rule:** the listing's size is lowercased, parentheses
  are removed, and it's split on `/` and spaces into tokens; it matches if the
  requested size is one of those tokens, so `"M"` matches `M`, `M/L` and `S/M`
  but not `XL`, and `"S"` does not match `US 9`. Listings whose size starts
  with "One Size" match any size.
- **Returns:** A `list[dict]` of up to 10 (`config.SEARCH_RESULT_LIMIT`)
  listing dicts, highest keyword score first. Each dict is a whole listing:
  `id`, `title`, `description`, `category`, `style_tags` (list), `size`,
  `condition`, `price` (float), `colors` (list), `brand` (str or None),
  `platform`. Listings that score 0 are dropped.
- **When it has nothing:** An empty list, `[]` — never `None` and never an
  exception. This is what the loop branches on.

### `suggest_outfit`

- **What it does:** Asks the model for one or two outfits built around the new
  item, naming specific pieces from the user's wardrobe.
- **Inputs:** `new_item` (dict) — one listing dict from `search_listings`;
  `wardrobe` (dict) — `{"items": [...]}`, where each item has `id`, `name`,
  `category`, `colors`, `style_tags`, `notes`. The list may be empty.
- **Returns:** A non-empty `str` of outfit suggestions that names the new item
  and at least one wardrobe piece by its `name`.
- **When it has nothing:** If `wardrobe["items"]` is empty, it doesn't fail —
  it asks the model for general styling advice for the item (what kinds of
  pieces it goes with) and returns that as a non-empty `str`.

### `create_fit_card`

- **What it does:** Asks the model to write a short, casual caption about the
  find, the way someone would post it, using the outfit and the item details.
- **Inputs:** `outfit` (str) — the string `suggest_outfit` returned;
  `new_item` (dict) — the same listing dict that went into `suggest_outfit`.
- **Returns:** A `str` caption of 2–4 sentences that mentions the item, its
  `price` and its `platform` once each. It only mentions `brand` when it isn't
  None.
- **When it has nothing:** If `outfit` is empty or only whitespace, it skips
  the model call and returns the message
  `"Can't write a fit card without an outfit suggestion."`

---

## Planning Loop

<!-- Your branch rule, stated as a rule — the condition AND both paths — plus
     the file and function that holds it.

     Like this:
       "If search_listings returns an empty list, put a message in the session
        and stop. Otherwise take the first result and go to suggest_outfit."
        — agent.py::run_agent

     The grader checks your code against what you claim here, so the file and
     function have to be real. -->

**Branch rule:** If `search_listings` returns an empty list, put a message in
`session["error"]` that says which filters were used (description, size, max
price) and suggests loosening one of them, then return the session without
calling `suggest_outfit` or `create_fit_card`. Otherwise, take the first
result as `session["selected_item"]` and go on to `suggest_outfit`.

**Where it lives:** `agent.py::run_agent`

**How the query is parsed:** Regex. `under $30` / `$30` → `max_price = 30.0`;
`size M` → `size = "M"`; whatever is left, minus filler words like "looking
for", becomes `description`.

**What moves through the session:** `query` → `parsed` (description, size,
max_price) → `search_results` → `selected_item` (the first result) →
`outfit_suggestion` → `fit_card`. On the empty path it stops after
`search_results`, with `error` set and `selected_item`, `outfit_suggestion`
and `fit_card` still `None`.

---

## Sample Run

<!-- Two things go here.

     1. One FULL query and its output, pasted as text.
     2. Your three per-tool terminal tests — the command and what it printed. -->

**One full query**

```
$ python app.py ask 'vintage graphic tee under $30'

  Found:    Y2K Baby Tee — Butterfly Print — $18.0 on depop

  Outfit:   **Outfit 1: Casual Streetwear**
Pair the **Y2K Baby Tee — Butterfly Print** with the **Baggy straight-leg jeans, dark wash** for an iconic early-2000s silhouette. Layer the **Black cropped zip hoodie** over top (left unzipped to show off the graphic), and finish the look with **Chunky white sneakers** and the **Black crossbody bag**. 

**Outfit 2: Edgy Contrast**
Combine the **Y2K Baby Tee — Butterfly Print** with the **Wide-leg khaki trousers** for a fun mix of earthy tones and playful Y2K graphics. Throw on the **Vintage black denim jacket** and ground the outfit with the **Black combat boots**. Accentuate the waist using the **Brown leather belt**.

  Fit card: finally found the ultimate early 2000s butterfly tee and i am literally never taking it off. snatched it on depop for just $18 and the fit is honestly unreal. gives major off-duty bratz doll energy.

0 model calls this session, 2 served from cache
```

**The three tools, tested one at a time**

```
$ python -c "from tools import search_listings; print(search_listings('graphic tee', max_price=30))"
[{'id': 'lst_002', 'title': 'Y2K Baby Tee — Butterfly Print', 'description': 'Super cute early 2000s baby tee with butterfly graphic. Fitted crop length. Tag says medium but fits like a small.', 'category': 'tops', 'style_tags': ['y2k', 'vintage', 'graphic tee', 'cottagecore'], 'size': 'S/M', 'condition': 'excellent', 'price': 18.0, 'colors': ['white', 'pink', 'purple'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_006', 'title': 'Graphic Tee — 2003 Tour Bootleg Style', 'description': 'Vintage-style bootleg tee with faded graphic. Slightly boxy fit. 100% cotton, soft and worn-in.', 'category': 'tops', 'style_tags': ['graphic tee', 'vintage', 'grunge', 'streetwear', 'band tee'], 'size': 'L', 'condition': 'good', 'price': 24.0, 'colors': ['black'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_017', 'title': 'Mesh Long-Sleeve Top — Black', 'description': 'Sheer black mesh long-sleeve. Great for layering under a graphic tee or over a bralette. Stretchy material, fits true to size.', 'category': 'tops', 'style_tags': ['y2k', 'grunge', 'goth', 'layering'], 'size': 'S/M', 'condition': 'excellent', 'price': 15.0, 'colors': ['black'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_033', 'title': 'Vintage Band Tee — Faded Grey', 'description': 'Faded grey band-style tee with distressed graphic. Crew neck. Fits boxy. Well-loved but no holes or major damage.', 'category': 'tops', 'style_tags': ['vintage', 'grunge', 'band tee', 'graphic tee', 'streetwear'], 'size': 'L', 'condition': 'fair', 'price': 19.0, 'colors': ['grey', 'charcoal'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_011', 'title': 'Low-Rise Cargo Pants — Khaki', 'description': 'Y2K era low-rise cargo pants. Lots of pockets. Khaki color, slightly distressed at the hems. Great for layering with a long tee.', 'category': 'bottoms', 'style_tags': ['y2k', 'cargo', '2000s', 'streetwear'], 'size': 'W29', 'condition': 'fair', 'price': 27.0, 'colors': ['khaki', 'tan'], 'brand': None, 'platform': 'poshmark'}, {'id': 'lst_015', 'title': 'Vintage Graphic Hoodie — Faded Black', 'description': 'Faded black pullover hoodie with barely-visible vintage graphic on the chest. Cozy interior. Some pilling but adds to the worn-in look.', 'category': 'tops', 'style_tags': ['vintage', 'grunge', 'graphic', 'streetwear'], 'size': 'L', 'condition': 'fair', 'price': 26.0, 'colors': ['black', 'charcoal'], 'brand': None, 'platform': 'depop'}]
```

```
$ python -c "from tools import suggest_outfit; from utils.data_loader import get_example_wardrobe, load_listings; print(suggest_outfit(load_listings()[0], get_example_wardrobe()))"
**Outfit 1: Casual Streetwear**
Pair the **Vintage Levi's 501 Jeans — Medium Wash** with the **White ribbed tank top** tucked in for a classic silhouette. Layer the **Oversized grey crewneck sweatshirt** on top for a cozy, relaxed vibe, and finish the look with the **Chunky white sneakers** and **Black crossbody bag**. 

**Outfit 2: Edgy Vintage**
Combine the **Vintage Levi's 501 Jeans — Medium Wash** with the **Black cropped zip hoodie** for a balanced proportions look. Add the **Vintage black denim jacket** as outerwear, tie it together with the **Brown leather belt**, and step into the **Black combat boots** to lean into a cool, grunge-inspired aesthetic.
```

```
$ python -c "from tools import create_fit_card; from utils.data_loader import load_listings; print(create_fit_card('jeans and white sneakers', load_listings()[0]))"
Finally found the holy grail of thrifted denim that actually fits right. These vintage Levi's have that perfect broken-in medium wash and look so good with crisp white sneakers. Snagged them on depop for just $38 and I’m never taking them off.
```

---

## How I Used AI

<!-- Two specific moments. What you asked, what came back, what you changed.

     "I used Claude to help me code" is not enough.

     "I gave Claude my search_listings spec. It returned None on no match
     instead of an empty list, so I changed it" is the level we want. -->

**Moment 1**

- *What I asked for:* I gave Claude my five criteria and asked it to say
  exactly how it would test each one from the sentence alone, without
  suggesting improvements.
- *What came back:* It said criterion 5 would fail as written. A query of just
  `size S` leaves no keywords, and my search drops anything that scores 0, so
  it returns `[]` and never includes the size S item. It also pointed out that
  my reason for criterion 1 says "the model might extract a size," but my
  README says the query is parsed with regex.
- *What I changed:* At first I kept the criteria as written. Once the tools
  were built I confirmed the problem — `search_listings('', size='S')` returns
  `[]` — so before running any evaluation I changed criterion 5's query to
  `classic streetwear size S`. Its keywords match both the size S denim jacket
  and the US 9 sneakers, so excluding the sneakers actually tests the size
  rule. I also fixed reason 1 to talk about my regex parser instead of a model.

**Moment 2**

- *What I asked for:* I had Claude build `create_fit_card` from my spec and
  run it three times on the same item, which is the check Milestone 4 asks
  for.
- *What came back:* Three word-for-word identical captions. My `TEMPERATURE`
  is 0.9, so that pointed at the response cache. Rerunning with
  `AI201_CACHE=0` gave three different captions. One cached caption also wrote
  the price as "thirty-eight dollars" even though the prompt asks for `$38`.
- *What I changed:* I used the cache-off output for the `create_fit_card`
  test in Sample Run. I'm keeping the "thirty-eight dollars" case in mind for
  criterion 4, since it counts the price.

<!-- ═══════════════════════ UNIT 4 — THE TEST ═══════════════════════

     Don't fill these in during unit 3.
     ═══════════════════════════════════════════════════════════════════ -->

---

## Run Log — Before

<!-- Five criteria, five tries each, in this exact format.

     Five, because your criteria are written out of five. Mark each try PASS
     or FAIL, count the passes, and read that count against your target — a
     row targeting 4 of 5 with three PASS cells is MISSED (3/5).

     `python run_eval.py --label before` runs everything and writes the table
     into results/. Paste it here and fill in the verdicts. -->

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Real output from one try**, pasted as text, naming the file and function
that produced it:

```

```

---

## Verdicts and Diagnoses

<!-- MET or MISSED per criterion against LAST UNIT's target, plus a sentence on
     how you decided.

     Then, for every miss: which of the four places it happened — a tool, the
     loop's branch, the session, or the model's output — AND the mechanism.

     Not a diagnosis:  "The fit card was bad."
     A diagnosis:      "The fit card criterion missed on 2 of 5 items. Both had
                        an empty brand field. My prompt puts the brand in the
                        first sentence, so the card opened with a blank and read
                        like a fragment. The tool worked; the prompt assumed a
                        field that isn't always there."

     Look for a pattern. Three misses on the same tool is one problem, not
     three. -->

| # | Criterion | Target | Verdict | How I decided |
|---|---|---|---|---|
| 1 |  |  |  |  |
| 2 |  |  |  |  |
| 3 |  |  |  |  |
| 4 |  |  |  |  |
| 5 |  |  |  |  |

**Diagnoses**



---

## Loop Trace

<!-- One full run, printed step by step, with the MCP call visible in it.

     `python app.py ask '...' --trace` once you've added the trace.step()
     calls in Milestone 2.

     Worth pasting BOTH the happy path and the empty-search path. The empty
     one should be visibly shorter, because it stops. If your two traces are
     the same length, your branch isn't working — and this is the fastest way
     anyone will ever find that out. -->

**Happy path**

```

```

**Empty search**

```

```

**On the MCP move:** <!-- what changed in your code, and whether anything
behaved differently afterwards. If the rewire didn't work, say exactly where it
broke — the error text and the last thing that worked. That earns the point in
full. -->



---

## The Improvement

<!-- What you changed, why your diagnosis pointed at it, and the after-run in
     the same table format. One change, measured properly.

     `python run_eval.py --label after` -->

**What I changed:**

**Which failure it was meant to fix:**

### Run Log — After

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Did it help, and how do I know:**

<!-- If it made things worse, say that. Honestly reported, that earns full
     credit and is more interesting than one that worked. -->



---

## What's Still Broken

<!-- For each criterion still missed: what you'd do, and why you stopped where
     you did. "I ran out of time" is fine if it's true. Pretending nothing is
     left is not. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 3

       [ ] criteria.md has five numbered criteria, each with a target
       [ ] Each criterion has a reason underneath it
       [ ] All five unit 3 sections above have real content
       [ ] Tool Inventory: all three tools, inputs WITH TYPES, a specific
           return value, and the empty case
       [ ] Planning Loop names the branch rule and agent.py::run_agent
       [ ] Sample Run: one full query plus the three per-tool tests, as text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN, you submit the same one
           next unit

     SUBMISSION CHECKLIST — unit 4

       [ ] mcp_server.py exists with one tool registered
           (or a written record of exactly where the rewire broke)
       [ ] Run Log — Before, five criteria, five tries each
       [ ] Real output pasted underneath, naming file and function
       [ ] A verdict on every criterion
       [ ] A diagnosis for every miss, naming a place AND a mechanism
       [ ] Loop Trace, with the MCP call visible in it
       [ ] All three failure modes triggered and handled
       [ ] One improvement, with Run Log — After in the same format
       [ ] What's Still Broken
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository. Your commit history is what
     shows your criteria existed before your results did.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
