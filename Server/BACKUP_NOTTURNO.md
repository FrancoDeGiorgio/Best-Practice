# Protocollo — Backup notturno (stack self-hosted)

**Versione:** 1.0  
**Data:** 23 settembre 2026  
**Ambito:** prassi aziendale per backup notturni di applicazioni self-hosted con PostgreSQL in Docker, volumi Docker di dati persistenti e (se presenti) storage oggetti S3-compatibile.  
**Sistema di riferimento:** Linux (Ubuntu/Debian), Docker Compose, `rclone`, crontab di sistema, timezone `Europe/Rome`.  
**Fuori ambito:** vedi §10.  
**Implementazione di riferimento (esempio validato):** Omni Car Backend — copiare struttura e script da lì, sostituendo solo le variabili di progetto. Questo file non è il recap di Omni Car: è la policy da applicare su qualsiasi progetto.

---

## 1. Scopo e quando si applica

Questo protocollo definisce **come** allestire e operare i backup notturni su un server nuovo o su un progetto esistente da allineare, in modo che tutti gli sviluppatori:

1. controllino e interpretino i backup allo stesso modo;
2. creino stack di backup standard (stesse cartelle, stessi orari, stessa retention, stessi guardrail).

Si applica quando l’applicazione è **self-hosted** e persiste almeno un database PostgreSQL (tipicamente in container). Le fasi volume, storage oggetti e Google Drive si aggiungono **solo se** il progetto le ha davvero: verificare prima di assumerle.

| Fase | Applicabile se | Se manca |
|---|---|---|
| **A — DB** | sempre | — |
| **C — Volume** | esiste un volume Docker con dati persistenti da archiviare | salta e documenta il perché |
| **B — Storage live** | esiste un bucket oggetti in produzione | salta; il wrapper resta senza questo step |
| **D — Drive** | esiste una cartella Google Drive condivisa con il service account aziendale | salta |
| **Dashboard** | il progetto ha una console operativa dove esporre lo stato | solo file di stato su disco; non inventare API |

Il wrapper notturno chiama **solo** le fasi applicabili, nell’ordine indicato in §3.

---

## 2. Principi non negoziabili

1. **Separazione rigorosa dei contenuti.** Il backup DB contiene **solo** il dump (`db.dump` + `backup_info.txt`). Nessun tarball di asset nella cartella del database. Gli asset vivono nel ramo volume e/o storage.
2. **Un solo entry in crontab.** Alle **03:00 Europe/Rome** gira **un solo** wrapper. Se esiste il wrapper, non devono coesistere cron separati per le singole fasi.
3. **Step indipendenti.** Un fallimento di una fase viene loggato; le fasi successive **continuano**. Il wrapper non usa `set -e` in modo da abortire la sequenza.
4. **Retention DB fissa:** daily remoto 30 giorni; monthly remoto 12 copie; copie locali daily 7 giorni.
5. **Mensile:** copia il daily del **1° del mese**; se quel run fallisce o il monthly del mese manca, promuove la **prima daily andata a buon fine** del mese.
6. **Staffetta volume/storage:** ogni notte si scrive sullo **slot attivo**; nel giorno di staffetta (default domenica, `STAFFETTA_WEEKDAY=0`) si copia l’attivo sullo slot fermo. Volume: sempre 2 slot (`slot_a`, `slot_b`). Storage: 2 di default; 3 solo se configurato esplicitamente.
7. **Freno anti-crollo (50%):** se dimensione (volume) o conteggio/byte oggetti (storage) scende sotto il **50%** dell’ultima run ok → niente upload/sync e niente staffetta; lo stato registra il motivo. Non azzerare a mano uno stato che conteneva run riuscite.
8. **Credenziali granulari:** Read sul bucket live, Write solo sul bucket freddo. Vivono in `rclone config`, non negli env di backup (gli env contengono **nomi di remote**).
9. **Job applicativi pesanti dopo le 03:00.** Cleanup e batch dalle **04:00** in poi, a scaglioni; mai in sovrapposizione al wrapper.
10. **Drive = copia umana, non secondo vault.** Solo ultimo daily DB (niente monthly/storico). DB e volume: staging+swap (`incoming` → verifica → `latest`, vecchia `latest` → `previous`). Storage: un solo specchio con `sync --delete-after` (mai `--delete-before`). Sorgente Drive = dati già sul freddo con `status=ok` nello stato della fase; altrimenti quel ramo non si tocca.
11. **Backup manuali solo su disco locale** (`/var/backups/{project_slug}/manual/`). Non salgono sul bucket freddo.
12. **Niente mescolare ambienti.** Dev e prod hanno compose, env, bucket e path distinti. Un restore di prova si fa su stack **dev**, non sovrascrivendo la produzione «per prova».
13. **Assert prima di cancellare.** Ogni `delete` / `purge` / `moveto` remoto richiede un `assert_*_path` sullo stesso flusso.
14. **Niente secret nel repository.** Env su server `chmod 600`; script installati `chmod 750`; state JSON `chmod 644`.

