# PR Draft: fix(run.lua): M.insert doubles backslashes in R code due to JSON transport mismatch

## Bug

`\o` (`RInsertLineOutput`) returns `character(0)` for pipe chains containing regex patterns with backslashes, e.g.:

```r
texte |>
  str_extract_all("\\w+") |>
  unlist()
# character(0)   ← inserted by \o, should be the extracted words
```

The same expression works correctly when run manually.

## Root cause

`M.insert` replaces `"` with byte `\x13` (to protect against premature JSON string termination in `cut_json_str`), then sends the code via `send_msg`, which serializes the Lua table to JSON. JSON encoding doubles every backslash (`\` → `\\`). However, `cut_json_str` on the C side finds the string boundary by scanning for a raw `"` byte — it does **no JSON unescaping** — so the doubled backslashes arrive at `nvimcom_eval_expr` as-is. `unscape_str` only handles `\u0012`/`\u0013`; it passes `\\` through unchanged. R's parser therefore sees `"\\\\w+"` (4 backslashes) instead of `"\\w+"` (2), producing the string `\\w+` — a regex matching a literal backslash, not word characters.

## Fix (applied here)

Halve backslashes in `M.insert` before JSON encoding to compensate:

```lua
cmd = cmd:gsub("\\\\", "\\")
```

This restores the correct count after the full Lua → JSON → `cut_json_str` → `unscape_str` → `R_ParseVector` round-trip.

The deeper fix would be to make `cut_json_str` properly unescape `\\` → `\`, but that requires a C change and would need care to not affect other callers. Happy to submit whichever approach you prefer.
