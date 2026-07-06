---
description: Genera documentazione deep-dive (IT) dei moduli di un progetto — un file Markdown per modulo + indice + registro dubbi — orchestrata con un workflow multi-agente
argument-hint: [dir-moduli] [dir-output]
---

# Documenta i moduli del progetto

Genera la documentazione completa dei moduli di questo progetto: un file Markdown per modulo, in **italiano**, con taglio **ibrido** (sezione funzionale + sezione tecnica) e profondità **deep dive**. Più un indice e un registro dei dubbi.

## Parametri

- **Directory dei moduli** = `$1` — se vuoto, usa `private/master/Modules` (struttura Master Laravel Enesi). Se quel percorso non esiste, **rileva** la struttura del progetto (cerca cartelle tipo `Modules/`, `app/Modules/`, `src/modules/`, `packages/`) e **chiedi conferma** del percorso prima di procedere.
- **Directory di output** = `$2` — se vuoto, usa `docs/moduli`.

## Procedura

1. **Determina** `MODULES_DIR` e `OUTPUT_DIR` come sopra. Crea `OUTPUT_DIR` se non esiste.

2. **Rileva se è un Master Laravel Enesi**. È un Master Laravel se vale almeno uno di: `composer.json` richiede un pacchetto `enesisrl/laravel-master-*` (in particolare `enesisrl/laravel-master-core`); esiste `private/master/Modules/`; è usato il namespace `Master\`. Se sì → attiva la **Modalità Master** (vedi sezione dedicata sotto).

3. **Chiedi quale modello usare** per gli agenti del workflow, con una domanda strutturata (AskUserQuestion). Opzioni:
   - **Sonnet** (`sonnet`) — buon compromesso qualità/velocità/costo. *(consigliato come default)*
   - **Opus** (`opus`) — massima qualità, più lento e costoso.
   - **Haiku** (`haiku`) — veloce ed economico, adatto a progetti piccoli o moduli semplici.
   - **Misto** — `opus` sui moduli core del dominio, `sonnet` sul resto.
   Applica la scelta come `opts.model` di **ogni** chiamata `agent()` del workflow (per "Misto", scegli il modello per modulo). Se l'utente ha già indicato il modello nel messaggio di invocazione, non richiederlo.

4. **Elenca i moduli**: le sottocartelle dirette di `MODULES_DIR`. Mostra quanti moduli sono stati trovati. Escludi le parti front-end/sito pubblico se l'utente lo indica o se sono in un albero separato.

5. **Genera la documentazione con un workflow multi-agente** (tool `Workflow`): **un agente per modulo**, in parallelo, con il `model` scelto al punto 3. Cabla la lista dei moduli direttamente nello script (NON passarla via `args`: è soggetta a problemi di serializzazione). Ogni agente:
   - legge i file del proprio modulo (vedi "Cosa analizzare"),
   - scrive `OUTPUT_DIR/<NomeModulo>.md` con lo strumento Write,
   - restituisce un output strutturato via `schema`: `{ name, area, oneLiner, status (completo|sintetico|da rivedere), tables[], doubts:[{ref, question}] }`.
   Fase finale di **sintesi** (un agente) che, dai dati strutturati aggregati, scrive:
   - `OUTPUT_DIR/README.md` — indice dei moduli **raggruppati per area funzionale**, con tabella `| Modulo | Descrizione | Stato |` e link relativi;
   - `OUTPUT_DIR/DUBBI.md` — registro aggregato di tutti i dubbi, raggruppato per modulo, ogni voce con riferimento `path:riga` e domanda; in testa il totale.
   Su pochi moduli (≲ 8) puoi procedere in sequenza senza workflow.

6. **Verifica l'output**: tutti i file presenti, nessun file troncato, nessun artefatto di sintassi tool (`</content>`, `</invoke>`, `<parameter>`) finito per errore nei `.md`; correggi quelli che trovi.

7. **Riepilogo finale**: numero di moduli documentati, eventuali falliti e perché, numero di voci in `DUBBI.md`.

8. **Commit/push**: NON committare automaticamente. Al termine **proponi** commit e push, ed eseguili solo se l'utente lo conferma (o l'ha già richiesto nel messaggio di invocazione).

## Modalità Master (solo se il progetto è un Master Laravel Enesi)

Quando attiva, sfrutta le skill e gli agent del plugin `master-laravel-enesi-plugin` per documentare in modo più accurato e coerente con l'ecosistema:

- **Convenzioni assodate** (NON discuterle, dalle per scontate nella documentazione): PK **UUID**, namespace `Master\`, classi base dal core (`enesisrl/laravel-master-core`), campi di audit `string(36)` (`created_by`/`updated_by`/`deleted_by`), pattern multilingua (model `*Translation` e valori `*Value`/EAV), label admin via `__('admin::label.*')`. La skill `master-review` ne è il riferimento.
- **Custom vs pacchetto ufficiale**: per ogni modulo indica se è **custom** (definito in `private/master/Modules/`) o **fornito/derivato da un pacchetto ufficiale** `enesisrl/laravel-master-*`. Per verificarlo consulta `composer.json` e `private/vendor/enesisrl/`, oppure usa l'agent **`master-laravel-enesi-plugin:master-package-scout`**.
- **Comandi/cron**: per la sezione 9 dei documenti, appoggiati alla skill **`master-laravel-enesi-plugin:master-artisan`** per riconoscere i comandi artisan dell'ecosistema (nativi e dei pacchetti).
- Quando usi gli agent del workflow puoi assegnare `agentType: 'master-laravel-enesi-plugin:master-package-scout'` a una fase di classificazione custom/pacchetto, mantenendo gli agenti documentatori come default.

Se il progetto **non** è un Master Laravel, ignora questa sezione e documenta in modo generico.

## Cosa analizzare per ogni modulo

Adatta ai file effettivamente presenti. Per un modulo Master Laravel Enesi tipicamente:
- `config.php` → chiave `model`, closure in `register` (rotte admin), blocco `crud` (`form` e `list`), permessi.
- `Models/*.php` → `$table`, `$fillable`, `$casts`, relazioni Eloquent, trait, scope, accessor/mutator, costanti.
- `Controllers/*.php` → azioni custom oltre al CRUD ereditato dalla classe base.
- `Facades/*.php` e la classe sottostante → metodi pubblici esposti ad altri moduli.
- `Form/` e `Form/Fields/`, `Classes/Module.php`.
- `Views/` → solo elenco delle viste principali (non trascrivere i template).
- **Migrazioni** del progetto relative alla/e tabella/e del modulo → ricostruisci lo schema completo dei campi con tipo.
- **Comandi/cron/job** collegati e relativo scheduling.
- **Dipendenze**: altri moduli importati e pacchetti vendor.

## Struttura di ogni documento `<NomeModulo>.md`

1. Intestazione (nome + path del modulo).
2. Scopo funzionale — cosa rappresenta nel dominio applicativo del progetto, a cosa serve, chi lo usa.
3. Entità e schema dati — per ogni tabella TUTTI i campi con tipo e significato; evidenzia PK, soft delete, audit. Usa tabelle markdown.
4. Relazioni — relazioni con altri modelli/moduli, con descrizione del legame.
5. Rotte — metodo HTTP, URI, azione, scopo (custom + eventuali rotte ereditate).
6. Form e lista — campi del form (con tab) e colonne della lista.
7. Facade e metodi pubblici — firma e scopo.
8. Integrazioni e dipendenze — da cosa dipende E chi dipende da questo modulo (in Modalità Master: anche custom vs pacchetto ufficiale).
9. Comandi, cron e job collegati — con frequenza di schedulazione se presente.
10. Note e dubbi — sintesi locale.

## Regole

- Scrivi tutto in **italiano**; nomi di file/classi/tabelle/rotte/comandi restano invariati.
- Cita le fonti con `path:riga`.
- **Non inventare**: ciò che non è deducibile dal codice va riportato come dubbio (nel campo `doubts` e in `DUBBI.md`), mai scritto come fatto.
- **Non interrompere** il lavoro per fare domande durante la generazione: ogni ambiguità in `DUBBI.md`, poi prosegui.
- Calibra la profondità: massimo dettaglio sui moduli core del dominio; più sintetico (ma con schema) sulle tabelle di lookup/anagrafiche.
- Mantieni coerenza terminologica tra i documenti.

Argomenti grezzi ricevuti: `$ARGUMENTS`
