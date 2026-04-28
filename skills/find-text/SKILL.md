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

## Procedure

1. **Resolve the reference.** If the user's wording is ambiguous, partial, or transliterated unusually, call `clarify_name_argument` (with `type_filter: "ref"` when looking for a textual citation) to get the canonical Sefaria reference string before fetching. Examples that benefit from clarification: "the part of Avot about Hillel", "Sanhedrin daf 90", "Rambam on repentance".

2. **Fetch the text.** Call `get_text` with the canonical reference. Pass `version_language`:
   - `"source"` — Hebrew/Aramaic only
   - `"english"` — English translation only
   - `"both"` — side-by-side (default for most reading requests)
   - omit — every available version

3. **Translation choice.** If the user wants a particular English translation or wants to see all available translations side-by-side, call `get_english_translations` instead of (or in addition to) `get_text`.

4. **Commentary on the passage.** If the user wants to read what commentators say *on* this verse/sugya (Rashi, Ramban, Tosafot, Steinsaltz, etc.), call `get_links_between_texts` with `with_text: "1"` to pull the linked commentary text in one go. Filter or summarise the returned links by commentator if the response is large.

5. **Manuscripts.** If the user asks about historical manuscript witnesses or wants to see a scanned page, call `get_available_manuscripts` and offer `get_manuscript_image` for a specific image URL.

6. **Bibliographic / structural questions.** If the user asks "how is this book organised" or "who wrote this and when", use `get_text_catalogue_info` (index data) or `get_text_or_category_shape` (hierarchy) — not `get_text`.

## Presentation

- For Hebrew + English, render Hebrew right-aligned in a quote block followed by the English. Preserve verse numbers / daf-amud markers from Sefaria's response.
- Hebrew text from Sefaria includes nikkud (vowel points) by default — keep them. Strip only if the user explicitly asks for unvocalized text.
- Always cite the canonical reference Sefaria returned, plus a Sefaria URL of the form `https://www.sefaria.org/<reference-with-underscores>` so the user can open it in a browser.
- For long passages (full chapters, full daf), summarise structure first and ask whether to dump the full text, unless the user explicitly asked for the complete passage.

## Edge cases

- **Talmud daf references** use side notation: `Berakhot 2a`, `Berakhot 2b`. If a user says "Berakhot 2", clarify which side, or fetch both and present them.
- **Mishneh Torah / Shulchan Aruch** require the full path: `Mishneh Torah, Teshuvah 3:4`, `Shulchan Aruch, Orach Chayim 1:1`. Use `clarify_name_argument` if unsure.
- **Zohar** references use the section + page format: `Zohar 1:15a`.
- If `get_text` returns empty for a valid-looking reference, the version may not exist in that language — fall back to `version_language: null` to see what's available, or call `get_english_translations` to enumerate.
