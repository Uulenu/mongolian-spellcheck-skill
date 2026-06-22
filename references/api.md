# spellcheck.mn API reference

Reverse-engineered contract for the two endpoints the skill uses, plus the
request-key algorithm. The bundled scripts (`scripts/spellcheck.py`,
`scripts/spellcheck.js`) implement everything below — this file documents it so
the behavior is auditable and portable.

## The request `key`

Every request body carries a `key`. If it does not match the content, the server
responds with the plain string `"Invalid hash."` (HTTP 200). The key is the
site's client-side `$encrypt(content)` plugin (extracted from `/_nuxt/9d5e31a.js`,
a Paul-Johnston SHA-256 implementation):

```
n   = ( Σ over each Unicode code point c of the content of (c + 1) ) mod 10000000008
key = sha256_hex( decimal_string_of(n) )
```

Notes:
- Iterate by **Unicode code point** (not UTF-16 unit). For Mongolian Cyrillic all
  chars are in the BMP, so it rarely matters, but the scripts handle it correctly.
- `10000000008` = `1e10 + 8`.
- Hash the **decimal string** of `n` (e.g. `"123456"`), not the raw bytes of the text.

### Worked examples (used as regression assertions)

| content | key |
|---------|-----|
| `уулзалтандаа` | `93af61ba9f335c505b55ca64a7f917f24f8f45a3a12480a5836237e11026e0d2` |
| `Сайн уу, юу байна даа? Маргаашийн уулзалтандаа бэлдээд ирээрэй хүү.` | `9b1bb74fc9ad291c23996a4ad455080ab91034bda14730d3e899e956e9c1973e` |

## Endpoints

Base: `https://spellcheck.mn/cms-client/modules/spellchecker`

Required headers on both:
```
content-type: application/json
origin:  https://spellcheck.mn
referer: https://spellcheck.mn/
```

### POST /check

Request:
```json
{"text": "<text to check>", "key": "<encrypt(text)>"}
```
Response: JSON array of misspelled words, e.g. `["уулзалтандаа"]`. Empty array
(or empty string) means no errors. The official client strips `-` characters from
each returned word — the scripts do the same.

### POST /suggest

Request (one word at a time):
```json
{"word": "<misspelled word>", "key": "<encrypt(word)>"}
```
Response: JSON array of up to ~10 candidate corrections, **ordered by spelling
similarity, not contextual correctness**:
```json
["уулзалтайндаа", "уулзалтдаа", "уулзалтындаа", "..."]
```

## Failure modes

| Symptom | Cause |
|---------|-------|
| Response body `"Invalid hash."` | `key` doesn't match the content — regenerate it from the exact string sent. |
| Empty response on `/check` | No misspellings detected. |
| Texts shorter than 2 trimmed chars | Site skips checking; the scripts return an empty result. |
