# Spec: `retrieve()`

**File:** `retriever.py`
**Status:** Spec incomplete — fill in all blank fields before implementing

---

## Purpose

Given a user's natural language query, find the most relevant chunks from the vector store using semantic similarity search. Return them ranked by relevance so that `generate_response()` can use them as context.

---

## Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `query` | `str` | The user's natural language question |
| `n_results` | `int` | Maximum number of chunks to return (default: `N_RESULTS` from `config.py`) |

**Output:** `list[dict]`

Each dict in the returned list must contain exactly these keys:

| Key | Type | Description |
|-----|------|-------------|
| `"text"` | `str` | The chunk text |
| `"game"` | `str` | The game name this chunk came from |
| `"distance"` | `float` | Cosine distance score — lower means more similar to the query |

Results should be ordered from most to least relevant (lowest to highest distance). Returns an empty list `[]` if the collection contains no documents.

---

## Design Decisions

*Complete the fields below before writing any code. Use your AI tool in Plan or Ask mode to help you reason through what belongs here — but the decisions are yours.*

---

### Query approach

*Describe how you will use `_collection.query()` to find relevant chunks. What arguments will you pass, and why?*

```
I will need to pass the following:
- the list of given query strings `query="What is the capital of France?"`
- the optional limit for the number of results to return: `n_results=3`
- what fields to include: `include_documents=True`, `include_distances=True`, etc
```

---

### Return structure

*Sketch out what one item in your return list looks like as a concrete example. Where does each field come from in the query results?*

```
Each item would be a dict, looking like this:
{
    "text": "a player rolls a seven, the robber must be moved to any hex of their choice, blocking that hex’s resource production, and the player who moved it may steal one random resource card from
any opponent who has a settlement or city adjacent to the robber’s new location, which may shift
trade flow.",
    "game": "Catan",
    "distance": 0.43
}

The value for "text" would come from the field "documents", "game" as dug from "metadatas" (created during the embed_and_store step), and "distance" from "distances". These fields should be requested to be included when we query from the collection.
```

---

### Handling the nested result structure

*`_collection.query()` returns nested lists. Describe what index you need to access to get the actual list of results for a single query, and why the nesting exists.*

```
For a single query, there would be only 1 list of results, so we can just use 0 to get the actual result list in the nested list of each key in the response (e.g. ids, documents, metadatas).
The nesting serves such case when user may pass in multiple query strings, resulting in multiple embedding vectors. Each sublist in the nested result list corresponds to all the matching chunks for each query string.
```

---

### Relevance threshold

*Will you filter out results above a certain distance score, or return all `n_results` regardless of how relevant they are? What are the tradeoffs of each approach?*

```
Depending on the quality of the chunks stored in the collection.
If most chunks tend to have low semantic signal, many of the results will be below a fixed distance score. Filtering them out may leave out the answer hidden between the boundaries of the chunks.
On the other hand, returning all top k results of all distances may create noise for the models with loose context that can pull the response off-track.
```

---

### Edge cases

*How does your implementation behave when: (a) the collection is empty, (b) the query matches no chunks well, (c) the query matches chunks from multiple games?*

```
(a) returns an empty list []
(b) returns an empty list [] if all distances are exceptionally high (over 0.7 for example) so avoid non-existing answer
(c) returns a list consisting of multiple dicts, each representing a matching chunk to the single query given 
```

---

## Implementation Notes

*Fill this in after implementing, before moving to Milestone 3.*

**Test query and top result returned:**

```
Query: "What happens if you roll a 7 in Catan?"
Top result game: Catan
Distance score: 0.471 for the top result
Does it make sense? [yes / no / explain] Yes. The top result clearly contains a text mentioning "rollling a 7" for the game of "Catan". The last chosen result is a chunk of "Uno" game, which was a far shot with distance score of 0.62, but the text does contain rule about card number 7.
```

**One thing about the query results that surprised you:**

```
It includes a chunk about Uno game even though the query string contains "Catan" explicitly.
```
