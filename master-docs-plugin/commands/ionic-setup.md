---
description: Setup one-shot di un'app Ionic + Capacitor per il flusso graphify+docs — costruisce il grafo (scope src/), installa l'autosync post-commit e (opzionale) genera il vault Obsidian della documentazione. Idempotente.
argument-hint: [root-progetto] [--no-vault] [--model sonnet|haiku|opus]
---

# Setup completo dell'app Ionic (grafo + autosync + vault Obsidian)

Prepara un'app Ionic/Angular/Capacitor in **un colpo solo** per il nostro flusso: knowledge graph `graphify` in root, refresh automatico del grafo a ogni commit, e vault Obsidian navigabile della documentazione già presente. È il comando da lanciare **una volta** su un progetto nuovo (o per riallineare uno esistente). È **idempotente**: rilanciarlo non duplica nulla, aggiorna.

> Gemello Ionic di `/master-docs:master-setup`. La differenza sostanziale è il passo "docs": nei Master si generano deep-dive per-modulo (LLM, costosi); qui la documentazione è **scritta a mano** (le schede in `.claude/features/`), quindi non la rigeneriamo — ci limitiamo a renderla un **vault Obsidian** navigabile (operazione locale, gratis).

## Cosa fa, in ordine
1. **graphify** installato (altrimenti lo installa).
2. **Grafo**: build completo se assente, altrimenti refresh incrementale (`graphify update .`), con scope `src/`.
3. **Autosync**: installa il git hook `post-commit` che aggiorna il grafo in background a ogni commit (gratis, solo AST).
4. **Vault Obsidian**: se esiste documentazione in `.claude/features/` (e/o `.claude/analysis/`), genera/aggiorna il vault Obsidian via `/master-docs:ionic-vault` (indice MOC + wikilink + config `.obsidian`). Non tocca il contenuto delle schede.

## Parametri

- `$1` = **root del progetto** — default `.` (la cwd).
- `--no-vault` → salta del tutto il passo 4 (solo grafo + hook).
- `--model <m>` → modello per il naming delle community (default `sonnet`).

## Procedura

### 1. Grafo
Determina la root (primo argomento che non è un flag; default `.`) ed entra nella cartella.

```bash
export PATH="$HOME/.local/bin:$PATH"
ROOT="."; for a in "$@"; do case "$a" in --*) ;; *) ROOT="$a"; break;; esac; done
cd "$ROOT" || { echo "root non valida"; exit 1; }
if [ -f graphify-out/graph.json ]; then echo "GRAFO: presente → refresh"; else echo "GRAFO: assente → build completo"; fi
```

- Se il grafo **NON** esiste → esegui l'**intera procedura del comando `/master-docs:ionic-graph`** (Step 0→5 di quel file: verifica/installa graphify, rileva `APP=src`, genera `.graphifyignore`, build, gate anti-leak, naming community, verifica finale). Passa `--model` se indicato.
- Se il grafo **esiste già** → basta il refresh incrementale: `graphify update .` (AST-only, gratis).

### 2. Autosync post-commit
Installa il git hook (stesso blocco di `/master-docs:ionic-graph --install-hook`). Idempotente e non distruttivo su hook esistenti.

```bash
export PATH="$HOME/.local/bin:$PATH"
git rev-parse --is-inside-work-tree >/dev/null 2>&1 || { echo "Non è un repo git: niente hook."; SKIP_HOOK=1; }
if [ -z "$SKIP_HOOK" ]; then
  HOOKS_DIR="$(git config core.hooksPath 2>/dev/null)"; [ -z "$HOOKS_DIR" ] && HOOKS_DIR="$(git rev-parse --git-path hooks)"
  mkdir -p "$HOOKS_DIR"; HOOK="$HOOKS_DIR/post-commit"
  S="# >>> master-docs graphify autosync >>>"; E="# <<< master-docs graphify autosync <<<"
  if [ -f "$HOOK" ] && grep -qF "$S" "$HOOK"; then
    awk -v s="$S" -v e="$E" '$0==s{skip=1} skip!=1{print} $0==e{skip=0}' "$HOOK" > "$HOOK.tmp" && mv "$HOOK.tmp" "$HOOK"
  fi
  [ -s "$HOOK" ] || printf '#!/bin/sh\n' > "$HOOK"
  cat >> "$HOOK" <<EOF
$S
# Aggiorna il knowledge graph graphify dopo ogni commit (AST-only, gratis, in background).
if command -v graphify >/dev/null 2>&1 && [ -f graphify-out/graph.json ]; then
  ( export PATH="\$HOME/.local/bin:\$PATH"; graphify update . >/dev/null 2>&1 & )
fi
$E
EOF
  chmod +x "$HOOK"; echo "OK: autosync post-commit installato in $HOOK"
fi
```

> **Nota permessi:** in modalità auto, l'installazione di un git hook può essere bloccata dal classifier come "persistence". È corretto: se accade, riporta all'utente il comando `/master-docs:ionic-graph --install-hook` da lanciare/approvare a mano. Il grafo funziona anche senza hook (basta `--update` manuale).

### 3. Vault Obsidian (salta se è stato passato `--no-vault`)

```bash
DOCS_FEAT=".claude/features"; DOCS_ANA=".claude/analysis"
if [ -d "$DOCS_FEAT" ] || [ -d "$DOCS_ANA" ]; then
  echo "VAULT: documentazione trovata → genero/aggiorno il vault Obsidian"
else
  echo "VAULT: nessuna doc in .claude/features o .claude/analysis → salta (niente da indicizzare)"
fi
```

- Se c'è documentazione → esegui la procedura di **`/master-docs:ionic-vault`** (config `.obsidian`, nota indice `Home.md`, wikilink additivi tra schede correlate e verso le analisi). Operazione locale, nessun LLM, non modifica il contenuto delle schede.
- **`--no-vault`** → salta.

## Report finale all'utente

Riepiloga in modo conciso:
- **Grafo**: nodi/edge/community nominate (se costruito) oppure "aggiornato" (se refresh).
- **Autosync**: hook installato / saltato (non-git o bloccato dal classifier — in tal caso indica il comando manuale).
- **Vault**: generato/aggiornato (N schede indicizzate) / saltato.
- Comandi d'uso quotidiano: `graphify query|explain|path`, `/master-docs:ionic-graph --update`, `/master-docs:ionic-vault`.

Ricorda che gli hook `MANDATORY: graphify` scattano per le sessioni Claude lanciate dalla root che contiene `graphify-out/`.
