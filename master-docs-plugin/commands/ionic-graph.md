---
description: Crea/aggiorna il knowledge graph graphify per un'app Ionic + Capacitor (Angular/TypeScript) — radicato sulla root, scope automatico a src/, community nominate gratis via claude-cli
argument-hint: [root-progetto] [--update|--relabel|--no-label] [--install-hook|--uninstall-hook] [--model sonnet|haiku|opus]
---

# Crea il grafo graphify per un'app Ionic + Capacitor

Costruisci (o aggiorna) il knowledge graph `graphify` di **questa app Ionic/Angular/Capacitor**, con la configurazione corretta per il nostro flusso: **Claude viene sempre lanciato dalla root del progetto**, quindi il grafo deve stare nella root (`./graphify-out/`) — così scattano gli hook `PreToolUse` e `graphify query` trova il grafo di default, e i path risultano relativi alla root (es. `src/app/services/...`).

> È il gemello Ionic di `/master-docs:master-graph` (che invece è per i Master Laravel). Stessa logica di grafo/autosync; cambia lo **scope** (l'app sta in `src/`, non in `private/`) e le esclusioni (build artifact JS/native invece di `vendor`).

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

## Gotcha da NON dimenticare (specifici delle app Ionic/Capacitor)

1. **Backend LLM gratis**: usa `--backend claude-cli` (via CLI Claude Code, sul piano). NON `--backend claude` (quello pretende `ANTHROPIC_API_KEY`). Il modello si sovrascrive con l'env `GRAPHIFY_CLAUDE_CLI_MODEL`.
2. **Scope = solo `src/`**: l'app vera è il sorgente Angular/Ionic in `src/`. Tutto il resto della root è build artifact (`www/`, `.angular/`, `dist/`), toolchain nativa (`android/`, `ios/`, `packages/`), dipendenze (`node_modules/`) o rumore (config, doc, log, tracce): va **tutto escluso**.
3. **I file non-codice fanno abortire il build AST-only** chiedendo una API key. In un progetto Angular questo significa escludere non solo `.md`/immagini ma anche i **template `.html`, gli stili `.scss/.css`, i `.json`** (assets, i18n) e i `.svg`. Solo i `.ts` devono entrare nel grafo.
4. **Escludi i `*.spec.ts`**: i test raddoppiano i nodi e sporcano le community con roba che non è architettura. Restano fuori.
5. **`node_modules` enorme**: se per errore non lo escludi, il build esplode a decine di migliaia di nodi. Il gate anti-leak allo Step 3 lo intercetta.
6. **Symlink / plugin `file:`**: i plugin Capacitor custom possono essere linkati via `file:../...` dentro `node_modules` — già escluso. Se la root ha altri symlink, l'exclude-list (che esclude *tutto tranne* `src/`) li tiene comunque fuori.

## Procedura

### 0. Verifica e (se manca) installa graphify

Esegui questo blocco per primo. Se `graphify` non è presente lo installa in automatico (via `uv`, con fallback a `pip` e all'installer standalone). Se al termine `graphify` non è disponibile, **fermati e segnala** all'utente il comando manuale.

```bash
export PATH="$HOME/.local/bin:$PATH"

if ! command -v graphify >/dev/null 2>&1; then
  echo "graphify non trovato — installazione in corso…"
  if ! command -v uv >/dev/null 2>&1; then
    pip install --user -q uv 2>/dev/null \
      || python3 -m pip install --user -q uv 2>/dev/null \
      || pip install --user -q --break-system-packages uv 2>/dev/null \
      || { command -v curl >/dev/null 2>&1 && curl -LsSf https://astral.sh/uv/install.sh | sh; }
    export PATH="$HOME/.local/bin:$PATH"
  fi
  if command -v uv >/dev/null 2>&1; then
    uv tool install graphifyy
  else
    pip install --user graphifyy || python3 -m pip install --user graphifyy \
      || pip install --user --break-system-packages graphifyy
  fi
  export PATH="$HOME/.local/bin:$PATH"
fi

if command -v graphify >/dev/null 2>&1; then
  echo "graphify OK: $(graphify --version 2>/dev/null) ($(command -v graphify))"
else
  echo "ERRORE: impossibile installare graphify in automatico."
  echo "Installa a mano:  pip install --user uv && uv tool install graphifyy"
  exit 1
fi

if ! command -v claude >/dev/null 2>&1; then
  echo "NOTA: binario 'claude' non nel PATH → niente naming community via claude-cli. Il grafo AST si costruisce comunque; usa --no-label o una API key."
fi
```

Se lo Step 0 fallisce l'installazione, non proseguire con il build.

### Modalità `--update` (refresh incrementale — scorciatoia)

Se è passato `--update`: dopo lo Step 0 esegui **solo** questo blocco e termina (NON rigenerare lo `.graphifyignore`, NON ricostruire da zero). Presuppone un build precedente.

```bash
export PATH="$HOME/.local/bin:$PATH"
ROOT="."; for a in "$@"; do case "$a" in --*) ;; *) ROOT="$a"; break;; esac; done
cd "$ROOT" || exit 1
if [ ! -f graphify-out/graph.json ]; then
  echo "Nessun grafo in $(pwd)/graphify-out/ → serve un build completo: ignora --update e prosegui dallo Step 1."
else
  graphify update .            # incrementale: solo i file cambiati, AST-only, nessun LLM
  # NB: dopo refactor con MOLTE cancellazioni, se update rifiuta di ridurre i nodi -> graphify update . --force
  echo "OK: struttura del grafo aggiornata. Le community NON sono state rinominate"
  echo "    (se hai aggiunto interi moduli/pagine usa: /master-docs:ionic-graph --relabel)."
fi
```

Se il grafo **non** esiste, ripiega sulla procedura completa (Step 1 in poi). Altrimenti salta al **Report finale**.

### Modalità `--install-hook` / `--uninstall-hook` (autosync del grafo — scorciatoia)

Se è passato `--install-hook` (o `--uninstall-hook`): dopo lo Step 0 esegui **solo** questo blocco e termina. Installa (o rimuove) un git hook `post-commit` che tiene il grafo sempre fresco lanciando `graphify update .` **in background** dopo ogni commit — gratis, AST-only. Idempotente e non distruttivo su hook esistenti.

```bash
export PATH="$HOME/.local/bin:$PATH"
ROOT="."; for a in "$@"; do case "$a" in --*) ;; *) ROOT="$a"; break;; esac; done
cd "$ROOT" || exit 1
UNINSTALL=0; case "$*" in *--uninstall-hook*) UNINSTALL=1;; esac
git rev-parse --is-inside-work-tree >/dev/null 2>&1 || { echo "Non è un repo git: niente hook."; exit 1; }
HOOKS_DIR="$(git config core.hooksPath 2>/dev/null)"; [ -z "$HOOKS_DIR" ] && HOOKS_DIR="$(git rev-parse --git-path hooks)"
mkdir -p "$HOOKS_DIR"; HOOK="$HOOKS_DIR/post-commit"
S="# >>> master-docs graphify autosync >>>"; E="# <<< master-docs graphify autosync <<<"
if [ -f "$HOOK" ] && grep -qF "$S" "$HOOK"; then
  awk -v s="$S" -v e="$E" '$0==s{skip=1} skip!=1{print} $0==e{skip=0}' "$HOOK" > "$HOOK.tmp" && mv "$HOOK.tmp" "$HOOK"
fi
if [ "$UNINSTALL" = "1" ]; then echo "OK: blocco autosync rimosso da $HOOK"; exit 0; fi
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
# Ionic/Capacitor/Angular: l'app sta in src/ (main.ts). Rileva anche i marker di progetto.
if [ -f src/main.ts ] || [ -f src/app/app.module.ts ] || [ -f src/app/app.config.ts ]; then
  APP=src
elif [ -f ionic.config.json ] || [ -f capacitor.config.ts ] || [ -f capacitor.config.json ] || [ -f angular.json ]; then
  APP=src   # marker di progetto presenti ma layout non standard: prova comunque src/
else
  echo "ATTENZIONE: non trovo src/main.ts né marker Ionic/Capacitor/Angular. Non sembra un'app Ionic. Fermati e chiedi conferma del path dell'app."
fi
echo "ROOT=$(pwd)  APP=$APP"
[ -d "$APP" ] || { echo "La cartella app '$APP' non esiste — chiedi conferma."; }
```

Se non è un'app Ionic/Angular, **fermati e chiedi** all'utente il percorso dell'app.

### 2. Genera lo `.graphifyignore` di root (scope = solo `src/`)

Rigeneralo ad ogni run (è derivato). Se esiste già un `.graphifyignore` **senza** la riga `# managed-by: /ionic-graph`, potrebbe essere fatto a mano: **mostra il contenuto e chiedi conferma** prima di sovrascrivere.

```bash
APP="${APP:-src}"
{
  echo "# managed-by: /ionic-graph  (rigenerato dal comando; non modificare a mano)"
  echo "# Scope: solo l'app Angular/Ionic in ./$APP/. Path del grafo relativi alla root del progetto."
  # Escludi OGNI voce top-level tranne l'app dir: build artifact (www, .angular, dist), nativo
  # (android, ios, packages), node_modules, config, docs, log, tracce, symlink...
  ls -A1 | grep -vxF "$APP" | grep -vxF ".graphifyignore" | grep -vxF "graphify-out" \
    | while IFS= read -r e; do printf '/%s\n' "$e"; done
  # Dentro src/: solo i .ts entrano nel grafo. Escludi non-codice (abort AST-only) + test.
  printf '%s\n' '*.md' '*.mdx' '*.rst' '*.txt' '*.html' '*.scss' '*.css' '*.sass' '*.less' \
    '*.json' '*.svg' '*.png' '*.jpg' '*.jpeg' '*.gif' '*.webp' '*.bmp' '*.ico' '*.pdf' \
    '*.mp4' '*.gpx' '*.wasm' '*.woff' '*.woff2' '*.ttf'
  printf '%s\n' '*.spec.ts' '*.e2e-spec.ts'
} > .graphifyignore
echo "--- .graphifyignore generato ---"; cat .graphifyignore
```

> Nota: se in `src/` c'è codice utile in `.js` (raro), aggiungi tu i `.js` allo scope. Di default consideriamo solo TypeScript.

### 3. Build (salta se `--relabel`)

```bash
rm -rf graphify-out
graphify .
```

**Verifica anti-leak** (gate obbligatorio): i path devono iniziare tutti con `src/`, il conteggio deve essere ragionevole (qualche migliaio, non decine di migliaia) e **zero** riferimenti a `node_modules`, `www`, `android`, `ios`, `packages`:

```bash
python3 - <<'PY'
import json,collections
d=json.load(open('graphify-out/graph.json')); n=d.get('nodes',[])
srcs=[(x.get('source_file') or x.get('src') or '') for x in n]; srcs=[s for s in srcs if s]
print('nodes',len(n),'edges',len(d.get('edges',d.get('links',[]))))
print('prefissi:',dict(collections.Counter(s.split('/',1)[0] for s in srcs).most_common(8)))
bad=('node_modules','www/','android','ios/','packages/','dist/','.angular')
leak=[s for s in srcs if s.startswith('/') or any(b in s for b in bad)]
print('LEAK fuori-scope:',len(leak),leak[:5])
PY
```

Se `LEAK > 0` o il conteggio esplode a decine di migliaia → **fermati**: manca un'esclusione nello `.graphifyignore` (probabile `node_modules` o una cartella build). Correggi e ripeti il build. Se il build abortisce con *"no LLM API key found (N doc/paper/image…)"*, quei file sono `.html`/`.scss`/`.json`/immagini non esclusi: aggiungili all'ignore e ripeti.

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
graphify query "come avviene la connessione principale al dispositivo / al backend" | head -15
```

## Report finale all'utente

Riepiloga in modo conciso: percorso del grafo (`<root>/graphify-out/`), n. nodi/edge/community (e quante nominate), file scansionati e scope (`src/`), eventuali esclusioni notevoli (build artifact, native, spec), e i comandi d'uso quotidiano:

- `graphify query "<domanda>"` · `graphify explain "<simbolo>"` · `graphify path "<A>" "<B>"`
- dopo modifiche al codice, refresh incrementale: `/master-docs:ionic-graph --update` (equivale a `graphify update .` — solo AST, gratis, ~secondi)
- **una tantum per progetto**, rendi il refresh automatico: `/master-docs:ionic-graph --install-hook` (git hook `post-commit` → `graphify update .` in background a ogni commit). Rimuovi con `--uninstall-hook`.
- ri-nomina community (dopo aver aggiunto pagine/servizi): `/master-docs:ionic-graph --relabel`
- vault Obsidian navigabile della documentazione (`.claude/features` + `.claude/analysis`): `/master-docs:ionic-vault`

Ricorda che gli hook `MANDATORY: graphify` scattano solo per sessioni Claude lanciate dalla cartella che contiene `graphify-out/` (la root del progetto, dove hai appena costruito il grafo).
