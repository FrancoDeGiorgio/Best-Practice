# Protocollo - Dipendenze fissate e aggiornamento delle librerie

**Versione:** 1.0  
**Data:** 1 ottobre 2026  
**Ambito:** prassi aziendale per i progetti che usano librerie di terze parti (Python, Node, Dart/Flutter) e immagini Docker, con una suite di test e un deploy lanciato da uno sviluppatore. Definisce che cosa si fissa, da dove si leggono le versioni giuste e come si aggiorna **una libreria alla volta**.  
**Sistema di riferimento:** pip, npm, pub, Docker/Docker Compose, Git. La prova pulita richiesta al §4 è quella di `CI_LOCALE_E_DOCKER_DI_TEST.md`.  
**Fuori ambito:** vedi §9.  
**Implementazione di riferimento (esempio validato):** Omni Car Backend - `backend/requirements.txt`, `backend/requirements.lock`, `backend/Dockerfile`, ADR-021. Questo file non è il recap di Omni Car: è la policy da applicare su qualsiasi progetto.

---

## 1. Scopo e quando si applica

Se le versioni delle librerie non sono scritte in un file, esistono **tante versioni diverse dello stesso software**: quella dell'ambiente di sviluppo, quella che una build nuova scaricherebbe oggi, quella che gira in produzione. Una suite verde dice qualcosa solo sulla prima. Il caso reale: Omni Car Backend, 1 ottobre 2026. Una build da zero ha scaricato SQLAlchemy 2.1.1 (il file chiedeva solo `>=`) e la suite ha dato 156 errori; leggendo l'immagine di produzione si è scoperto che anche lo sviluppo era su versioni diverse (due salti di versione maggiore). **La suite non era mai stata eseguita sulle versioni che girano in produzione.**

Il protocollo ha due obiettivi:

1. **Provare il codice sulle stesse librerie che andranno in produzione.**
2. **Cambiare le librerie in modo controllato**: un aggiornamento che rompe qualcosa deve essere scoperto sul PC, con la causa isolata a una sola libreria, **prima** di toccare il server.

Si applica a ogni progetto con dipendenze di terze parti.

---

## 2. Principi non negoziabili

1. **Quello che si prova è quello che gira.** Le versioni provate in sviluppo e nella corsa pulita sono le stesse della produzione.
2. **Ogni versione è scritta in un file versionato**: esatta per le dipendenze dirette, e completa (transitive comprese) in un file di vincoli o di lock.
3. **La fonte delle versioni è la produzione**, non lo sviluppo. Si leggono in sola lettura dall'immagine o dall'ambiente che gira davvero (§5).
4. **Un aggiornamento alla volta.** Un commit tocca una libreria, o un gruppo inseparabile dichiarato nel messaggio (§4.2).
5. **Prima i test, poi il server.** Nessun aggiornamento arriva in produzione senza una corsa pulita verde su quel commit.
6. **Se è rosso si torna indietro.** Non si correggono i test per far passare la nuova versione, a meno che il cambio di comportamento sia voluto e documentato.
7. **Niente aggiornamenti automatici non provati.** Nessuno strumento che alza o fonde versioni senza passare per il §4.
8. **Il lock non si rigenera per abitudine.** Si rigenera da produzione dopo un aggiornamento verde, mai per "sistemare" una build.

---

## 3. Che cosa si fissa

| Ambiente | Cosa fissare | Come si installa nel rilascio | Attenzione |
|---|---|---|---|
| Python | `requirements.txt` con `==` per le dirette + file di vincoli con **tutte** le versioni (da `pip freeze`) | `pip install -c <vincoli> -r requirements.txt`, anche per gli strumenti dello stadio di sviluppo | Il file di vincoli va letto da un ambiente Linux se il rilascio è Linux: i pacchetti nativi dipendono dalla piattaforma |
| Node | `package-lock.json` committato | `npm ci`, mai `npm install` nel rilascio | Un lock generato su un solo sistema operativo può non contenere i moduli nativi di un altro: si segnala, non si aggira disattivando i controlli o rigenerando il lock |
| Dart / Flutter (app) | `pubspec.lock` committato; versione di Flutter dichiarata (per esempio `.fvmrc`) | `flutter pub get` con il lock | Il progetto che non dichiara la versione di Flutter ha lo stesso problema di fondo |
| Immagini Docker | Tag preciso invece di `latest` | `FROM` e `image:` con versione | Un tag mobile (per esempio `3.11-slim`) è una deriva residua: si dichiara nei limiti del progetto, oppure si fissa il digest |