---

## 3. Architettura

Un solo cron alle **03:00 Europe/Rome** esegue il wrapper, che chiama gli script in sequenza:

| Ordine | Fase | Cosa fa | Destinazione primaria | Perché esiste |
|---|---|---|---|---|
| 1 | **A — DB** | `pg_dump` del database applicativo | Bucket freddo `db_backups/auto/daily` (+ `monthly` se dovuto) + copia locale 7 gg | Recupero schema/dati; retention lunga e mensile |
| 2 | **C — Volume** | tar del volume Docker dati (esclusioni solo per path rigenerabili documentati) | Freddo `volume_backups/auto/slot_*` | Disaster recovery disco / file non in object storage |
| 3 | **B — Storage** | sync incrementale bucket live → slot attivo sul freddo | Freddo `space_backups/auto/slot_*` | Replica off-site degli asset live; staffetta per versioni “ferme” |
| 4 | **D — Drive** | copia umana dall’ultimo stato ok sul freddo | Cartella Drive del progetto | Accesso da browser per ops umane; **non** sostituisce il freddo |

Destinazioni a tre livelli:

| Livello | Ruolo |
|---|---|
| **Locale host** | Copie DB daily recenti (7 gg), area `volume_tmp`, backup **manuali**, file di stato JSON |
| **Bucket freddo** (condiviso tra progetti, prefisso `{project_slug}/`) | Vault primario: retention DB, slot volume, replica storage |
| **Google Drive** (opzionale) | Specchio umano dell’ultimo buono; retention e storico restano sul freddo |

Ogni fase scrive (anche in errore) un file di stato JSON su host. Se esiste una dashboard, i file si montano **read-only** nel container che li espone. Devono esistere come **file regolari prima** del primo `docker compose up` del servizio che li monta: altrimenti Docker crea una directory e il backup fallisce (`tee: Is a directory`).

---

## 4. Struttura obbligatoria delle cartelle

### 4.1 Bucket freddo

Il bucket freddo è **condiviso**; ogni progetto vive sotto `{project_slug}/`.

```
{cold_bucket}/
└── {project_slug}/
    ├── db_backups/auto/
    │   ├── daily/{backup_prefix}_YYYYMMDD_HHMMSS/{db.dump, backup_info.txt}
    │   └── monthly/{backup_prefix}_YYYYMMDD_HHMMSS/{db.dump, backup_info.txt}
    ├── volume_backups/auto/
    │   ├── state.json
    │   ├── slot_a/{volume.tar.gz, backup_info.txt}
    │   └── slot_b/{volume.tar.gz, backup_info.txt}
    └── space_backups/auto/
        ├── state.json
        ├── slot_a/…replica gerarchia bucket live…
        └── slot_b/…idem…
```

| Ramo | Retention / modello | Note |
|---|---|---|
| `db_backups/auto/daily` | oggetti > 30 giorni eliminati | naming `{backup_prefix}_YYYYMMDD_HHMMSS` |
| `db_backups/auto/monthly` | 12 cartelle più recenti | stessa naming |
| `volume_backups/auto/slot_*` | sovrascrittura per slot | 2 slot; `state.json` remoto |
| `space_backups/auto/slot_*` | sovrascrittura per slot | 2 default; `slot_c` solo se 3 slot; exclude prefissi non produttivi |

Path deprecati (non devono più esistere):

