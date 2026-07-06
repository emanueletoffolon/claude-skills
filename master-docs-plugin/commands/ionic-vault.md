---
description: Genera/aggiorna un vault Obsidian navigabile sulla documentazione già scritta a mano (.claude/features + .claude/analysis) — indice MOC, wikilink additivi e config .obsidian. Locale, gratis, non riscrive il contenuto delle schede.
argument-hint: [root-progetto] [--dry-run]
---

# Vault Obsidian della documentazione (app Ionic)

Rende la documentazione **già esistente e curata a mano** un **vault Obsidian** navigabile: una nota indice (MOC) che collega tutte le schede, wikilink `[[...]]` tra schede correlate e verso le analisi, e la config `.obsidian` minima. È l'analogo Ionic di `/master-docs:master-docs-sync`, ma con una differenza fondamentale:

> **NON genera né riscrive documentazione.** Nei Master i deep-dive di modulo sono auto-generati (LLM). Qui le schede in `.claude/features/` sono **scritte a mano** e sono la fonte di verità: questo comando è **puramente meccanico e locale** (nessun LLM, nessun costo) e si limita ad aggiungere navigabilità. Le modifiche al contenuto delle schede si fanno a mano.

Le uniche scritture nelle schede sono un blocco **"Collegamenti"** delimitato da marker (`<!-- vault:links -->`), rigenerato in modo idempotente ad ogni run: il resto del file non viene mai toccato.

## Scelte di scope (perché così)

- **Root del vault = `.claude/`.** È dove vive la conoscenza curata (`features/`, `analysis/`). Un vault Obsidian è una cartella singola: non può risalire sopra la propria root, quindi non si può radicare sull'intera repo (ci sarebbe `node_modules/` → Obsidian collasserebbe). Le cartelle di rumore dentro `.claude/` (`tmp/`, `logs/`, `commands/`, `analisigpx/`) vengono escluse via `userIgnoreFilters`.
- **`docs/` (root) resta fuori**: nell'app è tipicamente l'output TypeDoc auto-generato (migliaia di file API reference), non conoscenza narrativa. Se in un progetto `docs/` contiene invece doc curati, spostali/linkali sotto `.claude/` o dimmelo.

## Parametri

- `$1` = **root del progetto** — default `.` (la cwd).
- `--dry-run` → mostra cosa verrebbe creato/aggiornato (Home + elenco schede + footer) senza scrivere nulla.

## Procedura

### 0. Individua la documentazione

```bash
ROOT="."; for a in "$@"; do case "$a" in --*) ;; *) ROOT="$a"; break;; esac; done
cd "$ROOT" || { echo "root non valida"; exit 1; }
VAULT=".claude"; FEAT="$VAULT/features"; ANA="$VAULT/analysis"
[ -d "$FEAT" ] || [ -d "$ANA" ] || { echo "Nessuna doc in $FEAT o $ANA — niente da indicizzare. Fermati."; exit 1; }
echo "VAULT root: $(pwd)/$VAULT"
echo "  features: $(ls "$FEAT"/*.md 2>/dev/null | wc -l) schede"
echo "  analysis: $(ls "$ANA"/*.md 2>/dev/null | wc -l) analisi"
```

Se `--dry-run`, dopo aver mostrato l'elenco (e il preview della Home allo Step 2) **termina** senza scrivere.

### 1. Config `.obsidian` (idempotente)

