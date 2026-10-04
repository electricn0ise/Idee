# Handoff: rilevamento "qualcuno è a letto" in Home Assistant

> Stato: **hardware deciso, in attesa di acquisto e verifica sul campo. Poi pronta per la sessione di implementazione.**
> Nessuna implementazione è stata eseguita in questo branch, solo progettazione.
> **Fase corrente (decisione utente): ci si occupa prevalentemente di scelta, ricerca, valutazione e acquisto dell'hardware** (sez. 3). Logica HA, automazioni e fase 2 restano per dopo.

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

**Candidati concreti e prezzi trovati in ricerca (da riverificare al momento dell'acquisto, cambiano e le disponibilità sono basse):**

| Componente | Candidato | Prezzo trovato | Note |
|---|---|---|---|
| FSR striscia lunga (classica) | Interlink **FSR 408** (24" × 0.4", cioè circa 61 cm × 1,5 cm, area sensibile continua; forza 0,2–20 N) | **Tinytronics (NL) €25,00** IVA incl. (€20,66 senza IVA), a magazzino a Eindhoven; Melopero (IT) €27,97, 2 pezzi; RobotShop EU €20,69; Electrokit (SE) 199 SEK IVA svedese incl.; Adafruit $19,95 (USA, + spedizione/dogana); **Interlink diretto $6,99** per il 610 mm secondo uno snippet di ricerca (+ spedizione USA, IVA/dogana, da verificare al checkout; il fornitore originale evita il rischio di cloni) | Pololu e Pi Hut lo danno come fuori produzione/discontinuato. Interlink la produce anche in lunghezze 50, 100, 200, 300, 400, 500 mm. Tinytronics ha anche la versione da 100 mm a €7,75. |
| FSR striscia lunga (serie UX) | Interlink **FSR UX 408** (forza **0,5–150 N**, confermata da più snippet del sito Interlink; i PDF non erano apribili) | **OpenELAB (EU) €2,75** per il 300 mm, ma il prodotto si chiama **"FSR X 408"**, non UX: nella nomenclatura Interlink la serie X ha range intermedio **0,3–50 N**, quindi è un terzo modello da verificare prima di ordinarlo (negozio spagnolo, spedizione €4,95–7,95); Interlink diretto $4,99 (300 mm) e $7,99 (400–610 mm) + spedizione USA e dogana | Molto più economico. Range di forza più ampio: **ipotesi mia, da verificare:** può evitare la saturazione dovuta al peso del materasso, mentre la classica 408 (fino a 20 N) potrebbe saturarsi. Non è lo stesso componente della 408 classica: la taratura sarà diversa. |
| FSR quadrato | Interlink **FSR 406** (area attiva 39,6 mm quadrata) | Melopero (IT) **€11,74** | Area piccola: più dipendente dalla posizione. Più economico. |
| ESP32 | **ESP32-DevKitC** (o equivalente, anche AZ-Delivery) | da **€10,90** (SOS Electronic) a €10,99–15,99 (AZ-Delivery) | Basta una scheda con Wi-Fi e USB. Usare pin **ADC1**: gli ADC2 non sono utilizzabili con Wi-Fi attivo. |
| ADC esterno (opzionale, consigliato) | **ADS1115** 16 bit, 4 canali, I2C | circa **€3,50** (modulo semplice) fino a ~€16 | Meno rumore dell'ADC interno dell'ESP32, che è non lineare. Per sola soglia presenza non è indispensabile, ma costa poco. |
| Alternativa DIY economica | Velostat + nastro di alluminio/rame | meno di 1 € a sensore (fonti community; prezzo materiali non verificato) | Segnalate forti oscillazioni di tensione quando ci si gira nel letto; ok per esperimenti, meno per produzione. |
| Accessori | resistori fissi per il partitore (valore da tarare, circa 10 kΩ come punto di partenza), cavi, nastro di carta/biadesivo, alimentatore USB per l'ESP32 | pochi euro | Resistori a precisione 1% e condensatore ceramico 100 nF sull'ingresso analogico come buona pratica. |

**Range di forza delle tre serie Interlink (ricerca, snippet dei siti Interlink):** classica 400 **0,2–20 N** (rip. singolo ±2%, tra pezzi ±6%, isteresi circa +10%, resistenza a riposo circa 10 MΩ); serie X 400 **0,3–50 N**; serie UX 400 **0,5–150 N** (rip. ±2%, tra pezzi ±5%, a riposo >10 MΩ). Oltre il range la risposta **satura** (la resistenza continua a scendere ma pochissimo). Stima mia, non verificata: sotto le doghe la striscia vede solo una frazione del peso del materasso e della persona, quindi la forza sulla striscia potrebbe cadere tra pochi N (a vuoto) e qualche decina di N (occupato). Se così, la classica (20 N) rischia di saturare, mentre X e UX sono più sicure. Il range nominale vale per un attuatore rigido: sul materasso va misurato. Conviene un ADC a 16 bit (ADS1115) per leggere variazioni piccole anche vicino a saturazione, e tarare ogni striscia singolarmente (variazione tra pezzi ±5–6%).

**Piano d'acquisto a fasi (riduce il rischio di comprare tutto prima di sapere se funziona):**
- **IDEA DELL'UTENTE (da provare come "fase 0", poco costosa): secondo radar mmWave dedicato, montato nella nicchia dell'armadio a ponte sopra il letto e puntato in basso.** Modulo candidato: **HLK-LD2410C** (24 GHz, ESP32 via UART con il componente ESPHome `ld2410`; circa €5,60 su eBay.de, £5,50 Pi Hut, da verificare; i pin potrebbero richiedere saldatura). Dati dalle fonti: 9 gate di distanza con risoluzione **selezionabile 0,2 m o 0,75 m**, portata fino a 5–6 m, angolo di rilevazione circa **±60°**, energia separata per "movimento" e "fermo" (rileva anche il respiro), soglie di energia regolabili per gate e gate massimo regolabile. **Limite geometrico chiave (ragionamento mio, da verificare sul campo):** i gate sono gusci a distanza obliqua, quindi non delimitano l'area laterale: con il radar a circa 1 m sopra il materasso e cono di ±60° il raggio al livello del letto è di circa 1,7 m, più largo di un letto matrimoniale; una persona in piedi accanto al letto o seduta sul bordo può quindi comparire come "a letto". Rimedi da provare: (1) gate min/max per escludere soffitto e pavimento; (2) **schermare con alluminio** (le microonde sono bloccate dal metallo; legno e cartongesso invece sono trasparenti) costruendo un piccolo cono/imbuto attorno al modulo per stringere il fascio, tecnica non verificata su questo modulo; (3) soglie di energia per gate; (4) debounce lungo in uscita. Il radar non può distinguere sdraiato da seduto sul letto. Non separa i lati (basta, perché di norma dorme una persona). Banda 24 GHz contro i 5,8 GHz dello SNZB-06P: non dovrebbero interferire (non verificato). **Sinergia:** l'ESP32 serve anche al piano FSR, quindi non è un acquisto sprecato. **Protocollo di prova:** sdraiato supino e di lato, seduto sul bordo, in piedi a 0,5 m dal letto, camera con persona ma letto vuoto, ventilatore acceso; registrare energia per gate. **Dati da chiedere all'utente:** altezza tra materasso e soffitto della nicchia, larghezza/profondità della nicchia, materiale delle pareti, dove può andare montato il modulo.
- **Aggiornamento radar con dati dell'utente (altezza materasso-soffitto nicchia circa 1 m, montaggio puntato verso il basso, possibilità di ridurre il raggio d'azione): stime geometriche mie, indicative, da confermare con la prova.** Con il modulo a circa 1 m dal materasso e risoluzione gate **0,2 m** (nove gate che arrivano a circa 1,6–1,8 m, sufficiente) si ottiene anche una discriminazione verticale, che modifica la conclusione precedente "non distingue sdraiato da seduto": le distanze oblique indicative sono

  | Bersaglio | Distanza dal modulo (circa) | Gate a 0,2 m (circa) |
  |---|---|---|
  | Torace di chi è sdraiato (0,25 m sopra il materasso) | 0,75 m | 4 |
  | Torace di chi è seduto sul letto (0,55 m sopra) | 0,45 m | 3 |
  | Superficie del materasso | 1,0 m | 5–6 |
  | Pavimento accanto al letto | oltre 1,5 m | 8 e oltre |

  Quindi: **gate minimo** impostato per escludere chi è seduto (parte alta del busto) e **gate massimo** per escludere pavimento e area oltre il letto; chi sta **in piedi accanto al letto** ha testa e busto sopra o oltre il bordo del cono rivolto verso il basso, quindi in gran parte fuori fascio (da verificare). Limiti rimasti: anche da seduti bacino e gambe restano a gate "da sdraiato" e il respiro rende comunque il bersaglio rilevabile; il ±60° nominale non è un confine netto (lobi laterali e riflessioni); biancheria, piumone e pareti influenzano il segnale. Valori di partenza da provare: gate minimo 4, massimo 6–7, soglie di energia da tarare per gate con le misure del protocollo di prova. Dettagli da verificare nell'annuncio d'acquisto: passo dei pin del modulo (potrebbe non essere 2,54 mm) e presenza del cavetto incluso. Costo stimato di questa fase: ESP32 €10,90–15,99 + LD2410C circa €5,60–8 + cavi €3–5 = circa **€20–30**, più saldatore (€35–50) solo se serve saldare i pin.