- `db_backups/auto/{prefix}_*/` flat (senza `daily/`);
- tarball di asset dentro la cartella del backup DB;
- backup manuali sul bucket freddo.

Il ramo storage **deve** escludere i prefissi non produttivi (sandbox, scratch, temporanei): l’elenco è valore di progetto; l’esclusione è obbligatoria. Preferire variabile env (es. `LIVE_EXCLUDE`); se lo script di riferimento ha l’exclude hardcoded, parametrizzarlo quando si adatta a un progetto nuovo.

### 4.2 Filesystem sull’host

```
/var/backups/{project_slug}/
├── auto/daily/{backup_prefix}_YYYYMMDD_HHMMSS/   # DB locali, 7 giorni
├── manual/                                        # solo emergenza, mai remoti
├── volume_tmp/                                    # area lavoro tar volume
├── db_backup_status.json                          # 644
├── volume_backup_status.json                      # 644
├── space_backup_status.json                       # 644 (se fase B)
└── gdrive_backup_status.json                      # 644 (se fase D)
```

Permessi tipici: directory progetto backup `700`; state `644`; env in `/root/.config/` `600`.

### 4.3 Google Drive (se fase D)

Drive è condiviso; ogni progetto ha una cartella dedicata (`root_folder_id` nel remote rclone).

```
{drive_root}/
├── db/incoming/     ├── db/latest/     ├── db/previous/
├── volume/incoming/ ├── volume/latest/ ├── volume/previous/
└── space/           # unico specchio incrementale
```

Niente monthly, niente 30 daily, niente secondo slot storage su Drive. Mai scrivere diretto su `latest` per DB/volume: sempre staging+swap. Mai `purge` di `latest` prima dell’upload verificato su `incoming`.

### 4.4 Script e env sul server

| Cosa | Path | Permessi |
|---|---|---|
| Script installati | `/usr/local/sbin/{project_slug}-*.sh` | `750` |
| Env backup | `/root/.config/{project_slug}-*.env` | `600` |
| Lock | `/var/run/{project_slug}-….lock` | gestiti da `flock` |
| Log wrapper | `/var/log/{project_slug}-nightly-backup.log` | append da cron |
| Chiave service account Drive (condivisa) | `/etc/omni-backup/gdrive.json` | `600`, mai in git |

Nomi script consigliati (da repository → installazione):

| Nel repo `scripts/` | Su server |
|---|---|
| `{slug}_backup_cron.sh` | `{project_slug}-backup.sh` |
| `{slug}_volume_backup_cron.sh` | `{project_slug}-volume-backup.sh` |
| `{slug}_space_backup_cron.sh` | `{project_slug}-space-backup.sh` |
| `{slug}_gdrive_backup_cron.sh` | `{project_slug}-gdrive-backup.sh` |
| `{slug}_nightly_backup.sh` | `{project_slug}-nightly-backup.sh` |

---

## 5. Checklist — server nuovo

Completare in ordine. Non inventare bucket, slug, nomi volume o ID cartella Drive: se non sono nel repository, chiederli.

### 5.0 Fase 0 — variabili di progetto

Compilare la tabella in §9 **prima** di scrivere codice o toccare il server. Decidere quali fasi sono applicabili.

### 5.1 Prerequisiti host

- [ ] Accesso admin/ops sul server secondo il protocollo utenti (`PROTOCOLLO_GESTIONE_UTENTI_SERVER.md`).
- [ ] Docker Compose di produzione operativo; volume e DB noti.
- [ ] `rclone` e `python3` installati (usati dagli script).
- [ ] Timezone host coerente con `Europe/Rome` per il crontab (o TZ esplicita se documentata sul progetto).
- [ ] Bucket freddo aziendale già esistente; chiavi granulari create (Read live / Write freddo).

### 5.2 Cartelle e file di stato

```bash
sudo mkdir -p /var/backups/{project_slug}/auto/daily \
              /var/backups/{project_slug}/manual \
              /var/backups/{project_slug}/volume_tmp
sudo chmod 700 /var/backups/{project_slug}

for f in db_backup_status.json volume_backup_status.json space_backup_status.json gdrive_backup_status.json; do
  # creare solo i file delle fasi applicabili
  sudo install -m 644 /dev/null "/var/backups/{project_slug}/$f"
  echo '{}' | sudo tee "/var/backups/{project_slug}/$f" > /dev/null
done
```

