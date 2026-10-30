# Protocollo — CI locale e Docker di test (corsa pulita prima del deploy)

**Versione:** 1.2  
**Data:** 30 ottobre 2026  
**Ambito:** prassi aziendale per i progetti che hanno una suite di test e un deploy lanciato da uno sviluppatore, ma **nessun runner CI remoto** a disposizione. Definisce gli ambienti Docker di test e la **corsa pulita** locale che il deploy richiede.  
**Sistema di riferimento:** Docker (Desktop o Engine), Docker Compose, Git; script bash (Git Bash su Windows) e Node per i progetti web.  
**Fuori ambito:** vedi §9.  
**Implementazione di riferimento (esempio validato):** Omni Car — backend (`Omni Car Backend`) e web (`Omni Car Frontend`). Questo file non è il recap di Omni Car: è la policy da applicare su qualsiasi progetto.

---

## 1. Scopo e quando si applica

Un gate locale (hook `pre-push`, test su un Docker di test) gira **sempre su una macchina che qualcuno ha già scaldato**: container avviato giorni prima, cache piene, variabili d'ambiente di una versione vecchia del compose. Una suite può essere verde solo per questo. Il caso reale è nel report di audit di Omni Car (sezione C.6.2): il gate risultava verde perché il container in esecuzione era stato avviato prima che una variabile esistesse; su un container pulito era rosso.

Questo protocollo definisce **come** ottenere, senza un runner remoto, almeno una corsa che parte da zero **prima di rilasciare**, in modo che tutti gli sviluppatori:

1. distinguano la corsa veloce quotidiana dalla corsa pulita pre-deploy;
2. costruiscano le corse pulite con gli stessi requisiti (nomi, pulizia, guardie);
3. sappiano che cosa il meccanismo garantisce e che cosa **no** (§8).

Si applica a ogni progetto con test automatici e uno script di deploy. Se il progetto ha una CI remota che parte da zero e blocca il merge, questo protocollo non serve: la CI remota prevale.

---

## 2. Principi non negoziabili

1. **Tre ambienti distinti per progetto**, mai sovrapposti (§3).
2. **La corsa pulita non parte mai da un ambiente esistente.** Costruisce tutto da una copia del commit, non dalla cartella di lavoro.
3. **Nome del progetto unico, con prefisso `<prodotto>-ci-`.** Il suffisso è l'hash (abbreviato) del commit provato. Nessun nome fisso, nessuna collisione con sviluppo, test veloce o produzione.
4. **Pulizia garantita.** Container, volumi, rete, immagine costruita e cartella temporanea vengono cancellati a fine corsa, **anche su errore e su Ctrl+C**: `trap` in bash, `finally` più gestore `SIGINT` in Node.
5. **Guardia sui nomi.** Prima di ogni cancellazione si verifica che il nome del progetto abbia il prefisso `<prodotto>-ci-`. Se è uno degli ambienti protetti (sviluppo, test veloce, produzione) lo script esce con errore. La cancellazione dei volumi (`down -v`) è ammessa **solo** su progetti `<prodotto>-ci-*`.
6. **Il Docker di test veloce si spegne da solo** se lo ha acceso il hook; se era già acceso, resta com'era. Mai `down -v` su di esso.
7. **Immagini pubbliche di base in cache, tutto il resto cancellato.** Le immagini ufficiali (database, cache, runtime del linguaggio) restano per non riscaricarle ogni volta; l'immagine costruita dalla corsa si elimina.
8. **Segreti fittizi.** La corsa usa valori scritti in chiaro con `ci-only` nel nome. Nessuna credenziale reale, nessun `.env` copiato.
9. **Il timbro attesta una corsa, non un'intenzione.** Vale per l'hash per cui è stato scritto. Si può riusare su un commit successivo solo se l'esito è `ok` e ogni path diverso sta fuori da ciò che la corsa pulita costruisce e prova; `fail` e `partial` non si riusano (§5).
10. **Il deploy chiede la corsa e accetta una sola scappatoia scritta:** `SENZA-CI` (§6).