- **Radar mmWave SENZA ESPHome (richiesta dell'utente): candidato principale Sonoff SNZB-06P24** (24 GHz, Zigbee, stesso ecosistema dello SNZB-06P già installato). Dati dalle ricerche: portata fino a **4 m**, **7 zone indipendenti da 0,5 m** (zona 1 = 0–1 m, zone 2–7 a incrementi di 0,5 m, attivabili/disattivabili), **sensibilità da −6 a +6**, **occupancy timeout** (ritardo prima di "libero"), calibrazione "spatial learning", base magnetica con testa orientabile per montaggio a soffitto/parete/incasso, **prezzo €22,90** da expert4house.com (Italia, listino €24,90), presente anche nel catalogo OpenELAB (stesso ordine, spedizione condivisa). Zigbee2MQTT lo supporta ed espone occupancy, illuminance, timeout, sensibilità e le sette zone; per **ZHA serve un quirk fornito da Sonoff** (file personalizzato da installare nella configurazione di HA: l'implementazione ha accesso API, ma il file va messo nella cartella config; verificare se il quirk è già incluso nella versione di HA dell'utente). Recensioni miste sulla sensibilità (un thread del forum eWeLink la giudica scarsa; per una persona ferma funziona meglio entro 3 m e rivolti verso il sensore): a circa 1 m dovrebbe andare bene (stima mia). **Limite rispetto a LD2410C:** zone da 0,5 m, con la zona 1 che copre 0–1 m, quindi **non separa sdraiato da seduto** (entrambi in zona 1); si può comunque disattivare le zone lontane (da 3 in su, cioè oltre 1,5 m, dove c'è il pavimento) e ridurre la sensibilità. Altre opzioni valutate e scartate: Tuya ZY-M100 e simili (distanza minima/massima regolabile, ma ZHA richiede quirk personalizzato e le varianti sono molte), WenzhiIoT da soffitto (alimentato a rete 110/220 V), Aqara FP2 (zone e rilevazione di più persone richiedono montaggio a parete, a soffitto no; prezzo non verificato), Everything Presence Lite (€39,99 più circa €13 di spedizione e tasse; usa LD2450 che traccia x/y su un piano, pensato per parete, poco adatto al montaggio verso il basso).
- **ALTERNATIVA SENZA DIY (da valutare per prima, perché l'utente non ha saldatore né ESP32 e giudica il DIY complicato):** esistono **sensori di pressione Zigbee già pronti**, strisce sottili da mettere sotto il materasso, in versione **40 e 80 cm** (l'80 cm è quella indicata per i letti), alimentati da una **CR2032** (fino a circa un anno), "Tuya Zigbee Seat Pressure Sensor" (Zigbee2MQTT lo elenca come "TS0601 bed presence sensor"). Il negozio olandese Slimhuisje (prezzo non trovato) dichiara compatibilità con Zigbee2MQTT e ZHA; SmartHomeScene lo indica come reperibile solo su AliExpress. Elimina ESP32, ADS1115, resistori, saldature, ESPHome e tarature analogiche; l'utente ha già Zigbee/ZHA. Limiti noti dalle fonti: con **materassi duri a molle** serve più pressione (meglio schiuma/lattice); molto **sensibile ai movimenti** nel letto (servirà il debounce sull'uscita); uscita **binaria**, niente dati di forza da tarare; per i due lati ne servono **due da 80 cm** (uno per lato). **Da verificare prima di comprare:** supporto reale in **ZHA** (i dispositivi Tuya TS0601 spesso richiedono un quirk), tipo di materasso dell'utente, prezzo e tempi di consegna in Italia. Se funziona bene, il piano FSR diventa un ripiego e questo handoff va semplificato (livello 1 = due binary_sensor Zigbee).
  - **Ricerca prezzo (esito):** **prezzo non trovato.** Le pagine Slimhuisje e shop.app non erano apribili dalla sessione (accesso bloccato) e gli snippet non riportano cifre per questo prodotto. Da controllare dall'utente sul sito del negozio e su AliExpress.
  - **Aggiunta tecnica:** Zigbee2MQTT elenca per il "TS0601 bed presence sensor" (Tuya, "Pressure Sensing Strap/Bed Occupancy Sensor") le entità `occupancy`, `battery`, `illuminance`, `sensitivity`, `interval_time`, `presence_delay`, `presence_time`, `work_state`: quindi sensibilità e ritardi sarebbero regolabili, a differenza di quanto scritto prima, **ma solo se ZHA espone le stesse entità** (dipende dal quirk). Esistono più varianti con identificatori diversi (es. `_TZE204_w2vunxzm`, `_TZE200_seq9cm6u`, segnalate in issue di Zigbee2MQTT e Homey): prima di comprare confermare l'ID produttore esatto e cercare il quirk in zigpy/zha-device-handlers.
