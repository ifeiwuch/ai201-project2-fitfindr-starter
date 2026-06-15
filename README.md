# FitFindr — Starter Kit

This starter kit contains everything you need to begin Project 2.

## What's Included

```
ai201-project2-fitfindr-starter/
├── data/
│   ├── listings.json          # 40 mock secondhand listings
│   └── wardrobe_schema.json   # Wardrobe format + example wardrobe
├── utils/
│   └── data_loader.py         # Helper functions for loading the data
├── planning.md                # Your planning template — fill this out first
└── requirements.txt           # Python dependencies
```

## Setup

```bash
pip install -r requirements.txt
```

Set your Groq API key in a `.env` file (get a free key at [console.groq.com](https://console.groq.com)):
```
GROQ_API_KEY=your_key_here
```

## The Mock Listings Dataset

`data/listings.json` contains 40 mock secondhand listings across categories (tops, bottoms, outerwear, shoes, accessories) and styles (vintage, y2k, grunge, cottagecore, streetwear, and more).

Each listing has: `id`, `title`, `description`, `category`, `style_tags`, `size`, `condition`, `price`, `colors`, `brand`, and `platform`.

Load it with:
```python
from utils.data_loader import load_listings
listings = load_listings()
```

## The Wardrobe Schema

`data/wardrobe_schema.json` defines the format your agent uses to represent a user's existing wardrobe. It includes:

- `schema`: field definitions for a wardrobe item
- `example_wardrobe`: a sample wardrobe with 10 items you can use for testing
- `empty_wardrobe`: a starting template for a new user

Load an example wardrobe with:
```python
from utils.data_loader import get_example_wardrobe
wardrobe = get_example_wardrobe()
```

## Where to Start

1. **Read `planning.md` and fill it out before writing any code.**
2. Verify the data loads correctly by running `python utils/data_loader.py`.
3. Build and test each tool individually before connecting them through your planning loop.

Your implementation files go in this same directory. There's no required file structure for your agent code — organize it however makes sense for your design.

---

## Running the app

```bash
python app.py
```

Open the URL shown in the terminal (usually `http://localhost:7860`). Type a query like `vintage graphic tee under $30` and choose a wardrobe mode, then click **Find it**.

---

## Tool inventory

### `search_listings(description: str, size: str | None, max_price: float | None) → list[dict]`
Searches `data/listings.json` for items matching the description. Scores each listing by counting how many description keywords appear in its title, description, category, style tags, and brand. Filters by `size` (case-insensitive substring match) and `max_price` (inclusive) when provided. Returns results sorted by relevance score, highest first. Returns an empty list if nothing matches — never raises an exception.

### `suggest_outfit(new_item: dict, wardrobe: dict) → str`
Calls the Groq LLM (`llama-3.1-8b-instant`) to suggest 1–2 complete outfits pairing the thrifted item with pieces from the user's wardrobe. The wardrobe items are formatted into the prompt by name, category, color, and any notes. If the wardrobe is empty, the LLM is prompted for general styling advice instead (what kinds of items pair well, what vibe it suits). Always returns a non-empty string.

### `create_fit_card(outfit: str, new_item: dict) → str`
Calls the Groq LLM (`llama-3.1-8b-instant`, temperature 0.9) to write a 2–4 sentence Instagram/TikTok-style caption for the outfit. The prompt instructs the LLM to mention the item name, price, and platform once each, capture the outfit vibe in specific terms, and sound casual rather than like a product description. Returns a descriptive error string — not an exception — if `outfit` is empty or `new_item` is missing.

---

## Planning loop explanation

`run_agent()` in `agent.py` runs the three tools in a fixed sequence and checks session state after each step to decide whether to continue.

**Step 1 — Parse the query** using regex to extract:
- `max_price`: matches phrases like `under $30`, `max $25`, `less than $40`
- `size`: matches `size M` or standalone tokens (`XS`, `S`, `M`, `L`, `XL`, `XXL`, `S/M`)
- `description`: everything left after stripping price and size phrases

**Step 2 — `search_listings()`** is called with the parsed parameters. If the result is an empty list, `session["error"]` is set to a helpful message and the function returns immediately. `suggest_outfit` and `create_fit_card` are never called.

**Step 3 — `suggest_outfit()`** is called with `session["selected_item"]` (the top search result) and the user's wardrobe. The result is stored in `session["outfit_suggestion"]`.

**Step 4 — `create_fit_card()`** is called with `session["outfit_suggestion"]` and `session["selected_item"]`. The result is stored in `session["fit_card"]`.

The loop is done when it hits the early return after `search_listings`, or after `create_fit_card` completes.

---

## State management approach

Each call to `run_agent()` initializes a fresh session dict:

```python
{
    "query": str,             # original user query
    "parsed": dict,           # extracted description, size, max_price
    "search_results": list,   # all matching listings returned by search_listings
    "selected_item": None,    # top result — passed into suggest_outfit and create_fit_card
    "wardrobe": dict,         # user's wardrobe passed in from the UI
    "outfit_suggestion": None, # string returned by suggest_outfit
    "fit_card": None,         # string returned by create_fit_card
    "error": None,            # set on early exit; all other output keys stay None
}
```

Each tool writes its output into the corresponding session key. The next tool reads its inputs directly from the session — no values are re-fetched from the dataset or re-prompted from the user between steps. At the end of any interaction, the full session dict is returned and can be inspected to see exactly what each step produced.

---

## Error handling

| Tool | Failure mode | What the agent does | Example |
|------|-------------|---------------------|---------|
| `search_listings` | No listings match the query | Sets `session["error"]` to `"No listings found for that description/size/price. Try loosening the price or size filter."` and returns early — `suggest_outfit` and `create_fit_card` are never called | Query: `"designer ballgown size XXS under $5"` → `search_results: []`, `fit_card: None` |
| `suggest_outfit` | Wardrobe is empty | LLM is prompted for general styling advice instead of outfit pairings — always returns a non-empty string, loop continues normally | Query with `Empty wardrobe` mode → suggestion describes what kinds of items pair well, not specific wardrobe pieces |
| `create_fit_card` | `outfit` is empty or whitespace-only | Returns `"Could not generate a fit card: outfit suggestion is missing or empty."` without calling the LLM | `create_fit_card("", item)` → error string, no exception |
| `create_fit_card` | `new_item` is `None` | Returns `"Could not generate a fit card: new_item is missing."` without calling the LLM | `create_fit_card(outfit, None)` → error string, no exception |

---

## Spec reflection

**One way the spec helped:** The architecture diagram in `planning.md` made the early-exit condition unambiguous before a single line of code was written. Because the diagram explicitly showed `search_listings` branching into a `RETURN session` path when `results = []`, there was no temptation to call `suggest_outfit` with empty input or to let the loop continue silently. That single decision — stop early and set `session["error"]`, skip the remaining tools entirely — was the most important correctness constraint in the whole agent, and having it drawn out meant the implementation matched it exactly on the first attempt.

**One way implementation diverged from the spec:** The planning.md spec did not specify how the user's natural language query would be parsed into `description`, `size`, and `max_price` — it only said "extract" those values. During implementation, this became a concrete choice: use an LLM call or use regex. The spec left it open, but an LLM call would add latency, cost, and a new failure mode to a step that precedes all three tools. Regex was used instead — patterns like `under $30`, `size M`, and standalone size tokens cover the expected query formats without any API call. This was documented in `planning.md` Milestone 4 after the decision was made, since the spec required documenting the parsing approach but didn't prescribe it.

## AI Usage

**Tool used:** Claude (claude-sonnet-4-6) via Claude Code

---

**Instance 1 — Implementing `search_listings`**

Input given to Claude: the Tool 1 spec from `planning.md` (inputs, return value, failure mode) and the architecture diagram. Also pointed it to `load_listings()` in `utils/data_loader.py` so it could use the correct data source.

What it produced: a working `search_listings()` that loads all listings, filters by `max_price` and `size` (case-insensitive substring), scores each remaining listing by keyword overlap across title, description, category, style tags, and brand, drops zero-score listings, and returns results sorted by score descending.

What I verified and changed: ran 4 test queries — a broad match, a size-filtered match, a price-filtered match, and an impossible query. Tests 2 and 3 initially returned 0 results, which I investigated. The filtering logic was correct; the dataset simply has no jeans sized "M" (jeans use waist sizes like W30) and no jackets under $30. Revised the test inputs to use values that actually exist in the data and confirmed all filters work. No logic changes were needed.

---

**Instance 2 — Implementing `suggest_outfit` and `create_fit_card`**

Input given to Claude: the Tool 2 and Tool 3 specs from `planning.md`, the example wardrobe structure from `wardrobe_schema.json`, and the `suggest_outfit` output from a live test run as sample input for `create_fit_card`.

What it produced: `suggest_outfit()` that builds a prompt including the item summary and a formatted list of wardrobe items (name, category, colors, notes), with a separate prompt branch for the empty-wardrobe case. `create_fit_card()` that guards against empty/missing inputs before calling the LLM, and uses temperature 0.9 for caption variety.

What I verified and changed: the first call to `suggest_outfit` failed with a `model_decommissioned` error — the model `llama3-8b-8192` Claude initially used no longer exists on Groq. Switched to `llama-3.1-8b-instant`. Then ran the full 3-test suite for each tool (populated wardrobe, empty wardrobe, minimal wardrobe for `suggest_outfit`; valid inputs, empty outfit string, `None` item for `create_fit_card`) and confirmed all failure modes return error strings rather than raising exceptions.

---

**Instance 3 — Implementing `run_agent()` (the planning loop)**

Input given to Claude: the Planning Loop section, State Management section, and Architecture diagram from `planning.md`, along with the instruction to follow the numbered TODO steps in `agent.py` exactly.

What it produced: `run_agent()` with regex-based query parsing (extracting `max_price`, `size`, and `description` from the natural language query), the session dict initialized via `_new_session()`, and the four-step tool sequence with an early return when `search_listings` returns an empty list.

What I verified and changed: ran a traced end-to-end test using monkey-patched wrappers to confirm the same dict object (`is` check) flows from `search_listings` into `suggest_outfit` and `create_fit_card` with no copying or re-fetching. Also ran `python agent.py` directly to confirm the no-results path sets `session["error"]` and leaves `session["fit_card"]` as `None`. Then cross-checked all 10 planning.md requirements with assertions — all passed. No logic changes were needed.
