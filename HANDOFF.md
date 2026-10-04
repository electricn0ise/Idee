# Handoff: rilevamento "qualcuno è a letto" in Home Assistant

> Stato: **idea in brainstorming, pronta per la sessione di implementazione dopo aver risolto le domande aperte (sez. 8).**
> Nessuna implementazione è stata eseguita in questo branch, solo progettazione.

## 1. Obiettivo

Avere in Home Assistant (HA) uno stato affidabile e riutilizzabile che dica:

- **se qualcuno è a letto** (almeno una persona),
- **se tutti sono a letto** (utile per "buonanotte" / arma allarme),
- **quante persone / quale lato** (opzionale, ma lo schema lo rende gratuito),
- in una seconda fase, **a letto vs. sta davvero dormendo**.

Lo stato deve essere *locale* (nessun cloud), *robusto* (niente falsi cambi di stato quando ci si gira nel sonno) e *consumabile* da automazioni, scene e dashboard senza che ognuna riscriva la logica.

## 2. Principio di progetto

Sensor fusion a livelli: i sensori grezzi non vengono mai usati direttamente nelle automazioni. Si costruisce una catena di entità derivate, e le automazioni leggono solo l'ultimo livello.

```
 sensori grezzi ──► occupazione per lato ──► aggregato casa ──► (fase 2) "dormendo"
 (pressione, mmWave)   (soglia + debounce)     (group any/all)      (+ orario, luci, tempo a letto)
```

## 3. Opzioni hardware (da scegliere in base a cosa l'utente ha già)

| Opzione | Affidabilità | Costo / sforzo | Note |
|---|---|---|---|
| **Sensori di pressione/peso sotto il materasso** (ESP32 + celle di carico HX711, oppure FSR/strisce a pressione) con ESPHome | Alta, rileva davvero il peso | Basso-medio, DIY | Un sensore per lato. Il segnale è un numero: serve una soglia con isteresi. Rileva anche animali pesanti. |
| **mmWave di presenza** (Aqara FP2, o ESP32 + LD2410/LD2450 con ESPHome) puntato sul letto | Alta per "presenza ferma", sensibile alla posizione/zone | Medio | Con FP2 si definiscono zone per lato. Il mmWave vede anche chi è seduto sul letto o in piedi accanto se la zona è larga. |
| **Sensori letto commerciali** (es. Withings Sleep Analyzer, Eight Sleep) | Alta | Alto | Spesso cloud, quindi in contrasto col requisito "locale" e dipendente dall'integrazione disponibile. |
| **Euristica senza hardware nuovo** (telefono in carica in camera, Wi-Fi/BLE connesso, luci camera spente, ora) | Bassa, solo come *segnale di supporto* | Zero | Non dice se si è a letto. Va usata solo per rafforzare/smorzare gli altri segnali. |

**Raccomandazione:** pressione per lato come segnale principale, mmWave come conferma/fallback se già presente. Se non c'è hardware, la sessione di implementazione deve prima chiedere all'utente cosa possiede.

## 4. Modello delle entità (architettura target)