Il file dei vincoli porta in testa un commento con: da dove è stato letto, quando, e quale commit era in produzione.

---

## 4. Come si aggiorna una libreria

### 4.1 Procedura

1. **Scegliere una libreria e una versione di arrivo.** Leggere le note di rilascio e cercare i cambi incompatibili. Per un salto di versione maggiore, pianificare più tempo.
2. **Annotare il punto di ritorno**: commit e lock attuali. Il rollback è tornare a quel lock e ricostruire.
3. **Alzare la versione** nel file delle dirette e nel file di vincoli. Una libreria che ne trascina altre cambia anche quelle righe nel lock, e il commit lo dice.
4. **Ricostruire l'ambiente di sviluppo** e far girare la corsa veloce (la suite del `pre-push`).
5. **Lanciare la corsa pulita** su quel commit (`CI_LOCALE_E_DOCKER_DI_TEST.md`).
6. **Se è rossa**: tornare al punto di ritorno. Si può indagare la causa, ma l'aggiornamento non procede finché non è verde senza cambiare la sostanza dei test.
7. **Se è verde**: per le librerie che parlano con servizi esterni, fare la prova mirata del §4.3.
8. **Deploy.** Il timbro verde è quello del commit che si rilascia.
9. **Dopo il deploy** controllare che l'ambiente di produzione coincida con il lock (§5).

### 4.2 Gruppi inseparabili

Due librerie si aggiornano nello stesso commit solo se una non si installa o non funziona senza l'altra (per esempio un framework e la sua libreria di validazione, quando la versione di una impone quella dell'altra). Il messaggio di commit dice quali sono e perché non si possono separare. Non è un gruppo inseparabile "aggiorno tutto insieme perché sono tante".

### 4.3 Librerie che parlano con servizi esterni

I test mockano il confine verso i servizi esterni (provider di intelligenza artificiale, pagamenti, storage): è la scelta giusta per una suite stabile, ma significa che **un cambio di contratto di un SDK non emerge dalla suite**. Per queste librerie, dopo la corsa verde, serve una prova reale e mirata (per esempio una chiamata in un ambiente di prova o un percorso usato davvero in sviluppo) prima del deploy. L'esito si scrive nel messaggio di commit.

### 4.4 Che cosa scrivere nel commit

- libreria, versione di partenza e di arrivo (e le altre righe del lock cambiate);
- esito della corsa veloce e hash del commit con timbro `ok` della corsa pulita;
- eventuale prova reale del §4.3;
- eventuale gruppo inseparabile e perché.

### 4.5 Correzioni di sicurezza

Seguono gli stessi passi, ma passano in cima all'ordine del lavoro. Non esiste una via più breve che salta la corsa pulita.

---

## 5. Controllo della deriva

Dopo ogni deploy che cambia le librerie, e ogni volta che si sospetta una differenza, si confronta l'elenco installato in produzione con il file di vincoli.

- Python: `pip freeze` dentro l'immagine di produzione confrontato con il file di vincoli.
- Node: l'albero installato dal lock (`npm ls`) o il `package-lock.json` usato nella build.

Una differenza non si "sistema" in produzione a mano: si decide quale dei due ha ragione, si corregge il file e si riprova con la corsa pulita.

L'accesso al server per questo controllo rispetta le regole di accesso del progetto: sola lettura, richiesto e autorizzato per quello scopo.

---

## 6. Checklist

