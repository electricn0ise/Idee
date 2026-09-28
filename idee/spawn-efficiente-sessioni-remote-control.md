# Idea: spawn efficiente di sessioni Claude Code con Remote Control locale

## Problema

Claude Code supporta già `claude remote-control`: un comando lanciato in locale
(sul proprio PC, in una cartella di progetto) che espone quella sessione alla
Claude Code app, così da poterla guidare da remoto (telefono, browser, altra
macchina). Il problema è che oggi questo è un gesto manuale e 1:1 — un
terminale, una cartella, un comando — quindi non scala bene quando si vogliono
tenere pronte più sessioni locali (progetti diversi, branch diversi, task
diversi) da poter riprendere o avviare al volo da remoto senza dover essere
fisicamente al PC a digitare i comandi.

Obiettivo dell'idea: un modo efficiente per **spawnare, tenere vive e
identificare** più sessioni `claude remote-control` locali, così che da remoto
sia immediato scegliere "quale PC / quale progetto / quale task" riprendere in
mano.

## Proposta

Un piccolo **fleet manager locale** (demone/CLI, es. `ccr-fleet`) che gira sul
PC dell'utente e si occupa di:

1. **Manifest dichiarativo delle sessioni**
   Un file (`fleet.yaml`) che elenca le sessioni desiderate:
   ```yaml
   sessions:
     - name: sito-lavoro
       dir: ~/progetti/sito
       branch: main
       autostart: true
     - name: idee-personali
       dir: ~/progetti/idee
       branch: dev
       autostart: false
       idle_timeout: 30m
   ```
   Idea analoga a un `docker-compose.yml`, ma per sessioni Claude Code in
   Remote Control.

2. **Process supervisor**
   Il demone avvia un `claude remote-control` per ogni voce con
   `autostart: true` (via systemd user unit su Linux, launchd su macOS,
   Windows service/Task Scheduler su Windows), ne monitora il PID, lo
   riavvia se crasha, e ne salva i log separatamente per sessione.

3. **Naming/labeling per il riconoscimento da remoto**
   Ogni sessione viene registrata con un nome leggibile
   (`hostname:progetto[:branch]`) invece del nome auto-generato attuale,
   così nella Claude Code app / lista sessioni risulta subito chiaro quale
   PC e quale cartella si sta guardando quando ce ne sono molte.

4. **Spawn on-demand da remoto**
   Le voci con `autostart: false` non consumano risorse finché non
   servono: da remoto si manda un comando ("avvia la sessione
   idee-personali") che il demone locale riceve (via un piccolo canale —
   webhook locale, polling, o lo stesso meccanismo di Remote Control) e
   traduce nell'avvio del processo corrispondente, che compare poco dopo
   nella app.

5. **Auto-stop per efficienza**
   Sessioni idle oltre `idle_timeout` vengono terminate automaticamente
   per liberare CPU/RAM, mantenendo però lo stato di lavoro (branch, dir,
   history) così da poter essere "riaccese" identiche al bisogno.

6. **Isolamento per evitare conflitti**
   Se più sessioni puntano allo stesso repo, usare git worktree per dare a
   ciascuna sessione una working copy dedicata, evitando che due sessioni
   Claude Code si pestino i piedi sullo stesso checkout.

## Rischi / domande aperte

- **Sicurezza**: esporre più sessioni locali al controllo remoto amplia la
  superficie d'attacco; serve un modo per revocare l'accesso remoto a una
  singola sessione senza spegnere le altre.
- **Limiti di risorse**: quante sessioni `claude remote-control` è
  ragionevole tenere vive insieme su una macchina normale (CPU/RAM per
  processo)?
- **Cross-platform**: il process supervisor deve funzionare in modo
  affidabile su Linux/macOS/Windows senza richiedere setup complesso
  all'utente.
- **UX di scoperta**: come si presenta all'utente, nella Claude Code app,
  la lista di sessioni "spawnabili on-demand" ma non ancora attive?

## Prossimi passi (per una sessione di implementazione)

1. Validare se esiste già un meccanismo ufficiale di Claude Code per
   registrare/avviare `remote-control` in modo scriptabile (flag, API, o
   solo interattivo).
2. Prototipo del fleet manager come script CLI (bash o Node) che legge
   `fleet.yaml` e gestisce solo l'autostart + naming, senza demone
   persistente, per validare l'idea end-to-end prima di costruire il
   supervisor completo.
3. Definire il formato di naming e verificare come appare nella Claude
   Code app.
