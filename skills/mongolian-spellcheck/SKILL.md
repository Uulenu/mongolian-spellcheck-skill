---
name: mongolian-spell-check
description: Use when checking, correcting, or proofreading Mongolian (Cyrillic) text — fixing typos and misspellings, or validating that Mongolian writing is spelled correctly.
---

# Mongolian Spell Check

## Overview

Spell-checks and auto-corrects Mongolian (Cyrillic) text using the `spellcheck.mn`
service. A bundled script finds misspelled words and fetches candidate
corrections; **you** then pick the right correction using sentence context and
produce the final corrected text.

The service requires a per-request `key` (a hash of the content) or it rejects
the call with `"Invalid hash."`. The bundled script computes this key for you —
you never construct API requests by hand.

## When to Use

- The user asks to spell-check, proofread, or fix typos in Mongolian text.
- You are writing/editing Mongolian Cyrillic content and want to validate spelling.
- Not for: transliteration, translation, or grammar rewriting beyond fixing misspellings.

## Workflow

1. **Run the helper** on the text (prefer Python; fall back to Node):
   ```bash
   python3 scripts/spellcheck.py "<the Mongolian text>"
   # or:  node scripts/spellcheck.js "<the Mongolian text>"
   # long text: pipe via stdin →  echo "<text>" | python3 scripts/spellcheck.py
   ```
   It prints JSON: `{"text", "misspelled": [...], "suggestions": {word: [...]}}`.

2. **If `misspelled` is empty**, report that no spelling errors were found. Done.

3. **For each misspelled word, choose the best candidate from its suggestion
   list using context** — the surrounding sentence, grammatical case/agreement,
   and meaning. The list is ranked by surface similarity, **not** by correctness,
   so do **not** blindly take the first item.

4. **If no candidate fits** the intended meaning, rewrite that word/phrase
   minimally yourself rather than forcing a wrong suggestion.

5. **Output two things:**
   - the full **corrected text**, and
   - a **change list** of `original → fixed` for every word you changed.

## Quick Reference

| Action | Command |
|--------|---------|
| Check + suggest (default) | `python3 scripts/spellcheck.py "<text>"` |
| Check only (no suggestions) | `python3 scripts/spellcheck.py --check-only "<text>"` |
| One word's suggestions | `python3 scripts/spellcheck.py --suggest "<word>"` |
| Node equivalents | `node scripts/spellcheck.js ...` (Node 18+) |

Underlying endpoints and the key algorithm: see `references/api.md`.

## Common Mistakes

- **Taking `suggestions[word][0]` automatically.** Suggestions are ordered by
  spelling similarity, not contextual fit. Read the sentence and pick deliberately.
- **Ignoring grammar.** Mongolian suffixes encode case/possession; the correct
  fix must agree with the sentence (e.g. `-даа/-дээ` vowel harmony, declension).
- **Forgetting the no-error case.** Empty `misspelled` means the text is clean —
  say so instead of inventing changes.
- **Hand-building API calls.** Always go through the script so the `key` is valid.