Se il progetto monta gli state nel container: creare i **file** **prima** di `docker compose up` / `--force-recreate`. Se esiste già una directory phantom: fermare e rimuovere il container, `sudo rm -rf` della directory, ricreare il file, ripulire residui nel volume di config se presenti, poi riavviare.

### 5.3 Env rclone (nomi variabili, non valori secret)

Creare in `/root/.config/` (chmod `600`) i file delle fasi applicabili. Contengono **nomi di remote e path**, non le chiavi.

**DB** — `{project_slug}-backup.env`:

```bash
RCLONE_REMOTE={rclone_remote_cold}
SPACES_BUCKET={cold_bucket}
REMOTE_PREFIX_DAILY={project_slug}/db_backups/auto/daily
REMOTE_PREFIX_MONTHLY={project_slug}/db_backups/auto/monthly
RETENTION_LOCAL_DAYS=7
RETENTION_REMOTE_DAYS=30
RETENTION_MONTHLY_COUNT=12
```

**Volume** — `{project_slug}-volume-backup.env`:

```bash
RCLONE_REMOTE={rclone_remote_cold}
SPACES_BUCKET={cold_bucket}
COLD_PREFIX={project_slug}/volume_backups/auto
VOLUME_NAME={volume_name}
BRAKE_THRESHOLD_PCT=50
STAFFETTA_WEEKDAY=0
```

**Storage** — `{project_slug}-space-backup.env`:

```bash
RCLONE_REMOTE_LIVE={rclone_remote_live_ro}
RCLONE_REMOTE_COLD={rclone_remote_cold}
LIVE_BUCKET={live_bucket}
COLD_BUCKET={cold_bucket}
COLD_PREFIX={project_slug}/space_backups/auto
LIVE_EXCLUDE={live_exclude}
SLOTS=2
BRAKE_THRESHOLD_PCT=50
STAFFETTA_WEEKDAY=0
```

**Drive** — `{project_slug}-gdrive-backup.env`:

```bash
RCLONE_REMOTE_COLD={rclone_remote_cold}
SPACES_BUCKET={cold_bucket}
REMOTE_PREFIX_DAILY={project_slug}/db_backups/auto/daily
VOLUME_COLD_PREFIX={project_slug}/volume_backups/auto
SPACE_COLD_PREFIX={project_slug}/space_backups/auto
RCLONE_REMOTE_DRIVE={rclone_remote_drive}
```

Dal `.env` applicazione gli script leggono **solo** i valori necessari (es. password DB) via `grep`, **mai** `source` dell’intero file. Path tipico: `/etc/{progetto}/…` (definirlo per progetto).

Config rclone sul remote freddo (chiavi senza `CreateBucket`):

```bash
rclone config update {rclone_remote_cold} no_check_bucket true
```

Gli script passano comunque `--s3-no-check-bucket`.

### 5.4 Service account Google Drive (se fase D)

- [ ] Chiave SA nel path condiviso `/etc/omni-backup/gdrive.json`, `chmod 600`, mai in git.
- [ ] Cartella Drive del progetto condivisa con il service account (Editor o Gestore dei contenuti) **prima** del test.
- [ ] Remote rclone `{project_slug}-gdrive` con `service_account_file` e `root_folder_id`.

### 5.5 Script nel repository e installazione

- [ ] Script + `*.env.example` nel repo sotto `scripts/` (una fase = un file + wrapper).
- [ ] Partire dall’implementazione di riferimento (Omni Car), adattare **solo** variabili/esclusioni di progetto.
- [ ] Ogni script passa lo standard di qualità §8.
- [ ] Installazione:

```bash
cd {server_project_path}
# dopo git pull del codice aggiornato:
sudo cp scripts/{slug}_backup_cron.sh        /usr/local/sbin/{project_slug}-backup.sh
sudo cp scripts/{slug}_volume_backup_cron.sh /usr/local/sbin/{project_slug}-volume-backup.sh
sudo cp scripts/{slug}_space_backup_cron.sh  /usr/local/sbin/{project_slug}-space-backup.sh
sudo cp scripts/{slug}_gdrive_backup_cron.sh /usr/local/sbin/{project_slug}-gdrive-backup.sh
sudo cp scripts/{slug}_nightly_backup.sh     /usr/local/sbin/{project_slug}-nightly-backup.sh
sudo chmod 750 /usr/local/sbin/{project_slug}-*.sh
```

