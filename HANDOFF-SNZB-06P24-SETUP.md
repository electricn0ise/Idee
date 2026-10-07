# Handoff per la sessione locale: messa in servizio del sensore SNZB-06P24 sopra il letto

> Destinatario: sessione Claude Code **locale** (con accesso a Home Assistant via API/MCP e, se disponibile, a Notion) che lavora **insieme all'utente**, presente fisicamente in camera.
> Contesto completo e storico delle decisioni: `HANDOFF.md` nello stesso branch (`claude/dazzling-wright-p6vc4a`). Questo file contiene solo ciò che serve per eseguire il lavoro.
> Stato: il sensore è **arrivato** (2026-10-07). Nessuna configurazione è stata ancora fatta. Lingua con l'utente: italiano.

## 1. Obiettivo

Mettere in servizio un **Sonoff SNZB-06P24** (radar 24 GHz, Zigbee, ZHA) nella nicchia sopra il letto, in modo da avere un'entità HA nuova che dica **"qualcuno è a letto"**, e **verificare con prove fisiche** se è abbastanza affidabile.

**Requisito dell'utente (unico criterio qualitativo):** non serve distinguere seduto da sdraiato; basta che stare **in prossimità del letto** (in piedi accanto, camminare, vestirsi) **non** attivi il sensore, e che chi dorme fermo **resti** rilevato.

Questo lavoro finisce con una **decisione go/no-go** (sez. 7). La costruzione della catena di entità finale (sez. 8) si fa solo dopo un "go" esplicito dell'utente.

## 2. Vincoli non negoziabili