---

## 3. I tre ambienti

| Ambiente | Quando | Stato | Chi lo accende / spegne |
|---|---|---|---|
| **Docker di sviluppo** | Lavoro quotidiano, hot reload | Persistente (volumi di dati) | Lo sviluppatore |
| **Docker di test veloce** | Hook `pre-push`, corsa rapida sul codice del disco | Può essere già caldo; dati effimeri | Il hook lo accende se serve e lo **rispegne se lo ha acceso lui**, sia con esito verde sia con esito rosso |
| **Corsa pulita usa e getta** | Prima del deploy | Nasce e muore nella stessa corsa | Lo script della corsa, con pulizia automatica |

Il test veloce resta veloce proprio perché parte caldo: non lo si trasforma in corsa pulita. La corsa pulita è un **secondo** passaggio, più lento, prima del rilascio.

---

## 4. Requisiti della corsa pulita

### 4.1 Cosa deve fare, in ordine

1. Identificare il commit (`HEAD`) e avvisare se l'albero di lavoro è sporco: le modifiche non committate **non** entrano nella corsa.
2. Calcolare il nome del progetto `<prodotto>-ci-<hash abbreviato>` e superare la guardia (§2.5).
3. Estrarre una **copia del commit** in una cartella temporanea (`git archive`), non copiare la cartella di lavoro.
4. Registrare la pulizia (`trap` / `finally`) **prima** di creare qualsiasi risorsa.
5. Costruire da zero (`build --no-cache` o container nuovo), avviare le dipendenze, eseguire **tutti** i controlli del gate del progetto.
6. Scrivere il timbro (§5).
7. Pulire tutto e stampare un riepilogo: esito, commit, durata.

### 4.2 Cosa non deve fare

- Pubblicare porte sull'host: la corsa non deve poter entrare in conflitto con lo sviluppo.
- Riusare volumi, reti o immagini di altri progetti.
- Montare la cartella di lavoro al posto della copia del commit.
- Scrivere `ok` nel timbro se ha eseguito solo una parte dei controlli (usare `partial`).
- Aggirare un arresto d'ambiente (errore TLS, modulo nativo mancante) disattivando i controlli di sicurezza o rigenerando i lock: si ferma e si riporta.

### 4.3 Checklist prima di dichiarare una corsa «fatta»

```
- [ ] Sintassi script verificata (bash -n / node --check / parser PowerShell)
- [ ] Guardia sul prefisso presente all'inizio e dentro la funzione di pulizia
- [ ] Nessuna porta pubblicata sull'host
- [ ] Corsa intera: timbro ok
- [ ] Corsa interrotta con Ctrl+C: pulizia fatta
- [ ] Corsa rossa: timbro fail (o partial), pulizia fatta, codice di uscita non zero
- [ ] Dopo ogni prova: nessun container, volume, rete o immagine con il prefisso <prodotto>-ci-
- [ ] Cartella temporanea inesistente
- [ ] Verifica del timbro: 0 con l'hash giusto; 0 anche con un hash diverso se il diff tocca solo path esclusi dalla corsa; 1 se l'altro hash cambia ciò che la corsa costruisce, o se l'esito non è `ok`
```

---

## 5. Il timbro

Ogni corsa scrive un file di una riga nella cartella git del repository (`.git/ci-local-result`), quindi mai versionato e specifico della macchina. Contiene almeno: hash del commit, istante (epoch) ed esito.

| Esito | Significato |
|---|---|
| `ok` | Tutti i controlli del gate eseguiti e passati |
| `fail` | Almeno un controllo rosso |
| `partial` | Corsa ridotta a mano (sottoinsieme di test): **non vale** come via libera al deploy |

Lo script di verifica confronta il timbro con il **commit che verrà davvero rilasciato** ed esce con codice distinto:

