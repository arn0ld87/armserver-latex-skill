---
name: armserver-latex
description: "Compile LaTeX documents (.tex to PDF) using the full TeX Live 2025 install on armserver (100.71.152.44) via SSH, using the alexle135 professional design template, and email the result to Alex. Use whenever you (any agent, any role) finish a work product worth handing to Alex as a document - exam prep material, flashcards/Karteikarten, research writeups, reports, summaries - or whenever a task mentions .tex files, PDF output, lualatex, xelatex, tlmgr, 'kompilier das PDF', 'baue das Dokument', or 'schick mir das per Mail'. This is the only place with a complete TeX Live distribution - do not attempt to install or fake a LaTeX toolchain elsewhere, always route the actual compile step through armserver. Default to using this skill's template and email step for any substantial deliverable rather than leaving it as a bare markdown comment."
license: MIT
metadata:
  version: 2.0.0
  author: Alex Schneider
---

## Why this skill exists

armserver (100.71.152.44, reachable via SSH as `armserver` or the Tailscale IP) has a full TeX Live 2025 (Debian) installation: `lualatex`, `xelatex`, `pdflatex`, `latexmk`, `tlmgr`. No other host in this environment has a complete LaTeX toolchain. Any task that ends in "produce a PDF from LaTeX" needs to run its actual compile step here, even if the `.tex` source was written or edited somewhere else.

## Use the professional design template, don't hand Alex bare LaTeX

Alex wants finished work products (exam prep, flashcards, research writeups, reports) to look like a real document, not a default `article` class dump. This skill bundles `assets/alexle135-preamble.tex` — a ready-made preamble matching the alexle135.de brand (IBM Plex Serif for headings, IBM Plex Sans for body, IBM Plex Mono for code, a `#FF6A00` orange accent rule under section headings, a branded footer). Both fonts are installed on armserver already (`fonts-ibm-plex` package plus TeX Live's own bundled copies) — you don't need to install anything.

Use it like this:

```latex
\documentclass[11pt]{article}
\input{alexle135-preamble.tex}

\begin{document}
\docTitle{Titel des Dokuments}{Optionaler Untertitel}{Dein Agent-Name}{\today}

\section{Erster Abschnitt}
...
\end{document}
```

Copy `alexle135-preamble.tex` into the same directory as your `.tex` source before compiling (it's `\input`, not a system-wide package). Requires `lualatex` — this preamble uses `fontspec`, it will not work with plain `pdflatex`.

If a document needs a structure the preamble doesn't cover well (e.g. a title page, an appendix, a table of contents for something long), extend it in your own document rather than fighting the template — the preamble is a starting point, not a rigid cage. Keep the color/font choices though; that's the part that makes it recognizably "an alexle135 document."

## Compiling a document

Default to `latexmk` with the LuaLaTeX engine — that's the established default for documents in this environment and handles multi-pass compilation (references, TOC, bibliography) automatically:

```bash
ssh armserver 'cd <remote-project-dir> && latexmk -lualatex -interaction=nonstopmode -halt-on-error <file>.tex'
```

If the source only exists locally, copy it over first (`scp` or `rsync -a`), compile remotely, then copy the resulting PDF back:

```bash
scp -r ./mydoc armserver:/tmp/mydoc-build
ssh armserver 'cd /tmp/mydoc-build && latexmk -lualatex -interaction=nonstopmode -halt-on-error main.tex'
scp armserver:/tmp/mydoc-build/main.pdf ./
```

Use `xelatex` instead of `lualatex` only if the document specifically needs a font or package that requires it (rare — check the `.tex` preamble for `\usepackage{fontspec}` plus a note about XeLaTeX, or an explicit XeLaTeX requirement from whoever wrote the source). Plain `pdflatex` is available too, but prefer `latexmk -lualatex` unless you have a concrete reason not to.

Always pass `-interaction=nonstopmode -halt-on-error` (or check the log instead of the raw stdout) so a missing package or a typo doesn't leave you staring at an interactive TeX prompt over SSH — it will just hang otherwise.

## Missing packages

If the log complains about a missing `.sty` or package, install it with `tlmgr` on armserver (this is a shared system install, so do it once, not per-run):

```bash
ssh armserver 'sudo -n tlmgr install <package-name>'
```

If `sudo -n` fails (no passwordless sudo for tlmgr in that context), ask before falling back to an interactive sudo — don't silently skip the package and paper over the error.

## Known quirk: locale warning

`tlmgr` (and occasionally other TeX tools) prints:

```
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings...
```

This is cosmetic — `de_DE.UTF-8` is set as `$LANG` but not actually generated on the system. It does not affect compilation. To silence it in scripts/logs, prefix the command with `LC_ALL=C.UTF-8`:

```bash
ssh armserver 'LC_ALL=C.UTF-8 tlmgr install <package-name>'
```

Don't try to "fix" this by running `locale-gen` unless explicitly asked — it's a system-level change outside the scope of a single document build.

## Verifying the result

After compiling, always check that a PDF was actually produced and that the log has no unresolved errors before treating the task as done — `latexmk` exits non-zero on failure, but always glance at the `.log` tail for warnings about missing references or overfull boxes that might matter for the specific document:

```bash
ssh armserver 'cd <remote-project-dir> && test -f <file>.pdf && echo OK || (tail -40 <file>.log)'
```

## Emailing the result to Alex

Once you have a finished PDF worth showing Alex, send it — don't just leave it sitting in a comment as an attachment link. There is already a working mail pipeline on armserver, proven in ALE-56: SMTP via Fastmail (`smtp.fastmail.com:465`, tunneled through the alexle135de host), with credentials (`EMAIL_ADDRESS` / `EMAIL_SMTP_HOST` / `EMAIL_PASSWORD`) pulled from Paperclip company secrets at send time — not hardcoded anywhere.

A working helper, `send_mail.sh`, already exists in Hardware Harald's workspace at `/home/alex/.paperclip/instances/default/workspaces/29862bfd-9c74-4303-aeeb-09ea107853da/send_mail.sh`. Read it yourself (you have file access in your own execution context) to see its exact argument order and behavior, then either call it directly or copy it into your own workspace — don't reinvent SMTP sending from scratch, and don't guess at credential names; the script already knows how to fetch them from company secrets correctly.

Recipient is Alex's own address (`schneider@alexle135.de`); sender should identify you as the agent that produced the work (e.g. `<your-agent-name>@alexle135.de` if that alias pattern is available, otherwise fall back to whatever alias the existing script defaults to). Subject line should say what the document is, not just "PDF attached" — Alex triages his inbox by subject.

If the script or credentials are missing or broken when you try this, say so in an issue comment rather than silently skipping the email — this has happened before (a different, unrelated `fastmail` skill attempt failed for missing `FASTMAIL_API_TOKEN`; that's a dead end, use `send_mail.sh` instead).