Copiare solo le fasi applicabili. Il deploy Docker **non** aggiorna `/usr/local/sbin/`: dopo un `git pull` che tocca gli script, ripetere `cp` + `chmod`.

### 5.6 Crontab e job schedulati

```cron
0 3 * * * /usr/local/sbin/{project_slug}-nightly-backup.sh >> /var/log/{project_slug}-nightly-backup.log 2>&1
```

- [ ] Una sola riga attiva; rimuovere i cron backup precedenti nello stesso `crontab -e`.
- [ ] Verificare: `crontab -l` senza commenti/vuoti mostra solo il wrapper (e altri job non-backup legittimi).
- [ ] Spostare cleanup/batch applicativi alle 04:00+ (quali job esistano dipende dal progetto: leggerli, non assumerli).
- [ ] Se la config job è un JSON in container: patch con `docker compose … exec -T` (**`-T` obbligatorio**), restart del servizio, **rileggere** il file per conferma.

### 5.7 Primo run e verifica

Ordine:

1. Smoke DB: `sudo /usr/local/sbin/{project_slug}-backup.sh` → stato `status: ok`, cartella in `auto/daily/`.
2. Smoke volume (se applicabile).
3. Wrapper completo.
4. `rclone lsf` sul freddo: sotto `db_backups/auto/` solo `daily/` e `monthly/`; slot volume/storage presenti (il secondo slot compare dopo la prima staffetta).
5. Drive (se applicabile): dopo A/B/C ok, smoke DB+volume su Drive; la **prima** sync storage verso Drive (potenzialmente molti GB) si lancia **a mano fuori** dalla finestra 03:00.

Atteso nel log wrapper: un blocco `=== OK: … ===` per ogni fase attiva e la riga di chiusura. I NOTICE rclone tipo `mkdir … does nothing` su S3 e alcuni `Entry doesn't belong in directory` sono rumore noto: la verifica valida è il **contenuto elencato**, non l’assenza di NOTICE.

Upload DB fallito → **nessun prune** quella notte (comportamento richiesto).

---

## 6. Operatività quotidiana

### 6.1 Controllo «stanotte è andato?»

1. Leggere il log wrapper: `/var/log/{project_slug}-nightly-backup.log` (ultime righe / blocco della notte).
2. Leggere gli state JSON in `/var/backups/{project_slug}/` (`status`, `last_ok_at`, `error`, `blocked_reason`).
3. Se esiste dashboard: confrontare con gli endpoint/schede di stato (mount RO degli stessi file).
4. Opzionale: `rclone lsf` sulle cartelle `daily/` e sugli slot attivi.

### 6.2 Interpretare successo e fallimento

| Segnale | Significato |
|---|---|
| Wrapper: `=== OK: … ===` per ogni fase attiva | Step terminato con exit 0 |
| Wrapper: `WARN: … terminato con errore` | Quella fase fallita; le altre possono essere ok |
| State `status: ok` | Fase ok; Drive può copiare da quel ramo |
| State `blocked` / `blocked_reason` valorizzato | Freno 50%: niente upload/staffetta |
| State `partial` / rami Drive `skipped` | Drive: sorgente non ok o ramo saltato di proposito |
| Upload daily fallito | Nessun prune remoto/locale quella notte |

### 6.3 Se scatta il freno

1. **Non** forzare upload e **non** lanciare a mano la sync live→freddo se il vivo è sospetto di wipe.
2. Investigare la causa sul vivo (cancellazioni, bucket sbagliato, exclude errato, volume vuoto).
3. Per recovery asset: preferire lo **slot fermo** sul freddo, non riscrivere il vivo da uno slot incompleto.
4. Solo dopo che il vivo è di nuovo sano, lasciare che il cron successivo aggiorni lo stato ok.

### 6.4 Cosa non fare