Crea la config minima solo se assente (non sovrascrivere preferenze dell'utente). Usa **wikilink** (`useMarkdownLinks:false`) e nasconde le cartelle di rumore.

```bash
OBS="$VAULT/.obsidian"; mkdir -p "$OBS"
[ -f "$OBS/app.json" ] || cat > "$OBS/app.json" <<'JSON'
{
  "useMarkdownLinks": false,
  "newLinkFormat": "shortest",
  "alwaysUpdateLinks": true,
  "attachmentFolderPath": "assets",
  "userIgnoreFilters": ["tmp/", "logs/", "commands/", "analisigpx/", "dev_plans/", "reports/"]
}
JSON
# stato per-utente volatile fuori da git
cat > "$OBS/.gitignore" <<'GI'
workspace.json
workspace-mobile.json
workspace.json.bak
cache
GI
echo "OK: config Obsidian in $OBS"
```

### 2. Genera la nota indice `Home.md` (MOC)

Prova a riusare la **categorizzazione autorevole** delle tabelle feature del `CLAUDE.md` di root (colonna "Feature" → titolo, link `.claude/features/<slug>.md`, heading `###` sopra la tabella → categoria). Se non trova quelle tabelle, ripiega elencando tutte le schede con il loro titolo H1.

```bash
python3 - "$VAULT" <<'PY'
import os, re, sys, glob
vault = sys.argv[1]
feat_dir = os.path.join(vault, "features"); ana_dir = os.path.join(vault, "analysis")
home = os.path.join(vault, "Home.md")

def slug(p): return os.path.splitext(os.path.basename(p))[0]
def h1(p):
    try:
        for ln in open(p, encoding="utf-8"):
            ln = ln.strip()
            if ln.startswith("# "): return ln[2:].replace("Feature:", "").strip()
    except OSError: pass
    return slug(p)

feat_files = sorted(glob.glob(os.path.join(feat_dir, "*.md")))
ana_files  = sorted(glob.glob(os.path.join(ana_dir, "*.md")))
present = {slug(p) for p in feat_files}

# --- prova a leggere categorie + titoli dalle tabelle del CLAUDE.md di root ---
categories = []  # [(cat_title, [(slug, title), ...])]
seen = set()
if os.path.exists("CLAUDE.md"):
    cur = None; rows = []
    for ln in open("CLAUDE.md", encoding="utf-8"):
        m = re.match(r"\s*#{2,4}\s+(.*)", ln)
        if m:
            if cur and rows: categories.append((cur, rows))
            cur = m.group(1).strip(); rows = []; continue
        if "|" in ln and ".claude/features/" in ln:
            cells = [c.strip() for c in ln.strip().strip("|").split("|")]
            sm = re.search(r"features/([a-z0-9\-]+)\.md", ln)
            if sm and sm.group(1) in present and sm.group(1) not in seen:
                sg = sm.group(1); title = cells[0] if cells and cells[0] else h1(os.path.join(feat_dir, sg+".md"))
                rows.append((sg, title)); seen.add(sg)
    if cur and rows: categories.append((cur, rows))

# schede non ancora categorizzate (nuove, non ancora in CLAUDE.md)
extra = [(slug(p), h1(p)) for p in feat_files if slug(p) not in seen]

out = []
out.append("# 🏠 Home — Documentazione polini-e-bike\n")
out.append("Vault Obsidian delle schede feature e delle analisi. "
           "La categorizzazione autorevole è nel `CLAUDE.md` di root.\n")
out.append("> Nota generata da `/master-docs:ionic-vault` — l'elenco si aggiorna rilanciando il comando.\n")

if categories:
    out.append("\n## Feature\n")
    for cat, rows in categories:
        if not rows: continue
        out.append(f"\n### {cat}\n")
        for sg, title in rows: out.append(f"- [[{sg}|{title}]]")
        out.append("")
else:
    out.append("\n## Feature\n")
    for p in feat_files: out.append(f"- [[{slug(p)}|{h1(p)}]]")

if extra:
    out.append("\n### Altre schede\n")
    for sg, title in extra: out.append(f"- [[{sg}|{title}]]")

if ana_files:
    out.append("\n## Analisi\n")
    for p in ana_files: out.append(f"- [[{slug(p)}|{h1(p)}]]")

open(home, "w", encoding="utf-8").write("\n".join(out) + "\n")
print(f"OK: {home}  ({len(feat_files)} schede, {len(ana_files)} analisi, "
      f"{sum(len(r) for _,r in categories)} categorizzate, {len(extra)} extra)")
PY
```

### 3. Inietta il blocco "Collegamenti" in ogni scheda (additivo, idempotente)

Per ogni file in `features/` e `analysis/` aggiunge — o rigenera — **solo** il blocco delimitato dai marker `<!-- vault:links start/end -->` in fondo al file: backlink a `[[Home]]` e collegamento incrociato feature↔analisi quando i nomi corrispondono per stem (es. `gpx-recording` ↔ `gpx-recording-analysis`). Nessun'altra riga del file viene toccata.

```bash
python3 - "$VAULT" <<'PY'
import os, re, sys, glob
vault = sys.argv[1]
feat = sorted(glob.glob(os.path.join(vault, "features", "*.md")))
ana  = sorted(glob.glob(os.path.join(vault, "analysis", "*.md")))
def slug(p): return os.path.splitext(os.path.basename(p))[0]
feat_slugs = {slug(p) for p in feat}
ana_slugs  = {slug(p) for p in ana}
START, END = "<!-- vault:links start -->", "<!-- vault:links end -->"

def cross_for(sg, is_feature):
    links = ["[[Home]]"]
    if is_feature:
        for cand in (sg + "-analysis", sg + "-analisi"):
            if cand in ana_slugs: links.append(f"[[{cand}|Analisi di dettaglio]]")
    else:  # è un'analisi → collega alla feature corrispondente
        base = re.sub(r"-(analysis|analisi)$", "", sg)
        if base in feat_slugs: links.append(f"[[{base}|Scheda feature]]")
    return links

def apply(path, is_feature):
    txt = open(path, encoding="utf-8").read()
    block = START + "\n## Collegamenti\n" + " · ".join(cross_for(slug(path), is_feature)) + "\n" + END
    if START in txt and END in txt:
        new = re.sub(re.escape(START) + r".*?" + re.escape(END), block, txt, flags=re.S)
    else:
        new = txt.rstrip() + "\n\n" + block + "\n"
    if new != txt:
        open(path, "w", encoding="utf-8").write(new); return True
    return False

n = sum(apply(p, True)  for p in feat) + sum(apply(p, False) for p in ana)
print(f"OK: blocco Collegamenti aggiornato in {n} file (su {len(feat)+len(ana)})")
PY
```

### 4. `.gitignore` dello stato Obsidian volatile

Assicura che lo stato per-utente di Obsidian non finisca nei commit (già coperto da `.claude/.obsidian/.gitignore`, ma verifica che nessuna regola più in alto forzi l'add).

```bash
git -C "$(pwd)" check-ignore -q "$VAULT/.obsidian/workspace.json" 2>/dev/null \
  && echo "OK: workspace Obsidian ignorato" \
  || echo "NOTA: verifica che $VAULT/.obsidian/workspace.json resti fuori dai commit"
```

## Report finale all'utente

Riepiloga: root del vault (`.claude/`), n. schede + analisi indicizzate, quante categorie riprese dal `CLAUDE.md`, quante schede hanno ricevuto/aggiornato il blocco "Collegamenti", e come aprirlo:

- **Aprire il vault**: in Obsidian → *Open folder as vault* → seleziona `.../<progetto>/.claude` (dal Mac se la repo è sincronizzata lì). La nota `Home.md` è la porta d'ingresso; il *Graph view* mostra i collegamenti tra schede.
- **Aggiornare** dopo aver aggiunto/rinominato schede: rilancia `/master-docs:ionic-vault` (rigenera Home e i blocchi Collegamenti; idempotente).
- Il comando è **locale e gratis**: non usa LLM e non riscrive il contenuto delle schede (solo il blocco marcato).
