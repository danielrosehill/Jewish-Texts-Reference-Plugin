# Jewish-Texts-Reference

A Claude Code plugin for looking up Jewish texts and references through the [Sefaria](https://www.sefaria.org) library, through its public REST API (no key, no MCP server).

Covers Tanakh, Mishnah, Talmud Bavli & Yerushalmi, Halakha (Mishneh Torah, Shulchan Aruch), Kabbalah (Zohar), Midrash, Liturgy, and the full commentary tradition — plus Hebrew/Aramaic dictionaries and a topic graph.

## Skills

- **find-text** — retrieve the actual text of a known passage. Trigger when the user names a specific reference ("Genesis 1:1", "Berakhot 2a", "Mishneh Torah, Teshuvah 3:4") and wants to read it in Hebrew, English, or both. Includes commentary lookup, manuscript images, and bibliographic info.

- **find-reference** — discover *where* in the library something is said. Combines keyword search, in-book search, dictionary lookup, and topic browsing for users who have a concept or phrase but not a citation.

- **strip-nikkud** — remove Hebrew vowel points (nikkud) and/or cantillation marks (te'amim). Two paths: offline regex (default) or Dicta's `removenikud.dicta.org.il` public API.

- **add-nikkud** — vocalise unvocalized Hebrew (the reverse of strip-nikkud). Defaults to Dicta's nakdan public API (best quality, supports modern / rabbinic / poetry genres); falls back to the offline [`unikud`](https://github.com/morrisalp/unikud) Python package on explicit request.

## Route

The skills call `https://www.sefaria.org/api/...` with `curl`: texts, names, search, dictionaries, topics, links,
manuscripts and calendars. Nothing to install or authenticate. Sefaria's MCP server (`https://mcp.sefaria.org/mcp`)
is no longer bundled. The one capability lost with it is English semantic search, which has no public REST equivalent.

## Installation

```bash
claude plugins marketplace add danielrosehill/Claude-Code-Plugins
claude plugins install jewish-texts-reference@danielrosehill
```

Restart Claude Code after installation.

## License

MIT
