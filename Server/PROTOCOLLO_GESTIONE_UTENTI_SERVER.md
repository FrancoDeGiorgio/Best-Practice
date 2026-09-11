# Protocollo — Gestione utenti e accessi su server Linux

**Versione:** 1.1  
**Data:** 11 settembre 2026  
**Ambito:** prassi aziendale replicabile su qualsiasi server Linux (VPS, droplet, bare metal).  
**Sistema di riferimento:** Ubuntu / Debian. Dove un comando o un gruppo cambia (RHEL, Rocky, Alma), è indicato.  
**Fuori ambito:** rotazione secret applicativi, pipeline CI/CD del publisher, hardening di rete, sicurezza del codice, patch OS. Vedi §14.

Questo file è il protocollo da seguire quando allestisci o riallinei un server. Non è il recap di una macchina specifica. I nomi romani (`legatus`, `centurio`, `miles`) sono l’esempio aziendale dei tre ruoli; su un server nuovo puoi usare gli stessi nomi o altri, purché **alias SSH = utente Linux** e i permessi restino quelli di questo documento.

---

## 1. Scopo

Separare **chi entra** e **cosa può toccare**, così un errore di path, uno script sbagliato o una chiave rubata non equivalgono a root sulla macchina intera.

Il caso tipico da evitare: due persone (o due script) entrano come `root` sullo stesso host. Chi pubblica una landing statica ha gli stessi poteri di chi gestisce API, Docker e secret. Un `rsync --delete` sul path sbagliato cancella l’applicazione.

Il controllo non è «lo script è scritto bene». Il controllo è: **permessi sul disco + sudo ristretto + ForceCommand**. Anche se lo script punta al path sbagliato, il sistema deve rifiutare la scrittura.

---

## 2. Principi (non negoziabili)

1. **Un grado = un utente Linux = un alias SSH.** `ssh alias-ops` deve dare `whoami` uguale al nome dell’utente ops. Niente alias che nasconde `User root` o `User ubuntu`.
2. **Scala fissa:** admin > ops > publisher. L’admin amministra la macchina. L’ops opera l’applicazione. Il publisher scrive **una** docroot statica e nient’altro.
3. **Un umano = un utente Linux in regime**, per ciascun grado che gli serve. Lo stesso operatore ha **due** utenti (admin e ops), non un unico account con entrambi i cappelli. Eccezione accettata: secondo umano di emergenza con **chiave propria** sullo **stesso** utente admin (§11).
4. **Niente account `root` condiviso** per il quotidiano. Root SSH si spegne solo dopo che admin e console del provider sono stati provati (§5.6).
5. **Break-glass = console del provider** (DigitalOcean Droplet Console, Hetzner Console, AWS EC2 Serial/SSM, o equivalente), non un alias SSH permanente verso root.
6. **Admin: sudo ALL con password.** La password sta nel password manager, mai negli script. Chiave rubata senza password ≠ macchina tua. Cache sudo tipica: ~15 minuti (`timestamp_timeout`).
7. **Ops non conosce la password admin.** Ha Docker e write sui tree applicazione. Sudo solo verso wrapper espliciti (es. manutenzione). Docker è un privilegio alto (il socket vale quasi root): basta contro un typo di path, non contro un attaccante già dentro ops.
8. **Publisher: niente shell, niente Docker, niente sudo.** Solo ForceCommand + `rrsync` (o equivalente) su **una** docroot.
9. **Script di deploy e backup puntano all’ops, mai all’admin.**
10. **Secret runtime fuori dal tree pubblicabile.** Permessi stretti. L’ops legge se deve fare recreate. Il publisher no.
11. **Fidarsi dello script non è un controllo.**
12. **Niente alias SSH nudo** verso l’host (un `ssh nomemacchina` senza suffisso di ruolo deve fallire o non esistere).
13. **Una modifica alla volta**, con prova di **affermazione** e di **negazione** prima di togliere i ponti (chiavi su root, `PermitRootLogin`).

---

## 3. Catalogo ruoli

I nomi tra parentesi sono l’esempio aziendale. Scegli un set e usalo uguale su Linux e sugli alias SSH.

| Ruolo | Esempio nome | Shell login | sudo | Docker | Cosa scrive | Chiave SSH |
|---|---|---|---|---|---|---|
| **Admin** | `legatus` | `/bin/bash` | **ALL, con password** | no (non gli serve nel quotidiano) | niente di applicativo in automatico; a mano: apt, sshd, utenti, nginx, `chattr` sui secret | dell’operatore titolare; eventuale seconda chiave di emergenza (§11) |
| **Ops** | `centurio` | `/bin/bash` | solo wrapper elencati in sudoers | sì, gruppo `docker` | tree applicazione (repo API, compose, SPA se la pubblica l’ops) | dell’operatore titolare (può essere la **stessa** coppia usata su admin) |
| **Publisher** | `miles` | shell presente per `su` da root, ma SSH **senza** PTY | no | no | **una** docroot statica | dell’umano (o della CI) che pubblica solo quello |
| **root** | `root` | locale / console | — | — | emergenza da console | SSH **spento** a regime; `authorized_keys` di root **non** si svuota |

### 3.1 Cosa fa ciascun ruolo, in pratica

**Admin** — lo usi a mano, da terminale, quando serve il privilegio di macchina:

- pacchetti, `sshd`, utenti, gruppi, sudoers;
- certificati, vhost nginx, `chattr` sul file env;
- riattivare root SSH per una finestra, poi rispegnerlo.

