# BOM draft - HiGrow 2.0

Bozza allineata alla sezione 3 del [readme](../readme.md). Codici LCSC da
assegnare in KiCad (vedi [README di questa cartella](README.md)).

Legenda Tipo previsto: B = Basic/Preferred (no fee), E = Extended (feeder fee),
P = passivo jellybean (quasi sempre Basic), - = non assemblato / meccanico.

## A. Cuore logico e interfaccia

| Funzione | Q.ta | MPN / Valore | Package | LCSC | Tipo | Note |
| :--- | :-- | :--- | :--- | :--- | :-- | :--- |
| Microcontrollore | 1 | ESP32-C6-WROOM-1 | modulo SMD | TBD | E | Variante 1U compatibile (antenna U.FL). |
| Connettore USB-C | 1 | USB Type-C 16 pin | SMT | TBD | B/E | Ricarica + USB Serial/JTAG nativo. |
| R pull-down CC | 2 | 5.1k | 0402/0603 | TBD | P | Sui pin CC1/CC2. |
| Pulsanti | 2 | tactile switch | SMD | TBD | B | EN (Reset) e GPIO9 (Boot). |
| R pull-up EN | 1 | 10k | 0402/0603 | TBD | P | |
| C debounce EN | 1 | 1uF | 0402/0603 | TBD | P | |
| Decoupling MCU | n | 100nF + bulk | 0402 / vari | TBD | P | Quantita da schematico. |

## B. Ricarica buck-boost, MPPT e ingresso solare

| Funzione | Q.ta | MPN / Valore | Package | LCSC | Tipo | Note |
| :--- | :-- | :--- | :--- | :--- | :-- | :--- |
| Charger buck-boost | 1 | BQ25798RQMR | VQFN | TBD | E | MPPT + ADC + I2C, 1-4S. |
| Induttore di potenza | 1 | ~1uH (da datasheet) | SMD power | TBD | B/E | Buck-boost, corrente elevata. |
| FET ingresso ACFET/RBFET | 4 | N-ch low Rds_on | SOT-23 / dual | TBD | B | 2 coppie su VAC1 (USB) e VAC2 (solare). |
| Connettore solare | 1 | JST-PH 2.0mm | SMD/TH | TBD | B | Ingresso pannello. |
| C ingresso/uscita | 2+ | 10uF (+ bulk) | 0805 / vari | TBD | P | VAC/VBUS e SYS/BAT. |
| C boot / REGN | 2+ | da datasheet | 0402/0603 | TBD | P | Bootstrap switch + bypass REGN. |
| R programmazione | n | da datasheet | 0402 | TBD | P | PROG/ILIM/indirizzo I2C. |
| LED stato carica | 1 | LED | 0603 | TBD | P | Su pin STAT (stato anche via I2C). |
| R LED stato | 1 | 2.2k-4.7k | 0402/0603 | TBD | P | |

## C. Batteria, protezione (BMS) e regolazione 3.3V

| Funzione | Q.ta | MPN / Valore | Package | LCSC | Tipo | Note |
| :--- | :-- | :--- | :--- | :--- | :-- | :--- |
| Socket batteria | 1 | socket plastico nero | SMT | TBD | - | 18650 / 21700. |
| Pad negativi | 2 | (rame PCB) | - | - | - | A 65mm e 70mm, no BOM. |
| Controller protezione | 1 | DW01A | SOT-23-6 | TBD | B | Protezione cella. |
| MOSFET protezione | 1 | FS8205A | SOT-23-6 | TBD | B | Dual N-ch, ramo negativo. |
| R rete DW01A | 1 | 100-470 ohm | 0402/0603 | TBD | P | |
| C rete DW01A | 1 | 0.1uF | 0402/0603 | TBD | P | |
| NTC temperatura | 1 | 10k B3950 | 0402/0603 | TBD | B | Su pin TS, nella fessura sotto la cella. |
| R rete pin TS | n | da datasheet | 0402 | TBD | P | Bias del pin TS. |
| LDO 3.3V | 1 | XC6220 / ME6211 | SOT-23-5 | TBD | B/E | Iq microampere. Alimentato da SYS. |
| C filtri LDO | 2 | 10uF | 0805 | TBD | P | In/out. |

> Rimossi rispetto allo schema precedente: partitore di lettura V-BAT e MOSFET
> 2N7002 (la tensione di cella la legge l'ADC del BQ25798 via I2C).

## D. Sensori ambientali e bus I2C

| Funzione | Q.ta | MPN / Valore | Package | LCSC | Tipo | Note |
| :--- | :-- | :--- | :--- | :--- | :-- | :--- |
| Sensore aria | 1 | BME280 | LGA | TBD | E | Temp/umidita/pressione, I2C. |
| MOSFET taglio sensori | 1 | AO3401 | SOT-23 | TBD | B | P-ch, scollega VCC sensori in deep sleep. |
| R pull-up I2C | 2 | 4.7k | 0402/0603 | TBD | P | Sul rail commutato dei sensori. |
| Connettori espansione | 2 | JST-SH 1.0mm 4p | SMD | TBD | B | VCC/GND/SCL/SDA (Qwiic/STEMMA). |
| ADC dedicato | 1 | ADS1115 | MSOP-10 | TBD | E | 16 bit, per umidita ed EC. |
| Oscillatore umidita | 1 | TLC555 | SOIC/SOT | TBD | B/E | Onda quadra per il capacitivo (alt: LEDC). |
| Diodo raddrizz. umidita | 1 | BAT54 | SOT-23 | TBD | B | Rilevatore di picco. |
| C rilevatore umidita | 1 | da taratura | 0402/0603 | TBD | P | |
| R serie umidita | 1 | da taratura | 0402/0603 | TBD | P | Sulla piastra capacitiva. |
| Elettrodi EC | 2 | gold-fingers ENIG | rame PCB | - | - | Pad inerti, no BOM. |
| R riferimento EC | 1 | 10k | 0402/0603 | TBD | P | Partitore noto per EC. |

## E. Automazione attiva (pompe e luci) e step-up

| Funzione | Q.ta | MPN / Valore | Package | LCSC | Tipo | Note |
| :--- | :-- | :--- | :--- | :--- | :-- | :--- |
| Chip boost 5V | 1 | SX1308 / MT3608 | SOT-23-6 | TBD | B | Abilitato via pin EN dall'ESP32. |
| Induttore boost | 1 | 4.7uH | SMD power | TBD | B | |
| Diodo boost | 1 | SS34 | SMA | TBD | B | Raddrizzamento step-up. |
| R feedback boost | 2 | da datasheet | 0402/0603 | TBD | P | Fissano l'uscita a 5V. |
| C filtri boost | 2 | 10-22uF | 0805/1206 | TBD | P | Output 5V pulito. |
| MOSFET carichi | 2 | AO3400 | SOT-23 | TBD | B | N-ch, fino a 5A, PWM. |
| Diodi flyback | 2 | SS34 / 1N5819 | SMA | TBD | B | Anodo al drain, catodo al +5V. |
| R pull-down gate | 2 | 10k | 0402/0603 | TBD | P | Carichi spenti durante reset. |
| Uscite carichi | 2 | JST-XH 2.54mm 2p | TH/SMD | TBD | B | Pompa o rele strisce LED. |
