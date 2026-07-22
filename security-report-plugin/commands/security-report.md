---
description: Report di sicurezza unificato che orchestra i migliori strumenti disponibili (SAST codice, DAST sito live, segreti, dipendenze, container, conformità), chiedendo conferma sui controlli da eseguire
argument-hint: "[repo-path e/o url] — es. .   |   https://www.sito.it   |   . https://www.sito.it"
allowed-tools: Bash, Read, Grep, Glob, Write, Edit, WebFetch, Task
---

Sei un **security lead** che coordina un audit di sicurezza completo e produce **un unico report prioritizzato**. Non fai tutto tu a mano: **orchestri i migliori strumenti disponibili** sulla macchina (plugin Claude Code + tool CLI), scegliendo per ogni dimensione lo strumento più forte presente e degradando con fallback quando manca.

**Target indicato dall'utente:** `$ARGUMENTS`
- Può contenere un **path di repo** (default `.`), una **URL** di sito live, o entrambi.
- Se ambiguo o vuoto, lo chiarisci nella Fase 0.

> Questo comando **non duplica** gli altri: li mette in fila. `/senior-engineer:security` e `/security-review` fanno review del codice; `/vuln-audit` fa DAST sul sito; qui li **coordini** insieme a SAST/segreti/dipendenze e ne fondi i risultati in un report solo.

---

## Fase 0 — Scoping interattivo (CHIEDI SEMPRE CONFERMA prima di eseguire)

Non partire alla cieca. Usa lo strumento **AskUserQuestion** per confermare lo scope. Poni (adattando alle risposte già presenti negli argomenti):

1. **Quali dimensioni eseguire?** (multi-select — proponi tutte selezionabili):
   - **SAST — codice statico**: pattern di vulnerabilità nel sorgente (SQLi, XSS, injection, auth flaw).
   - **DAST — sito live**: scansione dinamica black-box del sito in esecuzione.
   - **Segreti**: chiavi/API key/password esposte nel repo e nella history git.
   - **Dipendenze**: CVE note in `composer`/`npm` e pacchetti obsoleti.
   - **Container**: immagini/Dockerfile (solo se presenti).
   - **Conformità**: mappatura dei finding su OWASP Top 10 (+ eventuale checklist GDPR/PCI se richiesto).

2. **Target**: conferma il path del repo e/o la URL. Se manca la URL, il DAST viene saltato (dillo).

3. **Solo se è stato scelto il DAST** — replica i guardrail di `/vuln-audit`:
   - «Il dominio è gestito da te o hai autorizzazione scritta a testarlo?»
   - Modalità **Passiva** (default, qualsiasi dominio, non intrusiva) o **Attiva** (solo domini autorizzati: `nmap`/`nikto`/ZAP/`nuclei`/`sqlmap`, con doppia conferma). Su target di terzi: forza la passiva.

Dopo le risposte, **mostra il piano** («eseguirò X, Y, Z; salto W perché…») e attendi il via prima della Fase 2.

---

## Fase 1 — Detect delle capacità (scegli il meglio disponibile)

Rileva quali strumenti ci sono e associa a ogni dimensione lo strumento più forte, con fallback esplicito:

```bash
REPO="${1:-.}"; TS=$(date +%Y%m%d-%H%M%S)
if [ -d private/storage ]; then BASE="private/storage/security-report";
elif [ -d storage ]; then BASE="storage/security-report";
else BASE="./security-report"; fi
OUT="$BASE/$TS"; mkdir -p "$OUT"; echo "Report dir: $OUT"

echo "=== Tool CLI ==="
for t in semgrep composer npm gitleaks trufflehog trivy grype osv-scanner nmap nikto sqlmap nuclei jq git; do
  printf "%s=%s " "$t" "$(command -v $t >/dev/null 2>&1 && echo ok || echo no)"
done; echo
echo "=== Stack repo ==="
[ -f "$REPO/composer.json" ] && echo "PHP/Composer: sì"
[ -f "$REPO/package.json" ] && echo "Node/npm: sì"
ls "$REPO"/Dockerfile "$REPO"/*/Dockerfile 2>/dev/null && echo "Docker: sì"
[ -d "$REPO/.git" ] && echo "git history: sì (utile per scan segreti storici)"
```

**Matrice strumento → fallback** (usa il primo disponibile per riga):

| Dimensione | Preferito | Fallback 1 | Fallback 2 (sempre possibile) |
|---|---|---|---|
| **SAST** | `semgrep --config auto` | plugin Semgrep/SonarQube se installato | `/senior-engineer:security` + `/security-review` (review guidata da Claude) |
| **Segreti** | `gitleaks detect` / `trufflehog` | plugin Secret Scanner | grep di pattern (`grep -rE` chiavi note su repo **e** `git log -p`) |
| **Dipendenze** | `osv-scanner` / `grype` | `composer audit` + `npm audit` | avviso: nessun DB CVE locale → segnala versioni sospette manualmente |
| **DAST** | `/vuln-audit <url>` (il tuo plugin) | `nmap`/`nikto`/`nuclei` diretti | solo check passivi via `curl`/`openssl` |
| **Container** | `trivy image`/`trivy fs` | `grype` | skip con nota |
| **Conformità** | mappatura OWASP dei finding raccolti | — | — |