1. **Non toccare** `binary_sensor.sonoff_snzb_06p` (friendly name "Presenza Camera", SNZB-06P a 5,8 GHz, stessa nicchia, estremità sud vicino al comodino sud) né le automazioni/script `sveglia_potenziata_*`. Il progetto "Sveglia potenziata" lo usa per il failsafe audio (nessun suono se non è `on`) e per il dismiss (uscire dalla camera = `off`, poi arrivo in sala `binary_sensor.sonoff_snzb_06p_2` → `on` entro 3 minuti). Nessuna modifica di nome, sensibilità, timeout. Se qualcosa lo richiedesse: fermarsi e chiedere all'utente.
2. Il nuovo sensore ha **entità proprie e nome distinto** (proposta: friendly name "Presenza Letto").
3. Non rinominare entità esistenti, non cancellare helper orfani (regola di progetto dell'utente), fare **backup** prima di modifiche rilevanti come l'utente fa negli altri sotto-progetti.
4. Linee guida HA dell'utente: helper nativi (Group, Threshold, Schedule, History Stats) prima dei template; Template Helper da UI/config flow, non YAML; trigger/condizioni nativi; `entity_id` non `device_id`; modo `restart` per timer che si azzerano. Gestire la configurazione via API/config flow, non cercando file sul disco (eccezione: il file del quirk, se necessario, sez. 4 passo 2).
5. Questo task è **di sola configurazione e prova**. Nessuna automazione che agisca sulla casa (luci, allarme, riscaldamento) in questa fase.

## 3. Fatti sul dispositivo (da ricerche web, non verificati sul dispositivo reale: confermare)

- 24 GHz, Zigbee, alimentato presumibilmente via USB-C 5 V (come l'SNZB-06P; **non verificato** per il 06P24: controllare cosa è arrivato nella confezione).
- Portata fino a 4 m, **7 zone indipendenti**: zona 1 = 0–1 m, zone 2–7 a incrementi di 0,5 m (zona 2 = 1–1,5 m, zona 3 = 1,5–2 m, ...), ciascuna attivabile/disattivabile.
- Sensibilità da −6 a +6; "occupancy timeout" (ritardo prima di "libero"); calibrazione "spatial learning" per filtrare interferenze fisse; base magnetica con testa orientabile.
- Le recensioni sulla sensibilità sono miste (un thread del forum eWeLink la giudica scarsa); per una persona ferma funziona meglio entro 3 m e rivolta verso il sensore.
- **ZHA:** serve un quirk. Esiste una pull request in `zigpy/zha-device-handlers` (n. 4907, aperta il 7 aprile 2026, **stato di merge non verificato**) che aggiunge sensibilità fine, calibrazione e interruttori di abilitazione per zona. Sonoff dichiara anche un quirk proprio da installare.
- Geometria stimata (altezza materasso-soffitto nicchia ≈ **1 m**, montaggio rivolto in basso): torace di chi è sdraiato ≈ 0,75 m dal modulo, materasso ≈ 1,0 m, pavimento accanto al letto oltre ≈ 1,5 m. Chi è in piedi accanto al letto ha testa e busto sopra o oltre il bordo del cono, quindi in gran parte fuori fascio (da verificare).

## 4. Passi di messa in servizio

Eseguire in ordine; ogni passo ha una verifica.

1. **Verifica preliminare di sicurezza.** Annotare lo stato attuale di `binary_sensor.sonoff_snzb_06p` e delle automazioni `sveglia_potenziata_*` (nessuna modifica; serve da riferimento di non regressione). Verificare che non ci sia una sveglia armata.
2. **Quirk ZHA.** Versione di HA e di ZHA in uso. Controllare se il 06P24 è già supportato nativamente (cercare le entità al passo 3). Solo se mancano zone e sensibilità fine: installare il quirk (da PR 4907 o dal file Sonoff), il che richiede un file nella cartella config di HA e l'attivazione dei quirk personalizzati di ZHA, più riavvio. **Chiedere conferma all'utente prima del riavvio** e fare backup.
3. **Abbinamento.** Mettere ZHA in modalità aggiunta, abbinare il sensore, **elencare tutte le entità create** (occupancy, illuminance, timeout, sensibilità, switch per zona, calibrazione, ...) e mostrarle all'utente. Rinominare il dispositivo "Presenza Letto" (nome solo del nuovo dispositivo). Assegnare un'etichetta HA coerente con le convenzioni dell'utente (es. `presenza_letto`).
4. **Montaggio provvisorio (con l'utente).** Soffitto della nicchia, sopra il busto di chi dorme, testa orientata **dritta verso il basso**. Base magnetica o nastro, nessun foro definitivo. Annotare altezza reale dal materasso e posizione (centro / lato).
5. **Configurazione iniziale (valori di partenza, da tarare):** abilitare **zona 1** (0–1 m) ed eventualmente **zona 2** (1–1,5 m); **disabilitare zone 3–7**; sensibilità intermedia o bassa (partire da 0 o −2 e salire solo se chi dorme fermo non viene visto); **occupancy timeout lungo** (alcuni minuti; capire l'intervallo consentito).
6. **Calibrazione "spatial learning"** con camera ferma, **letto vuoto**, nessuno nella stanza (l'utente esce e chiude la porta se serve). Registrare esito/stato.

## 5. Prova di accettazione (con l'utente)

Usare uno storico/registro delle entità (logbook o history) per ogni caso e annotare i risultati in una tabella. Ripetere i casi dubbi 2–3 volte.

**Non deve attivarsi (o tornare `off` entro il timeout):**
- Letto vuoto, utente in piedi accanto al letto a **0,5 m**, poi a **1 m**.
- Camminare intorno al letto, vestirsi/spogliarsi vicino, rifare il letto da un lato.
- Persona in camera con letto vuoto (anche porta aperta, qualcuno in corridoio).
- Ventilatore o tende in movimento, se presenti.

**Deve attivarsi e restare attivo:**
- Sdraiato supino, sdraiato su un fianco (entrambi i lati), sotto il piumone.
- **Fermo 10–20 minuti** (simulare il sonno): nessun falso "libero".
- Due persone nel letto, se possibile.

**Deve tornare libero:** dopo che l'utente si alza, entro il timeout impostato + un margine ragionevole.

Per ogni caso: stato osservato, tempi (quanto ci mette ad attivarsi e a liberarsi), eventuali oscillazioni. Se i casi del primo elenco falliscono: ridurre sensibilità e/o restringere le zone (solo zona 1), riprovare; se si può, provare un'altra posizione di montaggio.

## 6. Verifica di non regressione

Al termine, confermare che `binary_sensor.sonoff_snzb_06p`, `binary_sensor.sonoff_snzb_06p_2` e le automazioni `sveglia_potenziata_*` hanno lo stesso stato e la stessa configurazione di prima (stesso `last_updated` del config, nessun riavvio che le abbia lasciate in stato anomalo). Se il riavvio di HA è stato necessario per il quirk, ricordare che il progetto ha già `automation.sveglia_potenziata_ripresa_dopo_riavvio_ha`: controllare che non scatti nulla di indesiderato con la sveglia disarmata.

## 7. Decisione go/no-go (da prendere con l'utente)

- **GO** se i casi "non deve attivarsi" passano in modo ripetibile e il caso "fermo 10–20 minuti" non produce falsi "libero". Si procede alla sez. 8.
- **GO con riserva** se i falsi positivi si risolvono solo con soglie molto restrittive che compromettono la rilevazione del sonno: documentare i compromessi e chiedere all'utente.
- **NO-GO** altrimenti. Ripiego in ordine, da concordare con l'utente (vedi `HANDOFF.md`):
  1. **Celle di carico sotto le gambe del letto** (HX711 + ESP32 con ESPHome). Prima domandare: il letto ha gambe accessibili e quante (anche centrali)? Si solleva da soli o servono due persone? È ancorato al muro o all'armadio a ponte? Peso approssimativo di letto + materasso? (Stima: 50 kg per cella potrebbero non bastare, servono celle da 100 kg o una cella per gamba.)
  2. **Modulo LD2410C con ESP32** (gate da 0,2 m che separano meglio sdraiato da seduto); serve saldatura.
  3. Sensori di pressione Zigbee già pronti (strisce da 80 cm): da verificare compatibilità ZHA e ID produttore.

## 8. Dopo un "GO" esplicito: catena di entità (non iniziare prima)

Proposta, da adattare ai dati reali raccolti:

1. `binary_sensor.presenza_letto` (grezzo del 06P24, nome da concordare).
2. **Debounce:** prima verificare se il solo "occupancy timeout" del dispositivo basta; aggiungere ritardo lato HA solo se serve, con strumenti nativi (non assumere cosa permetta la UI dei Template Helper nella versione in uso).
3. **Aggregato** `binary_sensor.qualcuno_a_letto`, con queste regole:
   - `unavailable`/`unknown` del sensore ⇒ l'aggregato è `unknown`, **mai `off`** (le automazioni di sicurezza non devono scattare su `unknown`).
   - **Controllo incrociato** con `binary_sensor.sonoff_snzb_06p` (Presenza Camera): letto `on` ma camera `off` per più di un tempo ragionevole ⇒ anomalia (`unknown`). Camera `on` e letto `off` è normale.
4. Etichette HA, backup, voce nel catalogo dispositivi dell'utente.
5. Una sola fonte di verità: le automazioni future usano `qualcuno_a_letto`, mai il sensore grezzo. **Cosa deve far scattare** lo stato (luci, riscaldamento, allarme, ...) **è da definire dopo**, non in questa sessione.
6. Fase 2 ("sta dormendo": orario, luci spente, tempo a letto con History Stats) **rimandata** per scelta dell'utente.

## 9. Documentazione da lasciare

- Se hai accesso a Notion: **leggere prima** la pagina "🧠 Sistema Memoria" per le convenzioni dell'utente (database Dispositivi con campi Area, Entity ID, Protocollo, Stato, Note, Sotto-progetto, "Marker mappa"; Sotto-progetti con "Label HA"; Decisioni, Changelog, Conoscenza, Attività). Poi, **chiedendo conferma**, registrare il nuovo sensore in "🔌 Dispositivi", le decisioni (go/no-go e valori scelti) e un changelog. Il sensore di camera esistente compare anche nel modello 3D (SH3D): chiedere all'utente se vuole aggiungere il nuovo pezzo.
- Se non hai accesso a Notion: riportare all'utente un riepilogo da incollare.
- Aggiornare **questo repository** (branch `claude/dazzling-wright-p6vc4a`) con i risultati: tabella della prova di accettazione, valori finali (zone, sensibilità, timeout, posizione), esito go/no-go. Il README del repository richiede che qui ci sia solo documentazione/handoff, non implementazione.

## 10. Cosa chiedere all'utente all'inizio

1. Il sensore è già fuori dalla scatola? Cosa c'era nella confezione (cavo USB, alimentatore, base)? Dov'è una presa vicino alla nicchia?
2. Hai accesso alla cartella config di HA (File editor, Samba o SSH) nel caso serva il quirk?
3. Puoi fare le prove fisiche adesso (letto, camera libera, un'altra persona per il caso "due persone")?
4. Conferma esplicita prima del riavvio di HA, se necessario.
