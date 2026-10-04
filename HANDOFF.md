# Handoff: rilevamento "qualcuno è a letto" in Home Assistant

> Stato: **hardware deciso, in attesa di acquisto e verifica sul campo. Poi pronta per la sessione di implementazione.**
> Nessuna implementazione è stata eseguita in questo branch, solo progettazione.

## 1. Obiettivo

Avere in Home Assistant (HA) uno stato affidabile e riutilizzabile che dica:

- **se qualcuno è a letto** (almeno una persona),
- **se tutti sono a letto** (utile per "buonanotte" / arma allarme),
- **quale lato è occupato** (best effort, vedi sez. 5: su letto matrimoniale la separazione dei lati è la parte meno certa),
- in una seconda fase, **a letto vs. sta davvero dormendo**.

Lo stato deve essere *locale* (nessun cloud), *robusto* (niente falsi cambi di stato quando ci si gira nel sonno) e *consumabile* da automazioni, scene e dashboard senza che ognuna riscriva la logica.

## 2. Contesto della casa (dal sistema Notion dell'utente)

- Letto **matrimoniale a doghe**. Impianto HA con **ZHA (Zigbee)**. Nessun ESP32/ESPHome posseduto: va acquistato.
- Sensore già installato: **Sonoff SNZB-06P** (radar a microonde 5.8 GHz, Zigbee, alimentato via USB-C), entità `binary_sensor.sonoff_snzb_06p`, friendly name "Presenza Camera", montato nella nicchia sopra il letto, estremità sud, vicino al comodino sud.
  - Espone (da verificare nelle entità del dispositivo) solo presenza on/off: **vede la camera, non il letto** e non distingue lati o persone.
- **Vincolo critico:** `binary_sensor.sonoff_snzb_06p` è già usato dal progetto **"Sveglia potenziata"**:
  - *failsafe:* nessun audio parte se il sensore non è `on`;
  - *dismiss:* uscire dalla camera (`off`) + arrivo in sala (`binary_sensor.sonoff_snzb_06p_2` → `on`) entro 3 minuti.

  Quindi **non rinominarlo, non cambiarne sensibilità/timeout, non riadattarlo** a "letto". Se serve una modifica, prima analisi d'impatto sulle automazioni `sveglia_potenziata_*`. Il rilevamento del letto usa **entità nuove**.
- Il sensore di camera si riusa solo come **controllo incrociato** (sez. 5, regola 6).

## 3. Decisione hardware

**Scelta: sensori di pressione FSR (Force Sensitive Resistor) sulle doghe, letti da un ESP32 con ESPHome.**

Perché questa scelta e non le altre:

| Opzione | Esito |
|---|---|
| **FSR su doghe + ESP32/ESPHome** | **Scelta.** Misura direttamente il peso sul materasso, quindi "a letto" a prescindere da come si sta. Nessun falso positivo da seduti o in piedi accanto al letto. Le esperienze della community HA indicano FSR affidabili e senza drift, a differenza di estensimetri/celle di carico che richiedono ritaratura per le variazioni delle doghe di legno con temperatura/umidità. |
| Celle di carico (HX711) | Scartata: ritaratura periodica, e su letto matrimoniale con doghe la risoluzione per lato è dubbia. |
| Tappetini a pressione | Scartata: per esperienze community si comprimono e finiscono per dare falsi trigger, e costano di più. |
| Radar con zone (Aqara FP2, LD2450) | Scartata come sensore principale: stessa famiglia del sensore già in casa; per l'FP2 alcuni utenti HA riportano difficoltà a rilevare chi dorme e una precisione sull'assenza molto inferiore a quella sulla presenza. |
| Secondo SNZB-06P o simile | Scartata: darebbe lo stesso segnale binario di camera. |

**Lista acquisti (quantità consigliate, prezzi non verificati, ordine di grandezza: poche decine di euro in tutto):**
- 1 × scheda ESP32 (con Wi-Fi e USB; usare pin ADC1 per la lettura analogica perché gli ADC2 non sono utilizzabili con Wi-Fi attivo).
- FSR **a larga area** (strisce o quadrati grandi, non i piccoli dischi da 1 cm): **almeno 2 per lato, 4 totali; conviene comprarne 6 per margine** e per provare posizioni diverse.
- Resistori fissi per il partitore di tensione di ciascun FSR (valore da scegliere in fase di taratura), cavi, nastro adesivo di carta / biadesivo, alimentatore USB per l'ESP32 vicino al letto.
- Verificare la copertura Wi-Fi in camera.

