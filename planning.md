# FitFindr — planning.md

> Complete this document before writing any implementation code.
> Your spec and agent diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Your planning.md will be reviewed as part of your submission.
> Update it before starting any stretch features.

---

## Tools

List every tool your agent will use. For each tool, fill in all four fields.
You must have at least 3 tools. The three required tools are listed — add any additional tools below them.

### Tool 1: search_listings

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
This tool searches the listings dataset for items matching the description, optional size, and optional price ceiling.

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `description` (str): Keywords describing the item to be searched
- `size` (str): the size of the item (format depends on the item type and is case insensitive). Can be none to skip size filtering
- `max_price` (float): the max price the user is willing to accept for the item (inclusive). Can be none to skip price filtering

**What it returns:**
<!-- Describe the return value — what fields does a result contain? -->
Returns a list of matching listing dicts, sorted by relevance. Each dict has the following fields: id, title, description, category, style_tags (list), size, condition, price (float), colors (list), brand, platform

**What happens if it fails or returns nothing:**
<!-- What should the agent do if no listings match? -->
Returns an empty list if nothing matches. The agent should notify the user that there are no matches on the current listings

---

### Tool 2: suggest_outfit

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
Given the new found item and the user's wardrobe, suggest 1–2 complete outfits.

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `new_item` (dict): The item the user is considering buying
- `wardrobe` (dict): A list of all the items in the user's current wardrobe (can be empty)

**What it returns:**
<!-- Describe the return value -->
A non-empty string with outfit suggestions.

**What happens if it fails or returns nothing:**
<!-- What should the agent do if the wardrobe is empty or no outfit can be suggested? -->
If the wardrobe is empty, offer general styling advice for the item rather than raising an exception or returning an empty string.

---

### Tool 3: create_fit_card

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
Generates a short, shareable outfit caption for the thrifted find.

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `outfit` (string): The outfit suggestion from suggest_outfit()
- `new_item` (dict): The item the user is considering buying

**What it returns:**
<!-- Describe the return value -->
A 2–4 sentence string usable as an Instagram/TikTok caption.

**What happens if it fails or returns nothing:**
<!-- What should the agent do if the outfit data is incomplete? -->
If outfit is empty or missing, return a descriptive error message string — do NOT raise an exception.

---

### Additional Tools (if any)

<!-- Copy the block above for any tools beyond the required three -->

---

## Planning Loop

**How does your agent decide which tool to call next?**
<!-- Describe the logic your planning loop uses. What does it look at? What conditions change its behavior? How does it know when it's done? -->

The loop runs tools in a fixed candidate order (search_listings → suggest_outfit → create_fit_card), but checks session state after each call to decide whether to continue.

Call search_listings(description, size, max_price). If the result is an empty list, return a message telling the user no matches were found and suggesting they loosen the size or price filter. suggest_outfit() and create_fit_card() are never called. If results are non-empty, store the selected item and call suggest_outfit(selected_item, wardrobe). This always returns a usable string. Next, call create_fit_card(outfit_suggestion, selected_item). If either input is missing/empty, the tool returns an error string without calling the LLM. Otherwise it stores the generated caption and returns it. The loop knows when it's done either when it hits the early-return after calling search_listings(), or after create_fit_card() is complete.
---

## State Management

**How does information from one tool get passed to the next?**
<!-- Describe how your agent stores and accesses state within a session. What data is tracked? How is it passed between tool calls? -->
The loop creates a session dict at the start of each query, with keys for selected_item, outfit_suggestion, fit_card, and error (initialized to None). As each tool runs, its output is written into the corresponding key. The next tool reads its inputs directly from session rather than the user re-providing them.

---

## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | The agent returns a message telling the user no matches were found and suggesting they loosen the size or price filter |
| suggest_outfit | Wardrobe is empty | The agent uses the tool's generic styling advice as input for the next step |
| create_fit_card | Outfit input is missing or incomplete | The agent returns an error message describing what went wrong |

---

## Architecture

