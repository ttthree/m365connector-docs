---
name: Research
description: Combine M365 work data and Lumina web search to answer research questions comprehensively. Supports --high flag for exhaustive mode.
alwaysAllow:
  - mcp__m365-connector__search
  - mcp__m365-connector__open
  - mcp__m365-connector__find_people
  - mcp__m365-connector__count_people
  - mcp__m365-connector__whoami
  - mcp__lumina-search__LuminaSearch
---

# Research

Combine two search backends to answer user questions thoroughly:

- **M365 Connector MCP** (`mcp__m365-connector__search`, `mcp__m365-connector__open`, `mcp__m365-connector__find_people`, `mcp__m365-connector__count_people`, `mcp__m365-connector__whoami`) — searches the user's Microsoft 365 environment (emails, files, Teams chats, calendar events, people, external connectors).
- **Lumina Search MCP** (`mcp__lumina-search__LuminaSearch`) — searches the public web (backed by Bing).

**IMPORTANT: Always use BOTH backends for every research query.** Fire M365 Connector and Lumina Search in parallel, then filter and weight results by relevance. Never skip one backend — internal and external context together produce the most complete answers.

## M365 Connector MCP Tools

### search

```
mcp__m365-connector__search
```

| Parameter | Required | Description |
|---|---|---|
| `query` | Yes | Search query string |
| `conversation_id` | Yes | Stable identifier for the conversation — reuse the same value across all calls in a single user request |
| `source` | No | `all` (default), `email`, `files`, `chat`, `events`, `people`, `external` |
| `connector` | No | Specific external connector name when `source=external` |
| `size` | No | Results to return (default 10; use 25+ for comprehensive searches) |
| `from` | No | Pagination offset (default 0) |

### open

```
mcp__m365-connector__open
```

| Parameter | Required | Description |
|---|---|---|
| `read_handle` | Yes | Opaque handle from a search result |
| `conversation_id` | Yes | Same conversation ID used in the originating search |

Returns content with completeness metadata: `full`, `partial`, or `snippet`.

### find_people

```
mcp__m365-connector__find_people
```

| Parameter | Required | Description |
|---|---|---|
| `query` | No | Free-text query matched across person fields |
| `conversation_id` | Yes | Same conversation ID used across all calls |
| `displayName` | No | Filter by display name |
| `department` | No | Filter by department (prefix match) |
| `title` | No | Filter by job title (prefix match) |
| `office` | No | Filter by office location (prefix match) |
| `alias` | No | Filter by alias (mailNickname) |
| `principalName` | No | Filter by user principal name |
| `size` | No | Max results to return (default 20, max 50) |
| `include_direct_reports` | No | Include direct reports for each result |

### count_people

```
mcp__m365-connector__count_people
```

Returns only the count of people matching the same filters as `find_people`. Use for quick headcount queries.

### whoami

```
mcp__m365-connector__whoami
```

| Parameter | Required | Description |
|---|---|---|
| `conversation_id` | Yes | Conversation identifier |

Returns the current signed-in user's profile from Microsoft Graph.

## Lumina Search MCP Tool

### LuminaSearch

```
mcp__lumina-search__LuminaSearch
```

| Parameter | Required | Description |
|---|---|---|
| `query` | Yes | Search query string |

Returns search results with citations and semantic documents from the public web.

## Search Strategy

### 1. Always search both backends in parallel

**Every research query fires both M365 Connector MCP and Lumina Search MCP simultaneously.** Assess the question to decide weighting and source selection, but never skip a backend:

- **Internal-leaning question** (e.g. "What time is my meeting with Alice tomorrow?"): M365 is primary, but still fire a Lumina search for supplementary context.
- **External-leaning question** (e.g. "What's the latest on React 19?"): Lumina is primary, but still fire an M365 search to catch any internal discussions or docs.
- **Open-ended question** (e.g. "Summarize what's been happening on Project X"): Both backends equally weighted.
- **People question** (e.g. "Who is Alice Smith?"): Use `find_people` / `count_people` in addition to `search`, plus Lumina for public profile info.

### 2. Choose sources and parallelize

For M365 searches, unless the user's question clearly implies a single source (e.g. "check my email" -> `email`), search `source=all` **plus** other likely specific sources in parallel:

- Question about a project: `all` + `files` + `chat` in parallel
- Question about a person: `all` + `people` + `email` in parallel
- Question about a meeting: `all` + `events` in parallel
- Ambiguous question: `source=all`

Fire parallel searches with different query phrasings or sources to maximize recall. **Always include Lumina searches in every parallel batch.**

### 3. Open results for depth (M365)

Search results are summaries. Use `open` to read the full content of promising results. Open multiple items in parallel when several look relevant. Prioritize opening:

- Results whose snippets are most relevant to the question
- More recent items when timeliness matters
- Items from authoritative sources (e.g. official docs over chat messages)

**Cache opened content**: After opening an item, persist its content to a local file (e.g. `research-cache/<conversation_id>/<item-id>.md` or `.txt`). This enables:
- Reuse in follow-up questions without re-calling `open`
- Reference during artifact generation (documents, slides, summaries)
- Reduced API calls and faster responses

Include metadata at the top of each cached file: source type, title, date, and the original `read_handle` for traceability.

### 4. Deepen with follow-up searches

Since both backends are always queried upfront, use this step to fill remaining gaps:
- If M365 results are thin, try rephrasing queries or searching additional M365 source types.
- If Lumina results are thin, try alternate phrasings or more specific sub-queries.
- Use entities discovered in initial results (people, project names, dates) to launch targeted follow-up searches on both backends.

