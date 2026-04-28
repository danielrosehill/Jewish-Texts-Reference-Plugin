---
name: strip-nikkud
description: Remove Hebrew vowel points (nikkud) and/or cantillation marks (te'amim) from Hebrew text. Use when the user asks for "unvocalized", "consonantal", "without nikkud", "ktiv male", "strip vowels", "remove te'amim/trope", or asks to clean Hebrew text fetched from Sefaria (which ships with full nikkud and often te'amim) for plain reading, search, or copy-paste.
---

# Strip Nikkud

Two paths — pick based on environment:

- **Offline regex** (default) — pure Python, no network. Fast and deterministic.
- **Dicta API** — `https://removenikud.dicta.org.il/`'s public backend. Use when the user explicitly asks for "Dicta", or when handling unusual edge cases (Yiddish, mixed corpora) where you want their tool's behaviour rather than a raw regex.

## When to invoke

- "Strip the nikkud from this verse"
- "Give me Genesis 1:1 without vowels"
- "Remove the trope marks"
- "Unvocalized Hebrew please"
- After a `find-text` call where the user wants the consonantal text only
- Preparing Hebrew text for a system that doesn't render nikkud well (search engines, plain-text exports, some fonts)

## Path A — Offline regex (default)

### Unicode reference

| Range | Block | Default action |
|---|---|---|
| `U+0591`–`U+05AF` | Cantillation marks (te'amim / trope) | Remove |
| `U+05B0`–`U+05BD` | Nikkud (sheva, vowels, dagesh, meteg, etc.) | Remove |
| `U+05BF` | Rafe | Remove |
| `U+05C1`, `U+05C2` | Shin/sin dot | Remove |
| `U+05C4`, `U+05C5` | Upper/lower dot (rare, masoretic) | Remove |
| `U+05C7` | Qamats qatan | Remove |

**Preserve** these — they are punctuation, not diacritics:

- `U+05BE` maqaf (`־`)
- `U+05C0` paseq (`׀`)
- `U+05C3` sof pasuk (`׃`)
- `U+05C6` nun hafukha (`׆`)
- `U+05F3`, `U+05F4` geresh, gershayim (`׳`, `״`)

### Procedure

1. Confirm scope — strip `nikkud` only, `teamim` only, or `both` (default for "unvocalized").

2. Run via `python3 -c` in Bash. Use **explicit `\u` escapes** — the literal-character form silently includes punctuation code points and strips them too. The character class below has deliberate gaps at U+05BE / U+05C0 / U+05C3 / U+05C6 to preserve maqaf, paseq, sof pasuk, and nun hafukha:

   ```python
   import re

   # Te'amim (cantillation) only — U+0591 to U+05AF
   TEAMIM = r'[֑-֯]'

   # Nikkud only — vowels, dagesh, sheva, meteg, rafe, shin/sin dot, qamats qatan
   NIKKUD = r'[ְ-ׇֽֿׁׂׅׄ]'

   # Both, in one pass — note the gaps that preserve Hebrew punctuation
   BOTH   = r'[֑-ׇֽֿׁׂׅׄ]'

   def strip(text, mode='both'):
       pattern = {'nikkud': NIKKUD, 'teamim': TEAMIM, 'both': BOTH}[mode]
       return re.sub(pattern, '', text)
   ```

   Verified: applied to `בְּרֵאשִׁ֖ית בָּרָ֣א אֱלֹהִ֑ים אֵ֥ת הַשָּׁמַ֖יִם וְאֵ֥ת הָאָֽרֶץ׃` (Genesis 1:1), `BOTH` mode yields `בראשית ברא אלהים את השמים ואת הארץ׃` — sof pasuk preserved.

3. Normalise whitespace afterwards: `re.sub(r' {2,}', ' ', text).strip()`.

4. **Don't** use NFD-decompose-and-drop-combining-marks. It over-strips and breaks mixed-script text. Always use the explicit ranges above.

## Path B — Dicta API

Public, no auth, no key. The frontend at `removenikud.dicta.org.il` calls this directly:

```bash
curl -sS -X POST https://remove-nikud-2-0.loadbalancer2.dicta.org.il/api \
  -H 'Content-Type: application/json' \
  -d '{"task":"remove_nikud","data":"<HEBREW TEXT>"}'
# => {"results":"<stripped text>"}
```

Notes:

- Undocumented API. Treat it as best-effort — if the URL 404s, fall back to Path A.
- Be polite: one request per user action, no tight batch loops.
- Strips both nikkud and te'amim — no mode toggle exposed.

## Edge cases

- **Mixed-script input** (Hebrew + English + transliteration) — the Path A regex only matches Hebrew code points, so Latin/Arabic/etc. characters pass through untouched. Safe to apply globally.
- **Final letters** (ך ם ן ף ץ) are separate code points from their non-final forms and are not affected — don't try to "fix" them.
- **Yiddish text** uses some of the same code points (e.g. U+05B7 patah, U+05BC dagesh) as semantic letters, not vowels. If the source is Yiddish, ask before stripping — the user may want to keep them, or prefer Path B which is trained on Hebrew specifically.
- **Te'amim-only mode** (Tanakh with vowels but no trope) is Path A only; Dicta's endpoint doesn't expose it.

## Output

Return the cleaned text in a fenced code block so RTL rendering doesn't reflow the user's terminal. Cite which path was used.