- **AGGIORNAMENTO PRECEDENTE: Interlink con spedizione costa $129 (riferito dall'utente), quindi si scartano Interlink diretto e la UX.** Nei negozi UE trovati la UX non c'è: l'unica serie "lunga" con range superiore alla classica reperibile in UE è la **FSR X 408 (0,3–50 N)**, venduta da **OpenELAB a €2,75 per il 300 mm** (spedizione €4,95–7,95). Piano: **4 × X 408 300 mm (circa €11 + spedizione)**, due per lato sulla stessa doga, in fila a coprire circa 60 cm, collegate in parallelo allo stesso canale ADC (da provare). Se i dati mostrano saturazione con la persona a letto, rimedi in ordine: (1) abbassare il resistore di carico del partitore per spostare la sensibilità verso forze alte (tecnica comune, da verificare); (2) ridurre meccanicamente la forza sulla striscia (es. strato che ripartisce il carico); (3) solo allora riconsiderare la UX. La classica 610 mm da Tinytronics (€25) è utile solo come riferimento di confronto, non come primo acquisto.
- **Preventivo del piano FSR (prezzi da ricerche web, i marcati "stima" non sono verificati; l'utente ha dichiarato di apprezzare la manualità e non ha saldatore):**

  | Voce | Prezzo | Note |
  |---|---|---|
  | 4 × FSR X 408 300 mm (OpenELAB) | €11,00 | verificato (€2,75 l'una) |
  | ESP32-DevKitC | €10,90–15,99 | SOS Electronic / AZ-Delivery |
  | ADS1115 | €3,50–13,37 | OpenELAB a €6,49 ma **esaurito**; generico da eBay.de circa €3,50; Farnell €13,37 |
  | Kit resistori 1% | €4,50–7,00 | eBay.de / negozio NL |
  | Condensatori ceramici 100 nF | €3–5 | stima |
  | Cavetti Dupont (OpenELAB) | €3,25–5,01 | verificato |
  | Breadboard MB-102 (OpenELAB) | da €9,34 | verificato |
  | Cavo bipolare sottile 2–3 m per lato | €3–6 | stima |
  | Nastro Kapton / guaina termorestringente | €3–6 | stima |
  | Nastro biadesivo/di carta | €2–3 | stima |
  | Spedizione OpenELAB | €4,95–7,95 | altre spedizioni non verificate (fino a circa €10) |
  | **Subtotale materiali** | **circa €58–100** | |
  | Saldatore a temperatura regolabile con stagno | €34,88–49,99 | ManoMano; attrezzo riusabile |
  | Multimetro (opzionale) | €18,74–27,70 | opzionale: ESPHome registra già la tensione dell'ADS1115 |
  | **Totale con saldatore** | **circa €93–150**; con multimetro **circa €112–177** | USB e alimentatore: si presume già in casa |

  Tipicamente il cavo USB corretto e un caricabatterie da telefono bastano per alimentare l'ESP32.
- **Lista acquisti completa per il test (BOM), oltre alle strisce:**
  - **Da OpenELAB** (confermati dalla ricerca come presenti in catalogo; prezzi non verificati): modulo ADS1115 (versione generica "ADC Module 16 bit 4 channels" o Gravity di DFRobot, la seconda probabilmente più cara), cavetti Dupont (40 pin F-F e M-F), breadboard MB-102, cavi USB-USB-C. Non confermati in catalogo: ESP32-DevKitC, resistori e condensatori, morsettiere, alimentatore USB. Controllare direttamente il sito per questi.
  - **Collegamento delle strisce (da non dimenticare):** la FSR X 408 ha piazzole da saldare sensibili al calore. Serve un saldatore con punta a bassa temperatura e saldatura rapida, oppure connettori a crimp/pinzette a coccodrillo per il test. Isolare con nastro Kapton o guaina termorestringente.
  - **Cavo bipolare sottile** (circa 2–3 m per lato) per portare il segnale dalle doghe all'ESP32; i Dupont sono troppo corti e fragili per l'uso fisso.
  - **Altrove (Amazon.it o negozio di elettronica):** ESP32-DevKitC (cavo USB giusto per il connettore della scheda), kit resistori 1% (servono 2,2 kΩ, 10 kΩ, 47 kΩ), condensatori ceramici 100 nF, nastro biadesivo/di carta, eventuale scatola per l'ESP32 e una millefori per la versione definitiva.
  - **Da chiedere all'utente:** possiede già saldatore e multimetro? Il multimetro serve per misurare la resistenza delle strisce a vuoto e sotto carico in fase di taratura.
- **DECISIONE UTENTE PRECEDENTE (superata dall'aggiornamento sopra): comprare solo 2 × FSR UX 408 da 610 mm** (una per lato), da Interlink diretto ($7,99 l'una secondo gli snippet, più spedizione/IVA/dogana da verificare al checkout). Rationale: la UX (0,5–150 N) non satura, quindi le letture dicono il range di forza reale; se i valori restano bassi si passa alla X (0,3–50 N) o alla classica (0,2–20 N) con dati alla mano. Rischio opposto da controllare: la UX è la meno sensibile sotto 0,5 N, quindi se a letto vuoto/materasso solo il segnale è troppo debole serve una serie più sensibile.
  - **Vincolo emerso: Interlink ha un ordine minimo di $25** (segnalato dall'utente; non verificato se prima o dopo spedizione/tasse). Due UX = $15,98 non bastano. Opzioni per superare la soglia: 2 UX 610 mm + 2 classiche 610 mm = **$29,96** (il confronto classica contro UX, che evita anche un secondo ordine con un altro minimo e un'altra spedizione); oppure 2 UX 610 mm + 2 UX 300 mm = $25,96 (stesso modello, due strisce corte in più per una seconda doga). Prezzi da snippet, da verificare al checkout.
  - **Variante con serie X (0,3–50 N), proposta dall'utente:** esiste la **FSR X 408** sul sito Interlink in lunghezze 50 mm ($3,49), 100 mm ($4,49), 300 mm ($4,49), 500 mm ($7,49); una 610 mm non è comparsa nei risultati. Larghezza area attiva 10,2 mm secondo lo snippet (da verificare). 2 UX 610 mm + 2 X 500 mm = **$30,96**; 2 UX + X 500 + X 300 = $27,96; 2 UX + 2 X 300 = $24,96, **sotto il minimo di 4 centesimi**. La X copre la fascia intermedia tra la classica (probabilmente satura) e la UX (meno sensibile in basso), quindi è un confronto più informativo di UX contro classica. Anche OpenELAB (UE, €2,75 per il 300 mm) vende la X 408.
  - **Da registrare in fase di test (ESPHome, tensione grezza dell'ADS1115 e resistenza stimata):** letto vuoto, solo materasso, persona sdraiata (supino, laterale), seduto sul bordo, due persone, per ciascun lato. Servono a decidere il range necessario, la soglia e se i lati si separano.
  - **Lista completa da ordinare:** 2 × UX 408 610 mm; ESP32-DevKitC; ADS1115; resistori 2,2 kΩ, 10 kΩ, 47 kΩ (1%) e condensatori ceramici 100 nF; cavi; nastro di carta o biadesivo; alimentatore USB. Spedizione fissa Interlink: considerare una terza striscia di scorta solo se il costo aggiuntivo è minimo.
- **Fase A, test minimo (≈ €60–80 con la 408 classica; molto meno con la UX):** 1 ESP32 + 1 ADS1115 + **2 strisce FSR** (una per lato, su una doga sotto busto/fianchi) + accessori. Obiettivo: verificare che `qualcuno_a_letto` sia affidabile e capire se i due lati si separano.
- **Variante economica consigliata per partire:** comprare **2 × FSR UX 408** (OpenELAB 300 mm €2,75 l'una + spedizione €4,95–7,95, oppure 610 mm da Interlink diretto) **e 1–2 × FSR 408 classica 610 mm da Tinytronics (€25 l'una)**, così si confrontano i due modelli per circa €60 totali invece di comprare 4 strisce classiche. Dopo il test si ordina il resto. Stock e prezzi vanno verificati sui siti (non apribili dalla sessione: ricerca solo da snippet).
- **Fase B, se la A funziona:** aggiungere un secondo FSR per lato (su un'altra doga) per robustezza alla posizione (≈ +€40–56).
- **Disegno del confronto classica contro UX (2 + 2):** per ogni lato fissare **una classica e una UX affiancate sulla stessa doga** (la doga è larga abbastanza per due strisce da circa 1,5 cm), così vedono lo stesso carico e il confronto è pulito. Metterle su doghe diverse o la classica a sinistra e la UX a destra confonderebbe il confronto con la posizione o il lato. Comprare qualche resistore di valori diversi (es. 2,2 kΩ, 10 kΩ, 47 kΩ) per tarare il partitore di ciascun modello.
- **Esperimento parallelo opzionale (≈ €10–15):** una striscia DIY in Velostat da confrontare con gli FSR, solo per curiosità/ripiego.
- Il costo totale prima stimato "poche decine di euro" era ottimistico: con FSR a larga area realistico ≈ **€60–80 per la fase A**, ≈ **€100–140** con 4 strisce.
- Verificare la copertura Wi-Fi in camera.

**Limiti di questa ricerca:** non sono riuscito ad aprire i thread della community HA (accesso bloccato), quindi la scelta del modello/dimensione FSR si basa su snippet di ricerca e non sulla lettura integrale delle esperienze. Prima dell'acquisto vale la pena rileggere "FSR - the best bed occupancy sensor" e "Bed occupancy DIY sensor" su community.home-assistant.io per confermare modello, dimensioni e posizione sulle doghe.

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
4. **Animali:** l'utente non ne ha in casa, quindi non serve gestirli. Se cambiasse, con la sola pressione un animale pesante può superare la soglia.
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

Risolte dopo:
- ~~Persone nel letto~~ → di norma **una**, a volte **due**. `tutti_a_letto` ha quindi senso solo come "tutte le persone presenti", da definire in implementazione; la modalità normale è una persona (conta `qualcuno_a_letto`).
- ~~Animali~~ → **no**. Si può non gestire il caso animale (regola 4 della sez. 5 declassata).
- ~~Accesso API/MCP a HA~~ → **sì**, l'implementazione può creare gli helper.
- ~~Fase 2 ("dormendo")~~ → **dopo**. Implementare solo i livelli 1-3.
- ~~Integrazione con "Sveglia potenziata"~~ → per ora **fuori scope**: ci si occupa di scelta/acquisto hardware.

Ancora aperte:
1. **Cosa deve far scattare** lo stato (luci, riscaldamento, allarme, aspirapolvere...): **"svariate cose, da definire dopo"**. Non blocca l'acquisto.
2. **Budget massimo** per la fase A e se ordinare da rivenditori italiani/UE (Melopero, RobotShop EU, SOS Electronic, AZ-Delivery) o da USA (Adafruit: più economico ma spedizione/dogana). Da confermare con l'utente.
3. **Quale striscia FSR** (408 contro 406) in base alla rilettura dei thread community (sez. 3, "Limiti di questa ricerca").

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
