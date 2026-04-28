# Jewish-Texts-Reference

A Claude Code plugin for looking up Jewish texts and references through the [Sefaria](https://www.sefaria.org) library, via the official [Sefaria MCP server](https://mcp.sefaria.org/sse).

Covers Tanakh, Mishnah, Talmud Bavli & Yerushalmi, Halakha (Mishneh Torah, Shulchan Aruch), Kabbalah (Zohar), Midrash, Liturgy, and the full commentary tradition — plus Hebrew/Aramaic dictionaries and a topic graph.

## Skills

- **find-text** — retrieve the actual text of a known passage. Trigger when the user names a specific reference ("Genesis 1:1", "Berakhot 2a", "Mishneh Torah, Teshuvah 3:4") and wants to read it in Hebrew, English, or both. Includes commentary lookup, manuscript images, and bibliographic info.

- **find-reference** — discover *where* in the library something is said. Combines keyword search, English semantic search, in-book search, dictionary lookup, and topic browsing for users who have a concept or phrase but not a citation.

## Bundled MCP server

This plugin attaches the Sefaria SSE MCP server (`https://mcp.sefaria.org/sse`) automatically. No API key required — Sefaria's MCP endpoint is public.

Tools exposed:

- `get_text`, `get_english_translations`, `get_links_between_texts`
- `text_search`, `english_semantic_search`, `search_in_book`, `search_in_dictionaries`
- `get_topic_details`, `get_current_calendar`
- `clarify_name_argument`, `clarify_search_path_filter`
- `get_text_catalogue_info`, `get_text_or_category_shape`
- `get_available_manuscripts`, `get_manuscript_image`

## Installation

```bash
claude plugins marketplace add danielrosehill/Claude-Code-Plugins
claude plugins install jewish-texts-reference@danielrosehill
```

Restart Claude Code after installation.

## License

MIT
