# security-report — Plugin Claude Code

Command `/security-report`: un **orchestratore di sicurezza** che produce **un unico report
prioritizzato** coordinando i migliori strumenti disponibili sulla macchina — plugin Claude Code
e tool CLI — invece di rifare tutto a mano. Sceglie per ogni dimensione lo strumento più forte
presente e **degrada con fallback** quando manca, senza mai ridurre la copertura in silenzio.

È il livello *orchestratore* sopra gli altri plugin di sicurezza: `/senior-engineer:security` e
`/security-review` fanno review del **codice**, `/vuln-audit` fa **DAST** sul sito live — qui
vengono messi in fila insieme a SAST/segreti/dipendenze/container e i risultati fusi in un report solo.

## Comando

```
/security-report [repo-path e/o url]
```

Esempi:
```
/security-report .                              # solo repo corrente
/security-report https://www.sito.it            # solo sito live
/security-report . https://www.sito.it          # repo + sito live
```

## Conferma interattiva (il punto chiave)

All'avvio il comando **chiede sempre conferma** (via `AskUserQuestion`) su:

1. **Quali dimensioni eseguire** (multi-select): SAST codice · DAST sito live · Segreti ·
   Dipendenze · Container · Conformità OWASP.
2. **Target**: path del repo e/o URL.
3. **Solo per il DAST**: autorizzazione e modalità **passiva** (default, non intrusiva) o
   **attiva** (solo domini autorizzati, con doppia conferma) — stessi guardrail di `/vuln-audit`.

Poi mostra il **piano** («eseguo X, Y, Z; salto W perché…») e attende il via prima di eseguire.

## Strumento preferito → fallback (per dimensione)

| Dimensione | Preferito | Fallback |
|---|---|---|
| **SAST** | `semgrep --config auto` | `/senior-engineer:security` + `/security-review` (review LLM) |
| **Segreti** | `gitleaks` / `trufflehog` | Secret Scanner plugin → grep di pattern su repo **e** history git |
| **Dipendenze** | `osv-scanner` / `grype` | `composer audit` + `npm audit` |
| **DAST** | `/vuln-audit <url>` | `nmap`/`nikto`/`nuclei` diretti → check passivi via `curl`/`openssl` |
| **Container** | `trivy` | `grype` → skip con nota |
| **Conformità** | mappatura OWASP Top 10 dei finding raccolti | — |

Se una dimensione scelta resta scoperta oltre il fallback base, il report lo **dichiara** (es.
«SAST eseguito con review LLM, non con Semgrep: copertura più debole, consiglio `brew install semgrep`»).

## Come lavora

1. **Fase 0** — scoping interattivo con conferma dello scope e (per il DAST) dell'autorizzazione.
2. **Fase 1** — detect delle capacità: quali tool CLI e plugin sono presenti.
3. **Fase 2** — esecuzione orchestrata su **subagenti paralleli read-only** (uno per dimensione):
   ognuno restituisce solo una tabella strutturata, mai i log grezzi.
4. **Fase 3** — sintesi: dedup tra fonti, gravità normalizzata (Critica→Info), report unico.

## Output

Report `private/storage/security-report/<TS>/report-<TS>.md` (in un progetto Master Laravel Enesi;
altrimenti fallback su `storage/security-report/<TS>/` o `./security-report/<TS>/`) con: contesto e
scope, riepilogo esecutivo, **finding prioritizzati** in tabella unica (con fonte e categoria OWASP),
mappatura **OWASP Top 10**, tabella di **copertura** con strumento usato e limiti, e separazione tra
**quick-win** (≤1h) e **interventi strutturali**. In chat un riepilogo di 5 righe.

I **segreti** e i dati sensibili sono sempre **oscurati** nel report; se un segreto reale è esposto,
la priorità #1 indicata è **ruotarlo**, non solo rimuoverlo dal codice.

## Dipendenze / integrazioni

Nessuna obbligatoria: funziona anche solo con i fallback base. Per il massimo della copertura:
- **Plugin Enesi consigliati**: `web-vuln-audit` (DAST), `senior-engineer` (review codice).
- **Tool CLI consigliati**: `semgrep`, `gitleaks`, `osv-scanner`/`trivy`, `composer`, `npm`.

## Installazione (da marketplace)

```
/plugin marketplace add <owner>/<repo-marketplace>
/plugin install security-report@<nome-marketplace>
```

## Struttura

```
security-report-plugin/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   └── security-report.md
└── README.md
```
