# Protocollo - Impianto di test dei progetti (pre-commit, pre-push, pre-deploy, suite complete)

**Versione:** 1.0  
**Data:** 1 ottobre 2026  
**Ambito:** prassi aziendale per dotare un progetto software, nuovo o esistente, di un impianto di test completo: controlli rapidi sul commit, suite sul push, corsa pulita prima del deploy, suite con i tipi di test che servono davvero e prove manuali dove non si può automatizzare. Questo file è la **mappa**: dice quali livelli esistono, in che ordine attivarli e che cosa deve contenere la suite. I dettagli dei due livelli più delicati stanno nei protocolli collegati (§10).  
**Sistema di riferimento:** Git con `core.hooksPath`, Docker/Docker Compose, shell POSIX per gli hook (su Windows la shell distribuita con Git), Python/pytest e Node/Vitest come stack già validati.  
**Fuori ambito:** vedi §9.  
**Implementazione di riferimento (esempio validato):** Omni Car - backend (`Omni Car Backend`) e web (`Omni Car Frontend`). Questo file non è il recap di Omni Car: è la policy da applicare su qualsiasi progetto.

---

## 1. Scopo e quando si applica

Un progetto senza impianto di test fa fatica a distinguere un cambiamento sicuro da uno rischioso. Un progetto con un impianto fatto male è peggio: dà la sensazione di essere protetto. In Omni Car, nel settembre 2026, sono emersi cinque gruppi di test verdi che non verificavano niente, un gate locale rosso su un ambiente pulito per ventiquattro ore senza che nulla fosse cambiato nel codice, e una suite mai eseguita sulle versioni delle librerie che girano in produzione. Tutte e tre le cose avevano la stessa radice: **un controllo provato solo sulla macchina in cui era stato costruito.**

Questo protocollo serve a:

1. avere **gli stessi livelli di controllo** su ogni software che gestiamo, con gli stessi nomi e le stesse regole;
2. attivarli **in un ordine sensato**, dal più economico al più completo (§4);
3. scrivere una suite che **protegge davvero** (§3) e non solo che passa;
4. sapere che cosa l'impianto garantisce e che cosa **no** (§8).

Si applica a ogni progetto con codice proprio e un deploy lanciato da uno sviluppatore. Se il progetto ha una CI remota che parte da zero e blocca il merge, la CI prevale sui livelli 1 e 2 e questo protocollo resta come guida alla suite (§3) e ai controlli manuali.

---

## 2. Principi non negoziabili

1. **Un test conta solo se si sa indicare la modifica del codice di produzione che lo fa diventare rosso.** Se non si riesce a produrre quella modifica, il test non protegge niente. Per le aree di sicurezza la prova si documenta.
2. **Una suite verde non è la prova che i test funzionino.** Test che non girano non proteggono, e test che girano senza asserire niente nemmeno. Si cerca di proposito, non per caso.
3. **Un presidio non è collaudato finché non è stato provato su un ambiente pulito.** Vale per gli hook, per la coverage, per gli script di deploy: la prova sulla macchina dove sono stati scritti non basta.
4. **Il controllo bloccante deve restare veloce.** Un gate lento insegna a usare `--no-verify`, e a quel punto non protegge più. La coverage strumentata e le prove lunghe non vanno nel gate.
5. **Un controllo che non può partire blocca, non salta.** Un hook che salta i test quando l'ambiente non è pronto dà falsa sicurezza.
6. **Si mocka al confine, non i dettagli interni.** Servizi esterni, rete, orologio, filesystem: sì. Nomi di funzioni interne: no. E **mai ciò che il test dichiara di verificare**.
7. **I test sono deterministici e non dipendono dalla macchina.** Niente valori letti dal `.env` locale, niente host del solo ambiente di sviluppo, niente ordine dei test.
8. **La suite di default non tocca rete né credenziali reali.** I test che lo richiedono hanno un marcatore e sono esclusi dal gate.
9. **Le versioni delle librerie sono scritte e coincidono con la produzione.** Vedi `DIPENDENZE_E_AGGIORNAMENTO_LIBRERIE.md`.
10. **Ogni bypass è consapevole e visibile** (`--no-verify`, `SENZA-CI`): un'eccezione deliberata, mai un'impostazione.
11. **Si dichiara quello che non c'è.** Niente CI remota, un gate aggirabile, una copertura bassa: sta scritto, non si lascia indovinare.

---

## 3. I livelli dell'impianto

