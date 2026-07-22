---
description: Audit di vulnerabilità a imbuto su un sito web live (DAST) — passivo su qualsiasi dominio, attivo solo su domini autorizzati, con report finale prioritizzato
argument-hint: "[url-target] [--active]  — es. https://www.sito.it   |   https://staging.sito.it --active"
allowed-tools: Bash, Read, Grep, Glob, Write, Edit, WebFetch
---

Stai eseguendo un **audit di vulnerabilità dinamico (DAST)** su un **sito web in esecuzione**. Sondi il sito dall'esterno (black-box) per trovare debolezze di sicurezza reali: security header, TLS, misconfigurazioni, esposizione di file/dati sensibili, e — solo se autorizzato — test attivi di iniezione e scansione delle porte.

**Target indicato dall'utente:** `$ARGUMENTS`
- Il **primo** valore è la URL/host bersaglio.
- Il flag `--active` (o `--deep`) richiede la **modalità attiva** (scansione intrusiva).
- Se vuoto: chiedi la URL. Senza un target non c'è audit.

> ⚠️ Questo NON è l'audit statico del codice (per quello c'è `/security` di `senior-engineer`). Qui il bersaglio è il **sito vivo**, non i sorgenti.

---

## 🔴 Regola zero — Autorizzazione (NON saltarla MAI)

Scansionare un sistema che non ti appartiene **senza autorizzazione può essere illegale** (in Italia art. 615-ter c.p. e affini). Prima di qualunque test:

1. **Chiedi esplicitamente all'utente: «Questo dominio è gestito da te / hai autorizzazione scritta a testarlo?»** Non procedere alla modalità attiva finché non hai una risposta chiara e affermativa.
2. **Determina la modalità in base alla risposta:**

   | Modalità | Quando | Cosa fa |
   |---|---|---|
   | **PASSIVA** (default) | Qualsiasi dominio, anche di terzi | Solo osservazione **non intrusiva**: legge risposte HTTP, header, TLS, contenuti pubblici. Nessun payload d'attacco, nessuna scansione di porte, nessun brute-force. Traffico indistinguibile da una normale navigazione. |
   | **ATTIVA** (`--active`) | **Solo** domini che l'utente gestisce o è autorizzato a testare | Aggiunge scansione porte (`nmap`), scanner web (`nikto`, ZAP, `nuclei`), test di iniezione mirati (`sqlmap`), fuzzing di path. Genera traffico anomalo e potenzialmente rilevabile/impattante. |

3. **La modalità attiva richiede DOPPIA conferma:** anche se l'utente ha passato `--active`, prima di lanciare i tool intrusivi (Fase 4) mostra cosa stai per eseguire e chiedi conferma finale. Su target di terzi la modalità attiva è **vietata** — resta in passiva e dillo.
4. **Mai** test distruttivi: niente DoS/stress, niente esaurimento risorse, niente modifica/cancellazione dati, niente prove su form di pagamento/checkout, niente brute-force di credenziali reali. Rate limit basso e rispettoso.
5. Se il target è dietro una **WAF/CDN di terzi** (Cloudflare, Akamai…), la scansione attiva può violare i loro ToS e attivare blocchi/ban IP: avvisa l'utente prima.

Documenta nel report: dominio, modalità usata, autorizzazione dichiarata dall'utente, data/ora, IP sorgente.

---

## Principio guida (NON saltarlo)
Procedi **a imbuto: ricognizione → superficie → profondità**. Prima mappa cosa è esposto con osservazione passiva (economica e sicura), poi scendi in profondità solo dove la superficie mostra debolezze. Ad ogni fase confronta con le **best practice di riferimento** e annota i *finding* nel formato:

`Vulnerabilità → Evidenza (richiesta/risposta concreta) → Gravità → Impatto (scenario d'attacco) → Fix`

Classifica la **gravità** in modo coerente (allineato OWASP / CVSS qualitativo):
- **Critica** — sfruttabile ora, impatto grave (RCE, SQLi confermata, dati sensibili esposti, auth bypass).
- **Alta** — sfruttabile con poco sforzo o alto impatto (XSS confermata, IDOR, `.env`/`.git` esposti, TLS rotto).
- **Media** — richiede condizioni o impatto limitato (header di sicurezza mancanti, cookie senza flag, info disclosure).
- **Bassa** — hardening/difesa in profondità (banner versione, header informativi, TLS subottimale).
- **Info** — osservazione, nessun rischio diretto.

