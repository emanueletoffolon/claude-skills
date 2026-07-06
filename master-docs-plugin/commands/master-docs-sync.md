---
description: Rigenera SOLO i deep-dive di docs/moduli/ dei moduli Master cambiati dall'ultima documentazione (stale o mancanti) — non tutti. Risparmio token, docs sempre freschi. On-demand, MAI automatico ai commit.
argument-hint: [root-progetto] [--dry-run] [--all] [--model sonnet|haiku|opus]
---

# Sync dei deep-dive di modulo (solo gli stale)

Riallinea `docs/moduli/` al codice **senza rigenerare tutti i ~46 moduli**: rileva quali moduli sono cambiati dall'ultima volta che il loro doc è stato scritto (o non hanno ancora un doc) e rigenera **solo quelli**. È il complemento on-demand del grafo: il grafo si aggiorna gratis a ogni commit (via `/master-docs:master-graph --install-hook`), la documentazione — che costa LLM — la rinfreschi tu quando serve con questo comando.

> **Mai in automatico.** Questo comando non va MAI messo in un git hook / cron: la rigenerazione dei deep-dive è un'operazione LLM costosa e va lanciata a mano. L'unica cosa che gira ai commit è `graphify update` (gratis).

## Parametri

- `$1` = **root del progetto** — default `.` (la cwd, da dove è lanciato Claude).
- `--dry-run` → solo **rilevazione**: elenca stale/mancanti/aggiornati e termina, senza rigenerare nulla (costo zero).
- `--all` → ignora la rilevazione e rigenera **tutti** i moduli (equivale a rilanciare `documenta-moduli` da zero). Usalo solo dopo un refactor massiccio o un cambio di formato dei doc.
- `--model <m>` → modello per la rigenerazione via `documenta-moduli` (se la skill lo supporta).

## Prerequisiti

- Repo **git**: la rilevazione dello stale usa i **timestamp dei commit** (`git log`), robusti ai checkout — non gli mtime del filesystem.
- `docs/moduli/` già esistente (prodotto in origine dalla skill `documenta-moduli`). Se manca del tutto, non è un sync: lancia prima `documenta-moduli` per il bootstrap completo.
- Skill `documenta-moduli` disponibile per la rigenerazione. Se non è installata (vive in un altro marketplace), il comando ripiega su **subagenti** che rigenerano ogni doc seguendo il formato di un doc esistente.

## Procedura

### 0. Rileva i moduli stale / mancanti (gratis)

```bash
export PATH="$HOME/.local/bin:$PATH"
# root = primo arg che NON è un flag
ROOT="."; for a in "$@"; do case "$a" in --*) ;; *) ROOT="$a"; break;; esac; done
cd "$ROOT" || { echo "root non valida"; exit 1; }

MODROOT="private/master/Modules"; DOCS="docs/moduli"
[ -d "$MODROOT" ] || { echo "Nessun $MODROOT — non sembra il Master. Fermati e chiedi conferma."; exit 1; }
[ -d "$DOCS" ] || { echo "ATTENZIONE: $DOCS assente → non è un sync ma un bootstrap. Lancia prima la skill documenta-moduli."; exit 1; }

stale=(); missing=(); dirty=(); ok=0; tot=0
for d in "$MODROOT"/*/; do
  [ -d "$d" ] || continue
  m="$(basename "$d")"; tot=$((tot+1))
  doc="$DOCS/$m.md"
  code_ct="$(git log -1 --format=%ct -- "$d" 2>/dev/null)"
  # modifiche non ancora committate nel modulo (il doc non può ancora rifletterle)
  [ -n "$(git status --porcelain -- "$d" 2>/dev/null)" ] && dirty+=("$m")
  if [ ! -f "$doc" ]; then missing+=("$m"); continue; fi
  doc_ct="$(git log -1 --format=%ct -- "$doc" 2>/dev/null)"
  if [ -n "$code_ct" ] && { [ -z "$doc_ct" ] || [ "$code_ct" -gt "$doc_ct" ]; }; then
    stale+=("$m")
  else
    ok=$((ok+1))
  fi
done

echo "Moduli totali: $tot | doc aggiornati: $ok"
echo "MANCANTI  (${#missing[@]}): ${missing[*]:-—}"
echo "STALE     (${#stale[@]}): ${stale[*]:-—}"
echo "modifiche non committate (${#dirty[@]}): ${dirty[*]:-—}   # committa prima, poi rilancia"
# elenco unificato da rigenerare (stale + mancanti), utile per lo step 2
printf '%s\n' "${missing[@]}" "${stale[@]}" | sed '/^$/d' | sort -u > /tmp/master-docs-sync.targets
echo "--- da rigenerare (stale+mancanti) ---"; cat /tmp/master-docs-sync.targets 2>/dev/null
```

### 1. Decisione

- Se è stato passato `--dry-run` → **termina** col report della rilevazione. Non rigenerare nulla.
- Se `--all` → ignora la lista e usa come target **tutti** i moduli.
- Se l'elenco stale+mancanti è **vuoto** → i doc sono allineati: **termina** con "nulla da fare".
- Ci sono moduli con modifiche **non committate**? Segnalalo: il doc non può riflettere codice non committato. Consiglia di committare prima (così scatta anche l'autosync del grafo) e rilanciare.

### 2. Rigenera SOLO i moduli target

**Preferito — via skill `documenta-moduli` (scoping).** Invoca la skill limitandola ESATTAMENTE all'elenco in `/tmp/master-docs-sync.targets` (es. "documenta solo questi moduli: X Y Z; non toccare gli altri"). Passa `--model` se richiesto. Questo riusa il formato e l'orchestrazione ufficiali.

**Fallback — se `documenta-moduli` non è disponibile o non supporta lo scoping.** Per **ciascun** modulo target lancia un subagent (in parallelo) che:
1. Legge un doc esistente e recente (es. `docs/moduli/` di un modulo simile) come **template di struttura/tono**.
2. Si orienta con `graphify explain "<Modulo>"` e `graphify query "<cosa fa il modulo X>"` (mappa dipendenze/entità del modulo).
3. Legge i sorgenti del modulo in `private/master/Modules/<Modulo>/`.
4. Scrive/aggiorna `docs/moduli/<Modulo>.md` nello **stesso formato** degli altri deep-dive.

> Includi SEMPRE nel prompt di ogni subagent la regola graphify: *"esiste `graphify-out/graph.json`: usa `graphify query`/`explain`/`path` PRIMA di leggere i file grezzi; leggi i sorgenti solo per il dettaglio."*

Rigenera **solo** i target: è qui il risparmio di token (N moduli invece di ~46).

### 3. Aggiorna l'indice e il registro dubbi

Se `documenta-moduli` non lo fa da sé: aggiorna `docs/moduli/README.md` (indice) con le voci nuove/modificate e, se presente, il registro `DUBBI.md`.

### 4. (Opzionale) riallinea il grafo

Se durante il lavoro hai toccato dei sorgenti, esegui `graphify update .` per riallineare anche il grafo. Se hai già installato l'autosync (`/master-docs:master-graph --install-hook`) succede da sé al prossimo commit.

## Report finale all'utente

Riepiloga: quanti moduli **rigenerati** (con nomi), quanti **doc mancanti creati**, quanti lasciati **intatti** perché già allineati, eventuali moduli con **modifiche non committate** rimandati, e il risparmio (rigenerati N su ~46). Ricorda che il comando è on-demand e non gira mai ai commit.
