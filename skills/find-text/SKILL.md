---
name: find-text
description: Retrieve the actual text of a known passage in the Jewish library via Sefaria — Tanakh, Talmud, Mishnah, Halakha, Kabbalah, commentaries. Use when the user names a specific reference (e.g. "Genesis 1:1", "Berakhot 2a", "Mishneh Torah, Hilchot Teshuvah 3:4") and wants to read it, or asks for a passage in Hebrew, English, or both.
---

# Find Text

Use this skill when the user wants to **read a specific passage** they can name or roughly describe. The goal is to return the text itself (Hebrew source, English translation, or both), not to discover new sources — for that, use `find-reference`.

## When to invoke

Trigger on requests like:

- "Show me Genesis 1:1"
- "What does Berakhot 2a say?"
- "Give me the Hebrew of Shema Yisrael" → `Deuteronomy 6:4`
- "Read me Pirkei Avot 1:1 in English"
- "What's the text of Rashi on Genesis 1:1?"
- "Show me Mishneh Torah, Hilchot Shabbat 1:1"

Skip for open-ended topical questions ("what does Judaism say about X") — that's `find-reference`.

## Route

Sefaria's public REST API with `curl`. No key, no MCP server. Base `https://www.sefaria.org`.
Send a `User-Agent` header and URL-encode references (spaces as `%20` or `_`). Verified 2026-11-05.

## Procedure

1. **Resolve the reference.** If the wording is ambiguous, partial, or transliterated unusually:
   `curl -s -A canopy-skill 'https://www.sefaria.org/api/name/Rashi%20on%20Genesis%201:1'`.
   `is_ref: true` gives the canonical `ref` and `url`. Otherwise read `completions` for candidates.

2. **Fetch the text.** `GET /api/v3/texts/<ref>?version=<v>&return_format=text_only`. Repeat `version` for several:
   - `version=source`: Hebrew/Aramaic only
   - `version=english`: English translation only
   - `version=source&version=english`: side by side (default for most reading requests)
   - `version=translation|all`: every translation in any language

   The response has `ref`, `heRef` and `versions[]`, each with `language`, `versionTitle` and `text`.
   Drop `return_format=text_only` if you need Sefaria's markup and footnotes.

3. **Translation choice.** `version=english|all` lists every English translation (17 for Genesis 1:1).
   `version=english|<versionTitle>` picks one.

4. **Commentary on the passage.** `GET /api/links/<ref>?with_text=1` returns linked text with `category`,
   `type`, `collectiveTitle`, `ref`, `text` and `he`. It is large (1,828 links for Genesis 1:1), so pipe it through
   `jq`, keep `category == "Commentary"`, and filter by the commentator asked for. Summarise unless asked for it verbatim.

5. **Manuscripts.** `GET /api/manuscripts/<ref>` returns `manuscript_slug`, `image_url` and `thumbnail_url` per witness.

6. **Bibliographic / structural questions.** Use `GET /api/v2/raw/index/<title>` for author, date, categories and
   schema, and `GET /api/shape/<title>` for the hierarchy, not the text endpoint.

## Presentation

- For Hebrew + English, render Hebrew right-aligned in a quote block followed by the English. Preserve verse numbers / daf-amud markers from Sefaria's response.
- Hebrew text from Sefaria includes nikkud (vowel points) by default — keep them. Strip only if the user explicitly asks for unvocalized text.
- Always cite the canonical reference Sefaria returned, plus a Sefaria URL of the form `https://www.sefaria.org/<reference-with-underscores>` so the user can open it in a browser.
- For long passages (full chapters, full daf), summarise structure first and ask whether to dump the full text, unless the user explicitly asked for the complete passage.

## Edge cases

- **Talmud daf references** use side notation: `Berakhot 2a`, `Berakhot 2b`. If a user says "Berakhot 2", clarify which side, or fetch both and present them.
- **Mishneh Torah / Shulchan Aruch** require the full path: `Mishneh Torah, Teshuvah 3:4`, `Shulchan Aruch, Orach Chayim 1:1`. Use `/api/name/` if unsure.
- **Zohar** references use the section + page format: `Zohar 1:15a`.
- If the text endpoint returns no text for a valid-looking reference, the version may not exist in that language. List what exists with `version=source|all` and `version=translation|all`. Plain `version=all` returns nothing.
