# master-docs — Plugin Claude Code

Strumenti di sviluppo per il flusso **`graphify` + docs**: costruzione del **knowledge graph** del
progetto, refresh automatico a ogni commit e **documentazione dei moduli** (deep-dive generati da
LLM o vault Obsidian navigabile). Copre due famiglie di progetti:

- **MASTER** — progetti **Master Laravel Enesi** (app in `private/`, deep-dive per-modulo in `docs/moduli/`);
- **IONIC** — app **Ionic + Capacitor / Angular** (app in `src/`, documentazione scritta a mano resa vault Obsidian).

Il grafo (`graphify`) si aggiorna **gratis** a ogni commit (git hook `post-commit`), mentre la
documentazione — che costa LLM — resta **on-demand**: la rigeneri tu quando serve, solo sui moduli
cambiati. Se `graphify` non è installato, i comandi lo installano in automatico.

## Comandi

### Famiglia MASTER (Laravel Enesi)

| Comando | Cosa fa |
|---|---|
| `/master-docs:master-setup` | **Setup one-shot** idempotente: costruisce il grafo (scope `private/`), installa l'autosync post-commit e (se i deep-dive mancano) bootstrappa `docs/moduli/`. |
| `/master-docs:master-graph` | Crea/aggiorna il **knowledge graph** `graphify` radicato sulla root, scope automatico all'app in `private/`, community nominate gratis via claude-cli, autosync post-commit opzionale. |
| `/master-docs:master-docs-sync` | Rigenera **solo** i deep-dive di `docs/moduli/` dei moduli cambiati (stale o mancanti) — non tutti. On-demand, **mai** automatico ai commit. |
| `/master-docs:documenta-moduli` | Genera **da zero** i deep-dive dei moduli — un `.md` per modulo + indice + registro dubbi, in italiano, via workflow multi-agente. |

### Famiglia IONIC (Ionic + Capacitor / Angular)

| Comando | Cosa fa |
|---|---|
| `/master-docs:ionic-setup` | **Setup one-shot** idempotente: costruisce il grafo (scope `src/`), installa l'autosync post-commit e (opzionale) genera il vault Obsidian della documentazione. |
| `/master-docs:ionic-graph` | Crea/aggiorna il **knowledge graph** `graphify` radicato sulla root, scope automatico a `src/`, community nominate gratis via claude-cli, autosync post-commit opzionale. |
| `/master-docs:ionic-vault` | Rende la documentazione **scritta a mano** (`.claude/features` + `.claude/analysis`) un **vault Obsidian** navigabile: indice MOC, wikilink additivi, config `.obsidian`. Locale, gratis, non riscrive le schede. |

## Flusso tipico

**Progetto nuovo (Master):**

```
/master-docs:master-setup
```

Fa tutto in un colpo: grafo + autosync + bootstrap docs. Da lì, uso quotidiano:

```
graphify query|explain|path        # interroga il grafo
/master-docs:master-graph --update # refresh incrementale del grafo (solo AST, gratis)
/master-docs:master-docs-sync      # rinfresca i deep-dive dei soli moduli cambiati
```

**Progetto nuovo (Ionic):**

```
/master-docs:ionic-setup
```

Grafo + autosync + vault Obsidian della documentazione già presente.

## Perché grafo gratis e docs on-demand

- Il **grafo** si costruisce con AST (nessun LLM): aggiornarlo a ogni commit costa secondi ed è
  gratis, quindi resta sempre fresco via git hook `post-commit`.
- La **documentazione deep-dive** dei moduli costa chiamate LLM: rigenerare tutti i ~46 moduli a
  ogni commit sarebbe insostenibile. Per questo `master-docs-sync` rileva **solo** i moduli
  cambiati dall'ultima documentazione e rigenera quelli, e non viene **mai** agganciata al commit.

## Requisiti

- **`graphify`** (installato in automatico dai comandi se assente).
- **`claude-cli`** per la nomina gratuita delle community del grafo.
- Git inizializzato nel progetto (per l'autosync `post-commit`).

> **Nota permessi.** In modalità auto l'installazione di un git hook può essere bloccata dal
> classifier come "persistence": in tal caso i comandi riportano lo slash command
> (`--install-hook`) da approvare a mano. Il grafo funziona anche senza hook (basta `--update`).

## Installazione (da marketplace)

```
/plugin marketplace add enesisrl/claude-skills
/plugin install master-docs@enesi-claude-skills
```

## Struttura

```
master-docs-plugin/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   ├── master-setup.md
│   ├── master-graph.md
│   ├── master-docs-sync.md
│   ├── documenta-moduli.md
│   ├── ionic-setup.md
│   ├── ionic-graph.md
│   └── ionic-vault.md
└── README.md
```
