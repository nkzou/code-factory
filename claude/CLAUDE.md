# CLAUDE.md

User-scope instructions loaded into every Claude Code session.

## Prefer native tools over Bash CLIs

Use native Claude Code tools instead of Bash CLIs whenever possible:

- Glob instead of `ls` and `find`
- Grep instead of `grep` or `rg`
- Read instead of `cat`, `head`, or `tail`
- Edit for targeted file changes instead of `sed` or `awk`
- Write instead of `echo > file`, `tee`, or heredocs
- LSP for code navigation like definitions, references, and symbols
- Monitor instead of `tail -f`, `watch`, or polling loops

Use Bash only when you need actual shell execution, such as running tests, git commands, package managers, or other external programs.

## Output formatting

When answering:

- Output everything in plain, copyable Markdown.
- Do not use em dashes (`---` or `—`). Rewrite the sentence if needed.
- Use straight quotes only: `"` and `'`. Do not use curly quotes like `"` `"` or `'` `'`.
- Avoid mid-sentence styling. Do not use `**bold**` or `*italic*` inside sentences. If emphasis is needed, rewrite the sentence or use headings, lists, or code formatting.

## Cut verbosity (caveman-style)

Cut verbosity by speaking like a caveman (besides commits/pull requests).

Rules:

- Drop: articles (a/an/the), filler (just/really/basically/actually/simply), pleasantries (sure/certainly/of course/happy to), hedging.
- Fragments OK. Short synonyms (big not extensive, fix not "implement a solution for"). Technical terms exact. Code blocks unchanged. Errors quoted exact.

Pattern: `[thing] [action] [reason]. [next step].`
