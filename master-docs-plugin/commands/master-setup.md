---
description: Setup one-shot di un progetto Master Laravel Enesi per il flusso graphify+docs — costruisce il grafo, installa l'autosync post-commit e (se i deep-dive mancano) bootstrappa docs/moduli/. Idempotente.
argument-hint: [root-progetto] [--no-docs] [--model sonnet|haiku|opus]
---

# Setup completo del progetto (grafo + autosync + docs)

Prepara un progetto Master in **un colpo solo** per il nostro flusso di lavoro: knowledge graph `graphify` in root, refresh automatico del grafo a ogni commit, e documentazione deep-dive dei moduli. È il comando da lanciare **una volta** su un progetto nuovo (o per riallineare uno esistente). È **idempotente**: rilanciarlo non duplica nulla, aggiorna.

## Cosa fa, in ordine
1. **graphify** installato (altrimenti lo installa).
2. **Grafo**: build completo se assente, altrimenti refresh incrementale (`graphify update .`).
3. **Autosync**: installa il git hook `post-commit` che aggiorna il grafo in background a ogni commit (gratis, solo AST — **mai** i docs).
4. **Docs**: se `docs/moduli/` è **assente** → bootstrap completo via skill `documenta-moduli`; se **presente** → rileva gli stale e rimanda a `/master-docs:master-docs-sync` (NON rigenera qui: è un'operazione LLM costosa e on-demand).

## Parametri

- `$1` = **root del progetto** — default `.` (la cwd).
- `--no-docs` → salta del tutto il passo 4 (solo grafo + hook).
- `--model <m>` → modello per il naming delle community e per l'eventuale bootstrap docs (default `sonnet`).

## Procedura

### 1. Grafo
Determina la root (primo argomento che non è un flag; default `.`) ed entra nella cartella.

```bash
export PATH="$HOME/.local/bin:$PATH"
ROOT="."; for a in "$@"; do case "$a" in --*) ;; *) ROOT="$a"; break;; esac; done
cd "$ROOT" || { echo "root non valida"; exit 1; }
if [ -f graphify-out/graph.json ]; then echo "GRAFO: presente → refresh"; else echo "GRAFO: assente → build completo"; fi
```

- Se il grafo **NON** esiste → esegui l'**intera procedura del comando `/master-docs:master-graph`** (Step 0→5 di quel file: verifica/installa graphify, genera `.graphifyignore`, build, gate anti-leak, naming community, verifica finale). Passa `--model` se indicato.
- Se il grafo **esiste già** → basta il refresh incrementale: `graphify update .` (AST-only, gratis).

### 2. Autosync post-commit
Installa il git hook (stesso blocco di `/master-docs:master-graph --install-hook`). Idempotente e non distruttivo su hook esistenti.

```bash
export PATH="$HOME/.local/bin:$PATH"
git rev-parse --is-inside-work-tree >/dev/null 2>&1 || { echo "Non è un repo git: niente hook."; SKIP_HOOK=1; }
if [ -z "$SKIP_HOOK" ]; then
  HOOKS_DIR="$(git config core.hooksPath 2>/dev/null)"; [ -z "$HOOKS_DIR" ] && HOOKS_DIR="$(git rev-parse --git-path hooks)"
  mkdir -p "$HOOKS_DIR"; HOOK="$HOOKS_DIR/post-commit"
  S="# >>> master-docs master-graph autosync >>>"; E="# <<< master-docs master-graph autosync <<<"
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

### 3. Docs (salta se è stato passato `--no-docs`)

```bash
MODROOT="private/master/Modules"; DOCS="docs/moduli"
if [ ! -d "$MODROOT" ]; then echo "DOCS: nessun $MODROOT (non è il Master o app in root) → salta"; 
elif [ ! -d "$DOCS" ]; then echo "DOCS: $DOCS assente → BOOTSTRAP completo necessario";
else
  # sync: rileva gli stale (git commit ts: modulo vs suo doc)
  n=0; for d in "$MODROOT"/*/; do m="$(basename "$d")"; doc="$DOCS/$m.md"; \
    cc="$(git log -1 --format=%ct -- "$d" 2>/dev/null)"; \
    [ -f "$doc" ] || { echo "  mancante: $m"; n=$((n+1)); continue; }; \
    dc="$(git log -1 --format=%ct -- "$doc" 2>/dev/null)"; \
    [ -n "$cc" ] && { [ -z "$dc" ] || [ "$cc" -gt "$dc" ]; } && { echo "  stale: $m"; n=$((n+1)); }; \
  done; echo "DOCS: $n moduli da riallineare"
fi
```

- **`docs/moduli/` assente** (prima installazione) → invoca la skill **`documenta-moduli`** per generare l'intero set di deep-dive (+ `README.md` indice + `DUBBI.md`). È un'operazione LLM: conferma col utente prima, o rispetta `--model`.
- **`docs/moduli/` presente** con moduli stale/mancanti → **non** rigenerare qui. Segnala il numero e rimanda a **`/master-docs:master-docs-sync`** (rigenera solo gli stale, on-demand).
- **`--no-docs`** → salta.

## Report finale all'utente

Riepiloga in modo conciso:
- **Grafo**: nodi/edge/community nominate (se costruito) oppure "aggiornato" (se refresh).
- **Autosync**: hook installato / saltato (non-git).
- **Docs**: bootstrappati (N moduli) / "N stale — usa `/master-docs:master-docs-sync`" / saltati.
- Comandi d'uso quotidiano: `graphify query|explain|path`, `/master-docs:master-graph --update`, `/master-docs:master-docs-sync`.

Ricorda che gli hook `MANDATORY: graphify` scattano per le sessioni Claude lanciate dalla root che contiene `graphify-out/`.