### 5. Handle recency and conflicts

- When information conflicts across sources, **favor the most recently modified item**.
- For time-sensitive questions, sort mentally by date and weight newer results higher.
- If search results seem stale or incomplete, try rephrasing the query or searching additional sources.

### 6. Iterate and paginate for comprehensive coverage

Search results are paginated. For complex topics or tasks requiring comprehensive information:

- **Request more results per page**: Increase `size` (at least 25) for broader initial coverage.
- **Paginate through results**: Use the `from` parameter to retrieve additional pages. E.g., first call `from=0, size=10`, then `from=10, size=10` for the next page.
- **When to paginate**:
  - Open-ended research questions that need thorough coverage
  - When first-page results are relevant but incomplete
  - When building a comprehensive summary across many items
  - When the user explicitly asks for "all" or "everything" about a topic
- **Refine queries**: Try synonyms, different phrasing, or narrower/broader terms if initial results miss the mark.
- **Follow-up searches**: For broad topics, perform additional searches based on entities or terms discovered in earlier results.

### 7. Know when to stop

- **Scoped questions**: Stop as soon as you have a confident answer. Do not over-search.
- **Open-ended questions**: Continue until you have covered the key facets of the topic or returns become repetitive/irrelevant.
- **`--high` mode**: Override default stopping criteria — see the `--high` Flag section below. Keep searching until results become clearly repetitive across multiple query phrasings.
- Always balance thoroughness against efficiency — do not perform redundant searches.

## Response Format — Internal vs. External

When presenting the final response, **separate internal and external findings** if both backends returned relevant results. This helps the user distinguish between their private work data and public information.

### When both sources have relevant results

Structure the response with clear section separation:

```
[Brief overview / direct answer to the question]

### Internal (Microsoft 365)
- Findings from emails, files, Teams chats, calendar events, people directory
- Include source attribution (e.g., "From an email dated...", "In a Teams chat...")

### External (Web)
- Findings from Lumina web search
- Include source URLs/citations where available
```

### When only one source has relevant results

Present findings normally without the Internal/External separation. No need to mention the empty source unless the user would expect results from it (e.g., "No internal documents were found about this topic.").

### Cross-referencing

When information from internal and external sources relates to the same topic:
- Highlight agreements or confirmations across sources
- **Flag conflicts clearly** — state which source says what and note which is more recent or authoritative
- Use internal data for company-specific facts, external data for industry/public context

## `--high` Flag — Exhaustive Research Mode

When the user includes `--high` in their request, switch to exhaustive information gathering mode. The goal is to cast the widest net possible and produce a deeply comprehensive answer.

### Detecting the flag

Look for `--high` anywhere in the user's message. Strip it from the search query itself — it is a mode flag, not a search term.

### How `--high` changes behavior

| Aspect | Normal mode | `--high` mode |
|---|---|---|
| **M365 `size` per query** | 10 (default) | 25–50 |
| **Pagination** | Only if first page is insufficient | Always paginate at least 2–3 pages per source |
| **M365 sources searched** | Most likely source(s) + `all` | **All source types** in parallel: `all`, `email`, `files`, `chat`, `events`, `people` |
| **Query phrasings** | 1–2 phrasings | 3+ phrasings per topic (synonyms, alternate terms, related concepts) |
| **Lumina web searches** | 1 search (if needed) | 2–4 searches with different angles/phrasings |
| **Results opened (M365)** | Top 2–3 relevant items | Top 5–10 relevant items |
| **Follow-up searches** | Only if initial results are thin | Always — mine entities, names, and terms from initial results to launch secondary searches |
| **Stopping threshold** | Stop when confident | Continue until results are clearly repetitive across multiple query phrasings |

### `--high` search workflow

1. **Broad initial sweep**: Fire parallel searches across ALL M365 source types with `size=25` or higher, plus 2+ Lumina searches with different phrasings. Do all of these in a single parallel batch.

2. **Deep dive**: Open the top 5–10 most relevant M365 results in parallel. Read full content, not just snippets.

3. **Entity extraction and follow-up**: From the opened results, identify key entities (people, projects, dates, terms) and launch secondary searches targeting those entities. For Lumina, try 1–2 additional queries based on what you learned.

4. **Paginate for coverage**: For each M365 source that returned a full page of results, fetch at least one more page (`from=25, size=25`). For Lumina, if initial results are rich, search with more specific sub-queries.

5. **Cross-reference and synthesize**: Compare information across sources. Note conflicts, flag uncertainty, and prefer the most recent data.

6. **Comprehensive output**: Provide a thorough, well-structured answer that covers multiple facets. Include source attribution. Don't truncate findings — the user wants depth.

### Example

User input: `What's been happening on Project Atlas? --high`

Normal mode might do: 1 M365 search (`source=all`, `size=10`), open 2–3 results, summarize.

`--high` mode should do:
- M365 parallel: `source=all` (size=25), `source=email` (size=25), `source=files` (size=25), `source=chat` (size=25), `source=events` (size=25) — all with query "Project Atlas"
- M365 parallel (alternate phrasing): same sources with query "Atlas project update"
- Lumina parallel: "Project Atlas" + "Atlas project latest news"
- Open top 8–10 results from M365
- Extract names/dates from results, run follow-up searches for key people or milestones
- Paginate M365 sources that returned full pages
- Synthesize a comprehensive multi-faceted summary

## Conversation ID

Generate a stable `conversation_id` once per user request (e.g. `"conv-<short-uuid>"`) and reuse it across all M365 search and open calls for that request. This enables the backend to group related queries.