- Cancellare a mano cartelle `daily/` / slot / `latest` «per fare spazio» senza assert e senza piano di restore.
- Azzerare uno state JSON che conteneva run riuscite (si perde il riferimento al freno fino al run successivo).
- Aggiungere cron separati per le fasi oltre al wrapper.
- Mischiare path/env/bucket di dev e prod.
- Usare `--delete-before` su Drive; fare `purge` di `latest` prima della verifica `incoming`.
- Caricare backup manuali sul bucket freddo.
- Trattare i NOTICE rclone `mkdir` come failure.
- `source` dell’intero `.env` applicazione negli script di backup.
- Sovrascrivere produzione per un restore di prova.

### 6.5 Backup manuale di emergenza

Se il progetto fornisce uno script manuale (esempio Omni Car: `backup_<slug>.sh` dal laptop via ops), scrive **solo** sotto `/var/backups/{project_slug}/manual/`. Non sostituisce il cron; non ha prune automatico sul freddo.

---

## 7. Restore (alto livello)

**Non esiste uno script di restore notturno standardizzato** nello stack di riferimento: il restore si ricava dai prodotti degli script di backup. Eseguire prove su **dev** (o ambiente isolato), mai «di prova» in sovrascrittura su prod senza approvazione esplicita.

### 7.1 Ordine consigliato

1. **Database** — recuperare `db.dump` dall’ultimo daily ok (locale, freddo `daily/`, o Drive `db/latest/`). Su stack di destinazione: `pg_restore` con utente admin del progetto, tipicamente `--clean --if-exists` (adattare a naming DB/utente del progetto).
2. **Volume** — recuperare `volume.tar.gz` dallo slot attivo (o fermo, se l’attivo è corrotto) o dal manuale locale; estrarre nel mount del volume di destinazione; sistemare ownership del processo applicativo (es. uid del container).
3. **Storage oggetti** — copiare oggetti selezionati dallo slot freddo verso un path di test o, in disaster recovery controllato, verso il live. Per file mancanti/sovrascritti sul vivo: preferire lo **slot fermo**. Se il vivo è vuoto/wipe: **non** lanciare il job di sync (il freno deve bloccare); ripristinare dallo slot sano.
4. **Verifiche** — app up, login, campione di record DB, campione di file volume e oggetti storage, confronto con `backup_info.txt` / state JSON della run usata.

### 7.2 Staging+swap Drive (implicazione restore)

Su Drive, `latest` è l’ultima pubblicazione verificata; `previous` è la generazione precedente dopo lo swap. Se `incoming` fallisce, `latest` non viene toccato: usare `latest` (o `previous`) come fonte umana, non `incoming`.

### 7.3 Non definito come procedura automatica

- Runbook di disaster recovery end-to-end con un unico comando;
- restore automatico da cron;
- restore del codice git o dei secret manager (fuori scope, §10).

---

## 8. Standard di qualità degli script bash

Uno script nuovo è accettabile solo se rispetta questi guardrail (stesso comportamento dell’implementazione di riferimento), non se è «bash simile».

### 8.1 Obbligatorio in ogni script

- Shebang `#!/usr/bin/env bash`. Fasi A–D: `set -euo pipefail`. Wrapper: `set -uo pipefail` (senza `-e` che abortisca gli step successivi).
- `require_root` come primo controllo in `main`.
- Env dedicato: `source` di `/root/.config/…`; variabili remote obbligatorie con `${VAR:?…}`.
- Dal `.env` app: solo `grep` del valore necessario.
- `flock -n` su lock dedicato `/var/run/{project_slug}-….lock`.
- `assert_*_path` **prima** di ogni `delete` / `purge` / `moveto`.
- State JSON scritto **anche** in errore; `chmod 644`; se il path è una **directory** (phantom mount) → log e non sovrascrivere.
- `log` con timestamp; `die` esce con codice 1.
- Scritture sul freddo con `--s3-no-check-bucket`.
- Pattern `case` su **una sola riga**. Vietato `|` a fine riga (syntax error bash).
- Nessun secret nel repo.

### 8.2 Per fase