Non lo metti in `REMOTE_HOST` di nessuno script.

**Ops** — quotidiano e automazione:

- `git pull`, `docker compose`, restart, health;
- publish della SPA/app se non è una jail statica separata;
- script di backup lanciati dal laptop o cron che girano come ops / root-cron già esistente;
- wrapper manutenzione (pagina 503, manutenzione on/off).

**Publisher** — un canale solo:

- `rsync` (GNU) verso l’alias publisher;
- destinazione `alias:` o `:.` (la jail è sul server);
- `--delete` è accettabile **dentro** la jail; il muro è `rrsync`, non la riga dello script.

### 3.2 Cosa non è un ruolo

- Un utente `ubuntu` / `debian` di immagine cloud con `NOPASSWD: ALL` non è l’admin di questo protocollo. Va sostituito o riallineato (password sudo, niente NOPASSWD ALL).
- Un utente `deploy` / `landing` / `sudo` con nome generico invita gli script a riusarlo come root. Meglio un nome di grado, uguale all’alias.
- Root non è «l’utente di squadra».

---

## 4. Matrice necessità → insieme minimo di utenti

Crea **solo** i ruoli che ti servono. Non creare il publisher «perché esiste nel protocollo» se nessuno pubblica statico da un canale separato.

| Necessità sul server | Admin | Ops | Publisher | Note |
|---|---|---|---|---|
| Server nuovo, solo tu, ancora niente app | sì | dopo | no | Tieni root SSH acceso finché admin + console non sono verdi. |
| App Docker / compose / git pull / backup | sì | sì | no | Caso «solo API». Script sul canale ops. |
| Come sopra + un altro umano pubblica solo HTML/statico | sì | sì | sì | Caso classico: landing statica vs API/SPA Docker. |
| Solo sito statico, lo pubblichi tu | sì | no | sì | Lo script di publish usa il publisher, non l’admin. |
| Solo sito statico + un esterno che pubblica | sì | no | sì | L’esterno **solo** publisher. Non dargli ops. |
| Tu operi l’app e un secondo umano deve poter entrare in emergenza | sì + seconda chiave | sì se c’è l’app | come da righe sopra | La seconda persona **non** va su ops, publisher o root (§11). |
| Due persone fanno il quotidiano come root | — | — | — | Vietato. Spezza subito: ognuno sul grado che gli compete. |

Stesso umano, due gradi: due utenti Linux (admin e ops). Non fondere i due in un unico login «per comodità».

Secondo operatore quotidiano (non emergenza): questo protocollo non usa un account ops condiviso. Se nasce il bisogno, si crea un **secondo utente di classe ops** con chiave propria e gli stessi vincoli sudo/Docker — non si condivide `centurio`. È un’estensione, non il default.

---

## 5. Procedura di setup (ordine sicuro)

Due percorsi. Non mescolarli.

**Percorso A — server nuovo o riallineamento senza urgenza.** Ordine per **non chiudersi fuori**:

1. Inventario + console provider provata (§5.1).  
2. Admin, provato da una sessione nuova.  
3. Ops, provato; script spostati sull’ops.  
4. Publisher, se serve; prova affermazione + negazione; poi si toglie la sua chiave da root.  
5. `PermitRootLogin no`.  
6. Alias nudi verso root rimossi dal laptop.

**Percorso B — incidente (chi pubblica lo statico entra ancora come root).** Qui il rischio è il wipe, non il lockout dell’admin titolare (che è già root). Ordine osservato e da ripetere:

1. Stop-gap sui secret (copia fuori tree, `chattr +i` sul live se serve) **senza** togliere ancora chiavi.  
2. Utente publisher + permessi sulla sola docroot.  
3. ForceCommand / `rrsync` verde (publish vero + dest sbagliata rifiutata).  
4. Allineare lo script del publisher al nuovo alias; **poi** togliere la sua chiave da root.  
5. Secret runtime fuori dal tree; recreate/cron puntano al nuovo path; solo allora si toglie il `.env` dal working tree.  
6. Creare admin e ops; spostare gli script dell’app sull’ops.  
7. Console provider (se non l’hai già aperta al punto 1) + `PermitRootLogin no`.

In B non spegnere root e non togliere chiavi da root prima che il canale ristretto del publisher sia verde. In A non creare il publisher «perché esiste nel protocollo» se la matrice (§4) non lo chiede.

Tieni **aperta** una sessione root (o admin, nelle fasi finali) finché la sessione nuova del passo appena fatto non è verde.

I comandi sotto sono da eseguire come root o come admin con sudo. Adatta i nomi (`legatus` / `centurio` / `miles`) e i path.

### 5.1 Prima di toccare gli utenti

Inventario, senza togliere ancora nessuno:

- Elenca `/root/.ssh/authorized_keys` (e di eventuali utenti cloud). Annota **commento + fingerprint**, non incollare le chiavi intere in chat o ticket.
- `ssh-keygen -lf /root/.ssh/authorized_keys`
- Capire quale riga è di chi. Due umani sulla stessa riga (stessa chiave) è un problema: non potrai offboardare uno solo.
- Se i secret dell’app stanno nel tree che qualcuno potrebbe `rsync --delete`, copiali **fuori** da quel tree (es. `/root/app-env-backup.env`, mode `600`) e valuta `chattr +i` sul live finché non li sposti in `/etc/...` (§10).
- Verifica di saper entrare nella **console del provider** (account + 2FA). Se non l’hai mai aperta, aprila ora: è il break-glass. Non è un passo opzionale da fare «dopo il flip».