**Montaggio (da provare, l'implementazione decide):** FSR fissati alle doghe con nastro di carta o sul lato inferiore delle doghe, uno o due per lato nella zona busto/fianchi. Se il segnale è debole o dipendente dalla posizione, provare l'altra posizione e/o parallelo di più FSR.

## 4. Principio di progetto

Sensor fusion a livelli: i sensori grezzi non vengono mai usati direttamente nelle automazioni. Si costruisce una catena di entità derivate, e le automazioni leggono solo l'ultimo livello.

```
 FSR per lato ──► occupazione per lato ──► aggregato casa ──► (fase 2) "dormendo"
 (ESP32/ESPHome)   (soglia + debounce)       (group any/all)     (+ orario, luci, tempo a letto)
                         ▲
       controllo incrociato con Presenza Camera (sonoff_snzb_06p)
```

### Modello delle entità (target)

Livello 1: grezzi (ESPHome)
- `sensor.letto_sx_pressione`, `sensor.letto_dx_pressione` (valore analogico, un valore per lato).

Livello 2: occupazione per lato
- `binary_sensor.letto_sx_occupato`, `binary_sensor.letto_dx_occupato`
- Helper **Threshold** con isteresi (soglia alta/bassa distanziate) sui sensori di pressione.
- **Debounce asimmetrico**: entrata veloce (pochi secondi), uscita lenta (decine di secondi/minuti), così girarsi nel sonno o un attimo in bagno non produce transizioni. Siccome l'hardware è ESPHome, farlo a monte con i filtri `delayed_on` / `delayed_off` del binary sensor. Non assumere cosa permettano i Template Helper dalla UI.

Livello 3: aggregati casa
- `binary_sensor.qualcuno_a_letto` → helper **Group** (binary sensor), "any on".
- `binary_sensor.tutti_a_letto` → helper **Group**, "all on" (richiede di sapere quante persone sono attese: vedi domande aperte).
- opzionale `sensor.persone_a_letto` → Template Helper (creato da UI/config flow) che conta i lati occupati.

Livello 4 (fase 2): "dormendo"
- `binary_sensor.sta_dormendo` = `qualcuno_a_letto` da più di N minuti **e** fascia oraria notturna (helper **Schedule**) **e** luci camera spente.
- `sensor.tempo_a_letto` con `history_stats` sul binary sensor aggregato.

## 5. Regole di robustezza (non negoziabili)

1. **`unavailable`/`unknown` non è "fuori dal letto".** Se un sensore o l'ESP32 è offline, l'aggregato deve diventare `unknown`, non `off`. Le automazioni di sicurezza (arma allarme, spegni tutto) non devono scattare su `unknown`.
2. **Isteresi sempre** sui segnali numerici; mai confrontare `> X` secco.
3. **Calibrazione:** la soglia dipende da materasso, doghe e posizione degli FSR. Va documentata e modificabile (opzione dell'helper o `input_number`), non cablata.
4. **Animali/bambini:** decidere se contano. Con la sola pressione un animale pesante può superare la soglia: usare soglia in peso relativa e/o il controllo incrociato.
5. **Lati su letto matrimoniale:** la separazione per lato non è garantita (la community segnala che la risoluzione sulle doghe è la parte difficile). **Criterio minimo di successo = `qualcuno_a_letto` affidabile.** I singoli lati sono best effort: se la separazione non è pulita dopo la taratura, tenere solo l'aggregato e non vincolare automazioni ai singoli lati.
6. **Controllo incrociato con la camera:** se il letto risulta occupato ma `binary_sensor.sonoff_snzb_06p` è `off` per più di un tempo ragionevole, trattare come anomalia (aggregato `unknown`), non come "a letto". L'inverso (camera `on`, letto vuoto) è normale.
7. **Una sola fonte di verità:** le automazioni usano `binary_sensor.qualcuno_a_letto` / `tutti_a_letto`, mai i sensori grezzi.
8. **Non toccare** `binary_sensor.sonoff_snzb_06p` né le automazioni `sveglia_potenziata_*` come effetto collaterale di questo progetto (sez. 2).

## 6. Linee guida HA da rispettare in implementazione

- Preferire **helper nativi** (Threshold, Group, History Stats, Schedule) a template sensor. Dove serve un template (es. conteggio), usare un **Template Helper** creato dalla UI/config flow, non YAML.
- Condizioni e trigger **nativi** (`state` con `for:`, `numeric_state`, `time`), niente `condition: template` dove esiste l'alternativa.
- Riferimenti via **`entity_id`, non `device_id`**.
- **Modo automazione:** `restart` per automazioni con timer che devono resettarsi; `single` per notifiche una tantum.
- HA è un sistema remoto: gestire la configurazione tramite API/config flow, non cercando o modificando file sul filesystem locale, e non chiedendo di modificare `configuration.yaml` per integrazioni configurabili da UI.
- Prima di rinominare entità già esistenti, fare l'analisi d'impatto (dashboard, script, scene, config entry di altri helper).
- Convenzioni di progetto dell'utente (da Notion): etichette HA per sotto-progetto, backup prima di modifiche rilevanti, mai cancellare helper orfani. Verificarle prima di creare entità.

## 7. Criteri di accettazione

- [ ] Sdraiandosi sul letto, `qualcuno_a_letto` passa a `on` entro pochi secondi, a prescindere da dove ci si mette sul materasso.
- [ ] Sedersi sul bordo o stare in piedi accanto al letto **non** porta a `on`.
- [ ] Girarsi o alzarsi per meno di N minuti **non** porta a `off`.
- [ ] Con due persone, `tutti_a_letto` è `on` solo quando entrambe sono a letto (se i lati sono separabili, altrimenti vedi regola 5).
- [ ] ESP32 offline ⇒ aggregato `unknown`, nessuna automazione di sicurezza scatta.
- [ ] Letto occupato con camera `off` a lungo ⇒ `unknown` (regola 6).
- [ ] "Sveglia potenziata" continua a comportarsi esattamente come prima (nessuna regressione).
- [ ] Almeno un'automazione dimostrativa end-to-end, es.: tutti a letto ⇒ luci spente + riscaldamento in modalità notte.
- [ ] Una card dashboard con stato aggregato, lati (se affidabili) e tempo a letto.
- [ ] Documentato: hardware, posizionamento degli FSR, valori di soglia/ritardi e come ricalibrare.

## 8. Domande aperte

Già risolte:
- ~~Hardware posseduto~~ → solo SNZB-06P (camera); ESP32/FSR da comprare.
- ~~Tipo di letto~~ → matrimoniale a doghe.
- ~~Preferenza contatto/non contatto~~ → indifferente, conta il risultato.

Ancora aperte:
1. **Quante persone dormono nel letto** (una sola o due?). Decide se `tutti_a_letto` ha senso o coincide con `qualcuno_a_letto`.
2. **Animali** che salgono sul letto?
3. **Cosa deve far scattare** lo stato (luci, riscaldamento, allarme, aspirapolvere)? Influisce su ritardi e gestione di `unknown`.
4. **Accesso API/MCP a HA** per creare gli helper dall'implementazione, altrimenti gli step diventano istruzioni manuali.
5. **Fase 2 ("dormendo")** subito o dopo? Se dopo, implementare solo i livelli 1-3.
6. **Integrazione con "Sveglia potenziata":** in futuro il sensore letto potrebbe rafforzare il suo dismiss/failsafe ("a letto" è più preciso di "in camera"). Fuori da questo progetto: se si vuole, va deciso a parte con le sue regole di modifica.

## 9. Piano di implementazione suggerito

1. Chiudere le domande aperte (sez. 8).
2. Acquistare l'hardware (sez. 3) e fissare gli FSR sulle doghe; provare più posizioni.
3. Flashare l'ESP32 con ESPHome, verificare che i valori compaiano in HA e osservarli a letto vuoto / occupato / seduti sul bordo / due persone.
4. Calibrare soglie e ritardi; decidere se i lati sono separabili (regola 5).
5. Creare gli helper di livello 2 (Threshold + debounce ESPHome) e il controllo incrociato con la camera.
6. Creare i Group di livello 3.
7. Automazione dimostrativa e card dashboard.
8. (Fase 2) Schedule + History Stats + `sta_dormendo`.
9. Test dei casi limite della sez. 7 (inclusa l'assenza di regressioni su "Sveglia potenziata") e documentazione finale.

## 10. Rischi noti

- **Posizione degli FSR:** il segnale dipende da dove sono sulle doghe e dal materasso. Comprarne in più e provare.
- **Separazione dei lati su matrimoniale:** può risultare poco pulita; il piano prevede un ripiego (solo aggregato).
- **Debounce di uscita:** troppo lungo ritarda le automazioni mattutine, troppo corto crea falsi "fuori dal letto" di notte. Partire da valori prudenti e regolare sull'uso reale.
- **Wi-Fi in camera:** un ESP32 offline degrada tutto a `unknown`; verificare la copertura.
- **Regressioni su "Sveglia potenziata":** il sensore di camera è già in produzione per quel progetto; ogni intervento va validato su quel fronte.