| Livello | Quando gira | Che cosa fa | Tempo (Omni Car) | Bypass | Dettaglio |
|---|---|---|---|---|---|
| **0 - pre-commit** | A ogni commit | Controlli rapidi sui soli file in commit (lint mirato) | Pochi secondi | `git commit --no-verify` | §5.1 |
| **1 - pre-push** | A ogni push | Suite completa su un Docker di test; blocca il push se rossa | Circa due minuti (backend), pochi secondi di test + build (web) | `git push --no-verify` | §5.2 |
| **2 - pre-deploy** | Prima di un deploy che ricostruisce il codice | Corsa pulita usa e getta su una copia del commit, timbro letto dal deploy | Circa sei minuti (backend) | `SENZA-CI` scritto a mano | `CI_LOCALE_E_DOCKER_DI_TEST.md` |
| **3 - pre-consegna** | A ogni rilascio verso gli utenti | Checklist manuale firmata per ciò che non si automatizza (camera, device, usabilità) | Sessione di una persona | - | §6 |
| **4 - CI remota** | A ogni merge | Parte da zero e blocca il merge | - | - | Quando disponibile, prevale su 1 e 2 |

In parallelo ai livelli, **le dipendenze sono fissate** e si aggiornano una alla volta (`DIPENDENZE_E_AGGIORNAMENTO_LIBRERIE.md`).

Perché due corse (1 e 2) e non una: la corsa veloce parte da un ambiente già caldo e per questo è veloce; la corsa pulita parte sempre da zero e per questo è lenta. Ciascuna copre un rischio che l'altra non vede.

---

## 4. Che cosa deve contenere una suite completa

Lo scopo non è la percentuale di coverage ma coprire **i punti in cui un errore costa di più**. Per un'applicazione web multi-utente o multi-tenant con API, questa è la lista da cui partire; ogni progetto toglie ciò che non ha e aggiunge ciò che è suo.

### 4.1 Fondamenta (da avere prima di tutto il resto)

1. **Database reale, non un sostituto.** La suite gira sullo stesso motore della produzione (nel nostro caso PostgreSQL), isolato dagli altri ambienti. Una suite su un database finto non vede vincoli, tipi, indici e transazioni veri.
2. **Test sulle migrazioni.** Una sola testa; il ciclo di salita e discesa fino alla prima revisione funziona; la generazione automatica dà un confronto vuoto (il modello e le migrazioni coincidono).
3. **Test di enumerazione.** Test che passano in rassegna **tutti** gli endpoint o i metodi e diventano rossi da soli se il codice nuovo dimentica la protezione: ogni endpoint che modifica dati ha la protezione CSRF; ogni rotta riservata alla piattaforma richiede il ruolo; ogni query su tabelle con isolamento per tenant filtra per tenant; ogni endpoint che costa (chiamate AI) ha il limite di richieste. Non si aggiunge un'esenzione per far passare il push: si corregge il codice.
4. **Isolamento tra utenti o tenant, sul comportamento HTTP.** Per ogni endpoint nuovo con dati di un tenant, la terna: il proprietario ottiene 200; l'estraneo ottiene 404 (mai 403, per non rivelare l'esistenza); l'estraneo sulla **propria** risorsa equivalente ottiene 200. Il terzo caso non è decorativo: senza di lui un endpoint rotto che risponde sempre 404 sembra protetto.
5. **Sicurezza con la catena reale.** Autenticazione, sessioni, token, CSRF e permessi si provano con la catena vera, mai sostituendola con un override. Un test che si chiama "CSRF" e disattiva il CSRF è un placebo.

### 4.2 Contratto verso i client

