# HiGrow 2.0 Avanzato

Sensore per piante open source di fascia avanzata. Evoluzione del classico
design "HiGrow" basato su ESP32, riprogettato per consumi ultra bassi,
ricarica solare con MPPT, automazione attiva (pompe e luci) e affidabilita
dei sensori.

> Crediti: basato sul design HiGrow originale, ideato da Luca Fabbri nel 2017.

> Stato: pianificazione tecnica. Questo documento descrive l'architettura
> hardware e la distinta base (BOM). Non e ancora un progetto finalizzato:
> vedere la sezione "Domande aperte e rischi di progetto".

## Indice

1. Introduzione e visione generale
2. Architettura hardware
   - 2.1 Cuore logico e connettivita
   - 2.2 Firmware e topologia di rete
   - 2.3 Alimentazione, ricarica e solare
   - 2.4 Protezione e gestione della batteria (BMS)
   - 2.5 Sensori e taglio energetico
   - 2.6 Automazione attiva (pompe e luci)
3. Distinta base (BOM) per macroblocchi
4. Domande aperte e rischi di progetto
5. Riferimenti e progetti open correlati
6. Licenza

---

## 1. Introduzione e visione generale

Il design originale HiGrow, ideato da Luca Fabbri nel 2017, combinava un
ESP32 con sensori capacitivi e ambientali, ma presentava limiti evidenti su
consumi, affidabilita dei sensori (es. DHT11) e routing dell'alimentazione.

Questa iterazione trasforma il dispositivo da semplice "monitor passivo" a
una vera centralina di giardinaggio smart: ricarica solare con tracciamento
del punto di massima potenza (MPPT), automazione attiva di pompe e luci,
protezione batteria completa e consumi ridotti al minimo in deep sleep.

Il form factor mantiene il classico design "a picchetto" da inserire nel
terreno, con la parte superiore ottimizzata per ospitare sul retro una cella
agli ioni di litio 18650 (o 21700). L'autonomia obiettivo (6-12 mesi) e
realistica solo con i protocolli mesh a basso consumo descritti al punto 2.2;
con Wi-Fi l'autonomia attesa e sensibilmente inferiore.

---

## 2. Architettura hardware

### 2.1 Cuore logico e connettivita

* **Microcontrollore:** modulo ESP32-C6-WROOM-1 (o variante 1U per antenna
  esterna U.FL). L'architettura single core RISC-V garantisce consumi
  irrisori in deep sleep (pochi microampere) e supporta Wi-Fi 6,
  Bluetooth 5, Zigbee 3.0 e Thread/Matter.
* **Interfaccia USB nativa:** il chip seriale dedicato (CH340, CP2104) e
  stato eliminato. Sfruttando il controller USB Serial/JTAG nativo
  dell'ESP32-C6, le piste D- e D+ del connettore Type-C vanno direttamente ai
  pin GPIO12 e GPIO13, abbattendo costi e consumi.
* **Pulsanti di emergenza:** due micro pulsanti SMT tattili sui pin EN
  (Reset) e GPIO9 (Boot) per forzare il flashing in caso di boot loop.

### 2.2 Firmware e topologia di rete

La scelta del protocollo radio determina direttamente l'autonomia. Sono
previsti due percorsi.