Non togliere la chiave di chi pubblica ancora come root: lo blocchi senza alternativa.

### 5.2 Admin (primo utente da creare)

```text
useradd --create-home --shell /bin/bash legatus
passwd legatus
usermod -aG sudo legatus
```

Su RHEL / Rocky / Alma il gruppo è `wheel`, non `sudo`. Su quelle distro: `usermod -aG wheel legatus` e verifica che `%wheel` in sudoers richieda la password.

Home e SSH:

```text
chmod 700 /home/legatus
mkdir -p /home/legatus/.ssh
chmod 700 /home/legatus/.ssh
```

Metti **solo** la pubblica dell’operatore titolare in `/home/legatus/.ssh/authorized_keys`. Riga con commento parlante (chi, quando). Mode `600`, owner `legatus:legatus`.

Password di login = password sudo, nel password manager, voce dedicata (es. `server-x/legatus`). Non è un secret da mettere negli script.

Laptop — alias dedicato, vedi §6. Poi, **seconda sessione** (non riusare il terminale root):

```text
ssh alias-admin
whoami          # deve stampare legatus
sudo whoami     # chiede la password, poi stampa root
sudo -l         # ALL, non NOPASSWD
```

Se `sudo whoami` non chiede password, ti sei portato dietro un `NOPASSWD: ALL` (immagine cloud o drop-in sudoers). Correggi **prima** di continuare.

Vietato in questo passo: `PermitRootLogin no`; svuotare `/root/.ssh/authorized_keys`; `NOPASSWD: ALL`; mettere gli script su questo alias.

### 5.3 Ops

```text
useradd --create-home --shell /bin/bash centurio
chmod 700 /home/centurio
usermod -aG docker,www-data centurio
```

`www-data` serve se nginx deve leggere i tree di cui l’ops è owner e tu usi gruppo `www-data` (schema owner ops, gruppo web, `750`/`640`). L’ops **non** va nel gruppo `sudo`.

Albero applicazione (adatta i path; esempio: API Docker + SPA):

```text
chown -R centurio:www-data /var/www/app-api
chown -R centurio:www-data /var/www/app-spa
find /var/www/app-api /var/www/app-spa -type d -exec chmod 750 {} \;
find /var/www/app-api /var/www/app-spa -type f -exec chmod 640 {} \;
```

Non fare `chown -R www-data:www-data`: l’ops smette di scrivere e gli script iniziano a `chown` da remoto (anti-pattern).

Chiave SSH dell’operatore in `/home/centurio/.ssh/authorized_keys` (`700` sulla dir, `600` sul file). Stessa coppia dell’admin: accettabile per lo stesso umano. Non metterci la chiave del publisher.

Se l’ops fa `git pull` verso un hosting, la chiave **di deploy** (solo read sul repo) sta in `/home/centurio/.ssh/`, owner `centurio`, `600`. `known_hosts` del remoto git va aggiunto da ops (`ssh-keyscan` o primo connect consapevole).

Sudoers **solo** wrapper, con `visudo`:

```text
visudo -f /etc/sudoers.d/centurio
```

Contenuto del file (path dei wrapper = quelli che installerai tu; nomi di esempio):

```text
centurio ALL=(root) NOPASSWD: /usr/local/bin/app-maintenance-on, /usr/local/bin/app-maintenance-off
```

I due wrapper sono script `root:root` `755`, **non** scrivibili dall’ops. Fanno una cosa sola (es. scambiare un symlink nginx verso una pagina 503 e ricaricare nginx). Non sono una shell. Esempio minimo:

```text
#!/bin/sh
set -eu
# /usr/local/bin/app-maintenance-on
ln -sfn /var/www/app-maintenance /run/app-maintenance-link
nginx -t && systemctl reload nginx
```

```text
#!/bin/sh
set -eu
# /usr/local/bin/app-maintenance-off
rm -f /run/app-maintenance-link
nginx -t && systemctl reload nginx
```

Il vhost deve testare l’esistenza di quel link: se c’è, serve la pagina 503; se non c’è, serve l’app. Adatta al tuo nginx: l’importante è che l’ops non abbia `sudo nginx` generico.

`visudo` deve chiudere pulito. Poi:

```text
ssh alias-ops
whoami                 # centurio
groups                 # centurio, docker, www-data — non sudo
docker ps              # elenco container, non denied sul socket
sudo -l                # solo i due wrapper
sudo -i                # deve fallire (niente password che funzioni da root via questo utente)
```

Secret: se l’ops deve leggere l’env per `docker compose --env-file`, applica lo schema di §10 **ora** (dir `750` `root:centurio`, file `640` `root:centurio`). Una dir `700` root blocca l’ops anche se il file è `640`.

Sposta gli script del laptop: `REMOTE_HOST` / `User` = alias ops. Smoke: un deploy o un restart vero, health ok.

Vietato: `NOPASSWD: ALL`; chiave del publisher su ops; owner dei tree al publisher.

### 5.4 Publisher (solo se la matrice lo richiede)

Conferma **prima** quale directory è la `root` del vhost statico (`nginx -T` o il file in `sites-enabled`). Non indovinare: due path che differiscono di poche lettere sono il modo classico per cancellare l’API.

```text
useradd --create-home --shell /bin/bash miles
chmod 700 /home/miles
```