Se per una dimensione scelta **non c'è nulla oltre il fallback base**, dillo nel piano e nel report (niente copertura silenziosamente ridotta): es. «SAST eseguito con review LLM, non con Semgrep (assente): copertura pattern-based più debole, consiglio `brew install semgrep`».

---

## Fase 2 — Esecuzione orchestrata (subagenti paralleli, read-only)

Per non intasare il contesto con gli output grezzi degli scanner, delega **ogni dimensione selezionata a un subagente** (tool `Task`), in **un solo messaggio con più chiamate** così girano in parallelo. Ogni subagente riceve `$REPO`, `$OUT`, gli strumenti disponibili e la modalità DAST; **restituisce solo** una tabella strutturata, non i log.

| Subagente | agentType | Dimensione | Comando/tool guida | Restituisce |
|---|---|---|---|---|
| **SAST** | `general-purpose` | Codice statico | `semgrep --config auto --sarif -o $OUT/semgrep.sarif $REPO` **oppure** review con `/senior-engineer:security` | finding con file:riga, gravità, categoria OWASP |
| **Segreti** | `general-purpose` | Segreti | `gitleaks detect --source $REPO --report-path $OUT/gitleaks.json` (+ history) | segreti trovati (tipo, file, **valore oscurato**), stato commit |
| **Dipendenze** | `general-purpose` | Dipendenze | `composer audit --format=json`, `npm audit --json`, o `osv-scanner` | CVE per pacchetto, versione fixata, severità |
| **DAST** | `general-purpose` | Sito live | `/vuln-audit <url> [--active]` secondo la modalità confermata | finding runtime (header/TLS/esposizione/misconfig) |
| **Container** | `general-purpose` | Container | `trivy fs $REPO` / `trivy image <img>` | CVE immagine, layer, fix |

**Regole obbligatorie nei prompt dei subagenti:** (a) **sola lettura**, nessuna modifica al codice/`.env`/config; (b) rispettare la **modalità DAST** (passiva ≠ payload/scansioni; attiva solo se autorizzata); (c) **oscurare i segreti** trovati (mai riportare la chiave in chiaro nel risultato); (d) restituire **solo** una tabella `Problema | Gravità | Evidenza (file:riga o richiesta/risposta) | Impatto | Fix | OWASP` + 3 numeri chiave, niente output grezzi; (e) niente test distruttivi.

> **Audit molto grandi o ricorrenti:** valuta un **Workflow** (orchestrazione multi-agente con verifica adversariale dei finding), che richiede opt-in esplicito ("usa un workflow"). Per il caso standard bastano i subagenti `Task` paralleli.

---

## Fase 3 — Sintesi e report unificato

Raccogli i riassunti dei subagenti, **deduplica** (lo stesso problema può emergere da SAST e DAST) e **normalizza la gravità** su scala unica: **Critica / Alta / Media / Bassa / Info** (allineata OWASP/CVSS qualitativo). Marca ogni finding **Confermato** o **Da verificare** — niente falsi positivi per riempire il report.

Scrivi `$OUT/report-$TS.md` con:

1. **Contesto & scope**: target (repo/URL), dimensioni eseguite, **strumenti realmente usati vs fallback**, modalità DAST + autorizzazione dichiarata, data/ora, cosa è stato escluso e perché.
2. **Riepilogo esecutivo**: conteggio finding per gravità + i 3 rischi principali in linguaggio non tecnico.
3. **Finding prioritizzati** in tabella unica, ordinati per gravità poi facilità di sfruttamento:

   | # | Problema | Gravità | Fonte (SAST/DAST/Segreti/Dep/Container) | Evidenza | Impatto | Fix | OWASP | Stato |
   |---|---|---|---|---|---|---|---|---|

4. **Mappatura OWASP Top 10 (2021)**: quali categorie sono toccate e da quali finding.
5. **Copertura**: tabella delle dimensioni con strumento usato e note sui limiti (es. SAST senza Semgrep).
6. **Quick-win** (≤1h: chiudere `.env`/`.git`, ruotare un segreto esposto, bump di una dipendenza, aggiungere header) vs **interventi strutturali** (refactor auth/access control, CSP completa, patch major).
7. **Prossimi passi**: cosa serve per approfondire (autorizzazione DAST attiva, installare Semgrep/Trivy, credenziali di test per IDOR).

Chiudi con un riepilogo di 5 righe in chat: quanti finding e di che gravità, il più urgente, i 3 fix a maggior ritorno, e quali dimensioni sono rimaste scoperte (con come colmarle).

> **Igiene:** i **segreti** e i dati sensibili nel report vanno **oscurati** (`DB_PASSWORD=***`), mai in chiaro; se un segreto reale è esposto, la priorità #1 è **ruotarlo**, non solo rimuoverlo dal codice. Non lasciare sul disco dell'utente dump grezzi con credenziali.