Distingui sempre **falla confermata** (riproducibile dall'evidenza raccolta) da **sospetto da verificare** (richiede contesto/accesso che non hai). Non gonfiare il report: un falso positivo erode la fiducia quanto un falso negativo.

---

## Fase 0 — Setup, autorizzazione e detect (sempre)

```bash
TARGET="https://www.sito.it"                      # <-- sostituisci con il primo argomento
HOST=$(echo "$TARGET" | sed -E 's#^https?://##; s#/.*$##')
TS=$(date +%Y%m%d-%H%M%S)
# Report sotto private/storage/ se sei in un progetto Master Laravel Enesi; altrimenti fallback locale.
if [ -d private/storage ]; then BASE="private/storage/vuln-audit";
elif [ -d storage ]; then BASE="storage/vuln-audit";
else BASE="./vuln-audit"; fi
OUT="$BASE/$TS"; mkdir -p "$OUT"; echo "Report dir: $OUT  (host=$HOST)"

echo "=== Tool nativi ==="
for t in curl openssl nmap nikto sqlmap dig host whatweb nuclei testssl.sh wpscan jq; do
  printf "%s=%s " "$t" "$(command -v $t >/dev/null 2>&1 && echo ok || echo no)"
done; echo
echo "=== Immagini Docker utili (se docker c'è) ==="
command -v docker >/dev/null 2>&1 && for img in \
  drwetter/testssl.sh zaproxy/zap-stable projectdiscovery/nuclei \
  sullo/nikto wpscanteam/wpscan; do
  docker image inspect "$img" >/dev/null 2>&1 && echo "$img=ok" || echo "$img=manca (docker pull $img)"
done || echo "docker non disponibile"
```

**Stabilisci cosa è disponibile** e adatta le fasi ai tool presenti. Regole di fallback:
- `curl` + `openssl` bastano per **tutta la Fase 1–3 passiva** — sono il minimo garantito.
- I tool attivi (`nmap`, `nikto`, `sqlmap`, ZAP, `nuclei`) sono opzionali: se mancano, usali via Docker (immagini sopra) oppure suggerisci l'installazione (`brew install nmap nikto sqlmap` su macOS; ZAP/nuclei via Docker) e prosegui con ciò che c'è.

**Fingerprint iniziale del target** (passivo, sempre):
```bash
curl -sSI --max-time 15 "$TARGET" | tee "$OUT/headers.txt"
echo "--- redirect chain ---"; curl -sSIL --max-time 20 "$TARGET" | grep -iE '^(HTTP/|location):'
command -v whatweb >/dev/null 2>&1 && whatweb -a3 "$TARGET" | tee "$OUT/whatweb.txt"
```
Annota: web server e versione, stack (PHP/Laravel? WordPress? Nginx/Apache?), presenza di **WAF/CDN** (header `cf-ray`, `server: cloudflare`, `x-akamai-*`), HTTP→HTTPS redirect.

---

## Fase 1 — Ricognizione passiva & superficie (qualsiasi dominio)

Tutto in questa fase è **non intrusivo**: è ciò che qualunque browser vede.

### 1a — Security header HTTP
```bash
curl -sSI --max-time 15 "$TARGET" | tr -d '\r' > "$OUT/hdr.txt"
for h in strict-transport-security content-security-policy x-frame-options \
         x-content-type-options referrer-policy permissions-policy \
         cross-origin-opener-policy cross-origin-resource-policy; do
  v=$(grep -i "^$h:" "$OUT/hdr.txt")
  [ -z "$v" ] && echo "MANCANTE  $h" || echo "presente  $v"
done
echo "--- header informativi (leak versione) ---"
grep -iE '^(server|x-powered-by|x-aspnet-version|x-generator|x-drupal|x-runtime):' "$OUT/hdr.txt"
```
**Lettura / soglie:**
- **HSTS** mancante o `max-age` basso (< 15768000) → downgrade a HTTP possibile → **Media**.
- **Content-Security-Policy** assente → nessuna difesa in profondità contro XSS/injection → **Media** (Alta su siti con contenuti utente/login).
- `X-Frame-Options`/`frame-ancestors` assenti → **clickjacking** → **Media**.
- `X-Content-Type-Options: nosniff` assente → MIME sniffing → **Bassa/Media**.
- `Server`/`X-Powered-By` che espongono versioni precise → facilita il matching CVE → **Bassa** (ma utile per la Fase 3).

### 1b — Cookie & flag di sicurezza
```bash
curl -sSI --max-time 15 "$TARGET" | grep -i '^set-cookie:'
```
Per ogni cookie di sessione/auth verifica: `Secure`, `HttpOnly`, `SameSite`. Cookie di sessione senza `HttpOnly` → furto via XSS (**Alta**); senza `Secure` su HTTPS → intercettabile (**Media**); `SameSite=None` senza motivo → CSRF (**Media**).

### 1c — TLS / SSL
```bash
# nativo, veloce:
echo | openssl s_client -connect "$HOST:443" -servername "$HOST" 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
# completo (consigliato): testssl.sh nativo o via Docker
command -v testssl.sh >/dev/null 2>&1 \
  && testssl.sh --quiet --color 0 "$HOST" | tee "$OUT/testssl.txt" \
  || (command -v docker >/dev/null 2>&1 && docker run --rm drwetter/testssl.sh --quiet --color 0 "$HOST" | tee "$OUT/testssl.txt")
```
**Cerca:** certificato scaduto/self-signed/hostname mismatch (**Alta**), protocolli deboli (SSLv3, TLS 1.0/1.1 abilitati → **Media**), cipher deboli/RC4/3DES, vulnerabilità note (Heartbleed, ROBOT, POODLE → **Alta/Critica** se confermate), assenza di forward secrecy.

### 1d — File di scoperta pubblici
```bash
for p in robots.txt sitemap.xml .well-known/security.txt humans.txt; do
  code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 10 "$TARGET/$p")
  echo "$code  $TARGET/$p"
done
curl -s --max-time 10 "$TARGET/robots.txt" | grep -iE 'disallow|sitemap' | head -20
```
`robots.txt` spesso rivela path admin/riservati (non è protezione!). Assenza di `security.txt` → **Info**. Path `Disallow` interessanti vanno esaminati (con cautela) nelle fasi successive.

### 1e — Sottodomini & superficie DNS (passivo, via Certificate Transparency)
```bash
# elenco sottodomini dai log CT pubblici (nessun contatto col target)
curl -s "https://crt.sh/?q=%25.$HOST&output=json" \
  | jq -r '.[].name_value' 2>/dev/null | sed 's/\*\.//' | sort -u | head -50 | tee "$OUT/subdomains.txt"
```
Sottodomini dimenticati (staging, dev, admin, old) sono superficie d'attacco frequente. **Non** scansionarli senza autorizzazione; segnalali come superficie da verificare.

---

## Fase 2 — Esposizione & misconfigurazioni (leggera, quasi passiva)

Probe **non distruttivi** di risorse che non dovrebbero essere pubbliche. Sono richieste GET singole a path noti — nessun payload d'attacco — ma verifica di essere in un ambito autorizzato se il volume cresce.

### 2a — File e directory sensibili esposti
```bash
for p in .git/config .git/HEAD .env .env.bak .env.save config.php.bak \
         backup.zip backup.sql db.sql dump.sql .DS_Store .htaccess \
         composer.json composer.lock package.json phpinfo.php info.php \
         storage/logs/laravel.log wp-config.php.bak web.config; do
  code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 10 "$TARGET/$p")
  [ "$code" = "200" ] && echo "🔴 ESPOSTO ($code)  $TARGET/$p" || echo "   $code  $p"
done
```
Qualsiasi `200` su `.env`, `.git/`, dump SQL, backup → **Critica** (spesso contiene credenziali DB, APP_KEY, chiavi API). Verifica scaricando solo l'header/prime righe per confermare, senza esfiltrare dati inutilmente.

### 2b — Directory listing
```bash
for d in uploads/ files/ backup/ storage/ images/ assets/ old/ tmp/; do
  body=$(curl -s --max-time 10 "$TARGET/$d")
  echo "$body" | grep -qiE 'Index of|<title>Directory listing' && echo "🔴 LISTING  $TARGET/$d"
done
```

### 2c — Info disclosure & debug mode
```bash
# pagina di errore: forza un 404/500 e cerca stack trace / debug
curl -s --max-time 12 "$TARGET/questa-pagina-non-esiste-$RANDOM" | \
  grep -iE 'stack trace|whoops|laravel|symfony|exception|fatal error|on line|APP_DEBUG|SQLSTATE' | head
```
Stack trace/Whoops (Laravel `APP_DEBUG=true`) in produzione → espone path, versioni, a volte query/credenziali → **Alta**.

### 2d — CORS misconfig
```bash
curl -sSI --max-time 12 -H "Origin: https://evil.example" "$TARGET" \
  | grep -iE 'access-control-allow-(origin|credentials)'
```
`Access-Control-Allow-Origin` che riflette l'`Origin` arbitrario **insieme** a `Allow-Credentials: true` → furto dati cross-origin → **Alta**.

### 2e — Metodi HTTP & endpoint di gestione
```bash
curl -sSI --max-time 12 -X OPTIONS "$TARGET" | grep -i '^allow:'
```
`TRACE`/`PUT`/`DELETE` abilitati senza motivo → **Media**. Verifica accessibilità di pannelli comuni (`/admin`, `/wp-admin`, `/phpmyadmin`, `/.git`, `/actuator`, `/server-status`) con GET singole e annota solo quelli che rispondono.

---

## Fase 3 — Check specifici per stack (in base al fingerprint)

Adatta in base a cosa hai rilevato in Fase 0.

**Se Laravel / Master Enesi:**
```bash
for p in telescope _debugbar/open .env storage/logs/laravel.log \
         vendor/composer/installed.json horizon; do
  code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 10 "$TARGET/$p")
  echo "$code  $TARGET/$p"
done
```
`/telescope` o `/horizon` pubblici → espongono richieste, query, job → **Alta/Critica**. `APP_DEBUG=true` (vedi 2c) → **Alta**. `.env` accessibile → **Critica**.

**Se WordPress** (fingerprint `wp-content`, generator meta):
```bash
curl -s --max-time 10 "$TARGET/" | grep -oE 'content="WordPress [0-9.]+"'   # versione
curl -s -o /dev/null -w '%{http_code}\n' --max-time 10 "$TARGET/wp-json/wp/v2/users"  # user enum via REST
# scansione dedicata (SOLO modalità attiva e autorizzata):
# wpscan --url "$TARGET" --enumerate vp,vt,u --random-user-agent   (o via docker wpscanteam/wpscan)
```
Enumerazione utenti via REST API (`200` con lista) → **Media**. Plugin/temi con CVE note → dipende.

**Matching CVE per versione:** con le versioni raccolte (server, framework, CMS, plugin), cerca CVE note. Usa `nuclei` (Fase 4, attiva) per confermare, oppure segnala come sospetto da verificare in passiva.

---

## Fase 4 — Scansione attiva (SOLO modalità `--active` + autorizzazione confermata)

> ⛔ **Non entrare in questa fase** senza: (a) flag `--active`, (b) conferma esplicita che il dominio è gestito/autorizzato, (c) la doppia conferma della Regola Zero. Su target di terzi: SALTA e dillo nel report.
> Prima di lanciare, **mostra all'utente i comandi esatti** e attendi il via. Rate limit basso, orari concordati se è produzione.

### 4a — Scansione porte/servizi (nmap)
```bash
nmap -sV -T3 --top-ports 1000 -oN "$OUT/nmap.txt" "$HOST"
# servizi web anche su porte alternative? controlla 8080/8443/etc dal risultato
```
Servizi inattesi esposti (DB, Redis, admin panel, SSH aperto al mondo) → finding per servizio.

### 4b — Scanner web (nikto)
```bash
nikto -h "$TARGET" -maxtime 300 -o "$OUT/nikto.txt" \
  || docker run --rm sullo/nikto -h "$TARGET" -maxtime 300
```
Segnala file pericolosi, config di default, header, versioni note vulnerabili.

### 4c — OWASP ZAP baseline (passiva profonda) → poi eventuale scan attiva
```bash
# baseline = spider passivo + regole passive (poco invasivo):
docker run --rm -v "$(pwd)/$OUT:/zap/wrk:rw" zaproxy/zap-stable \
  zap-baseline.py -t "$TARGET" -r zap-baseline.html
# full active scan (INVASIVO — solo con ok esplicito): zap-full-scan.py -t "$TARGET"
```

### 4d — Template CVE/misconfig (nuclei)
```bash
nuclei -u "$TARGET" -severity critical,high,medium -o "$OUT/nuclei.txt" \
  || docker run --rm projectdiscovery/nuclei -u "$TARGET" -severity critical,high,medium
```

### 4e — Test di iniezione mirati (sqlmap) — massima cautela
Solo su **parametri specifici già identificati** (querystring, form) e **solo** in autorizzazione. Mai su tutto il sito, mai con flag distruttivi.
```bash
# esempio mirato, livello basso, senza dump automatico:
sqlmap -u "$TARGET/pagina?id=1" --batch --level=2 --risk=1 --technique=BEUST \
  --output-dir="$OUT/sqlmap" --flush-session
```
Conferma la SQLi solo se sqlmap la riproduce; NON procedere a dump massivi di dati — basta la prova d'esistenza.

### 4f — Access control / IDOR (semi-manuale)
Con due utenti/sessioni di test (forniti dall'utente), verifica se l'utente A può accedere alle risorse di B cambiando ID negli URL/API. Questo è quasi sempre **manuale** e richiede credenziali di test: guida l'utente, non fabbricare credenziali.

---

## Esecuzione con subagenti (velocità + contesto pulito)

Per audit ampi **non eseguire tutto inline**: gli output degli scanner (ZAP HTML, nmap, nikto, testssl) sono enormi e intasano il contesto. Dopo la Fase 0 (che fai inline per fissare `$TARGET`, modalità e tool disponibili), delega le fasi **indipendenti** a **subagenti paralleli** (tool `Agent`), ognuno restituisce **solo un riassunto strutturato**, non i log grezzi.

**Fan-out consigliato (un solo messaggio con più chiamate `Agent`):**

| Subagente | agentType | Copre | Restituisce |
|---|---|---|---|
| **Header & TLS** | `general-purpose` | Fase 1a–1c | tabella finding header + verdetto TLS + 3 problemi peggiori |
| **Esposizione** | `general-purpose` | Fase 2 + 3 | file/endpoint esposti, debug mode, CORS, check stack-specifici |
| **Superficie** | `Explore` | Fase 1d–1e | robots/sitemap, sottodomini CT, pannelli raggiunti |
| **Attiva** | `general-purpose` | Fase 4 (**solo se autorizzata**) | sintesi nmap/nikto/ZAP/nuclei, niente log grezzi |

**Regole obbligatorie nei prompt dei subagenti:** (a) rispettare la **modalità** (passiva ≠ payload d'attacco/scansioni); (b) usare lo stesso `$TARGET`/`$OUT`; (c) restituire **solo** una tabella `Vulnerabilità | Evidenza | Gravità | Impatto | Fix`, niente output grezzi; (d) il subagente **Attiva** parte **solo** se il main gli passa il flag di autorizzazione confermata.

> **Audit molto grandi o ricorrenti:** valuta un **Workflow** (orchestrazione multi-agente con verifica adversariale dei finding), che però richiede opt-in esplicito ("usa un workflow"). Per il caso standard bastano i subagenti `Agent` paralleli.

---

## Report finale (sempre)

Scrivi `$OUT/report-$TS.md` (cioè `private/storage/vuln-audit/$TS/report-$TS.md` in un progetto Master, altrimenti il fallback locale scelto in Fase 0) con:

1. **Contesto & scope**: target, host/IP, **modalità usata** (passiva/attiva), **autorizzazione dichiarata**, data/ora, tool usati, cosa è stato escluso e perché.
2. **Riepilogo esecutivo**: conteggio finding per gravità (Critica/Alta/Media/Bassa/Info) + i 3 rischi principali in linguaggio non tecnico.
3. **Finding prioritizzati** in tabella, ordinati per gravità poi per facilità di sfruttamento:

   | # | Vulnerabilità | Gravità | Evidenza (richiesta/risposta concreta) | Impatto (scenario) | Fix | Stato |
   |---|---|---|---|---|---|---|

   dove **Stato** = `Confermata` oppure `Da verificare`.
4. **Mappatura OWASP**: associa i finding alle categorie OWASP Top 10 (2021) pertinenti.
5. **Quick-win** (fix ≤1h: aggiungere header, chiudere `.env`/`.git`, disattivare `APP_DEBUG`, flag cookie) vs **interventi strutturali** (CSP completa, refactor auth/access control, patch CVE).
6. **Cosa serve per approfondire**: es. autorizzazione alla modalità attiva, credenziali di test per IDOR, accesso staging.

Chiudi con un riepilogo di 5 righe in chat: quante falle e di che gravità, la più urgente, i 3 fix a maggior ritorno, e — se sei rimasto in passiva su un dominio autorizzato — proponi il passaggio alla modalità attiva.

> **Igiene:** non lasciare sul disco dell'utente dati sensibili esfiltrati dal target. Nel report cita l'evidenza minima necessaria (es. "prime righe di `.env` mostrano `DB_PASSWORD=...` — oscurato"), non l'intero contenuto.