Shell `/bin/bash` è voluta: serve per `su - miles` da admin/root in debug. Il muro SSH non è `nologin`; è ForceCommand. `nologin` come shell di login rompe alcuni flussi `su` e non aggiunge nulla se SSH è già ingabbiato.

```text
chown -R miles:www-data /var/www/sito-statico
find /var/www/sito-statico -type d -exec chmod 2750 {} \;
find /var/www/sito-statico -type f -exec chmod 640 {} \;
```

`2750` (setgid): i file nuovi restano gruppo `www-data` e nginx li legge. Il publisher **non** va in `www-data` (altrimenti legge anche i tree `750` dell’ops) e **non** va in `docker`.

```text
# atteso
sudo -l -U miles
# User miles is not allowed to run sudo
groups miles
# miles
```

Pacchetto `rsync` installato. Trova `rrsync`:

```text
command -v rrsync
# di solito /usr/bin/rrsync su Debian/Ubuntu recenti
```

Se manca l’eseguibile ma il pacchetto è installato, cerca sotto `/usr/share/rsync/` e installa una copia in `/usr/bin/rrsync` owned by root.

Wrapper `root:root` `755`, non scrivibile dal publisher:

```text
#!/bin/sh
# /usr/local/bin/deploy-static
set -eu
LOG=/var/log/static-deploy.log
ts=$(date -Is)
printf '%s user=%s cmd=%s\n' "$ts" "${USER:-?}" "${SSH_ORIGINAL_COMMAND:-}" >> "$LOG"
exec /usr/bin/rrsync -wo /var/www/sito-statico
```

```text
touch /var/log/static-deploy.log
chown miles:adm /var/log/static-deploy.log
chmod 640 /var/log/static-deploy.log
chmod 755 /usr/local/bin/deploy-static
chown root:root /usr/local/bin/deploy-static
```

`-wo` = write-only: il client non scarica la docroot. `--delete` **dentro** la jail resta consentito (è il publish). Un path fuori dalla jail viene annidato sotto la docroot o rifiutato: **non** scrive sull’API.

`authorized_keys` del publisher (`/home/miles/.ssh/authorized_keys`, `600`):

```text
command="/usr/local/bin/deploy-static",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty ssh-ed25519 AAAA...COMMENTO_UMANO
```

Sostituisci `AAAA...COMMENTO_UMANO` con la pubblica reale. Non riusare la chiave admin/ops.

Cintura e bretelle in `sshd` — drop-in, poi `sshd -t`:

```text
# /etc/ssh/sshd_config.d/publisher.conf
Match User miles
    ForceCommand /usr/local/bin/deploy-static
    AllowTcpForwarding no
    X11Forwarding no
    PermitTTY no
```

`ForceCommand` in sshd vince sulla riga `command=` e resta valido anche se qualcuno modificasse `authorized_keys`. Riavvia ssh **solo** se hai già una sessione admin aperta e verificata.

Lato client (chi pubblica):

- alias `User miles`, `IdentitiesOnly yes`, chiave sua;
- `rsync` **GNU** (su macOS: Homebrew `/opt/homebrew/bin/rsync` o `/usr/local/bin/rsync`; **non** `/usr/bin/rsync` = openrsync, che `rrsync` rifiuta);
- destinazione `alias-publisher:` o `:.` — **mai** il path assoluto della docroot (rrsync lo anniderebbe: `sito-statico/var/www/sito-statico`);
- niente `ssh … exit`, niente `mkdir` / `chown` / `scp` di fallback, niente `rm -rf` remoto.

SFTP + `ChrootDirectory` **non** è il modello di questo protocollo: la chroot deve essere root-owned e non scrivibile, la docroot deve essere scrivibile dal publisher. I due requisiti si urtano.

Prove: §7.3. Solo dopo il verde: togli la chiave del publisher da `/root/.ssh/authorized_keys` (backup prima, es. `authorized_keys.bak-YYYYMMDD`, mode `600`). Lo script vecchio (`User root`) deve diventare `Permission denied (publickey)`.

### 5.5 Script e laptop, prima di spegnere root

Controlla che **nessuno** script punti ancora a `User root` o all’alias nudo. Admin assente da `REMOTE_HOST`. Publisher assente dagli script dell’app. Ops assente dallo script della landing.

Togli dal `~/.ssh/config` del laptop gli Host che fanno `User root`. Non lasciare un alias «corto» comodo che è root.

### 5.6 Spegnere root SSH

Prerequisiti, tutti veri insieme:

- `ssh alias-admin` + `sudo whoami` verde (sessione **nuova**);
- `ssh alias-ops` verde se l’ops esiste; script smoke ok;
- publisher verde se esiste, e la sua chiave **non** è più su root;
- console del provider usata **almeno una volta** in questa campagna (login web, sessione root o admin da console).

Da una sessione **admin** (non chiudere questo terminale):

```text
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak-YYYYMMDD
sudo sed -n '1,120p' /etc/ssh/sshd_config
```

Imposta `PermitRootLogin no` nel file che comanda davvero (riga in `sshd_config` oppure drop-in). Niente `prohibit-password` a regime: root non entra nemmeno con chiave.

```text
sudo sshd -t
sudo sshd -T | grep permitrootlogin
# atteso: permitrootlogin no
sudo systemctl restart ssh
```

Su Debian/Ubuntu il servizio è `ssh` (a volte alias `sshd`). Su RHEL è `sshd`.

Senza chiudere il terminale admin, da **un altro** terminale:

```text
ssh alias-admin && whoami     # admin
ssh alias-ops && whoami       # ops, se esiste
ssh user@host -l root         # Permission denied (publickey)
```

