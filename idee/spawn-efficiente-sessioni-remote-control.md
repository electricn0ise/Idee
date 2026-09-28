# Idea: spawn efficiente di sessioni Claude Code con Remote Control locale

## Problema (workflow reale attuale)

Per operare da remoto oggi il workflow è: aprire VS Code, aprire un
terminale, lanciare `claude`, digitare `/rc` per abilitare il Remote
Control su quella sessione. Per seguire più progetti/task in parallelo
questo si ripete N volte → N terminali VS Code aperti in contemporanea,
ciascuno con la propria sessione Claude Code in remote control.

Problemi concreti di questo approccio:
- è manuale e ripetitivo per ogni nuovo progetto/task;
- N terminali aperti sono difficili da tenere in ordine e pesano sulla
  macchina;
- **soprattutto**: per aprire una sessione *nuova* bisogna comunque essere
  fisicamente al PC — Claude Code non offre oggi un modo per far partire
  da remoto una sessione locale che non esiste ancora. Il Remote Control
  permette di *guidare* una sessione già avviata, non di *avviarne* una
  nuova dal telefono.

Obiettivo dell'idea: un modo semplice per **aprire sessioni anche da
remoto**, non solo controllare quelle già aperte, senza dover moltiplicare
terminali VS Code gestiti a mano.

## Proposta principale (semplice, nessuna infrastruttura nuova)

Usare **una sola sessione "orchestratore"**, sempre accesa con `/rc`, il
cui unico compito è aprire le altre sessioni per conto dell'utente:

1. Si lascia una sessione Claude Code fissa (es. nella cartella
   `~/orchestratore`) sempre in remote control. È l'unico terminale che
   serve tenere aperto in modo permanente.
2. Da remoto (telefono/app) si chiede a questa sessione, in linguaggio
   naturale, di aprire un progetto: "aprimi una sessione su
   `~/progetti/sito`". L'orchestratore esegue qualcosa come:
   ```bash
   tmux new-session -d -s sito 'cd ~/progetti/sito && claude'
   ```
   usando **tmux** (o `screen`) così la nuova sessione sopravvive anche
   se il terminale/finestra da cui è stata lanciata viene chiuso.
3. Se si vuole guidare *direttamente* anche quella nuova sessione (non
   solo tramite l'orchestratore), basta far eseguire all'orchestratore lo
   stesso comando ma con `/rc` incluso nel prompt iniziale, oppure
   allegare `claude remote-control` invece di `claude` nel comando tmux.
4. Per riprendere il controllo diretto di una sessione già aperta in
   tmux: `tmux attach -t sito` (da un terminale locale) — l'orchestratore
   stesso può anche solo *pilotare* quella sessione via tmux
   (`tmux send-keys`) senza che l'utente debba aprirne un'altra a mano.

**Perché serve tmux (o equivalente) e non un semplice `&` in background:**
un processo lanciato solo in background dalla shell dell'orchestratore è
figlio di quella chiamata di comando e tende a morire (SIGHUP) quando la
chiamata termina, oltre a non avere un vero terminale (pty) a cui Claude
Code, essendo una CLI interattiva, si aspetta di essere agganciato. tmux
risolve entrambi i problemi: fornisce un pty persistente e indipendente
dal processo che l'ha avviato, quindi la sessione sopravvive e resta
riattaccabile. `screen` è un'alternativa equivalente; un servizio
systemd/launchd con pty dedicato è più solido ma più complesso da
configurare.

Vantaggi: zero nuovi servizi da installare, zero problemi di sicurezza
aggiuntivi (si usa lo stesso meccanismo di Remote Control già esistente,
solo un'unica volta come "punto di ingresso"), e risolve esattamente il
problema riportato — aprire sessioni da remoto — con quello che c'è già
oggi in Claude Code + tmux.

Limite: l'orchestratore deve restare sempre acceso (un piccolo processo
sempre vivo sul PC); se il PC è spento o senza rete, non si può comunque
aprire nulla di nuovo da remoto (limite fisico, non risolvibile lato
software).

## Estensione futura (se il pattern manuale sopra diventa scomodo)

Se in futuro servisse qualcosa di più strutturato di un semplice comando
detto in linguaggio naturale all'orchestratore, l'idea può evolvere in un
piccolo **fleet manager locale** con un manifest dichiarativo delle
sessioni da tenere pronte:

```yaml
sessions:
  - name: sito-lavoro
    dir: ~/progetti/sito
    autostart: true
  - name: idee-personali
    dir: ~/progetti/idee
    autostart: false
    idle_timeout: 30m
```

con naming leggibile (`hostname:progetto`), auto-stop delle sessioni
idle per liberare risorse, e isolamento via `git worktree` quando più
sessioni toccano lo stesso repo. Questa parte resta un'evoluzione
opzionale: il pattern "orchestratore + tmux" sopra è già sufficiente a
risolvere il problema descritto senza costruire nulla.

## Rischi / domande aperte

- **Sicurezza**: l'orchestratore, essendo sempre acceso in remote
  control, ha di fatto accesso shell completo al PC da remoto — è lo
  stesso livello di rischio di qualunque sessione Claude Code in remote
  control oggi, ma va tenuto presente essendo permanente.
- **Affidabilità**: se il processo `claude` dell'orchestratore crasha,
  serve un modo per farlo ripartire da solo (systemd user unit / launchd
  agent) senza intervento manuale.
- **Scoperta**: come si riconoscono a colpo d'occhio, nella Claude Code
  app, le sessioni tmux aperte dall'orchestratore rispetto a quelle
  avviate a mano.

## Prossimi passi (per una sessione di implementazione)

1. Verificare in pratica che una sessione Claude Code possa lanciare
   comandi `tmux new-session` e che le sessioni tmux risultanti restino
   vive e riattaccabili dopo la chiusura del terminale originale.
2. Provare il giro completo: chiedere all'orchestratore via Remote
   Control (dal telefono) di aprire un progetto e verificare che compaia
   una nuova sessione controllabile.
3. Solo se il pattern manuale risulta insufficiente, valutare il fleet
   manager descritto come estensione futura.
