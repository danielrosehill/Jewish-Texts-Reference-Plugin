---
name: find-reference
description: Find passages, sources, or topical references in the Jewish library via Sefaria when the user knows the concept but not the citation. Combines keyword search, semantic search, dictionary lookup, and topic browsing across Tanakh, Talmud, Halakha, Kabbalah, and commentary.
---

# Find Reference

Use this skill when the user wants to **discover where in the Jewish library** something is said — they have a phrase, concept, term, theme, or partial quote, but not a citation. The goal is to return the relevant references (and short excerpts), so the user can then drill into a passage with `find-text`.

## When to invoke

Trigger on requests like:

- "Where does the Talmud talk about lashon hara?"
- "Find sources about welcoming guests"
- "What does Sefaria say about the topic 'sabbath'?"
- "I'm looking for the verse that says 'love your neighbour as yourself'"
- "Define the term tikkun olam — show me primary sources"
- "Find passages in Mishneh Torah about repentance"

Skip when the user already names a specific reference — that's `find-text`.

## Route

Sefaria's public REST API with `curl`. No key, no MCP server. Base `https://www.sefaria.org`.
Send a `User-Agent` header and URL-encode the query. Verified 2026-11-05.

## Choosing the right search

| Query shape | Call |
|---|---|
| Hebrew/Aramaic phrase, exact wording, or distinctive English keyword | `POST /api/search-wrapper` |
| Searching only within one work or category | `POST /api/search-wrapper` with `filters` |
| Defining a term in Jastrow, Klein, BDB, Ben Yehuda | `GET /api/words/<word>` |
| Browsing a known topic by slug (`moses`, `sabbath`, `repentance`) | `GET /api/topics/<slug>?with_refs=1&with_links=1` |
| Validating a name, slug or book | `GET /api/name/<text>` |
| This week's parasha, today's daf | `GET /api/calendars?timezone=Asia/Jerusalem&diaspora=0` |

Search call:

```bash
curl -s -A canopy-skill -H 'Content-Type: application/json' https://www.sefaria.org/api/search-wrapper \
  -d '{"query":"לשון הרע","type":"text","size":10,"field":"naive_lemmatizer","filters":["Talmud"],"filter_fields":["path"]}'
```

Use `field: "naive_lemmatizer"` for Hebrew and `"exact"` for an English keyword. Hits are in `hits.hits[]`. The reference is
in `_id`, in the form `Yoma 22b:15 (William Davidson Edition - Aramaic [he])`, and the snippet is in `highlight.<field>`.
The same passage appears once per version, so deduplicate on the reference before the parenthesis.

**There is no semantic search on this route.** The MCP's `english_semantic_search` has no public REST equivalent. For an
English concept, search its Hebrew technical term, browse the topic page, and say the search was by keyword.

For broad questions, run the keyword search and the topic lookup together and merge the results.

## Procedure

1. **Disambiguate the request.** Decide whether the user wants:
   - A single source (the verse / sugya / halakha)
   - A topical survey (multiple sources across the corpus)
   - A definition (dictionary)
   - A specific work scoped (Mishneh Torah, Talmud Bavli, etc.)

2. **Resolve names if needed.** If the query mentions a book, topic slug, or author whose canonical Sefaria form is uncertain, call `/api/name/` first. For a book-scoped search, `filters` takes the category path (`["Talmud"]`, `["Tanakh"]`, `["Halakhah"]`) with `filter_fields: ["path"]`.

3. **Hebrew vs English.** Hebrew/Aramaic queries are markedly more reliable in search than English (translations vary). When the user gives an English concept that has a well-known Hebrew/Aramaic technical term (lashon hara, teshuvah, tzedakah, hachnasat orchim, kavod, kedushah), search the Hebrew term as well.

4. **Run the search.** Default `size: 10` is usually fine; raise to 25–50 for topical surveys. Apply `filters` (e.g. `["Tanakh"]`, `["Talmud"]`, `["Halakhah"]`) when the user has scoped the search.

5. **Topic browsing.** When the user names a recognised topic (sabbath, moses, prayer, charity), call `/api/topics/<slug>?with_refs=1&with_links=1` for canonical references (`refs`) plus related topics (`links`). Use `/api/name/` to validate the slug.

6. **Definitions.** For "what does X mean" lexical questions, prefer `/api/words/<word>` over search. It returns entries by `parent_lexicon` (Jastrow, Klein, BDB, Ben Yehuda).

7. **Drill down.** Once you have promising hits, fetch the top one or two references with `find-text` to confirm they actually answer the user's question, before returning the list. Don't blindly trust the search snippet — translations can mislead.

## Presentation

- Group results by work (Tanakh / Mishnah / Talmud / Halakha / Kabbalah / Commentary).
- For each hit, show: canonical reference, a short excerpt (Hebrew or English as appropriate), and a `https://www.sefaria.org/...` link.
- Lead with the strongest match (often the topic-page canonical sources), not raw search rank.
- If results are thin, say so explicitly and suggest reformulating in Hebrew or broadening the filter — don't fabricate sources.

## Edge cases

- **No results in English** is common and usually a translation-coverage issue, not a real absence. Re-run in Hebrew.
- **Apocryphal / "everyone knows the Talmud says…" quotes** — many popular quotations are misattributed. If you can't find a source after a careful search, say so plainly rather than guessing.
- **Ambiguous topics** (e.g. "Joseph" — Joseph the patriarch? Joseph Karo? Joseph ben Matityahu?) — call `/api/name/` and confirm with the user before committing to one branch.
- **Calendar-anchored requests** ("what's this week's parasha" / "what's today's daf yomi") — call `/api/calendars` first to anchor, then pull the text with `find-text`.