| Fase | Deve |
|---|---|
| **DB** | Solo `pg_dump` + `backup_info.txt`; prune solo sotto `daily/` e `monthly/`; mensile 1° o prima daily ok |
| **Volume** | Tar volume dati; 2 slot; freno 50%; staffetta solo nel giorno configurato |
| **Storage** | Sync incrementale con esclusioni di progetto; 2 slot (3 solo se richiesto); freno 50%; staffetta |
| **Drive** | Se source JSON ≠ `ok` → skip; DB/volume staging+swap; Space `sync --delete-after` |
| **Wrapper** | Step indipendenti; un solo cron; un solo lock wrapper |

### 8.3 Checklist prima di dichiarare uno script «fatto»

```
- [ ] bash -n passa
- [ ] flock + lock dedicato
- [ ] assert prima di ogni delete/purge/moveto
- [ ] case patterns su una riga
- [ ] state JSON anche su fail; guardia directory phantom
- [ ] env 600, script 750, state 644
- [ ] smoke: muore se una remote obbligatoria manca
- [ ] Drive: skip se lo stato sorgente non è ok
```

### 8.4 Anti-pattern

- Cron separati oltre al wrapper.
- Assumere fasi o job che nel progetto non esistono.
- Write diretto su `db/latest` o `volume/latest`.
- `--delete-before` verso Drive; secondo specchio/staffetta storage su Drive.
- Cleanup alle 03:00.
- Secret duplicati negli env se già in `rclone config`.
- Backup manuali sul freddo.
- Prune/purge senza assert.
- `set -e` nel wrapper che blocca i passi successivi.
- `source` dell’intero `.env` app.
- Trattare NOTICE `rclone mkdir` come errore.

---

## 9. Adattamento a un progetto nuovo

Compilare e usare solo questi placeholder. Non hardcodare un unico prodotto.

### 9.1 Infrastruttura condivisa (confermare all’utente al primo uso)

| Elemento | Valore tipico aziendale | Note |
|---|---|---|
| Bucket freddo | `omni-ml-backup` (FRA1) | condiviso; progetto sotto `{project_slug}/` |
| Service account Drive | account backup aziendale | invitato sulla cartella del progetto |
| Chiave SA sul server | `/etc/omni-backup/gdrive.json` | `600`, mai in git |
| Env / script / state | path in §4.4 | |

### 9.2 Valori fissi (non parametrizzare)

| Elemento | Valore |
|---|---|
| Orario wrapper | `0 3 * * *`, `Europe/Rome` |
| Sequenza | DB → volume → storage → Drive (solo fasi applicabili) |
| Retention DB | 30 daily / 12 monthly / 7 locali |
| Slot volume | 2 |
| Slot storage | 2 default (3 solo su richiesta) |
| Staffetta | domenica (`STAFFETTA_WEEKDAY=0`) |
| Freno | 50% |
| Job pesanti app | dalle 04:00 |

### 9.3 Tabella da compilare per il progetto

| Variabile | Significato | Valore |
|---|---|---|
| `{project_slug}` | Radice sul freddo e prefisso nomi server | |
| `{backup_prefix}` | Prefisso cartelle datate DB (di norma `{project_slug}_`) | |
| `{docker_project}` | Nome progetto Docker Compose prod | |
| `{compose_file}` | File compose di produzione | |
| `{server_project_path}` | Path repository sul server | |
| `{volume_name}` | Volume Docker da archiviare | |
| `{db_name}` | Database da dumpare | |
| `{db_user}` | Utente con privilegi di dump | |
| `{rclone_remote_cold}` | Remote write sul freddo | |
| `{live_bucket}` | Bucket asset live (se fase B) | |
| `{rclone_remote_live_ro}` | Remote Read-only sul live | |
| `{live_exclude}` | Prefissi da escludere dal sync | |
| `{drive_folder_id}` | ID cartella Drive (se fase D) | |
| `{rclone_remote_drive}` | Remote Drive (di norma `{project_slug}-gdrive`) | |
| Fasi applicabili | DB / volume / storage / Drive / dashboard | |

### 9.4 Esempio compilato (solo chiarimento)

Omni Car usa tipicamente: slug `omnicar`, volume `omnicar-prod_backend_data`, bucket live `omnicar-assets`, remote freddo dedicato, tutte le fasi attive, exclude storage `_sandbox/**`, esclusioni volume per cache/scratch/trash. Altri progetti compilano la stessa tabella con i **propri** valori.