- **Snapshot dell'API** (per esempio lo schema OpenAPI generato in processo e confrontato con un file versionato). Si aggiorna solo in modo esplicito. I campi che dipendono dall'ambiente (indirizzi del server, versione dell'applicazione) si tolgono dal confronto.
- **Catalogo di ciò che i client usano davvero** (web, mobile): nessun client chiama un percorso che non esiste, nessun campo usato manca nello schema. Un cambio di API che rompe un client diventa rosso prima del rilascio.
- Il client web verifica che i **tipi generati** dallo snapshot coincidano con quelli committati.

### 4.3 Denaro, effetti collaterali e operazioni irreversibili

- **Ledger e addebiti**: ogni percorso che produce qualcosa di fatturabile è agganciato al registro (test di enumerazione); idempotenza dimostrata; zero addebito se l'operazione fallisce.
- **Provider esterni** (intelligenza artificiale, pagamenti): ritentativi, timeout, risposte vuote o bloccate, errori del fornitore; sempre con il fornitore simulato.
- **Script operativi** (backup, migrazione di file, pulizie): provati con i comandi esterni finti; un test fa vedere che lo script **non cancella** quando non deve e che esce con errore se una condizione manca.

### 4.4 Pubblicazione e storage

- Distinguere accesso pubblico (indirizzo diretto) da accesso privato (passa dall'API autenticata); percorsi ostili (`..`, byte nullo) rifiutati; senza credenziali lo storage non parte.
- I test che scrivono sullo storage reale hanno un marcatore (`spaces_storage`) e sono esclusi dalla suite di default.

### 4.5 Web

Ordine di controllo dell'hook web: codifica UTF-8, lint, typecheck, test, tipi dell'API, build di produzione. I test girano in jsdom con un finto server di rete (Testing Library, MSW), senza browser. Gli errori di rendering di una pagina caricata in ritardo hanno un confine di errore con un test.

### 4.6 Ogni correzione di bug

Un test che **fallisce prima** della correzione. Senza averlo visto rosso non si sa se si corregge la causa o un sintomo.

### 4.7 Quando un test non serve

Refactor a comportamento invariato (basta che la suite resti verde), log, documentazione, rinomine, commenti, formattazione.

### 4.8 Marcatori

Si dichiarano e **si applicano**. Un marcatore dichiarato e non usato (`unit`, `slow`) è una promessa falsa. Utili: `security` (esegue solo i test di rifiuto) e un marcatore per i test che toccano servizi reali, escluso di default.

---

## 5. Come si attiva in un progetto

Ordine consigliato, dal più economico al più completo. Ogni passo è verificato prima di passare al successivo.

### 5.1 Passo 1 - Hook versionati e pre-commit

1. Creare `.githooks/` nel repository e committarlo. Una volta per macchina e per repository: `git config core.hooksPath .githooks` (gli hook in `.git/hooks/` non sono versionati). Va scritto nel README del progetto.
2. Il **pre-commit** fa solo cose da pochi secondi sui **file in commit** (`git diff --cached --name-only --diff-filter=ACM`): lint mirato (nel backend Omni Car, `ruff` con la sola regola degli import inutilizzati). Controlli più lunghi, come la codifica dell'intero repository, vanno nel pre-push.
3. Se lo strumento non è disponibile, il pre-commit **avvisa e non blocca**: è un aiuto, non un gate. (Il pre-push invece blocca: §5.2.)
4. Bypass: `git commit --no-verify`.

### 5.2 Passo 2 - Docker di test e pre-push

Il **Docker di test veloce** ha solo i servizi che servono ai test (nel backend: un PostgreSQL e un Redis di test), progetto Compose separato dallo sviluppo, con dati effimeri. La suite gira nel container di sviluppo già in esecuzione, che usa quei servizi.

Requisiti del pre-push, tutti imparati da un errore:

1. **Blocca se l'ambiente non è pronto** (container dell'applicazione non in esecuzione, `node_modules` assente): messaggio chiaro e uscita con errore. Mai saltare.
2. **Accende lo stack di test se serve e lo spegne alla fine** (`down`, mai `-v`), con esito verde o rosso, se lo ha acceso lui. Se era già acceso, resta com'era. Spegnimento garantito con `trap` su uscita, interruzione e terminazione.
3. **Lettura dei riferimenti da `stdin`.** Si salta la verifica solo se git ha passato almeno una riga e **tutte** sono cancellazioni di branch. Un `stdin` vuoto **non** significa "niente da verificare": il pulsante Push di alcuni editor non passa l'elenco dei riferimenti, e trattarlo come vuoto faceva arrivare commit sul remoto senza suite.
4. **Più indirizzi di push** (per esempio due remoti): git lancia l'hook una volta per indirizzo. Si usa un timbro in `.git/` con l'identità del push (`HEAD` più gli hash elencati) e un'età massima breve (300 secondi): la seconda destinazione non rilancia la suite per lo stesso commit.
5. **Comando senza terminale**: `docker compose exec -T`, uscita compatta (`pytest -qq`, perché la configurazione ha già `-v`).
6. **Niente coverage nel gate.** Nel backend Omni Car la suite strumentata dura circa 218 secondi contro 80.
7. **I test che toccano servizi reali sono esclusi** con il marcatore (`-m "not spaces_storage"` nella configurazione).
8. Su Windows gli hook girano con la shell distribuita con Git; non con altre shell nel `PATH`.
9. Bypass: `git push --no-verify`, da trattare come eccezione.

Per il web il pre-push esegue in ordine: codifica, lint, typecheck, test, tipi dell'API, build di produzione, e verifica `node_modules`.

### 5.3 Passo 3 - La suite minima

Costruire la suite del §4.1 prima di allargarla. Un progetto che ha solo i test di enumerazione e di isolamento ha già una protezione maggiore di uno con mille test unitari che mockano tutto.

### 5.4 Passo 4 - Dipendenze fissate

Seguire `DIPENDENZE_E_AGGIORNAMENTO_LIBRERIE.md`: leggere le versioni dalla produzione, fissarle, installare con il vincolo, ricostruire lo sviluppo, verificare che la suite sia verde su quelle versioni.

### 5.5 Passo 5 - Corsa pulita e controllo nel deploy

Seguire `CI_LOCALE_E_DOCKER_DI_TEST.md`: progetto Compose o container usa e getta con prefisso `<prodotto>-ci-`, copia del commit con `git archive`, pulizia garantita, timbro, verifica, integrazione nel deploy con `SENZA-CI`.

### 5.6 Passo 6 - Controlli manuali

Per tutto ciò che non si automatizza (camera reale, formato nativo, rotazione, permessi, memoria sotto pressione, messaggi di errore, usabilità, casi limite di business), una **checklist firmabile**: ogni voce ha chi ha provato, data ed esito (`ok` / `ko` / `non eseguito`). Senza firma e data non è una checklist. Le due righe di intestazione del rilascio riportano commit ed esito delle corse pulite.

### 5.7 Passo 7 - Documentazione

- un documento di **strategia** che dice come si scrivono e si lanciano i test e quali comandi usare;
- un **report di stato** con misure datate, falle aperte e piano; i numeri si annotano lì, perché i file di coverage sono volatili;
- una **regola per gli agenti e gli sviluppatori** che risponde alla domanda "questa modifica richiede un test?";
- un **ADR** per ogni decisione che vincola il seguito (corsa pulita, librerie fissate).

### 5.8 Passo 8 - Collaudo dell'impianto

Prima di dichiarare l'impianto "fatto":

```
- [ ] Hook attivati su una macchina pulita (git config core.hooksPath .githooks)
- [ ] Pre-commit: ok con un file valido, blocca un file con errore, non blocca se manca lo strumento (avviso)
- [ ] Pre-push: con la suite verde il push passa; con un test volutamente rosso esce con codice 1
- [ ] Pre-push: container non in esecuzione -> blocca con messaggio
- [ ] Pre-push: stack di test acceso dall'hook viene spento; se era già acceso resta
- [ ] Pre-push: Ctrl+C a metà non lascia lo stack acceso
- [ ] Due remoti: la seconda destinazione riusa il timbro
- [ ] Stdin vuoto (push da un editor): non salta la verifica
- [ ] Suite eseguita da zero su un ambiente pulito (corsa pulita): verde
- [ ] Almeno un test per tipo del §4.1 visto rosso con la modifica del codice che lo fa rompere
- [ ] Un test di enumerazione provato togliendo una protezione: diventa rosso
- [ ] Checklist manuale compilata almeno una volta
```

---

## 6. Esempio di riferimento: Omni Car

| Elemento | Backend | Web |
|---|---|---|
| Pre-commit | `.githooks/pre-commit`: `ruff` (regola F401) sui `.py` in commit; avvisa e non blocca se `ruff` manca | - |
| Pre-push | `.githooks/pre-push`: avvia `docker-compose.test.yml` se serve e lo spegne; `pytest -qq --tb=line --disable-warnings` nel container `backend` | `.githooks/pre-push`: `encoding:check`, `lint`, `typecheck`, `test:run`, `openapi:check`, `build` |
| Docker di test | `omni-car-test`: PostgreSQL e Redis di test | - |
| Pre-deploy | `scripts/ci_local.sh`, `docker-compose.ci.yml`, `deploy_omnicar.sh` (modalità 2, 3, 4) | `scripts/ci-local.mjs`, `deploy.ps1`, `deploy.sh` |
| Dipendenze | `requirements.txt` con `==` + `requirements.lock` | `package-lock.json` con `npm ci` |
| Contratto | snapshot OpenAPI + catalogo dei client (`client_api_contract.json`) | tipi generati e `openapi:check` |
| Pre-consegna | `Doc_test/CHECKLIST_RILASCIO.md` (14 voci) | - |
| Misure (1 ottobre 2026) | 1.726 test passati, 14 saltati, 4 esclusi; corsa pulita verde | 84 test in 14 file |
| Documenti | `Documents/6_TESTING/TESTING_STRATEGY.md`, `REPORT_AUDIT_TESTING.md`, `22-testing-requirements.mdc`, ADR-020 e ADR-021 | `README.md` |

L'app Flutter non ha ancora l'impianto sul repository: il controllo previsto (`flutter analyze --fatal-infos && flutter test`) va chiesto allo sviluppatore dell'app.

---

## 7. Adattamenti per stack

| Stack | Hook e gate | Corsa pulita | Note |
|---|---|---|---|
| Python / FastAPI | `pytest` nel container; `ruff` sul commit | `git archive` + Compose usa e getta | Database reale; marcatori dichiarati e applicati (§4.8) |
| Node / React | encoding, lint, typecheck, Vitest, tipi API, build | container `node` usa e getta, `npm ci` | Lock generato su un solo sistema operativo può mancare i moduli nativi di un altro |
| Flutter | `flutter analyze --fatal-infos` e `flutter test` | da definire | Dichiarare la versione di Flutter (per esempio `.fvmrc`) |
| Progetto con un solo sviluppatore | Gli stessi livelli: il rischio non è la coordinazione ma la fretta | Il timbro serve comunque | Il bypass resta un'eccezione consapevole |

---

## 8. Limiti dichiarati

Questo impianto **non è una CI**.

1. **Il pre-push è aggirabile** (`--no-verify`) senza lasciare traccia, e **non protegge chi non lo attiva** (senza `core.hooksPath` per quello sviluppatore il gate non esiste). Un editor che spedisce il push senza passare dall'hook può non eseguire la suite.
2. **Gira sullo stato di lavoro locale**, non sul commit che raggiunge il remoto: con file non committati che fanno passare i test, il push può essere verde in locale e rotto per gli altri.
3. **Il pre-push non parte da zero.** Per questo esiste la corsa pulita; ma anche questa è facoltativa (`SENZA-CI`).
4. **Nessun divieto di merge**: un commit non provato può arrivare sul ramo principale; il controllo scatta solo al deploy.
5. **I timbri sono locali**: non condivisi, non verificabili da terzi.
6. **La coverage è un numero, non una prova.** Una percentuale bassa non significa scoperto e una media alta non significa verificato. I test strutturali (che ispezionano il sorgente) danno coverage quasi nulla ai moduli che proteggono.
7. **Le prove manuali dipendono da una persona**: sono verifiche di giudizio, non regressioni automatiche.

Se e quando sarà disponibile una CI remota, diventa la fonte primaria; gli hook restano come feedback rapido.

---

## 9. Fuori scope

Questo protocollo **non** copre:

- una CI remota o il divieto di merge (servono un runner e una piattaforma);
- test di carico, di penetrazione e prove end-to-end contro la produzione;
- test su dispositivi fisici in automatico (restano nella checklist manuale);
- la scelta degli strumenti di lint e di test di un singolo linguaggio;
- il contenuto dei test di dominio: ogni progetto decide cosa proteggere, il protocollo dice dove guardare.

---

## 10. Riferimenti interni

- Corsa pulita prima del deploy: `Test/CI_LOCALE_E_DOCKER_DI_TEST.md`
- Dipendenze fissate e aggiornamento di una libreria alla volta: `Test/DIPENDENZE_E_AGGIORNAMENTO_LIBRERIE.md`
- Protocollo correlato: `Server/BACKUP_NOTTURNO.md` (stesso impianto: principi, checklist, limiti)
- Esempio completo: repository Omni Car Backend (`.githooks/`, `docker-compose.test.yml`, `docker-compose.ci.yml`, `scripts/ci_local.sh`, `backend/app/tests/`) e Omni Car Frontend (`.githooks/`, `scripts/ci-local.mjs`)
- Decisioni di progetto: ADR-020 e ADR-021 (Omni Car Backend, `Documents/7_PROJECT_DOCUMENTATION/decisions/`)
- Evidenza dei problemi: report di audit dei test di Omni Car (`REPORT_AUDIT_TESTING.md`), sezioni 0.1, 0.3, C.6
