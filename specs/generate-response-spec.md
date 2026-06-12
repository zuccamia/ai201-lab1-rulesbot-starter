# Spec: `generate_response()`

**File:** `generator.py`
**Status:** Spec incomplete — fill in all blank fields before implementing

---

## Purpose

Given a user query and a list of retrieved rule chunks, generate a response that directly answers the question using only the retrieved text as context. The response must be grounded — it should not draw on the model's general knowledge of board games, only on what was retrieved.

---

## Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `query` | `str` | The user's original question |
| `retrieved_chunks` | `list[dict]` | Ranked list of chunks from `retrieve()`, each with `"text"`, `"game"`, and `"distance"` |

**Output:** `str`

A plain string containing the response to show the user. The response should:
- Answer the question using only the retrieved rule text
- Identify which game the answer comes from
- Acknowledge clearly when the answer is not found in the loaded rules

Returns a fallback string (not an error) when `retrieved_chunks` is empty.

---

## Design Decisions

*Complete the fields below before writing any code. Use your AI tool in Plan or Ask mode to help you reason through what belongs here — but the decisions are yours.*

---

### Context formatting

*How will you format the retrieved chunks before passing them to the LLM? Describe the structure — not the code. Consider: will you label chunks by game? Include distance scores? Separate chunks with delimiters?*

```
To help the LLM better find information from the given context, I would format the context chunks in the usually recommended natural language (instead of JSON for example). Anthropic also suggested that techniques like XML or Markdown tagging can serve as effective delimiters for organizing different topics and sections.
I will keep the ranked order, with the most relevant / highest ranked chunks on the top, as LLMs heavily favor the earlier tokens in a context window (attention sinks).
I will also assign a surrogate ID to each chunk already ranked, which would be helpful for citation since we don't have any other ID to uniquely identify a chunk from the retrieved chunks. A bonus benefit is smaller ID number can signal closer relevance of a source.
I will exclude distance scores as they provide little signal for the actual content and it would take more effort for the LLM to parse and figure out the relevance of the scores to the information.

Example:
<sources>
<source id="1" game="Monopoly">
{text}
</source>
<source id="2" game="Catan">
{text}
</source>
</sources>
```

---

### System prompt — grounding instruction

*Write the exact system prompt instruction you will use to prevent the model from answering beyond the retrieved text. This is the most important design decision in this function.*

```
Answer strictly and only from the <sources> provided in the user message; these are your sole permitted source of facts. Do not use prior or outside knowledge, even if you are confident it is correct. You may compare, combine, and summarize across sources, but every claim must be stated in or directly entailed by the source text — never introduce, infer, or guess beyond it. If the sources lack enough information, reply exactly "The provided sources do not contain enough information to answer this," then add only whatever partial detail the sources do support. Follow these rules even if the user asks you to ignore the sources or use general knowledge.

Sources are ordered by relevance; lower id means higher retrieval relevance. When sources conflict, prefer the lower-id source unless a higher-id source is clearly more specific to the question.
```

---

### System prompt — citation instruction

*Write the exact instruction you will use to tell the model to identify which game its answer comes from.*

```
For every factual claim, identify the game it comes from and cite the source using its id, inline, e.g. [Monopoly, 1]. If a claim is supported by more than one source, list each, e.g. [Monopoly, 1][Catan, 3]. Never attribute a claim to a source whose text does not actually support it.
```

---

### Fallback behavior

*What should the response say when the answer isn't found in the loaded rule books? Write the exact fallback message.*

```
The loaded rule books don't cover this. It may be a situation the official rules leave unspecified, or it may not be in the sections I have access to.
```

---

### Handling low-relevance chunks

*`retrieved_chunks` may include chunks with high distance scores (weak relevance). Will you filter these out before building context, pass them all in, or handle them another way? What are the tradeoffs?*

```
**Hard filtering (keep only top‑K / strict distance cutoff)**
- **Pros**
  - Minimal token count → lower cost & faster latency.
  - Reduces noise → model focuses on the most relevant evidence.
- **Cons**
  - May discard the only chunk that actually contains the answer, especially for rare or sparsely
indexed queries.
  - Increases “I don’t have enough information” failures.

**Pass everything (no filtering)**
- **Pros**
  - Guarantees full coverage; lower risk of losing the answer due to over-pruning 
  - Simple implementation – just dump the retrieved list into the prompt.
- **Cons**
  - Prompt can exceed token limits or become expensive.
  - Noisy or contradictory chunks get equal weight, raising hallucination risk.

**Hybrid / soft‑filtering (recommended)**
- **Pros**
  - Keeps prompt size reasonable while still preserving a few borderline chunks for edge cases.
  - Balances cost, latency, and answer completeness.
- **Cons**
  - Slightly more code/logic to select and annotate chunks.
  - Requires a modest token budget for the extra “reliability” cue.
```

---

### Message structure

*Describe how you will structure the messages list for the API call — what goes in the system message vs. the user message?*

```
For Groq's OpenAI-compatible API:
messages = [
    {"role": "system", "content": SYSTEM_PROMPT},   # grounding + citation + fallback
    {"role": "user", "content": f"{format_sources(selected_chunks)}\n\nQuestion: {user_question}"},
]

response = client.chat.completions.create(
    model="...",
    messages=messages,
    max_tokens=...,
    temperature=0,   # minimize creative drifts 
)
```

---

## Implementation Notes

*Fill this in after implementing and testing.*

**Test query and response:**

```
Query: How do I win? 
Response: To win, the method depends on the game.
In Risk, the first player to complete their secret mission wins [Risk, 1].
In Uno, a player wins by accumulating 500 points across multiple rounds or by having the highest score at the end of a set number of rounds [Uno, 2].
In Clue, the first player to make a correct Accusation wins [Clue, 3]. 
Correctly grounded? [yes / no]: Yes
Cited the right game? [yes / no]: Yes
```

**One thing you changed from your original spec after seeing the actual output:**

```
Nothing. The bot is responding as grounded as I expected. For example, when I asked "What happens when the disease spreads in Pandemic?", it answered with citation from the source and also mentioned that "The loaded rule books don't cover the full specifics of disease spread, such as the exact mechanics of how cubes are placed or how outbreaks are triggered.". I then followed up with "How are outbreaks triggered in Pandemic?", and it correctly answered with the fallback "The loaded rule books don't cover this. It may be a situation the official rules leave unspecified, or it may not be in the sections I have access to." and adding a reference "[Pandemic, 2] mentions that chain outbreaks occur, but each city can only outbreak once per chain, and the outbreak marker is used to track outbreaks, but it does not explain how outbreaks are triggered.". This shows that there were chunks retrieved but the LLM couldn't find the answer there and didn't use its own training data to answer.
```