| Codice | Significato |
|---|---|
| 0 | Timbro `ok` per quel commit, oppure `ok` su un commit precedente il cui diff verso il bersaglio sta tutto fuori da ciò che la corsa costruisce e prova |
| 1 | Timbro assente, `fail`, `partial`, o riferito a un commit che cambia ciò che la corsa costruisce (il messaggio indica il motivo e i due hash) |
| 3 | Solo per i progetti che rilasciano dal disco: albero sporco, perché il deploy costruirebbe codice mai committato e mai provato |

**Quale commit confrontare** dipende da come il deploy ottiene il codice:

- il server fa `git pull` da un ramo remoto → si confronta con **quel ramo remoto** (dopo `git fetch`), non con `HEAD`; se `HEAD` è diverso la corsa non prova ciò che verrà rilasciato e il deploy si ferma finché non si allineano;
- il deploy costruisce dal disco locale → si confronta con `HEAD` e si pretende l'albero pulito.

---

## 6. Integrazione nel deploy

Il controllo va **prima di qualsiasi accesso al server** (nessuna connessione remota prima di sapere se si può procedere) e prima della conferma finale di produzione, che resta com'è.

```
verifica timbro per il commit da rilasciare
├── verde                      → deploy come oggi
├── mancante / rosso / codice diverso → «Lanciare ora la corsa pulita? (s/n)»
│     ├── s → corsa; pulizia; verde = si procede (con nuova verifica), rosso = deploy annullato
│     └── n → scrivere SENZA-CI
│              ├── testo esatto → si procede, con l'avviso che la corsa è stata saltata
│              └── altro        → deploy annullato
└── albero sporco (codice 3)   → nessuna proposta di corsa (non servirebbe): solo SENZA-CI oppure annullato
```

Regole:

- Il testo da scrivere è esattamente `SENZA-CI`. Qualunque altra risposta annulla il deploy.
- Le modalità di deploy che **non ricostruiscono il codice** (per esempio un semplice restart dei container) saltano il controllo, perché non rilasciano codice nuovo.
- `SENZA-CI` è una scelta consapevole dello sviluppatore, non un'impostazione: non va resa permanente (niente variabili d'ambiente o flag che la attivino in automatico).

---

## 7. Esempio di riferimento: Omni Car

| Elemento | Backend | Web |
|---|---|---|
| Docker di test veloce | `docker-compose.test.yml`, progetto `omni-car-test`, acceso e rispento dal hook `.githooks/pre-push` | Gate esistente (`lint`, `typecheck`, `test:run`) |
| Compose / script della corsa | `docker-compose.ci.yml` + `scripts/ci_local.sh` | `scripts/ci-local.mjs` (`npm run ci:local`) |
| Prefisso | progetto Compose `omnicar-ci-<hash 12>` | container `omnicar-web-ci-<hash 12>` |
| Come si isola | Copia del commit con `git archive` montata nel container dei test; PostgreSQL e Redis nuovi, solo in rete interna | Container `node:22-bookworm-slim` usa e getta; sorgente del commit copiato dentro con `docker cp` |
| Controlli | Intera suite pytest del gate | `encoding:check`, `lint`, `typecheck`, `test:run`, `openapi:check`, `build:prod` |
| Timbro | `.git/ci-local-result`: hash, epoch, `ok` / `fail` / `partial` | `.git/ci-local-result`: hash, epoch, `ok` / `fail`, hash del backend da cui è stato preso lo schema |
| Verifica | `scripts/ci_local_verify.sh` — confronto con `origin/main`; un `ok` precedente resta valido se il diff è solo `Documents/` e `.cursor/` (`scripts/ci_paths.sh`) | `npm run ci:verify` — confronto con `HEAD`, albero pulito |
| Deploy | `deploy_omnicar.sh`, modalità 2, 3 e 4 (la 1, restart, salta il controllo) | `deploy.ps1` e `deploy.sh`, prima del controllo SSH |
| Immagini pubbliche in cache | `postgres:15-alpine`, `redis:7-alpine`, `python:3.11-slim` | `node:22-bookworm-slim` |