Poi togli dal laptop ogni `Host` verso `User root`.

`/root/.ssh/authorized_keys` **non** si svuota. Se un giorno riattivi root SSH dalla console, quella chiave serve. Non ricreare un alias `…-root`.

Riattivazione temporanea (rara, a mano, da admin o da console):

```text
sudo editor /etc/ssh/sshd_config
# PermitRootLogin prohibit-password   (chiave, non password di root in rete)
sudo sshd -t && sudo systemctl restart ssh
```

Finestra breve, lavoro, poi di nuovo `PermitRootLogin no` e verifica. Non è un canale quotidiano.

---

## 6. SSH sul laptop (esempio)

Stesso `HostName`, tre `User` diversi. Sostituisci host, path chiavi, nomi.

```sshconfig
Host progetto-legatus
  HostName HOST_O_DNS
  User legatus
  IdentityFile ~/.ssh/OPERATORE
  IdentitiesOnly yes

Host progetto-centurio
  HostName HOST_O_DNS
  User centurio
  IdentityFile ~/.ssh/OPERATORE
  IdentitiesOnly yes

Host progetto-miles
  HostName HOST_O_DNS
  User miles
  IdentityFile ~/.ssh/PUBLISHER
  IdentitiesOnly yes
```

Regole:

- Non esiste `Host progetto` nudo.
- Non esiste `Host progetto-root`.
- `IdentitiesOnly yes` evita che l’agente offra la chiave sbagliata e «entri comunque».
- L’operatore può riusare `OPERATORE` su admin e ops. Il publisher ha una coppia **sua**.
- Se usi un secondo admin di emergenza, sul **suo** laptop c’è solo `progetto-legatus` con **la sua** `IdentityFile`. Non gli dai gli alias ops/publisher.

Sul server, `AllowUsers` è facoltativo (follow-up di hardening). Se lo metti, elenca esplicitamente admin, ops, publisher — e tieni una sessione aperta mentre riavvii sshd.

---

## 7. Prove di affermazione e di negazione

Non chiudere una fase sulle intenzioni. Ogni prova ha un esito atteso. Le negazioni valgono quanto le affermazioni.

### 7.1 Admin

| Prova | Atteso |
|---|---|
| `ssh alias-admin` ; `whoami` | nome admin |
| `sudo whoami` | chiede password, poi `root` |
| `sudo -l` | `(ALL : ALL) ALL` o equivalente **con** password, non `NOPASSWD` |
| `sudo visudo -c` dopo un drop-in | ok, nessun syntax error |
| Seconda sessione dopo restart sshd | entra ancora |

### 7.2 Ops

| Prova | Atteso |
|---|---|
| `whoami` | nome ops |
| `groups` | ops, `docker`, eventualmente `www-data`; **non** `sudo` |
| `docker ps` | elenco, non permission denied sul socket |
| write su un file di prova nel tree app | ok; owner ops |
| `sudo /usr/local/bin/app-maintenance-on` poi `off` | effetto previsto (503 poi sito normale) |
| `sudo -l` | solo quelle righe |
| `sudo -i` / `sudo bash` | rifiutato |
| `ls` sul secret (§10) | lettura ok se gli serve recreate |
| `sudo chattr -i` sul secret | rifiutato |
| deploy/restart da script laptop via alias ops | health ok |

### 7.3 Publisher

| Prova | Atteso |
|---|---|
| `ssh alias-publisher` senza comando | niente bash; `rrsync` esce (spesso «Not invoked via sshd» / exit 1); PTY denied |
| GNU rsync di un `probe.txt` verso `alias:` | file nella **docroot**; HTTP 200 sul vhost statico; poi `rm` → 404 |
| stesso rsync con dest path dell’API (`/var/www/app-api` o simile) | file **non** compare nell’API; al più è annidato sotto la jail; `DENY_OK` |
| `ls` tree API, `cat` env, `docker ps` | impossibili (niente shell). Se testi con `su - miles` da admin: `Permission denied` su API/env/socket |
| `sudo -l -U miles` | not allowed to run sudo |
| `groups miles` | solo sé stesso |
| client openrsync (macOS di sistema) | rifiutato da `rrsync` (syntax) — è corretto; si usa GNU rsync |
| dopo il taglio da root: `ssh` suo verso `User root` | `Permission denied (publickey)` |
| publish vero | vhost statico 200; vhost app invariato |

### 7.4 Root SSH spento

| Prova | Atteso |
|---|---|
| `sshd -T` → `permitrootlogin` | `no` |
| `ssh` verso `root` | `Permission denied (publickey)` |
| alias nudo | non esiste o fallisce |
| admin / ops / publisher | invariati rispetto alle prove sopra |
| console provider | la sai ancora aprire (l’hai già aperta prima del flip) |

### 7.5 Isolamento incrociato

| Da | Verso | Atteso |
|---|---|---|
| Publisher | tree ops, env, Docker, sudo, home admin | no |
| Ops | sudo ALL, `chattr` env, sshd, apt | no |
| Ops | docroot publisher | no write quotidiano (non è il suo canale; i permessi possono già negarlo) |
| Chiave publisher | admin o ops | `Permission denied` |
| Chiave admin | `User root` a regime | `Permission denied` |
| Script | alias admin | non deve comparire nel file |

---

## 8. sudo e sudoers

