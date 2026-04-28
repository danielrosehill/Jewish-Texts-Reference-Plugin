---
name: add-nikkud
description: Add Hebrew vowel points (nikkud) to unvocalized Hebrew text — vocalisation / niqqud restoration. Use when the user pastes consonantal Hebrew and asks to "add nikud", "vocalise", "add vowels", "nakdan", or wants to turn ktiv male / modern Hebrew into pointed text. Two backends — Dicta's nakdan (default, online, high quality) and the local `unikud` Python package (offline fallback).
---

# Add Nikkud

The reverse of `strip-nikkud`. Vocalisation requires a real model — a regex can't do this — so we call out to one of two trained models.

## When to invoke

- "Add nikud to this"
- "Vocalise this Hebrew sentence"
- "Run nakdan on…"
- "Put vowels on this paragraph"
- Preparing unvocalized text for reading aloud, study, or display

## Choosing a backend

| Backend | When | Tradeoffs |
|---|---|---|
| **Dicta nakdan** (default) | Always available; user has internet; user wants best quality | Online; depends on Dicta's free public API; supports modern + rabbinic + poetry genres |
| **`unikud`** (Python, [morrisalp/unikud](https://github.com/morrisalp/unikud)) | User explicitly wants offline, on-device, or air-gapped | First run downloads a CANINE model (~500 MB); modern Hebrew only; lower accuracy than Dicta |

Default to **Dicta** unless the user says "offline", "local", "no network", or has clearly asked for `unikud`.

## Path A — Dicta nakdan (default)

Public JSON API behind `nakdanpro.dicta.org.il`. No key.

```bash
curl -sS -X POST https://nakdan-2-0.loadbalancer.dicta.org.il/api \
  -H 'Content-Type: application/json' \
  -d '{
    "task": "nakdan",
    "data": "<UNVOCALIZED HEBREW>",
    "genre": "modern",
    "addmorph": false,
    "keepqq": false,
    "nodageshdefmem": false,
    "patachma": false,
    "keepmetagim": true
  }'
```

### Response shape

The response is a JSON array, one element per token (word or separator):

```json
[
  {"word": "שלום", "sep": false, "options": ["שָׁלוֹם", "שְׁלוֹם", "שִׁלּוּם", ...], "fconfident": true},
  {"word": " ",    "sep": true,  "options": []},
  ...
]
```

**Reconstruction rule:** for each element,
- if `sep == true`, append `word` verbatim
- if `sep == false`, append `options[0]` (the highest-ranked vocalisation)

Concatenate to get the vocalised string.

### Genre parameter

- `modern` — modern Hebrew (default)
- `rabbinic` — Talmudic / halakhic / responsa text
- `poetry` — piyut, biblical poetry style

If the user names a genre, use it. If the source is recognisably Talmudic or halakhic (Aramaic words, sugya structure), prefer `rabbinic`. For Tanakh, `poetry` often outperforms `modern` for prophetic books.

### Other flags

- `keepmetagim: true` — preserve meteg / ga'ya marks already present
- `keepqq: true` — keep existing qamats qatan distinctions
- `addmorph: true` — also return per-word morphological analysis (rarely needed; bigger response)

## Path B — `unikud` (offline)

```bash
pip install unikud
```

```python
from unikud.framework import Unikud
u = Unikud()  # downloads model on first call; cached locally
print(u.vocalize("שלום עולם"))
```

First run will pull the model from Hugging Face (~500 MB) and cache it under `~/.cache/huggingface/`. Subsequent runs are offline and fast.

If `pip install` fails or the model download fails, fall back to Path A and tell the user why.

## Procedure

1. **Detect input.** If the input already contains nikkud (any code point in `U+05B0–U+05C7` excluding punctuation), confirm with the user before re-vocalising — they may have meant `strip-nikkud` first, or they may want to refresh the existing pointing.

2. **Choose backend.** Default Path A. Switch to Path B only on explicit offline request.

3. **Pick genre** (Path A only) based on the text or the user's hint.

4. **Call the API / library.** For long text, chunk into paragraphs and concatenate — Dicta handles a few KB per request comfortably.

5. **Render.** Return the vocalised text in a fenced code block. If the user wants to see alternative vocalisations for ambiguous words (Path A only), expose `options[1:]` for the words where `fconfident` is false.

## Caveats

- Vocalisation is inherently ambiguous in Hebrew (homographs distinguished only by context). Even the best models will misvocalise occasionally — flag low-confidence tokens (`fconfident: false`) when the user is using the output for something high-stakes.
- The Dicta endpoint is undocumented and unversioned. Treat it as best-effort and fall back to `unikud` (or report failure) if it 404s or schema changes.
- Don't run `add-nikkud` on text fetched from Sefaria — Sefaria text is already vocalised by traditional sources (Masoretic for Tanakh, etc.) and re-vocalising would be both wasteful and likely worse than the source.