<!-- Draw a diagram of your agent showing how the components connect:
     User input → Planning Loop → Tools (search_listings, suggest_outfit, create_fit_card)
                                                                          ↕
                                                                   State / Session
     Show what triggers each tool, how state flows between them, and where error paths branch off.
     ASCII art, a Mermaid diagram (https://mermaid.js.org/syntax/flowchart.html), or an embedded
     sketch are all fine. You'll share this diagram with an AI tool when asking it to implement
     the planning loop and each individual tool. -->

User query
    │
    ▼
Planning Loop ─────────────────────────────────────────────────────────────┐
    │                                                                      │
    ├─► search_listings(description, size, max_price)                      │
    │       │                                                              │
    │       ├─ results = []                                                │
    │       │     └─► session["error"] = "No listings found for that       │
    │       │         description/size/price. Try loosening the price      │
    │       │         or size filter."                                     │
    │       │     └─► RETURN session (stop — do not call suggest_outfit)   │
    │       │                                                              │
    │       └─ results = [item, ...]                                       │
    │             └─► session["selected_item"] = results[0]                │
    │                 (proceed)                                            │
    │       │                                                              │
    ├─► suggest_outfit(selected_item, wardrobe)                            │
    │       │                                                              │
    │       └─ result = <string> (always non-empty by design —             │
    │            tool internally handles empty wardrobe by returning       │
    │            general styling advice instead of an outfit pairing)      │
    │             └─► session["outfit_suggestion"] = result                │
    │                 (proceed)                                            │
    │       │                                                              │
    └─► create_fit_card(outfit_suggestion, selected_item)                  │
            │                                                              │
            ├─ if outfit_suggestion or selected_item missing/empty:        │
            │     └─► return descriptive error string (no API call)        │
            │           session["fit_card"] = None                         │
            │           session["error"] = <that error string>             │
            │                                                              │
            └─ else: result = <fit card string> (call to Groq API)         │
                  └─► session["fit_card"] = result                         │
            │                                                              │
            ▼                                                              │
        RETURN session ◄───────────────────────────────────────────────────┘

---

## AI Tool Plan

<!-- For each part of the implementation below, describe:
     - Which AI tool you plan to use (Claude, Copilot, ChatGPT, etc.)
     - What you'll give it as input (which sections of this planning.md, your agent diagram)
     - What you expect it to produce
     - How you'll verify the output matches your spec before moving on

     "I'll use AI to help me code" is not a plan.
     "I'll give Claude my Tool 1 spec (inputs, return value, failure mode) and ask it to implement
     search_listings() using load_listings() from the data loader — then test it against 3 queries
     before trusting it" is a plan. -->

**Milestone 3 — Individual tool implementations:**

Tool 1 — search_listings:
I'll give Claude the Tool 1 spec from this file (inputs, return value, failure mode) and the architecture diagram, and ask it to implement `search_listings(description, size, max_price)` using `load_listings()` from the data loader. I expect it to produce a function that filters by keyword match on title/description, optional size (case-insensitive), and optional max_price, returning a sorted list of dicts with all required fields. I'll verify by running 4 manual test queries: (1) a description that matches multiple listings, (2) a description with a size filter, (3) a description with a max_price filter, and (4) a query that returns no results — confirming the empty list is returned and all dict fields are present.

Tool 2 — suggest_outfit:
I'll give Claude the Tool 2 spec and a sample wardrobe dict, and ask it to implement `suggest_outfit(new_item, wardrobe)`. I expect it to return a non-empty suggestion string in all cases. I'll verify by running 3 tests: (1) a populated wardrobe with a real `new_item` from search_listings output, (2) an empty wardrobe dict confirming it falls back to general styling advice, and (3) a minimal wardrobe (one item) to confirm it still produces a useful suggestion.

Tool 3 — create_fit_card:
I'll give Claude the Tool 3 spec, the suggest_outfit output from the test above, and the selected item dict, and ask it to implement `create_fit_card(outfit, new_item)`. I expect a 2–4 sentence caption string, or a descriptive error string if inputs are missing. I'll verify by running 3 tests: (1) valid outfit + item, confirming the caption is 2–4 sentences and reads like a social post, (2) empty outfit string, confirming an error string is returned without raising an exception, and (3) None item, confirming the same graceful error behavior.

**Milestone 4 — Planning loop and state management:**

**Query parsing approach:** regex (not LLM). Price is extracted with a pattern matching phrases like "under $30", "max $25", "less than $40". Size is extracted by matching "size M" or standalone tokens (XS/S/M/L/XL/XXL/S/M). The description is everything left after stripping those phrases and collapsing whitespace. Regex was chosen over LLM parsing because the query structure is predictable and a regex is faster, cheaper, and has no failure modes.

I'll give Claude the Planning Loop section, the State Management section, and the full Architecture diagram from this file, and ask it to implement the `run_agent(query, wardrobe)` function that initializes a session dict and calls each tool in order.

I expect it to produce a function that:
- Initializes `session = {selected_item: None, outfit_suggestion: None, fit_card: None, error: None}`
- Calls `search_listings()` with the parsed description, size, and max_price from the query
- Returns early (with `session["error"]` set) if results are empty, without calling the remaining tools
- Stores `results[0]` in `session["selected_item"]` and calls `suggest_outfit()`
- Stores the suggestion in `session["outfit_suggestion"]` and calls `create_fit_card()`
- Stores the final caption in `session["fit_card"]` and returns the session

I'll verify by running 3 end-to-end tests:
1. A query that matches listings — confirm all four session keys are populated and the fit card reads like a real caption
2. A query that matches nothing — confirm `session["error"]` is set and `suggest_outfit` / `create_fit_card` are never called (add a print or log to confirm)
3. A query with no wardrobe (empty dict) — confirm it completes successfully using the fallback styling path and still produces a fit card

---

## A Complete Interaction (Step by Step)

Write out what a full user interaction looks like from start to finish — tool call by tool call. Use a specific example query.

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1:**
<!-- What does the agent do first? Which tool is called? With what input? -->
The agent searches all current listings in the dataset using the search_listings() tool. The inputs for the tool are the description ("vintage graphic tee"), max price (30), and size of the item (M). The tool returns 3 matching listings sorted by relevance. Then, the agent picks the best result. If the search comes back empty, the agent will relay that back to the user.

**Step 2:**
<!-- What happens next? What was returned from step 1? What tool is called now? -->
The agent then calls suggest_outfit(), with the parameters of the new item that was found, and the user's current wardrobe. This tool returns a suggested outfit from the items in the current wardrobe and the the new item in natural language (e.g. "Pair this with your wide-leg jeans and platform Docs for a classic 90s grunge look. Roll the sleeves once and tuck the front corner slightly for shape"). If the wardrobe is empty or minimal, it will tell the user that the current wardrobe is empty/too small. This is only triggered if the search_listings() tool returns an item.

**Step 3:**
<!-- Continue until the full interaction is complete -->
The agent will take the new outfit suggestion and the new item as parameters for create_fit_card() tool. This tool will generate a uniqe short description of the new outfit based on the pieces in the wardrobe. This is only triggered if an adequate outfit is suggested.

**Final output to user:**
<!-- What does the user actually see at the end? -->
At the end the user may see something like: "thrifted this faded band tee off depop for $22 and honestly it was made for my wide-legs 🖤 full look in my stories."