Livello 1: grezzi (forniti dall'hardware/integrazione)
- `sensor.letto_sx_pressione`, `sensor.letto_dx_pressione` (valori numerici)
- opzionale: `binary_sensor.letto_sx_mmwave`, `binary_sensor.letto_dx_mmwave`

Livello 2: occupazione per lato
- `binary_sensor.letto_sx_occupato`, `binary_sensor.letto_dx_occupato`
- Da pressione: helper **Threshold** con isteresi (soglia alta/bassa distanziate per evitare flapping).
- Se c'è anche il mmWave: si combinano (es. occupato = soglia pressione ON, mmWave solo come tie-break o fallback quando la pressione è `unavailable`).
- **Debounce asimmetrico**: entrata veloce (pochi secondi), uscita lenta (decine di secondi/minuti) così girarsi nel sonno o un attimo in bagno non produce transizioni. Se l'hardware è ESPHome, farlo a monte con i filtri `delayed_on` / `delayed_off` del binary sensor. Altrimenti con trigger di stato con `for:` che pilotano un helper, scegliendo l'approccio dopo aver verificato cosa permette la UI dei Template Helper nella versione di HA dell'utente (non assumerlo).

Livello 3: aggregati casa
- `binary_sensor.qualcuno_a_letto` → helper **Group** (binary sensor), "any on".
- `binary_sensor.tutti_a_letto` → helper **Group**, "all on" (con le persone attese; vedi domande aperte).
- opzionale `sensor.persone_a_letto` → Template Helper che conta i lati occupati.

Livello 4 (fase 2): "dormendo"
- `binary_sensor.sta_dormendo` = `qualcuno_a_letto` da più di N minuti **e** fascia oraria notturna (helper **Schedule**) **e** luci camera spente.
- `sensor.tempo_a_letto` con `history_stats` sul binary sensor aggregato (per statistiche, dashboard, e per l'automazione di "sveglia dolce").

## 5. Regole di robustezza (non negoziabili)

1. **`unavailable`/`unknown` non è "fuori dal letto".** Se un sensore è offline, l'aggregato deve diventare `unknown`, non `off`. Le automazioni di sicurezza (arma allarme, spegni tutto) non devono scattare su `unknown`.
2. **Isteresi sempre** sui segnali numerici; mai confrontare `> X` secco.
3. **Calibrazione:** la soglia dipende da materasso e sensore. Va documentata e modificabile da un `input_number` o dall'opzione dell'helper, non cablata.
4. **Animali/bambini:** decidere se contano. Con la sola pressione un gatto pesante può superare la soglia; usare soglia in peso o confermare con mmWave.
5. **Una sola fonte di verità:** le automazioni usano `binary_sensor.qualcuno_a_letto` / `tutti_a_letto`, mai i sensori grezzi.

## 6. Linee guida HA da rispettare in implementazione

Derivate dalle best practice native di HA:

- Preferire **helper nativi** (Threshold, Group, History Stats, Schedule) a template sensor. Dove serve un template (es. conteggio), usare un **Template Helper** creato dalla UI/config flow, non YAML.
- Condizioni e trigger **nativi** (`state` con `for:`, `numeric_state`, `time`), niente `condition: template` dove esiste l'alternativa.
- Riferimenti via **`entity_id`, non `device_id`**.
- **Modo automazione:** `restart` per automazioni con timer che devono resettarsi (es. luci notte che si spengono dopo un ritardo); `single` per notifiche una tantum.
- HA è un sistema remoto: gestire la configurazione tramite API/config flow, non cercando o modificando file sul filesystem locale, e non chiedendo di modificare `configuration.yaml` per integrazioni configurabili da UI.
- Prima di rinominare entità già esistenti nella casa, fare l'analisi d'impatto (dashboard, script, scene, config entry di altri helper).

## 7. Casi d'uso che l'idea deve abilitare (criteri di accettazione)

- [ ] Sdraiandosi su un lato, `letto_*_occupato` passa a `on` entro pochi secondi.
- [ ] Girarsi o alzarsi per meno di N minuti **non** porta a `off`.
- [ ] Con entrambi i lati occupati, `qualcuno_a_letto` e `tutti_a_letto` sono `on`; con uno solo, solo il primo.
- [ ] Sensore offline ⇒ aggregato `unknown`, nessuna automazione di sicurezza scatta.
- [ ] Almeno un'automazione dimostrativa end-to-end, ad esempio: tutti a letto ⇒ luci spente + riscaldamento in modalità notte; ultimo che si alza al mattino ⇒ luce soffusa del corridoio.
- [ ] Una card dashboard con stato per lato, aggregato e tempo a letto.
- [ ] Documentato: hardware usato, posizionamento, valori di soglia/ritardi e come ricalibrare.

## 8. Domande aperte (da chiudere all'inizio della sessione di implementazione)

1. **Che hardware ha già?** (sensori di pressione, FP2/mmWave, ESPHome, nessuno?) Determina l'intero livello 1.
2. **Quante persone / lati** e c'è un letto singolo o matrimoniale? Servono persone o animali da escludere?
3. **Versione di HA** e se è disponibile accesso API/MCP per creare gli helper (altrimenti gli step diventano istruzioni manuali in UI).
4. **Cosa deve far scattare** (luci, riscaldamento, allarme, aspirapolvere)? Influisce sulla severità dei ritardi e su cosa fare con `unknown`.
5. **Fase 2 ("dormendo")** serve subito o dopo? Se dopo, implementare solo i livelli 1-3.

## 9. Piano di implementazione suggerito

1. Raccogliere le risposte alla sez. 8.
2. Installare/configurare i sensori grezzi (ESPHome se pressione/LD2410) e verificare che le entità compaiano in HA con valori sensati.
3. Calibrare le soglie osservando i valori a letto vuoto / occupato.
4. Creare gli helper di livello 2 (Threshold + debounce) e verificare il comportamento al volo.
5. Creare i Group di livello 3.
6. Costruire l'automazione dimostrativa e la card dashboard.
7. (Fase 2) Schedule + History Stats + `sta_dormendo`.
8. Test dei casi limite della sez. 7 e documentazione finale.

## 10. Rischi noti

- Posizionamento del sensore: pressione sotto doghe vs. materasso cambia molto il segnale; va provato.
- mmWave: falsi positivi con ventilatori/tende; zone da restringere al solo letto.
- Se il debounce di uscita è troppo lungo le automazioni mattutine arrivano in ritardo; troppo corto crea falsi "fuori dal letto" di notte. Va regolato sull'uso reale, partendo da valori prudenti.
