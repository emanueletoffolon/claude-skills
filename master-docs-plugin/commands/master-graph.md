---
description: Crea/aggiorna il knowledge graph graphify per un progetto Master Laravel Enesi — radicato sulla root, scope automatico all'app in private/, community nominate gratis via claude-cli
argument-hint: [root-progetto] [--update|--relabel|--no-label] [--install-hook|--uninstall-hook] [--model sonnet|haiku|opus]
---

# Crea il grafo graphify per un Master Laravel

Costruisci (o aggiorna) il knowledge graph `graphify` di **questo progetto Master Laravel Enesi**, con la configurazione corretta per il nostro flusso: **Claude viene sempre lanciato dalla root del progetto**, quindi il grafo deve stare nella root (`./graphify-out/`) — così scattano gli hook `PreToolUse` e `graphify query` trova il grafo di default, e i path risultano relativi alla root (es. `private/master/...`).

## Parametri

- `$1` = **root del progetto** — default `.` (la cwd, cioè da dove è lanciato Claude).
- `--update` → **refresh incrementale** di un grafo esistente: ri-estrae solo i file cambiati (AST-only, no LLM, ~secondi). Non ri-clusterizza né rinomina le community. Se il grafo non esiste, ripiega su un build completo.
- `--relabel` → salta il build, ri-nomina solo le community di un grafo esistente.
- `--no-label` → build AST soltanto, niente arricchimento LLM (velocissimo, deterministico, community senza nome).
- `--model <m>` → modello per il backend `claude-cli` (default `sonnet`; `haiku` = più veloce/economico, `opus` = più accurato).
- `--install-hook` → installa un **git hook `post-commit`** nel progetto che lancia `graphify update .` (AST-only, gratis, in background) dopo ogni commit: il grafo non resta mai stale. Idempotente, si concatena a un `post-commit` esistente senza sovrascriverlo.
- `--uninstall-hook` → rimuove **solo** il blocco autosync dal `post-commit` (lascia intatto il resto dell'hook).

## Prerequisiti

- `graphify` nel PATH (`~/.local/bin/graphify`) — **se manca lo installa lo Step 0**, non serve fare nulla a mano.
- Il binario `claude` nel PATH (serve al backend gratuito `claude-cli` per nominare le community — usa l'abbonamento, nessuna API key). Non installabile in automatico: se manca, il grafo si costruisce comunque (AST) ma senza nomi community (usa `--no-label` o una API key).
- Esegui sempre con `export PATH="$HOME/.local/bin:$PATH"` in testa ai comandi bash.

## Gotcha da NON dimenticare (li abbiamo pagati sul campo)

1. **Backend LLM gratis**: usa `--backend claude-cli` (via CLI Claude Code, sul piano). NON `--backend claude` (quello pretende `ANTHROPIC_API_KEY`). Il modello si sovrascrive con l'env `GRAPHIFY_CLAUDE_CLI_MODEL`.
2. **Scansione dalla root ≠ ignore annidati**: quando scansioni dalla root, graphify **non** applica i `.gitignore`/`.graphifyignore` dentro `private/`. Tutte le esclusioni vanno nello **`.graphifyignore` di root**.
3. **Whitelist gitignore non funziona**: `/*` + `!/private/` produce **0 file**. Usa una **exclude-list** (escludi tutto ciò che NON è l'app), mai una whitelist.
4. **Auto-follow dei symlink**: se la root ha anche un solo symlink diretto, graphify **segue i symlink**. Sulle nostre webroot ci sono symlink verso **altri siti** (es. `master-dev`, `rest2`): vanno esclusi, altrimenti finiscono nel grafo. Il generatore qui sotto li esclude in automatico.
5. **File non-codice** (`.md`, immagini, pdf) fanno **abortire** il build AST-only chiedendo una API key: escludili sempre.

## Procedura

### 0. Verifica e (se manca) installa graphify

Esegui questo blocco per primo. Se `graphify` non è presente lo installa in automatico (via `uv`, con fallback a `pip` e all'installer standalone). Se al termine `graphify` non è disponibile, **fermati e segnala** all'utente il comando manuale.

```bash
export PATH="$HOME/.local/bin:$PATH"

if ! command -v graphify >/dev/null 2>&1; then
  echo "graphify non trovato — installazione in corso…"

  # 1) assicura uv (installer di graphify)
  if ! command -v uv >/dev/null 2>&1; then
    pip install --user -q uv 2>/dev/null \
      || python3 -m pip install --user -q uv 2>/dev/null \
      || pip install --user -q --break-system-packages uv 2>/dev/null \
      || { command -v curl >/dev/null 2>&1 && curl -LsSf https://astral.sh/uv/install.sh | sh; }
    export PATH="$HOME/.local/bin:$PATH"
  fi

  # 2) installa graphify (pkg PyPI = graphifyy, CLI = graphify)
  if command -v uv >/dev/null 2>&1; then
    uv tool install graphifyy
  else
    pip install --user graphifyy || python3 -m pip install --user graphifyy \
      || pip install --user --break-system-packages graphifyy
  fi
  export PATH="$HOME/.local/bin:$PATH"
fi

# 3) verifica finale — blocco se ancora assente
if command -v graphify >/dev/null 2>&1; then
  echo "graphify OK: $(graphify --version 2>/dev/null) ($(command -v graphify))"
else
  echo "ERRORE: impossibile installare graphify in automatico."
  echo "Installa a mano:  pip install --user uv && uv tool install graphifyy"
  exit 1
fi

# 4) backend per nominare le community (opzionale)
if ! command -v claude >/dev/null 2>&1; then
  echo "NOTA: binario 'claude' non nel PATH → niente naming community via claude-cli. Il grafo AST si costruisce comunque; usa --no-label o una API key."
fi
```

Se lo Step 0 fallisce l'installazione, non proseguire con il build.

### Modalità `--update` (refresh incrementale — scorciatoia)

Se è passato `--update`: dopo lo Step 0 esegui **solo** questo blocco e termina (NON rigenerare lo `.graphifyignore`, NON ricostruire da zero). Presuppone un build precedente (quindi `.graphifyignore` e grafo già presenti).

```bash
export PATH="$HOME/.local/bin:$PATH"
ROOT="${1:-.}"; cd "$ROOT" || exit 1
if [ ! -f graphify-out/graph.json ]; then
  echo "Nessun grafo in $(pwd)/graphify-out/ → serve un build completo: ignora --update e prosegui dallo Step 1."
else
  graphify update .            # incrementale: solo i file cambiati, AST-only, nessun LLM
  # NB: dopo refactor con MOLTE cancellazioni, se update rifiuta di ridurre i nodi -> graphify update . --force
  echo "OK: struttura del grafo aggiornata. Le community NON sono state rinominate"
  echo "    (se hai aggiunto interi moduli usa: /master-docs:master-graph --relabel; per refactor grossi: rebuild completo)."
fi
```

Se il grafo **non** esiste, ripiega sulla procedura completa (Step 1 in poi). Altrimenti, dopo l'update, salta direttamente al **Report finale**.

### Modalità `--install-hook` / `--uninstall-hook` (autosync del grafo — scorciatoia)

Se è passato `--install-hook` (o `--uninstall-hook`): dopo lo Step 0 esegui **solo** questo blocco e termina. Installa (o rimuove) un git hook `post-commit` che tiene il grafo sempre fresco lanciando `graphify update .` **in background** dopo ogni commit — gratis, AST-only. È idempotente e si concatena a un `post-commit` già esistente senza sovrascriverlo.

> **Solo il grafo.** Il hook aggiorna esclusivamente `graphify` (operazione gratuita in secondi). La documentazione dei moduli (`docs/moduli/`) **non** viene mai rigenerata al commit — è costosa (LLM) e resta on-demand via `/master-docs:master-docs-sync`.

```bash
export PATH="$HOME/.local/bin:$PATH"
# root = primo argomento che NON è un flag (così `--install-hook` da solo non viene preso per root)
ROOT="."; for a in "$@"; do case "$a" in --*) ;; *) ROOT="$a"; break;; esac; done
cd "$ROOT" || exit 1
UNINSTALL=0; case "$*" in *--uninstall-hook*) UNINSTALL=1;; esac
git rev-parse --is-inside-work-tree >/dev/null 2>&1 || { echo "Non è un repo git: niente hook."; exit 1; }
HOOKS_DIR="$(git config core.hooksPath 2>/dev/null)"; [ -z "$HOOKS_DIR" ] && HOOKS_DIR="$(git rev-parse --git-path hooks)"
mkdir -p "$HOOKS_DIR"; HOOK="$HOOKS_DIR/post-commit"
S="# >>> master-docs master-graph autosync >>>"; E="# <<< master-docs master-graph autosync <<<"
# 1) rimuovi sempre un eventuale blocco precedente (idempotente)
if [ -f "$HOOK" ] && grep -qF "$S" "$HOOK"; then
  awk -v s="$S" -v e="$E" '$0==s{skip=1} skip!=1{print} $0==e{skip=0}' "$HOOK" > "$HOOK.tmp" && mv "$HOOK.tmp" "$HOOK"
fi
if [ "$UNINSTALL" = "1" ]; then echo "OK: blocco autosync rimosso da $HOOK"; exit 0; fi
# 2) (re)installa: shebang se il file è nuovo/vuoto, poi accoda il blocco marcato
[ -s "$HOOK" ] || printf '#!/bin/sh\n' > "$HOOK"
cat >> "$HOOK" <<EOF
$S
# Aggiorna il knowledge graph graphify dopo ogni commit (AST-only, gratis, in background).
if command -v graphify >/dev/null 2>&1 && [ -f graphify-out/graph.json ]; then
  ( export PATH="\$HOME/.local/bin:\$PATH"; graphify update . >/dev/null 2>&1 & )
fi
$E
EOF
chmod +x "$HOOK"
echo "OK: autosync post-commit installato in $HOOK"
```

Dopo l'installazione (o la rimozione) **termina** con una riga di conferma; non proseguire con il build.

### 1. Determina root e cartella app

```bash
export PATH="$HOME/.local/bin:$PATH"
ROOT="${1:-.}"; cd "$ROOT" || { echo "root non valida"; exit 1; }
if   [ -f private/artisan ]; then APP=private     # Master su hosting condiviso (app in private/)
elif [ -f artisan ];         then APP=.           # Laravel con app alla root
else echo "ATTENZIONE: non trovo artisan — non sembra un progetto Laravel. Fermati e chiedi conferma."; fi
echo "ROOT=$(pwd)  APP=$APP"
```

Se non è un Laravel, **fermati e chiedi** all'utente il percorso dell'app.

### 2. Genera lo `.graphifyignore` di root (scope = solo l'app)

Rigeneralo ad ogni run (è derivato). Se esiste già un `.graphifyignore` **senza** la riga `# managed-by: /master-graph`, potrebbe essere fatto a mano: **mostra il contenuto e chiedi conferma** prima di sovrascrivere.

```bash
{
  echo "# managed-by: /master-graph  (rigenerato dal comando; non modificare a mano)"
  echo "# Scope: solo l'app Laravel in ./$APP/. Path del grafo relativi alla root del progetto."
  if [ "$APP" != "." ]; then
    # Escludi OGNI voce top-level tranne l'app dir: gestisce symlink verso altri siti,
    # old/, assets/, ftp/, docs/, storage/, .claude/, file di root, ecc.
    ls -A1 | grep -vxF "$APP" | grep -vxF ".graphifyignore" | grep -vxF "graphify-out" \
      | while IFS= read -r e; do printf '/%s\n' "$e"; done
    BASE="$APP"
  else
    # App alla root: escludi i symlink diretti (potrebbero puntare ad altri siti) + dir non-codice
    find . -maxdepth 1 -type l -printf '/%f\n'
    printf '/%s\n' public storage
    BASE=""
  fi
  # Dentro l'app: vendored / generato / dati / rumore (le migrations restano)
  for d in vendor node_modules storage bootstrap scripts graphify-out database/seeders resources; do
    if [ -n "$BASE" ]; then printf '/%s/%s\n' "$BASE" "$d"; else printf '/%s\n' "$d"; fi
  done
  # Doc / testo / binari a QUALSIASI profondità (no slash iniziale → match ovunque)
  printf '%s\n' '*.md' '*.mdx' '*.rst' '*.txt' '*.png' '*.jpg' '*.jpeg' '*.gif' '*.svg' '*.webp' '*.bmp' '*.ico' '*.pdf' '*.mp4'
} > .graphifyignore
echo "--- .graphifyignore generato ---"; cat .graphifyignore
```

> Nota: `resources/` è escluso perché nel Master contiene per lo più array di traduzione (rumore). Se in un progetto lì c'è codice utile, rimuovi quella riga.

### 3. Build (salta se `--relabel`)

```bash
rm -rf graphify-out
graphify .
```

**Verifica anti-leak** (gate obbligatorio): i path devono iniziare tutti con `$APP/`, il conteggio deve essere ragionevole (centinaia, non migliaia) e **zero** riferimenti a `vendor`, `seeders`, o altri siti:

```bash
python3 - <<'PY'
import json,collections
d=json.load(open('graphify-out/graph.json')); n=d.get('nodes',[])
srcs=[(x.get('source_file') or x.get('src') or '') for x in n]; srcs=[s for s in srcs if s]
print('nodes',len(n),'edges',len(d.get('edges',d.get('links',[]))))
print('prefissi:',dict(collections.Counter(s.split('/',1)[0] for s in srcs).most_common(8)))
leak=[s for s in srcs if 'vendor' in s or '/seeders/' in s or s.startswith('/') or 'dev.internal' in s.split('private',1)[0]]
print('LEAK fuori-scope:',len(leak),leak[:5])
PY
```

Se `LEAK > 0` o il conteggio esplode a migliaia → **fermati**: manca un'esclusione nello `.graphifyignore` (probabile symlink o `vendor` non escluso). Correggi e ripeti il build. Se il build abortisce con *"no LLM API key found (N doc/paper/image…)"*, individua quei file (`find $APP -type f \( -iname '*.md' -o -iname '*.png' … \)`) e aggiungili all'ignore.

### 4. Nomina le community via claude-cli (salta se `--no-label`)

Operazione LLM economica (poche chiamate batch, sequenziali). Può richiedere qualche minuto: se supera ~1-2 min, lanciala in **background** e monitora il log.

```bash
export GRAPHIFY_CLAUDE_CLI_MODEL="${MODEL:-sonnet}"
graphify label . --backend claude-cli --max-concurrency 1
```

Con `--relabel` esegui **solo** questo passo (senza il build).

### 5. Pulizia + verifica finale

```bash
find graphify-out -maxdepth 1 -type d -name '20*' -exec rm -rf {} + 2>/dev/null   # backup datati
[ -f graphify-out/graph.json ] && echo "OK: graphify-out/graph.json in root → hook attivi lanciando Claude da qui"
python3 -c "import json;l=json.load(open('graphify-out/.graphify_labels.json'));print('community',len(l),'| nominate',sum(1 for v in l.values() if not str(v).lower().startswith('community')))"
# query di prova (path di default, nessun --graph):
graphify query "come viene creata e salvata un'entità principale del dominio" | head -15
```

## Report finale all'utente

Riepiloga in modo conciso: percorso del grafo (`<root>/graphify-out/`), n. nodi/edge/community (e quante nominate), file scansionati e scope (`$APP/`), eventuali esclusioni notevoli (symlink verso altri siti, seeders, resources), e i comandi d'uso quotidiano:

- `graphify query "<domanda>"` · `graphify explain "<simbolo>"` · `graphify path "<A>" "<B>"`
- dopo modifiche al codice, refresh incrementale: `/master-docs:master-graph --update` (equivale a `graphify update .` — solo AST, gratis, ~secondi)
- **una tantum per progetto**, rendi il refresh automatico: `/master-docs:master-graph --install-hook` (git hook `post-commit` → `graphify update .` in background a ogni commit; il grafo non resta mai stale). Rimuovi con `--uninstall-hook`.
- ri-nomina community (dopo aver aggiunto moduli): `/master-docs:master-graph --relabel`
- tieni freschi i deep-dive dei moduli (**solo** quelli cambiati, on-demand — mai automatico): `/master-docs:master-docs-sync`

Ricorda che gli hook `MANDATORY: graphify` scattano solo per sessioni Claude lanciate dalla cartella che contiene `graphify-out/` (cioè la root del progetto, dove hai appena costruito il grafo).