| Utente | Policy | Password |
|---|---|---|
| admin | `ALL=(ALL:ALL) ALL` via gruppo `sudo` (Debian/Ubuntu) o `wheel` (RHEL) | sì, ogni volta che la cache è scaduta |
| ops | file in `/etc/sudoers.d/` con **path assoluti** di wrapper | NOPASSWD **solo** su quelle righe |
| publisher | assente da sudoers | — |
| root | solo console / riattivazione temporanea | password root locale se la imposti; non è un canale SSH |

Dettagli:

- Modifichi sudoers **solo** con `visudo` / `visudo -f`. Un file rotto = macchina senza sudo (break-glass = console).
- `chmod 440` sui file in `/etc/sudoers.d/`.
- Cache: `Defaults timestamp_timeout=15` (minuti). È il secondo fattore: la chiave apre la sessione, la password sblocca root per un quarto d’ora. Non alzare il timeout per comodità. `timestamp_timeout=0` chiede la password ogni comando: più duro, usalo se l’admin è condiviso in emergenza e vuoi ridurre la finestra.
- `NOPASSWD: ALL` su admin **annulla** il secondo fattore. Vietato. `NOPASSWD: ALL` su ops è equivalente a root via script. Vietato.
- I wrapper ops non accettano argomenti liberi (`sudo wrapper $USER_INPUT`). Se ti servono argomenti, whitelist stretta nel wrapper, non `$@` inoltrato a una shell.

Verifica periodica:

```text
sudo -l -U legatus
sudo -l -U centurio
sudo -l -U miles
```

---

## 9. Chiavi SSH

- Solo ED25519 (o altro algoritmo moderno già in uso in azienda). Niente RSA maiale a 2048 se stai creando chiavi nuove.
- Una riga = una chiave = un commento parlante (`alice-laptop-2026-09`). Annota il fingerprint a lato (password manager o inventario server), **non** in questo file e non in ticket.
- `ssh-keygen -lf ~/.ssh/file.pub` in locale; sul server `ssh-keygen -lf /home/UTENTE/.ssh/authorized_keys`.
- Home `700`, `.ssh` `700`, `authorized_keys` `600`. Owner = l’utente del grado.
- L’ops/publisher non devono poter scrivere `authorized_keys` di un altro. L’admin può.
- Stesso umano su admin+ops: stessa coppia, accettabile. Non mescolare quella coppia col publisher.
- Secondo umano su admin: **altra** coppia, altra riga, commento e fingerprint annotati (§11).
- Chiave git/deploy dell’ops: solo sul repo che deve tirare, preferibilmente read-only, distinta dalla chiave interattiva se puoi. Non è la chiave del publisher.
- Non incollare chiavi private, `authorized_keys` intere o fingerprint vere in chat.

Offboarding di una chiave = togliere **quella riga**, non «rigenerare tutte le utenze» se non serve. Audit di chi è entrato: fingerprint e commento in `auth.log` / `journalctl -u ssh`. `whoami` su un admin condiviso **non** distingue due umani (§11).

---

## 10. Secret runtime

Due piani, non si sostituiscono:

| Piano | Dove | A cosa serve |
|---|---|---|
| **Runtime** | file sul server, fuori dal tree pubblicabile | compose, `--env-file`, cron |
| **Caveau umano** | password manager, **voci separate** | ripristino se il disco muore |

Il password manager non alimenta i container. Togliere il file dal disco senza averlo spostato spegne l’app al recreate.

### 10.1 Dove sta il file

Schema raccomandato:

```text
/etc/NOMEAPP/           root:centurio   750
/etc/NOMEAPP/prod.env   root:centurio   640  +  chattr +i
```

- Non sta sotto `/var/www/...`.
- Non sta nella docroot del publisher.
- Non sta in un `.env` committato.
- La directory `750` `root:ops` è obbligatoria: `700` root impedisce all’ops di attraversare la dir.
- `chattr +i` è un lucchetto anti-typo (ext4/xfs). Non è un confine contro root. Su filesystem senza `chattr`, fermati a owner/mode e path fuori tree.
- Il publisher deve prendere `Permission denied` su dir e file.
- L’ops **legge** (recreate / `grep` di debug). Non toglie `+i`.
- Solo l’admin modifica:

```text
sudo chattr -i /etc/NOMEAPP/prod.env
sudo editor /etc/NOMEAPP/prod.env
sudo chattr +i /etc/NOMEAPP/prod.env
```

Poi recreate **dal canale ops**. Un `restart` di container spesso **non** ricarica le variabili: serve recreate.

### 10.2 Altri env (backup, rclone, service account)

Se hai file distinti (dump DB, replica object storage, Drive), **non** unirli al `prod.env`. Lascia i path che i cron già usano se sono già fuori dal tree (es. sotto `/root/.config/`, home `700`). Owner root, mode `600`. L’ops e il publisher non li editano. Caveau: una voce per file, nome parlante.

### 10.3 Rotazione (raccomandazione, non obbligo di questa guida)

Questo protocollo **non** prescrive di ruotare JWT, password DB, chiavi object storage, API terze o secret CI. La rotazione è un follow-up distinto (§14).

È **consigliata** (non obbligatoria qui) quando:

- i secret sono stati in un `.env` sotto un tree toccato da root condiviso o da un `rsync --delete`;
- una chiave privata o un `authorized_keys` di root è stato in mano a chi non doveva;
- c’è evidenza di leak (chat, ticket, backup pubblico, laptop perso).