* **Percorso primario (consigliato per l'autonomia): Thread/Matter o Zigbee.**
  Il sensore a batteria opera come *Sleepy End Device*: dorme in deep sleep,
  si sveglia, invia la lettura in poche decine di millisecondi e torna a
  dormire. I nodi alimentati a muro (per esempio le centraline che pilotano
  pompe e luci) agiscono da router/relay della mesh. Un Border Router
  (Thread) o un Coordinator (Zigbee) fa da bridge verso Wi-Fi/IP e verso
  Home Assistant. Questa topologia mesh master/relay e cio che rende
  credibile l'autonomia di 6-12 mesi su una singola cella. Con Matter su
  Thread si ottiene integrazione nativa con i principali ecosistemi smart
  home. Richiede firmware custom (ESP-IDF con OpenThread o lo stack Zigbee).
* **Gateway self-contained (deployment senza hardware aggiuntivo):** se non
  si dispone di un Border Router o di un coordinator Zigbee dedicato (dongle,
  HA Yellow/Connect e simili), si puo dedicare un singolo nodo HiGrow
  alimentato a muro al ruolo di gateway: raccoglie i dati degli altri sensori
  sulla mesh e li espone a Home Assistant via Wi-Fi. Sull'ESP32-C6 questo
  ruolo si realizza con firmware da Thread Border Router o Zigbee Coordinator
  (Espressif fornisce firmware di riferimento). Da notare: ESPHome puro non
  implementa un Border Router Thread, quindi il nodo gateway usa firmware
  dedicato, non lo stesso firmware ESPHome dei nodi Wi-Fi.
* **Percorso opzionale (hackability): ESPHome su Wi-Fi.** Piu semplice da
  personalizzare e integrare, ma la sequenza di associazione Wi-Fi + DHCP +
  invio costa molta energia a ogni risveglio, quindi l'autonomia attesa
  scende a settimane o pochi mesi. Per mitigare, le letture possono essere
  bufferizzate nella memoria RTC dell'ESP32 e inviate in blocco a intervalli
  piu radi, riducendo il numero di associazioni Wi-Fi.

### 2.3 Alimentazione, ricarica e solare

L'intera sezione di alimentazione e ridisegnata per eliminare le correnti di
dispersione (quiescent current) che affliggevano i vecchi cloni.

* **Cella e socket:** cella 18650 o 21700 alloggiata in un **socket plastico
  nero SMT** (stesso tipo del HiGrow originale), saldato a montaggio
  superficiale sul retro del PCB. Il polo negativo prevede pad multipli
  posizionati a 65mm e 70mm per la compatibilita meccanica sia con 18650 sia
  con 21700, senza componenti through hole.
* **Sede per NTC sotto la cella:** lungo l'asse della cella resta una
  fessura longitudinale (circa 5mm di larghezza) in cui il PCB e esposto a
  circa 1mm dalla cella. Questa sede ospita l'**NTC di temperatura batteria**
  (vedi 2.4), collegato al pin TS del charger, che cosi misura la temperatura
  reale del pacco.
* **Charger buck-boost con MPPT e I2C:** core energetico affidato al
  **BQ25798** (Texas Instruments, VQFN, sigla ordinabile `BQ25798RQMR`). E un
  charger buck-boost 1-4 celle con **MPPT integrato**, ADC interno e controllo
  **I2C**. Vantaggi rispetto al precedente schema lineare/MPPT discreto:
  * gestisce nativamente sia USB-C 5V sia un pannello solare, con tensione di
    ingresso sopra o sotto la tensione di cella (buck-boost), senza compromessi
    sul setpoint MPPT;
  * **corrente di carica e input current limit programmabili da firmware**
    (via I2C), quindi nessun solder jumper sulla resistenza di sense: la
    corrente si adatta alla cella installata (18650/21700 o LiPo piu piccola);
  * **ADC integrato** (VBAT, VBUS, IBAT, IBUS, temperatura): legge la tensione
    e la corrente di batteria via I2C, rendendo inutili il partitore di lettura
    V-BAT e il relativo MOSFET;
  * **pin TS** per il termistore NTC: la qualificazione termica della carica
    (logica JEITA, vedi 2.4) e gestita in hardware dal charger.
* **Combinazione USB + solare (doppio ingresso ACFET/RBFET):** la porta USB-C
  (con resistenze di pull-down da 5.1kOhm sui pin CC1/CC2 per abilitare i 5V
  dai caricatori) e l'ingresso solare (JST-PH 2.0mm) sono collegati ai due
  ingressi del BQ25798 (VAC1 = USB, VAC2 = solare). Ogni ingresso ha una
  coppia di MOSFET N-channel in serie schiena-contro-schiena, pilotata dai
  driver ACDRV del charger: l'**ACFET** e l'interruttore di potenza (Rds_on di
  pochi mOhm) e l'**RBFET** blocca il ritorno di corrente verso la sorgente non
  attiva. Rispetto all'OR-ing a diodi questo azzera la caduta di tensione
  (decine di mV invece di 0.3-0.5V) e il relativo calore, recuperando il
  raccolto solare, e permette di scegliere la **priorita di sorgente via I2C**
  (es. preferire l'USB quando presente). E lo schema del reference design TI
  del BQ25798.
* **Regolatore LDO:** tensione logica stabilizzata a 3.3V tramite LDO ultra
  efficiente (XC6220 o ME6211) con quiescent current nell'ordine dei
  microampere, essenziale per il deep sleep. Alimentato dal rail di sistema
  (SYS) del BQ25798.

### 2.4 Protezione e gestione della batteria (BMS)

Attenzione a non confondere i due livelli di protezione. Il BQ25798 protegge
il **processo di carica** (incluso il pin TS, vedi sotto) ma **non** sostituisce
la protezione della cella: il pin TS fa solo qualificazione termica della
carica, non protegge da sovrascarica, sovracorrente o cortocircuito lato
batteria. In un dispositivo outdoor con cella 18650 sfusa nel socket, lasciato
mesi all'aperto, una protezione di cella indipendente e obbligatoria.

* **Protezione cella (BMS discreto):** accoppiata **DW01A** (controller di
  protezione) e **FS8205A** (doppio MOSFET N-channel) sul ramo negativo
  della batteria. Previene sovrascarica (taglio sotto circa 3.0V),
  sovraccarica (sopra circa 4.2V), sovracorrente e cortocircuito. E una rete
  di sicurezza hardware indipendente, che non dipende dal firmware ne dalla
  configurazione I2C del charger. La rete applicativa del DW01A prevede una
  resistenza serie (circa 100-470 Ohm) e un condensatore di filtro (0.1uF) sul
  pin di alimentazione del controller.
* **Protezione termica in carica (NTC, logica JEITA):** una Li-ion **non va
  caricata sotto 0 C** (rischio di lithium plating e degrado), condizione
  frequente per un dispositivo da giardino. Un NTC da 10kOhm (B 3950),
  alloggiato nella fessura sotto la cella (vedi 2.3), e collegato al **pin TS
  del BQ25798**: il charger qualifica e, se necessario, sospende o riduce la
  carica in base alla temperatura del pacco, senza coinvolgere il firmware.

### 2.5 Sensori e taglio energetico

* **Umidita del terreno (capacitiva):** sensore capacitivo integrato nel PCB
  (nessuna corrosione galvanica). La lettura non e in DC: serve
  un'eccitazione in alternata. Soluzione consigliata: oscillatore dedicato
  basato su timer **TLC555** che genera un'onda quadra (circa 1 MHz) iniettata
  nella piastra capacitiva tramite una resistenza serie; il segnale viene
  raddrizzato da un diodo Schottky di piccolo segnale (BAT54) e filtrato da un
  condensatore, producendo una tensione DC proporzionale all'umidita, letta
  dall'ADC. Approccio collaudato sul progetto open b-parasite e sui sensori
  capacitivi v1.2/v2.0. In alternativa l'onda quadra puo essere generata via
  LEDC/PWM dall'ESP32, risparmiando il 555 ma occupando un timer e un GPIO.
  La parte inferiore del picchetto richiede *conformal coating*
  (verniciatura poliuretanica o epossidica) per evitare infiltrazioni
  d'acqua ai lati della maschera di saldatura.
* **Conduttivita elettrica (EC / nutrienti) anti elettrolisi:** due elettrodi
  inerti ("gold-fingers" con finitura ENIG) alla base del picchetto, in serie
  a una resistenza di riferimento nota che forma un partitore letto
  dall'ADC. Best practice da rispettare:
  * eccitazione bipolare simmetrica (finta AC): l'ESP32 alterna la polarita
    su due GPIO a frequenza elevata per azzerare l'elettrolisi e il degrado
    dei pad;
  * misura sincrona, campionando l'ADC a meta di ciascun semiperiodo;
  * compensazione in temperatura: la conducibilita varia di circa 2%/C,
    quindi le letture vanno normalizzate a 25 C usando la temperatura del
    terreno o dell'aria;
  * calibrazione a due punti con soluzioni a conducibilita nota;
  * elettrodi inerti (oro su nichel) per limitare ossidazione e fouling, che
    restano comunque voci di manutenzione periodica.
* **Dati ambientali:** sensore **BME280** integrato per temperatura, umidita
  dell'aria e pressione atmosferica, su bus I2C. Sostituisce i vecchi e
  inaffidabili DHT11.
* **Espansione I2C:** connettori **JST-SH 1.0mm** (4 pin: VCC, GND, SCL, SDA)
  compatibili Grove/Qwiic/STEMMA QT, per moduli come VEML7700 (luce ambiente
  fino a 120000 lux, non satura sotto sole diretto), SI1145 (luce visibile,
  IR, stima UV) o SCD41 (CO2).
* **ADC dedicato ai segnali delle piante:** l'ADC interno dell'ESP32-C6 e
  rumoroso e non lineare, e l'ADC del BQ25798 e dedicato ai rail di potenza
  (VBAT, VBUS, correnti). Per i segnali analogici delle piante si usa quindi un
  convertitore **ADS1115** dedicato (16 bit, 4 canali, I2C) su umidita del
  terreno ed EC, per letture stabili e ripetibili. Va sul bus I2C condiviso,
  alimentato dal rail commutato dei sensori. La tensione di batteria non passa
  piu da qui: la legge il BQ25798 via I2C.
* **Isolamento energetico:** per evitare assorbimenti in deep sleep,
  l'alimentazione (VCC) dei sensori e dell'ADS1115 e tagliata da un MOSFET
  P-channel (AO3401), comandato da un GPIO. Non serve piu il partitore di
  lettura batteria ne il relativo MOSFET, perche la tensione di cella e letta
  dall'ADC interno del BQ25798.
  > Importante: i pull-up I2C (4.7kOhm) e l'ADS1115 devono essere
  > alimentati dal **rail commutato dei sensori**, non dal 3.3V sempre
  > acceso. Se i pull-up restano sul 3.3V principale mentre il VCC dei
  > sensori e tagliato, la corrente fluisce attraverso i diodi ESD dei
  > sensori spenti, vanificando il taglio energetico.

### 2.6 Automazione attiva (pompe e luci)

* **Boost 5V on demand:** un convertitore Step-Up (SX1308 o MT3608) eleva la
  tensione della cella a 5V per pilotare carichi a 5V in modo stabile sia a
  batteria sia con alimentatore. Il chip e normalmente spento (pin EN basso)
  per non consumare, e viene abilitato da un GPIO solo all'occorrenza.
  > Comportamento da tenere presente: anche con EN basso il boost presenta un
  > percorso passivo interno (induttore + diodo) che porta il rail 5V a circa
  > Vbatt meno la caduta del diodo (circa 3.4V). A vuoto non consuma, ma un
  > carico collegato verrebbe alimentato passivamente e drenerebbe la cella.
  > Per questo ogni uscita deve restare isolata dal proprio MOSFET (sotto).
* **Coordinamento EN e MOSFET:** il rail 5V e condiviso dalle due uscite,
  quindi EN deve stare alto se almeno una uscita e attiva. Schema consigliato:
  EN su un GPIO dedicato, tenuto statico per tutta la sessione di uscita, e i
  due MOSFET pilotati indipendentemente (anche in PWM). Il firmware accende
  prima il boost, attende un breve settle, poi pilota i carichi; allo stop
  spegne i carichi e infine il boost. Cosi un carico vede sempre e solo i 5V
  regolati e mai il 3.4V passivo. Da evitare il collegamento diretto di EN al
  gate di un singolo MOSFET: con due uscite non basta, e in PWM farebbe
  ripartire il boost a ogni ciclo (soft-start continuo e ripple). Se serve una
  garanzia hardware contro un errore firmware, EN puo essere derivato come OR
  (a diodi) dei due segnali di gate, accettando pero il limite sul PWM.
* **Uscite di potenza:** 2 connettori JST-XH (2 pin) serviti da 2 MOSFET
  N-channel (AO3400), capaci di gestire fino a 5A e di supportare il PWM.
* **Pull-down di gate:** ogni gate dei MOSFET di uscita ha un pull-down da
  10kOhm, in modo che pompe e luci restino spente durante reset e riavvii
  dell'ESP32.
* **Protezione induttiva (flyback):** ogni uscita pilota un carico
  potenzialmente induttivo (pompe, valvole). Con switch low-side (5V ->
  carico -> drain del MOSFET -> source a GND), allo spegnimento il nodo di
  drain genera uno spike positivo. Il diodo di freewheeling (SS34 o 1N5819)
  va quindi posto in antiparallelo al carico, con **anodo sul nodo di drain
  (lato switch) e catodo sul +5V**, cosi da richiudere la corrente
  dell'induttanza e sopprimere la back-EMF distruttiva. Questo schema rende
  le porte universali e sicure per pompe, valvole e strisce LED in PWM.

---

## 3. Distinta base (BOM) per macroblocchi

### 3.1 Cuore logico e interfaccia

Gestisce elaborazione, comunicazione e flashing diretto.

| Componente | Q.ta | Sigla / Valore | Note e funzione |
| :--- | :--- | :--- | :--- |
| Microcontrollore | 1 | `ESP32-C6-WROOM-1` | Modulo Wi-Fi 6 / BLE / Zigbee / Thread. Footprint compatibile anche con variante 1U. |
| Connettore USB | 1 | Type-C 16-pin SMT | Ricarica batteria e programmazione seriale nativa. |
| Resistenze PD | 2 | `5.1kOhm` | Pull-down sui pin CC1/CC2 per abilitare i 5V sui caricatori Type-C. |
| Pulsanti SMT | 2 | Tactile switch SMD | Pulsanti di emergenza su EN (Reset) e GPIO9 (Boot). |
| Resistenza EN | 1 | `10kOhm` | Pull-up standard sul pin EN. |
| Condensatore EN | 1 | `1uF` (MLCC) | Filtro di debounce sul pin EN. |

### 3.2 Ricarica buck-boost, MPPT e ingresso solare

Charger I2C con MPPT integrato e doppio ingresso (USB + solare). Valori dei
passivi e necessita esatta dei componenti di contorno da confermare sul
datasheet / reference design TI del BQ25798.

| Componente | Q.ta | Sigla / Valore | Note e funzione |
| :--- | :--- | :--- | :--- |
| Charger buck-boost | 1 | `BQ25798RQMR` | Buck-boost 1-4S con MPPT, ADC e I2C. VQFN. Extended part su JLCPCB. |
| Induttore di potenza | 1 | `~1uH` (da datasheet) | Induttore buck-boost, corrente elevata (alcuni A). |
| Connettore solare | 1 | `JST-PH 2.0mm` | Ingresso a 2 pin per pannello solare (VAC2). |
| FET ingresso (ACFET/RBFET) | 4 | N-channel low Rds_on (o 2 dual) | 2 coppie schiena-contro-schiena su VAC1 (USB) e VAC2 (solare), via ACDRV1/2. |
| Filtri ingresso/uscita | 2+ | `10uF` MLCC (+ bulk) | Condensatori VAC/VBUS e SYS/BAT per stabilita del buck-boost. |
| Condensatori boot/REGN | 2+ | (da datasheet) | Bootstrap degli switch e bypass del regolatore interno REGN. |
| Programmazione | n | (da datasheet) | Resistenze su PROG/ILIM/indirizzo I2C secondo reference design. |
| LED di stato | 1 | LED SMD | Su pin STAT del charger (stato carica). Opzionale, stato anche via I2C. |
| Resistenza LED | 1 | `2.2kOhm` o `4.7kOhm` | Limita l'assorbimento del LED di stato. |

> Nota di progetto: il combinatore di ingressi usa il doppio ingresso del
> BQ25798 (VAC1 = USB, VAC2 = solare) con i FET esterni ACFET/RBFET. L'OR-ing
> a diodi Schottky era l'alternativa semplice scartata, perche piu lossy e
> senza priorita programmabile.

### 3.3 Batteria, protezione (BMS), regolazione 3.3V e monitoraggio

Ritenzione cella, protezione, conversione a bassissimo consumo e analisi
dello stato.

| Componente | Q.ta | Sigla / Valore | Note e funzione |
| :--- | :--- | :--- | :--- |
| Socket batteria | 1 | Socket plastico nero SMT | Alloggiamento cella, stesso tipo del HiGrow originale. |
| Pad negativi | 2 | (PCB) | Pad a 65mm e 70mm per compatibilita 18650 / 21700. |
| Controller protezione | 1 | `DW01A` | Protezione cella: sovrascarica, sovraccarica, sovracorrente, corto. |
| MOSFET protezione | 1 | `FS8205A` | Doppio MOSFET N-channel pilotato dal DW01A, sul ramo negativo. |
| Rete DW01A | 1+1 | `100-470Ohm` + `0.1uF` | Resistenza serie e condensatore di filtro per il DW01A. |
| NTC temperatura batt. | 1 | `10kOhm` (B 3950) | Nella fessura sotto la cella. Collegato al pin TS del BQ25798 (logica JEITA). |
| Rete pin TS | n | (da datasheet) | Resistenze di bias del pin TS secondo reference design del charger. |
| Regolatore 3.3V | 1 | `XC6220` o `ME6211` | LDO con quiescent current (Iq) ridotto ai microampere. Alimentato da SYS. |
| Filtri LDO | 2 | `10uF` (MLCC) | Condensatori di ingresso/uscita per i 3.3V logici. |

> La tensione e la corrente di batteria sono lette dall'ADC interno del
> BQ25798 via I2C: non servono piu il partitore di lettura V-BAT ne il MOSFET
> N-channel (2N7002) che lo accendeva.

### 3.4 Sensori ambientali e bus I2C

Rilevamento delle metriche biologiche.

| Componente | Q.ta | Sigla / Valore | Note e funzione |
| :--- | :--- | :--- | :--- |
| Sensore aria | 1 | `BME280` | Temperatura, umidita, pressione (integrato, I2C). |
| MOSFET sensori | 1 | `AO3401` | P-channel, scollega i 3.3V a BME280 e porte I2C in deep sleep. |
| Pull-up I2C | 2 | `4.7kOhm` | Su SDA/SCL. Da alimentare dal rail commutato dei sensori, non dal 3.3V fisso. |
| Connettori esp. | 2 | `JST-SH 1.0mm` | 4 pin (VCC/GND/SCL/SDA) per moduli I2C esterni (es. VEML7700). |
| Eccitazione umidita | 1 | `TLC555` | Oscillatore onda quadra per il sensore capacitivo (alternativa: LEDC/PWM). |
| Raddrizzatore umidita | 1+1 | `BAT54` + cond. | Diodo Schottky di piccolo segnale e condensatore per il rilevatore di picco. |
| Resistenza serie umidita | 1 | (da taratura) | In serie alla piastra capacitiva, sull'eccitazione. |
| Elettrodi EC | 2 | Gold-fingers (ENIG) | Pad inerti esposti, gestiti in pseudo-AC via GPIO. |
| Resistenza riferimento EC | 1 | `10kOhm` | Partitore noto per la misura della conduttivita dal terreno. |
| ADC dedicato | 1 | `ADS1115` | 16 bit, 4 canali, I2C. Per umidita ed EC. Sul rail commutato dei sensori. |

### 3.5 Automazione attiva (pompe e luci) e step-up

Infrastruttura di potenza on demand per i carichi ausiliari.

| Componente | Q.ta | Sigla / Valore | Note e funzione |
| :--- | :--- | :--- | :--- |
| Chip boost 5V | 1 | `SX1308` o `MT3608` | Step-up comandato dall'ESP32 via pin EN per erogare 5V. |
| Induttore boost | 1 | `4.7uH` (SMT) | Induttore della fase di step-up. |
| Diodo boost | 1 | `SS34` | Schottky di raddrizzamento dello schema boost. |
| Resistenze boost | 2 | (da datasheet) | Partitore di feedback per fissare l'uscita a 5V. |
| Filtri boost | 2 | `10uF` o `22uF` | Condensatori ceramici per un output 5V pulito. |
| MOSFET carichi | 2 | `AO3400` | N-channel, interruttori per le uscite esterne (fino a 5A, PWM). |
| Diodi flyback | 2 | `SS34` o `1N5819` | In antiparallelo al carico: anodo al drain, catodo al +5V. Sopprimono la back-EMF. |
| Pull-down gate | 2 | `10kOhm` | Mantengono pompe e luci spente durante reset/riavvii dell'ESP32. |
| Uscite carichi | 2 | `JST-XH 2.54mm` | 2 pin per pompa dell'acqua o rele per strisce LED. |

---

## 4. Domande aperte e rischi di progetto

* **Dimensionamento di contorno del BQ25798:** architettura decisa (doppio
  ingresso ACFET/RBFET). Resta il lavoro di dettaglio: scelta esatta dei 4
  MOSFET N-channel (tensione, Rds_on, package), induttore buck-boost, passivi e
  rete del pin TS, da derivare dal datasheet / reference design TI (inclusa
  l'eventuale necessita di resistenze di sense).
* **Firmware e autonomia:** la claim di 6-12 mesi vale per il percorso
  Thread/Zigbee (Sleepy End Device + mesh relay), non per il Wi-Fi. Da
  validare con misure reali di consumo per ciclo di risveglio.
* **Sigillatura / enclosure:** la sezione superiore con la cella all'aperto
  necessita di un guscio stagno (gestione condensa ed escursioni termiche).
  Sara trattata insieme allo shell da stampare.
* **Calibrazione EC e umidita:** entrambe richiedono taratura e
  compensazione in temperatura; definire la procedura di calibrazione di
  fabbrica.

---

## 5. Riferimenti e progetti open correlati

* HiGrow / TTGO T-Higrow (design originale ESP32 per piante).
* b-parasite (rbaron): sensore open di umidita del suolo a basso consumo,
  riferimento per l'eccitazione capacitiva.
* Sensori di umidita capacitivi v1.2 / v2.0 (eccitazione + rilevatore di
  picco).
* ESPHome (firmware Wi-Fi opzionale).
* OpenThread / Matter / Zigbee (stack mesh a basso consumo).
* Datasheet: BQ25798 (charger buck-boost MPPT, I2C, reference design TI),
  DW01A + FS8205A (protezione cella), XC6220 / ME6211 (LDO),
  SX1308 / MT3608 (boost), ADS1115 (ADC).

---

## 6. Licenza

L'hardware di questo progetto (schematici, PCB, file CAD) e rilasciato sotto
**CERN Open Hardware Licence Version 2 - Strongly Reciprocal**
(SPDX: `CERN-OHL-S-2.0`). Il testo completo e nel file [LICENSE](LICENSE) ed e
disponibile anche su https://ohwr.org/cern_ohl_s_v2.txt

E una licenza reciproca forte (copyleft): chi distribuisce il progetto o
prodotti basati su di esso, anche modificati, deve rendere disponibili i
relativi file sorgente di progetto alle stesse condizioni.

Notice da apporre sui file di progetto:

```
Copyright (C) 2026 <titolare del copyright>

This source describes Open Hardware and is licensed under the CERN-OHL-S v2.
You may redistribute and modify this source and make products using it under
the terms of the CERN-OHL-S v2 (https://ohwr.org/cern_ohl_s_v2.txt).
This source is distributed WITHOUT ANY EXPRESS OR IMPLIED WARRANTY, INCLUDING
OF MERCHANTABILITY, SATISFACTORY QUALITY AND FITNESS FOR A PARTICULAR PURPOSE.
Please see the CERN-OHL-S v2 for applicable conditions.
```

> Nota: firmware e documentazione, quando aggiunti, potranno avere licenze
> proprie e piu adatte (es. GPLv3 per il firmware, CC BY-SA-4.0 per la
> documentazione).