Il web legge lo schema API dal **commit** del backend (non dal file sul disco), così la corsa non dipende da modifiche non committate di un altro repository.

Un nuovo progetto replica la stessa struttura con i propri nomi: stesso schema di prefisso, guardia, pulizia, timbro, verifica e parola `SENZA-CI`.

---

## 8. Limiti dichiarati

Questo meccanismo **non è una CI**. Chi lo adotta deve saperlo e dirlo:

1. **Non sostituisce un divieto di merge sul remoto.** Nessuno impedisce a un commit non provato di arrivare sul ramo principale: il controllo scatta solo quando qualcuno lancia il deploy.
2. **Si può saltare.** Con `SENZA-CI` il deploy prosegue. È un attrito deliberato e visibile, non un blocco; manca ogni traccia centrale di chi lo ha usato.
3. **Il timbro è locale.** Vale per la macchina e il repository in cui è stato scritto; non è condiviso fra sviluppatori e non è un artefatto verificabile da terzi.
4. **Non è identico alla produzione.** Prova il codice su dipendenze pulite, non sul server né con i dati reali. Le versioni delle librerie sono quelle di produzione solo se il progetto le ha fissate: senza, la corsa installa le ultime versioni disponibili e può risultare rossa (o verde) per motivi che non riguardano il codice. Vedi `DIPENDENZE_E_AGGIORNAMENTO_LIBRERIE.md`.
5. **Le immagini pubbliche di base restano in cache** e non vengono ricostruite: un'immagine base cambiata a monte non emerge finché non la si riscarica.
6. **Dipende da Docker locale.** Senza Docker funzionante la corsa non parte; il rimedio non è saltare la pulizia ma sistemare l'ambiente.

Se e quando sarà disponibile una CI remota (GitHub Actions, GitLab o equivalente), diventa la fonte primaria: la corsa locale resta come verifica rapida pre-deploy.

---

## 9. Fuori scope

Questo protocollo **non** copre:

- Una CI remota o un divieto di merge (richiedono un runner e una piattaforma che oggi non ci sono).
- Ambienti di test condivisi e sempre accesi fra più prodotti: accumulano stato e mescolano i prodotti (motivazione in ADR-020 di Omni Car).
- Test su dispositivi o app mobile (la corsa pulita copre backend e web).
- Prove di carico, di sicurezza o end-to-end contro la produzione.
- Il contenuto dei test: quali controlli compongano il gate lo decide ogni progetto; il protocollo dice solo che la corsa pulita li esegue **tutti**.

---

## 10. Riferimenti interni

- Visione d'insieme (livelli, suite completa, come attivarli in un progetto): `Test/IMPIANTO_DI_TEST_DEI_PROGETTI.md`
- Protocollo correlato: `Test/DIPENDENZE_E_AGGIORNAMENTO_LIBRERIE.md` (versioni fissate e aggiornamento di una libreria alla volta; usa questa corsa come prova)
- Protocollo correlato: `Server/BACKUP_NOTTURNO.md` (stesso impianto: principi, checklist, limiti)
- Codice da copiare: repository Omni Car Backend → `scripts/ci_local.sh`, `scripts/ci_local_verify.sh`, `scripts/ci_paths.sh`, `docker-compose.ci.yml`; repository Omni Car Frontend → `scripts/ci-local.mjs`, `scripts/ci-local-verify.mjs`
- Decisione di progetto: ADR-020 «Corsa pulita locale prima del deploy» (Omni Car Backend, `Documents/7_PROJECT_DOCUMENTATION/decisions/`)
- Evidenza del problema: report di audit dei test di Omni Car, sezione C.6.2