Se l’azienda accetta il rischio residuo di copie storiche, si annota la decisione e si va avanti: dopo admin/ops/publisher e root SSH spento, il publisher e il vecchio canale root **non** leggono più il live. Il caveau resta allineato ai valori attuali.

Non ricreare un `.env` nel working tree «per comodità di compose».

---

## 11. Emergenza e secondo admin

### 11.1 Break-glass

Se SSH non risponde (sshd rotto, regola cloud, chiave persa):

1. Console del provider (account cloud + 2FA), non un secondo alias root sul laptop.
2. Da console: login locale root o `su` / `sudo` come admin se la sessione console te lo dà.
3. Ripari sshd, o riattivi root SSH per una finestra (§5.6), lavori, rispegni.
4. Non aggiungere «per stavolta» la tua chiave su un utente `ubuntu` NOPASSWD e non lasciarla lì.

La console va **provata prima** del flip, non la prima volta in panico.

### 11.2 Secondo umano (emergenza, persona più esperta)

Eccezione accettata al «un umano = un utente»:

- Ha la **propria** chiave, riga propria, su `/home/legatus/.ssh/authorized_keys` (stesso utente admin).
- Commento + fingerprint annotati nell’inventario.
- Password sudo **uguale**, nel password manager (voce già esistente dell’admin), condivisa con quella persona.
- **Non** viene aggiunto a ops, publisher, root, `docker`.
- Sul suo laptop: solo l’alias admin. Niente script, niente alias ops.
- Audit: in `auth.log` / journal ssh vedi il **fingerprint** (e il commento se lo logghi). `whoami` è `legatus` per entrambi: non è un identificatore.

Offboarding di questa persona: togliere **quella riga**. Non si ricrea l’utente, non si tocca ops, non si ruota per forza la password sudo (la ruoti se sospetti che la conosca ancora qualcuno che non deve).

Non è un secondo operatore quotidiano. Se inizia a fare il quotidiano su quell’account, hai ricreato l’account condiviso: si torna alla matrice (§4) e gli si dà il grado che gli compete, con utente proprio.

---

## 12. Onboarding e offboarding

### 12.1 Onboarding operatore titolare (admin + ops)

1. Crea admin, password in caveau, chiave sua, prove §7.1.  
2. Crea ops, alberi, docker, sudoers wrapper, chiave sua, prove §7.2.  
3. Punta gli script all’ops.  
4. Console provider sul **suo** account cloud (o accesso condiviso cloud già regolato a parte).  
5. Spegnere root SSH solo a checklist §5.6 verde.

### 12.2 Onboarding publisher

1. Docroot confermata sul vhost.  
2. Utente, jail, wrapper, `authorized_keys` ristretta, drop-in sshd.  
3. Sul **suo** laptop: alias, GNU rsync, dest `alias:`.  
4. Prove §7.3, **poi** rimozione della sua chiave da root e da qualsiasi altro utente.  
5. Non riceve password sudo, non riceve alias admin/ops, non riceve Docker.

### 12.3 Onboarding secondo admin di emergenza

1. Genera (lui) una coppia nuova. Ti dà la **pubblica**.  
2. Aggiungi la riga su admin, commento + fingerprint in inventario.  
3. Condividi la voce password sudo dal password manager.  
4. Lui prova `ssh` + `sudo whoami` da una sessione sua.  
5. Stop. Niente ops, niente publisher, niente root.

### 12.4 Offboarding publisher

1. Togli la riga da `/home/miles/.ssh/authorized_keys`.  
2. Verifica che non resti su root, admin, ops.  
3. Lui riprova: `Permission denied`.  
4. Non serve cancellare l’utente Linux il giorno stesso (i file della docroot restano `miles:www-data`); se l’utente sparisce, riassegna l’owner **prima** o i publish successivi falliscono.  
5. Se quella chiave era anche in CI, toglierla dalla CI è follow-up (§14), non silenzioso.

### 12.5 Offboarding secondo admin

Togli quella riga. Fine. Controlla `auth.log` se vuoi l’ultima data di uso.

### 12.6 Offboarding dell’operatore titolare

È un cambio di titolarità, non una riga:

- nuovo umano: proprie chiavi su admin e ops (o nuovo utente ops se non riusi i nomi);
- password sudo admin **nuova**, nel caveau;
- chiavi vecchie tolte da admin, ops, root;
- script e laptop del nuovo titolare puntano all’ops;
- console provider: togliere l’account cloud di chi esce (fuori da questo file, ma va fatto);
- rotazione secret: raccomandata se l’ex titolare poteva leggere `prod.env` o i backup (§10.3), non obbligatoria da questo protocollo.

---

## 13. Cosa non fare

- Due persone su `User root` per il quotidiano.
- Alias SSH nudo (`Host progetto` → `User root`).
- `NOPASSWD: ALL` su admin o ops.
- Mettere il publisher nel gruppo `docker` o `sudo` o `www-data`.
- Far girare deploy/backup sull’alias admin.
- `PermitRootLogin no` prima di admin verde **e** console provider usata.
- Togliere la chiave del publisher da root prima che il canale ristretto sia verde.
- Svuotare `/root/.ssh/authorized_keys` dopo il flip.
- Fidarsi dello script come controllo (path «giusto», assenza di `rm -rf`, code review) al posto di permessi + ForceCommand.
- `chown -R www-data` sui tree ops, o `755`/`644` «perché nginx».
- Tenere il `.env` dell’app nel tree che un rsync può cancellare.
- Unire tutti gli env in una sola voce del password manager.
- Incollare env, chiavi private, `authorized_keys` intere in ticket/chat.
- Dare al secondo umano di emergenza anche ops o publisher «così è più veloce».
- Usare openrsync di macOS contro `rrsync`.
- Destinazione rsync con path assoluto quando la jail è `rrsync` (annida).
- `ChrootDirectory` SFTP come sostituto improvvisato di `rrsync`.
- Chiamare l’utente Linux `root` / `deploy` / `landing` e l’alias in un altro modo: gli script sbagliano User.