```
Fissare le versioni (una tantum, dal server)
- [ ] Versioni lette in sola lettura dall'ambiente di produzione
- [ ] File delle dirette con versioni esatte
- [ ] File di vincoli / lock completo, con commento su origine e data
- [ ] Il build installa con il vincolo (pip -c / npm ci / pub get con lock)
- [ ] Gli strumenti dello stadio di sviluppo passano dallo stesso vincolo
- [ ] Il file dei vincoli entra nell'immagine (non escluso da .dockerignore)
- [ ] Ambiente di sviluppo ricostruito: stesse versioni, suite verde
- [ ] Corsa pulita verde sulle versioni di produzione

Aggiornare una libreria
- [ ] Una sola libreria (o gruppo inseparabile motivato)
- [ ] Note di rilascio lette, cambi incompatibili noti
- [ ] Punto di ritorno annotato
- [ ] Corsa veloce verde
- [ ] Corsa pulita verde sul commit che si rilascia
- [ ] Prova reale se la libreria parla con un servizio esterno
- [ ] Messaggio di commit completo (§4.4)
- [ ] Dopo il deploy: produzione coincide con il lock
```

---

## 7. Esempio di riferimento: Omni Car Backend

| Elemento | Valore |
|---|---|
| File delle dirette | `backend/requirements.txt`, versioni `==` identiche all'immagine di produzione letta il 01/10/2026 (build del 29/09/2026, commit `3bd4c5f`) |
| File di vincoli | `backend/requirements.lock`, `pip freeze` completo (126 righe di versioni), commento in testa con origine e data |
| Installazione | `Dockerfile`: `pip install -c requirements.lock -r requirements.txt`; strumenti di sviluppo (`watchdog`, `ipdb`) con lo stesso `-c`; `.dockerignore` lascia entrare `requirements.lock` |
| Differenze trovate (sviluppo contro produzione) | `openai` 2.53.0 contro 3.17.0; `google-cloud-aiplatform` 1.163.0 contro 2.1.3; `google-genai` 2.17.0 contro 2.24.0; SQLAlchemy 2.0.51 contro 2.0.54; `pydantic` 2.13.4 contro 2.13.5 |
| Che cosa ha fatto emergere | Build da zero con SQLAlchemy 2.1.1 e URL sincrono senza driver: 156 errori |
| Risultato dopo il blocco | Corsa pulita con le versioni di produzione: due test rossi che non dipendevano dalle librerie (versione dello schema API letta dal `.env`, lock reale su un database presente solo in sviluppo), corretti nello stesso lavoro |
| Decisione | ADR-021 |

La lezione generale: i due test rossi sono comparsi solo perché la corsa è stata fatta su un ambiente pulito con le versioni giuste; una suite che gira sempre sullo stesso ambiente caldo non li vede.

---

## 8. Limiti dichiarati

1. **Un blocco delle versioni non è un controllo di sicurezza.** Fissare impedisce le sorprese, non rimuove le falle delle versioni vecchie: serve decidere una cadenza con cui si cercano aggiornamenti e correzioni di sicurezza, e scriverla nel progetto.
2. **Costa lavoro.** Ogni aggiornamento richiede una corsa pulita e la lettura delle note. È il prezzo della prevedibilità.
3. **I due file possono divergere.** File delle dirette e lock vanno aggiornati insieme; un controllo automatico non è previsto da questo protocollo.
4. **I tag mobili delle immagini base restano una deriva** finché non si fissa anche il tag preciso o il digest.
5. **La suite non vede i cambi di contratto dei servizi esterni** (§4.3): la prova reale è manuale.
6. **Il lock descrive una piattaforma.** Un lock letto da Linux non è valido per ricostruire un ambiente Windows e viceversa, dove ci sono pacchetti nativi.

---

## 9. Fuori scope

Questo protocollo **non** copre:

- scansione automatica delle vulnerabilità o bot che propongono aggiornamenti (richiedono una CI che li provi da zero);
- la scelta di quali librerie usare;
- gli aggiornamenti del sistema operativo del server, di Docker o di PostgreSQL.

---

## 10. Riferimenti interni

- Visione d'insieme (livelli, suite completa, come attivarli in un progetto): `Test/IMPIANTO_DI_TEST_DEI_PROGETTI.md`
- Protocollo correlato: `Test/CI_LOCALE_E_DOCKER_DI_TEST.md` (la corsa pulita richiesta al §4.1)
- Decisione di progetto: ADR-021 «Dipendenze fissate alle versioni di produzione e aggiornamenti uno alla volta» (Omni Car Backend, `Documents/7_PROJECT_DOCUMENTATION/decisions/`)
- Evidenza del problema: report di audit dei test di Omni Car, §0.1 (v3.38)
