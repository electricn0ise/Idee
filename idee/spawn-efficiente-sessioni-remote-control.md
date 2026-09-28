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

**Validato in pratica (test su Windows).** Un tentativo reale ha
confermato la teoria: il tool Bash/PowerShell della sessione orchestratore
gira con stdin collegato a `null` e nessun TTY, quindi lanciare `claude`
interattivo da lì fallisce subito o resta bloccato — esattamente il
problema descritto sopra. Su Windows l'equivalente di tmux è aprire una
finestra/processo indipendente con PID proprio e una vera console:
```powershell
Start-Process powershell -ArgumentList '-NoExit','-Command','cd "<cartella>"; claude remote-control'
```
Punto importante: conviene lanciare **direttamente `claude remote-control`**
(non `claude` seguito da `/rc` digitato a mano), così la sessione nasce
già registrata in remote control fin dal primo istante, senza bisogno di
interagire con la finestra dopo l'apertura. Questo elimina anche il falso
problema di "non posso leggere l'output né inviare input a quella
finestra": l'orchestratore non deve pilotare la sessione figlia via
stdin/stdout — la verifica e il controllo avvengono tramite il
meccanismo di Remote Control stesso (la sessione compare nella Claude
Code app ed è lì che la si guida), non tramite l'orchestratore.

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

## Prior art: strumenti già esistenti che risolvono questo problema

Prima di costruire qualunque cosa custom, vale la pena valutare progetti
open source che fanno già esattamente questo:

- **Happy / Happy Coder** — [github.com/slopus/happy](https://github.com/slopus/happy),
  [happy.engineering](http://happy.engineering/). Client mobile/web
  end-to-end criptato per Claude Code e Codex. Sul PC si lancia `happy`
  al posto di `claude`; quando serve controllo da remoto la sessione
  riparte in "remote mode".
- **Happier** — [github.com/happier-dev/happier](https://github.com/happier-dev/happier),
  [happier.dev](https://happier.dev/). Fork/evoluzione più ampia, con
  supporto a 13 agenti (Claude Code, Codex, Cursor, Gemini, ecc.). Qui il
  problema che stiamo descrivendo è già risolto nativamente: si installa
  un **daemon** sempre acceso sul PC (`happier`), e dall'app si preme
  *"New session"*, si sceglie **macchina → cartella → agente**, e il
  daemon locale la avvia — nessun terminale da aprire a mano. In più
  offre creazione automatica di **git worktree** per isolare sessioni
  sullo stesso repo, notifiche push per richieste di permesso, e setup
  anche via SSH su macchine remote (`happier machine setup --ssh
  user@host`).

Il daemon di Happy/Happier gioca esattamente il ruolo del "sessione
orchestratore" descritto sopra, ma pre-costruito, con pairing sicuro
(QR code) e crittografia end-to-end invece di gestione manuale. Per la
maggior parte dei casi conviene adottare uno di questi due invece di
costruire il pattern manuale o il fleet manager: il pattern
"orchestratore + tmux/Start-Process" resta utile solo come soluzione
zero-dipendenze quando non si vuole installare un tool di terze parti.

### Happy vs Happier: quale scegliere

| | Happy / Happy Coder | Happier |
|---|---|---|
| Origine | Progetto originale | Nato come contributo a Happy, poi separato per iterare più veloce |
| Maturità | ~23.9k star, ~2.5k commit — più maturo/rodato | ~1.8k star, ma 10.7k+ commit — sviluppo molto più rapido |
| Agenti supportati | Solo Claude Code + Codex | 30+ agenti via ACP (Claude Code, Codex, Cursor, Gemini, ecc.) |
| Git worktree per sessione | Non menzionato nella documentazione | Sì, nativo |
| Spawn multi-macchina da app | Sì | Sì, pensato esplicitamente per gestire tante sessioni parallele su più macchine |
| App mobile | Più matura/nativa, voice control | Setup guidato (l'app desktop configura CLI e daemon da sola) |
| Licenza | MIT | MIT |

**Raccomandazione per questo caso d'uso (spawnare sessioni multiple da
remoto isolate tra loro):** **Happier**, perché copre nativamente sia lo
spawn on-demand ("New session" → macchina/cartella/agente) sia
l'isolamento tra sessioni sullo stesso repo via git worktree, che è
proprio il rischio segnalato più sopra. Happy resta preferibile solo se
si usa esclusivamente Claude Code/Codex e si preferisce il prodotto più
maturo e con l'app mobile più rodata, senza bisogno di worktree o altri
agenti.

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

1. ~~Verificare in pratica che una sessione Claude Code possa lanciare un
   processo detached indipendente (tmux su Linux/macOS, `Start-Process`
   su Windows) che resti vivo e controllabile dopo la chiusura del
   comando/terminale originale.~~ Confermato su Windows con
   `Start-Process` + `claude remote-control`. Da confermare allo stesso
   modo con tmux/screen su Linux/macOS.
2. Provare il giro completo end-to-end: chiedere all'orchestratore via
   Remote Control (dal telefono) di aprire un progetto con
   `claude remote-control` diretto e verificare che la nuova sessione
   compaia e sia controllabile dalla Claude Code app.
3. Provare **Happier** (o Happy) come soluzione pronta all'uso e
   confrontarla col pattern manuale: se copre già naming, worktree e
   spawn on-demand da remoto, non ha senso costruire un fleet manager
   custom.
4. Solo se Happy/Happier non fossero adatti (per policy di sicurezza,
   necessità di non installare tool di terze parti, ecc.), valutare il
   fleet manager descritto come estensione futura del pattern manuale.

## Guida di setup rapida — Happier

1. **Installa il CLI/daemon sul PC.**
   - Windows (PowerShell): `iwr https://happier.dev/install.ps1 -useb | iex`
   - macOS/Linux: `curl -fsSL https://happier.dev/install | bash`
   - In alternativa: app desktop da `happier.dev/download` (macOS/Windows/Linux),
     configura da sola CLI + daemon.
2. **Pairing col telefono.** Avvia l'app desktop (o il login del CLI): mostra
   un QR code. Installa l'app mobile "Happier" (App Store / Google Play),
   fai login con lo stesso account e scansiona il QR.
3. **Avvia una sessione.**
   - Da terminale locale: `cd <cartella progetto>` poi `happier` al posto
     di `claude`.
   - Da telefono, senza toccare il PC: app → **"New session"** → scegli
     macchina → cartella → agente (Claude Code) → parte da remoto.
4. **Macchine remote via SSH (opzionale).**
   `happier machine setup --ssh user@host`, oppure, se la macchina remota
   non ha accesso browser: `happier auth pair-remote --ssh user@host`.

Nota: questi comandi vanno eseguiti sul PC/telefono dell'utente, non da
una sessione Claude Code Remote in un container cloud (che non ha
accesso a quella macchina).
