---
name: armserver-latex
description: "Compile LaTeX documents (.tex to PDF) using the full TeX Live 2025 install on armserver (100.71.152.44) via SSH. Use whenever a task asks for a PDF report, a LaTeX document, a rendered document, or mentions .tex files, lualatex, xelatex, tlmgr, or 'kompilier das PDF' / 'baue das Dokument'. This is the only place with a complete TeX Live distribution (lualatex, xelatex, pdflatex, latexmk, tlmgr) - do not attempt to install or fake a LaTeX toolchain elsewhere, always route the actual compile step through armserver."
license: MIT
metadata:
  version: 1.0.0
  author: Alex Schneider
---

## Why this skill exists

armserver (100.71.152.44, reachable via SSH as `armserver` or the Tailscale IP) has a full TeX Live 2025 (Debian) installation: `lualatex`, `xelatex`, `pdflatex`, `latexmk`, `tlmgr`. No other host in this environment has a complete LaTeX toolchain. Any task that ends in "produce a PDF from LaTeX" needs to run its actual compile step here, even if the `.tex` source was written or edited somewhere else.

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
