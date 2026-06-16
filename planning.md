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
The user's input is used by the agent to extract 3 things: the description of the clothing they want, the size they want for that clothing item, and the max_price they are willing to pay for it. The tool/function search_listings() then uses these 3 extracted attributes as input parameters and returns the top 3 clothing items from the listings which match the input parameters.

**Input parameters:**
- `description` (str): The description of the clothing the user wants.
- `size` (str): The size in which the user wants the clothing item.
- `max_price` (float): The maximum price the user is willing to pay for this clothing item.

**What it returns:**
The function/tool returns the top 3 matches as a list of dictionaries sorted by relevance (relevance will be measured by finding a score of how many paramters directly matched with the item) from the listings according to the input paramters it received. Each dictionary includes fields {"id", "title", "description", "category", "style_tags", "size", "condition", "price", "colors", "brand", "platform"}.

**What happens if it fails or returns nothing:**
If the tool fails or returns nothing, then the user is told to try some other attributes because the attributes they gave were not matched with any listings. Also, the agent stops the loop here and does not call the next function (suggest_outfit()).

---

### Tool 2: suggest_outfit

**What it does:**
The tool suggest_outfit() takes a listing dict and a wardrobe dict, then calls the LLM to generate 1–2 complete outfit suggestions using the new item paired with pieces from the wardrobe. If the wardrobe is empty, it falls back to general styling advice for the item instead.

**Input parameters:**
- `new_item` (dict): A dictionary containing all information about the new clothing item. Fields: `id` (str), `title` (str), `description` (str), `category` (str), `style_tags` (list[str]), `size` (str), `condition` (str), `price` (float), `colors` (list[str]), `brand` (str or None), `platform` (str).
- `wardrobe` (dict): A dictionary with a single key `items`, which maps to a list of wardrobe item dicts. Each wardrobe item has: `id` (str), `name` (str), `category` (str), `colors` (list[str]), `style_tags` (list[str]), `notes` (str or None). May be empty — `wardrobe["items"]` will be an empty list for a new user.

**What it returns:**
A non-empty string with 1–2 outfit suggestions. Each suggestion references specific wardrobe pieces by name (e.g. "pair with your dark wash baggy jeans and chunky white sneakers"), describes the overall aesthetic or vibe, and may include a small styling tip (e.g. tuck, roll sleeves). If the wardrobe is empty, returns general advice on what types of pieces pair well with the item and what aesthetic it suits.

**What happens if it fails or returns nothing:**
- Empty wardrobe (expected case): the tool does not fail — it returns general styling advice for the item instead of wardrobe-specific combinations.
- LLM/API error (actual failure): return the string "FitFindr found your item but was unable to generate a styling suggestion. The item found was: {title} — ${price} on {platform}." Do not raise an exception. 

---

### Tool 3: create_fit_card

**What it does:**
It takes the outfit suggestion string and the new item from the listings as inputs. It calls the LLM with higher temperature and then generates a short, shareable outfit caption for the thrifted find that the user can use on their Instagram/Tiktok post.

**Input parameters:**
- `outfit` (str): The outfit suggestion string. May be empty or whitespace-only if suggest_outfit() failed — the tool must check for this before calling the LLM.
- `new_item` (dict): The listing dict for the thrifted item. The tool uses `title` (str), `price` (float), and `platform` (str) from this dict to reference the item naturally in the caption.

**What it returns:**
It returns a 2-4 sentence long caption as a String for the user to use as a caption for their new instagram/tiktom post. The caption should feel casual and authentic (like a real OOTD post, not a product description). Mention the item name, price, and platform naturally (once each). Capture the outfit vibe in specific terms. Sound different each time for different inputs (use higher LLM temperature)

**What happens if it fails or returns nothing:**
- Empty/whitespace `outfit` string (input guard): return "No outfit suggestion was available to generate a caption from." Do not call the LLM.
- LLM/API error (actual failure): return "FitFindr found your item but was unable to generate a fit caption. The item was: {title} — ${price} on {platform}." Do not raise an exception.

---

### Additional Tools (if any)

No additional tools will be used

---

## Planning Loop

**How does your agent decide which tool to call next?**

- Step 1: First the agent looks at the user's input of what type of new clothing item they want. Then the agent asks the LLM (low temperature) to extract 3 things from it: the description of the clothing, the size of the item, and the maximum price the user is willing to pay. If the user did not mention a size or price, the LLM should return `None` for that field. The extracted values are stored in `session["parsed"]` as `{"description": str, "size": str or None, "max_price": float or None}`.

- Step 2: Then these attributes are given to `search_listings()` as input and the tool is run. After that, the agent checks the returned value from the the tool. 
     1. If it is empty or an error was caused, then `session["error"]`="Try some other attributes because the attributes you gave were not matched with any listings." and `session["error"]` is returned. Also, the agent stops the loop here and does not call the next function (`suggest_outfit()`). 
     2. If the `search_listings()` tool does return the matched items, then they are put into `session["search_results"]`, then the top item from that returned list is saved in a variable in `session["selected_item"]`. 

- Step 3: Then `suggest_outfit()` is called:
     1. If the LLM/API error, then `session["error"]`= "FitFindr found your item but was unable to generate a styling suggestion. The item found was: {title} — ${price} on {platform}." and `session["error"]` is returned. Do not raise an exception and stop the loop. 
     2. If it returns a string, then the suggestion is saved in `session["outfit_suggestion"]` and the agent calls the next function `create_fit_card()`. 

- Step 4: Then `create_fit_card()` is run and 
     1. If there was an LLM error, then `session["error"]`= "FitFindr found your item but was unable to generate a fit caption. The item was: {title} — ${price} on {platform}." and `session["error"]` is returned. Do not raise an exception. 
     2. If it does return a string, then save that in `session["fit_card"]`and return that to the user as the final output.


---

## State Management

**How does information from one tool get passed to the next?**
A session dict is initialized at the start of each run and acts as the shared state for the entire interaction. Each tool writes its output into a key in this dict, and the next tool reads from it rather than receiving values directly.

Keys tracked: `session["parsed"]` (extracted description/size/price), `session["search_results"]` (listings found), `session["selected_item"]` (top result, passed to suggest_outfit), `session["outfit_suggestion"]` (passed to create_fit_card), `session["fit_card"]` (final output), and `session["error"]` (set if the loop exits early, None on success).

---

## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | |
| suggest_outfit | Wardrobe is empty | |
| create_fit_card | Outfit input is missing or incomplete | |

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

**Milestone 4 — Planning loop and state management:**

---

## A Complete Interaction (Step by Step)

Write out what a full user interaction looks like from start to finish — tool call by tool call. Use a specific example query.

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1:**
It takes the input and uses that input to extract information from it to use it in the search_listings() function. This function takes the name of the product, the size and the price as a parameter.

**Step 2:**
<!-- What happens next? What was returned from step 1? What tool is called now? -->
It returns the top 3 listing matched according to the paramters given to the search_listings function. FitFindr then picks the top result from these 3 suggested listings and gives it to suggest_outfit() which takes this top listing and the user's wardrobe as a parameter. This function is supposed to use these 2 inputs and returns a suggested outfit using the new item and the user's current wardrobe. This output is then given to the next create_fit_card() function.

**Step 3:**
The create_fit_card() function will take the suggestion from suggest_outfit() and the new item as the paramter and use these to create a fit card which the user can use as a caption or something like that in their social media posts when they wear the item.

**Final output to user:**
"thrifted this faded band tee off depop for $22 and honestly it was made for my wide-legs 🖤 full look in my stories"
