# web-vuln-audit — Plugin Claude Code

Audit di vulnerabilità **dinamico (DAST)** **a imbuto** (ricognizione → superficie → profondità)
su un **sito web in esecuzione**. Sonda il sito dall'esterno (black-box) per trovare debolezze
di sicurezza reali e produce un **report finale prioritizzato** con mappatura OWASP, nel formato
`Vulnerabilità | Evidenza | Gravità | Impatto | Fix`.

> ⚠️ Questo **non** è l'audit statico del codice. Per l'analisi dei sorgenti usa `/security` del
> plugin `senior-engineer` (white-box, SAST). Qui il bersaglio è il **sito vivo**, non il codice.

## Comando

```
/vuln-audit [url-target] [--active]
```

Esempi:
```
/vuln-audit https://www.sito.it                 # modalità passiva (default, sicura ovunque)
/vuln-audit https://staging.sito.it --active    # modalità attiva (solo domini autorizzati)
```

## Le due modalità (il cardine del plugin)

| Modalità | Quando | Cosa fa |
|---|---|---|
| **Passiva** (default) | Qualsiasi dominio, anche di terzi | Solo osservazione **non intrusiva**: security header, TLS, cookie, esposizione file/dati, misconfig, info disclosure. Traffico indistinguibile da una normale navigazione. |
| **Attiva** (`--active`) | **Solo** domini che gestisci o sei autorizzato a testare | Aggiunge `nmap`, `nikto`, OWASP ZAP, `nuclei`, `sqlmap` mirato, fuzzing di path. Traffico anomalo e potenzialmente impattante. Richiede **doppia conferma**. |

## 🔴 Autorizzazione (regola zero)

Scansionare un sistema che non ti appartiene senza autorizzazione può essere **illegale**
(in Italia art. 615-ter c.p. e affini). Il comando:

1. **Chiede sempre** se il dominio è gestito dall'utente o se ha autorizzazione a testarlo.
2. Resta in **passiva** finché non c'è conferma; la **attiva** richiede il flag `--active` **e**
   una doppia conferma esplicita prima di lanciare i tool intrusivi.
3. Su target di terzi la modalità attiva è **vietata**: resta passiva e lo dichiara nel report.
4. **Mai** test distruttivi: niente DoS/stress, niente modifica/cancellazione dati, niente prove
   su form di pagamento, niente brute-force di credenziali reali.

## Cosa fa (fasi)

| Fase | Cosa controlla | Modalità |
|---|---|---|
| **0** | Setup, autorizzazione, detect tool, fingerprint target (stack, WAF/CDN) | — |
| **1** | Security header, cookie flag, **TLS/SSL**, robots/sitemap/security.txt, sottodomini (CT logs) | Passiva |
| **2** | Esposizione file sensibili (`.env`, `.git`, dump, backup), directory listing, debug mode, CORS, metodi HTTP | Passiva |
| **3** | Check specifici per stack: **Laravel** (`/telescope`, `APP_DEBUG`, `.env`), **WordPress** (versione, user enum), matching CVE | Passiva |
| **4** | Scansione porte (`nmap`), scanner web (`nikto`, ZAP), template CVE (`nuclei`), iniezione mirata (`sqlmap`), IDOR/access control | **Attiva** |
| **Report** | Finding prioritizzati per gravità + mappatura OWASP Top 10 + quick-win vs strutturali | — |

Audit ampi vengono delegati a **subagenti read-only paralleli** (Header & TLS / Esposizione /
Superficie / Attiva): ognuno restituisce solo un riassunto strutturato, senza intasare il
contesto con i log grezzi degli scanner.

## Gravità dei finding

Allineata a OWASP / CVSS qualitativo: **Critica** (RCE, SQLi, `.env`/`.git` esposti, auth bypass)
· **Alta** (XSS confermata, IDOR, TLS rotto, debug mode in prod) · **Media** (header mancanti,
cookie senza flag, CORS lasco) · **Bassa** (banner versione, hardening) · **Info**.
Ogni finding è marcato **Confermata** o **Da verificare**: niente falsi positivi per riempire il report.

## Strumenti usati

- **Sempre disponibili** (bastano per tutta la parte passiva): `curl`, `openssl`.
- **Consigliati**: `testssl.sh` (TLS), `whatweb`/`nuclei` (fingerprint/CVE), `jq` (parsing).
- **Solo modalità attiva**: `nmap`, `nikto`, OWASP ZAP, `nuclei`, `sqlmap`, `wpscan` — nativi o via
  Docker (`drwetter/testssl.sh`, `zaproxy/zap-stable`, `projectdiscovery/nuclei`, `sullo/nikto`,
  `wpscanteam/wpscan`). Se mancano, il comando lo rileva in Fase 0 e prosegue con ciò che c'è.

## Output

Report `private/storage/vuln-audit/<TS>/report-<TS>.md` (in un progetto Master Laravel Enesi;
altrimenti fallback su `storage/vuln-audit/<TS>/` o `./vuln-audit/<TS>/`) con: contesto e scope (target, **modalità**,
autorizzazione, tool), riepilogo esecutivo per gravità, **finding prioritizzati** in tabella,
mappatura **OWASP Top 10**, e separazione tra **quick-win** (≤1h) e **interventi strutturali**.
In chat un riepilogo di 5 righe con le falle trovate, la più urgente e i 3 fix a maggior ritorno.

I dati sensibili eventualmente esposti dal target vengono citati come evidenza **minima e oscurata**,
mai esfiltrati per intero sul disco dell'utente.

## Installazione (da marketplace)

```
/plugin marketplace add <owner>/<repo-marketplace>
/plugin install web-vuln-audit@<nome-marketplace>
```

## Struttura

```
web-vuln-audit-plugin/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   └── vuln-audit.md
└── README.md
```