### 9.5 Workflow verifica progetto esistente

1. Leggere crontab, env, script installati, job schedulati.
2. Confrontare con questo protocollo e col layout del freddo.
3. Produrre un **gap report** (conforme / devia / manca / non applicabile).
4. Proporre correzioni; attendere conferma prima di modificare il server.

### 9.6 Migrazione da backup misti

Se esistono cartelle `auto/{prefix}_*/` con dump **e** tarball asset:

1. Spostarle sotto `db_backups/auto/daily/`.
2. Non convertirle: i vecchi escono col prune a 30 giorni.
3. `monthly/` si popola dai run nuovi.
4. Verificare che il nuovo backup DB non produca più il tarball.

Su S3, `rclone mkdir` non crea directory vuote: i prefissi compaiono al primo upload.

---

## 10. Fuori scope

Questo protocollo **non** copre:

- Backup del **codice** (git remoti, mirror di repository).
- Backup di **secret manager**, vault, o del file `.env` applicativo come parte del wrapper notturno (eventuale copia umana offline del solo env è procedura ops separata, non fase del cron 03:00).
- Hardening OS, firewall, gestione utenti SSH (vedi `PROTOCOLLO_GESTIONE_UTENTI_SERVER.md`).
- Pipeline CI/CD, patch OS, monitoring generico oltre ai file di stato backup.
- Inventare una dashboard se il progetto non ne ha una.
- Backup di bucket o path non dichiarati nelle fasi applicabili (un nuovo bucket live richiede estensione esplicita della fase storage).
- Path solo-cache / scratch / trash usati come storage di dati utente irrecuperabili: vanno evitati in progettazione; se presenti, restano esclusi dal tar durable e dal sync dove previsto.

Copertura dei **nuovi contenuti di prodotto**: prima di persistere un nuovo tipo di dato, dichiarare dove vive (DB / volume / object storage) e se il cron notturno lo copre. Default: DB + volume durable + storage live. Se esce da questo perimetro, fermarsi e chiedere.

---

## 11. Scostamenti osservati (skill vs implementazione Omni Car)

Rilevati confrontando la skill aziendale `nightly-backup-stack` con gli script/documentazione Omni Car Backend (analisi da file locali, senza accesso SSH al server):

| Punto | Standard (skill / questo protocollo) | Omni Car (osservato nei file) | Cosa seguire |
|---|---|---|---|
| Exclude storage | Preferire `LIVE_EXCLUDE` da env, valorizzato per progetto | `--exclude "_sandbox/**"` **hardcoded** nello script space; l’env.example non espone `LIVE_EXCLUDE` | Per **nuovi** progetti: parametrizzare con `LIVE_EXCLUDE`. Omni Car esistente: allineare quando si tocca lo script |
| Tar volume | Skill: «tar completo» del volume | Script: esclude `scratch`, `.trash`, `_sandbox`, cache Photoroom `cutouts`; freno su `durable_bytes` | Accettato: esclusioni solo per path **rigenerabili** documentati; il freno deve basarsi sui byte durable, non sulla cache |
| Layout doc `manual/` sul freddo | Manuali **solo** su disco locale | In `COMANDI_SERVER.md` compare un ramo `db_backups/manual/` nel diagramma freddo; lo script manuale scrive solo in `/var/backups/…/manual/` | Policy: manuali solo locali. Il diagramma progetto non autorizza upload manuali sul freddo |
| Override slot storage | Env `SLOTS` + eventuale config esplicita | Omni Car può far vincere un setting dashboard su env | Consentito come estensione di progetto; default resta 2 |

Se skill e codice divergono su un punto non elencato qui, **prevale questo protocollo** (derivato dalla skill) per i progetti nuovi; per Omni Car in produzione verificare sul server prima di cambiare comportamento.

---

## 12. Riferimenti interni

- Skill: `nightly-backup-stack` (`SKILL.md`, `reference.md`, `project-vars.md`)
- Protocollo correlato: `PROTOCOLLO_GESTIONE_UTENTI_SERVER.md` (ruoli; cron backup tipicamente root sul server)
- Codice da copiare: repository Omni Car Backend → `scripts/omnicar_*backup*.sh` e relativi `*.env.example`