---

## 14. Fuori da questo protocollo (follow-up)

Non sono passi di questa guida. Si aprono a parte se servono:

- rotazione secret applicativi e chiavi object storage / pagamenti / AI;
- pipeline CI/CD del publisher (stesso utente e stessa ForceCommand; niente SSH umano);
- hardening di rete (firewall cloud, `ufw`/`nftables`, fail2ban, VPN, cambio porta);
- AllowUsers / cifrari SSH / `PasswordAuthentication no` se non è già così;
- auditd, SIEM, centralizzazione log;
- privilegi Docker (rootless, user namespace) e rischio escape dal socket;
- patch OS, backup strategy, cifratura disco;
- sicurezza dell’applicazione (auth, CSRF, rate limit).

---

## 15. Checklist copia-incolla

Usa i nomi reali al posto degli esempi. Non spegnere root finché i box 1–4 (e 5 se c’è il publisher) non sono verdi.

**0 — Inventario**

- [ ] Chiavi su root (e utenti cloud) elencate: commento + fingerprint, niente paste in chat
- [ ] Chiaro chi è chi; nessuna chiave condivisa tra due umani su root
- [ ] Secret di app copiati fuori dal tree se oggi stanno in un path rsync-abile
- [ ] Console del provider: login riuscito almeno una volta

**1 — Admin (`legatus`)**

- [ ] Utente creato, home `700`, gruppo `sudo` (o `wheel`)
- [ ] Password nel password manager, voce dedicata
- [ ] Pubblica dell’operatore in `authorized_keys`
- [ ] Alias laptop `…-legatus`, `User` = stesso nome, `IdentitiesOnly yes`
- [ ] Sessione nuova: `whoami` = admin; `sudo whoami` chiede password e dà `root`
- [ ] `sudo -l` senza `NOPASSWD: ALL`

**2 — Ops (`centurio`) se serve**

- [ ] Utente creato, home `700`, gruppi `docker` e se serve `www-data`, **non** `sudo`
- [ ] Tree app `owner:www-data` `750`/`640` (non `www-data:www-data`)
- [ ] Env: dir `750` `root:ops`, file `640` `root:ops`, fuori dal tree; `chattr +i` se il FS lo consente
- [ ] Ops legge l’env; publisher (se esiste) no
- [ ] Sudoers solo wrapper assoluti, `visudo` ok; wrapper `root:root` `755`
- [ ] Chiave operatore su ops; chiave git di pull in home ops se serve
- [ ] Sessione nuova: `docker ps` ok; `sudo -i` no; maintenance on/off ok
- [ ] Script laptop: `REMOTE_HOST` = alias ops; smoke deploy/restart + health

**3 — Publisher (`miles`) se serve**

- [ ] Docroot confermata sul vhost (`nginx -T` o equivalente)
- [ ] Utente creato, home `700`, **solo** gruppo proprio
- [ ] Docroot `publisher:www-data` dir `2750`, file `640`
- [ ] Wrapper `rrsync -wo` + log; `authorized_keys` con `command=` e `no-pty` / no-forwarding
- [ ] Drop-in sshd `Match User` + `ForceCommand` stesso wrapper; `sshd -t` ok
- [ ] Client: GNU rsync, dest `alias:`, niente `chown`/`scp`/`rm -rf`
- [ ] Affermazione: probe HTTP 200 sulla statica, poi rimosso
- [ ] Negazione: dest sul tree API → niente file lì; `User root` del publisher ancora da tagliare solo dopo questo verde
- [ ] Chiave publisher tolta da root (backup `authorized_keys` fatto); `User root` suo = denied

**4 — Root SSH**

- [ ] Box 1 verde; box 2 verde se ops esiste; box 3 verde se publisher esiste
- [ ] Console provider già usata in questa campagna
- [ ] Backup `sshd_config`; `PermitRootLogin no`; `sshd -T` = `no`; restart
- [ ] Sessione admin tenuta aperta finché una **nuova** admin (e ops) non entra
- [ ] `ssh` root = denied; alias nudi e `…-root` tolti dal laptop
- [ ] `/root/.ssh/authorized_keys` **non** svuotato

**5 — Igiene continua**

- [ ] Nessuno script punta all’admin o a root
- [ ] Nessun `NOPASSWD: ALL`
- [ ] Secondo umano (se c’è): solo riga su admin, inventario fingerprint, niente ops/publisher
- [ ] Offboarding = togliere la riga giusta, poi prova di negazione
- [ ] Rotazione secret e CI publisher: non mescolate a questa checklist; aprirle solo se c’è una decisione esplicita

---

## 16. Come si usa questo file

Si applica **un** server alla volta, nell’ordine di §5. Si spuntano i box di §15. Non si salta il publisher «perché è urgente» togliendo root. Non si salta l’admin «perché tanto c’è ops».

Se il prompt è `root@...` e non sei nella console del provider, sei nel canale sbagliato.

I nomi romani sono un’abitudine di squadra, non un requisito di sicurezza. I permessi sì.
