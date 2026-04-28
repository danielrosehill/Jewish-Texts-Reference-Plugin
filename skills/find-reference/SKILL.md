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

## Choosing the right search tool

Sefaria exposes several complementary search tools. Pick based on the shape of the query:

| Query shape | Tool |
|---|---|
| Hebrew/Aramaic phrase, exact wording, or distinctive English keyword | `text_search` |
| English concept where the user cares about *meaning*, not exact wording | `english_semantic_search` |
| Searching only within one named work (e.g. "in the Mishneh Torah", "in Berakhot") | `search_in_book` |
| Defining a term or looking it up in Jastrow / Klein / other lexica | `search_in_dictionaries` |
| Browsing a known topic by slug (e.g. `moses`, `sabbath`, `repentance`) | `get_topic_details` |
| Validating a slug, author name, or category before searching | `clarify_name_argument` |

For broad questions, run more than one in parallel — e.g. `text_search` for the Hebrew term and `english_semantic_search` for the concept — and merge results.

## Procedure

1. **Disambiguate the request.** Decide whether the user wants:
   - A single source (the verse / sugya / halakha)
   - A topical survey (multiple sources across the corpus)
   - A definition (dictionary)
   - A specific work scoped (Mishneh Torah, Talmud Bavli, etc.)

2. **Resolve names if needed.** If the query mentions a book, topic slug, or author whose canonical Sefaria form is uncertain, call `clarify_name_argument` first. For book-scoped searches, you may also need `clarify_search_path_filter` to convert a book name into the search filter path that `text_search` accepts via its `filters` argument.

3. **Hebrew vs English.** Hebrew/Aramaic queries are markedly more reliable on `text_search` than English (translations vary). When the user gives an English concept that has a well-known Hebrew/Aramaic technical term (lashon hara, teshuvah, tzedakah, hachnasat orchim, kavod, kedushah), search the Hebrew term as well.

4. **Run the search.** Default `size: 10` is usually fine; raise to 25–50 for topical surveys. Apply `filters` (e.g. `["Tanakh"]`, `["Talmud"]`, `["Halakhah"]`) when the user has scoped the search.

5. **Topic browsing.** When the user names a recognised topic (sabbath, moses, prayer, charity), call `get_topic_details` with `with_refs: true, with_links: true` for a structured view of canonical references plus related topics. Use `clarify_name_argument` with `type_filter: "Topic"` to find the slug if the user gives a phrase.

6. **Definitions.** For "what does X mean" lexical questions, prefer `search_in_dictionaries` over `text_search` — it returns Jastrow / Klein / Even-Shoshan / etc. entries directly.

7. **Drill down.** Once you have promising hits, call `get_text` on the top one or two references to confirm they actually answer the user's question, before returning the list. Don't blindly trust the search snippet — translations can mislead.

## Presentation

- Group results by work (Tanakh / Mishnah / Talmud / Halakha / Kabbalah / Commentary).
- For each hit, show: canonical reference, a short excerpt (Hebrew or English as appropriate), and a `https://www.sefaria.org/...` link.
- Lead with the strongest match (often the topic-page canonical sources), not raw search rank.
- If results are thin, say so explicitly and suggest reformulating in Hebrew or broadening the filter — don't fabricate sources.

## Edge cases

- **No results in English** is common and usually a translation-coverage issue, not a real absence. Re-run in Hebrew.
- **Apocryphal / "everyone knows the Talmud says…" quotes** — many popular quotations are misattributed. If you can't find a source after a careful search, say so plainly rather than guessing.
- **Ambiguous topics** (e.g. "Joseph" — Joseph the patriarch? Joseph Karo? Joseph ben Matityahu?) — call `clarify_name_argument` and confirm with the user before committing to one branch.
- **Calendar-anchored requests** ("what's this week's parasha" / "what's today's daf yomi") — call `get_current_calendar` first to anchor, then pull the text with `find-text`.
