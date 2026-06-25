# Workflow KiCad - JLCPCB

Obiettivo: progettare in KiCad e arrivare a JLCPCB (PCB + assembly SMT) senza
sorprese, sapendo gia in fase di disegno se un componente e disponibile, se e
Basic o Extended, e quanto costa. Tutto il lavoro su LCSC/stock si fa dentro
KiCad, cosi la BOM resta sempre allineata allo schematico.

## 1. Strumenti

* **KiCad 8 o 9** (https://www.kicad.org).
* **Plugin `kicad-jlcpcb-tools` (Bouni)**, installabile dal Plugin and Content
  Manager di KiCad (cerca "JLCPCB"). E lo strumento principale: mantiene un
  mirror locale del database parti assembly di JLCPCB e permette di:
  * cercare e assegnare il codice LCSC a ogni componente direttamente dal PCB;
  * vedere **stock, prezzo e flag Basic/Extended** per ogni parte;
  * generare i file pronti per l'ordine: Gerber, **BOM** e **CPL** (posizioni).
* **`easyeda2kicad`** (opzionale): importa simbolo e footprint di un componente
  LCSC dentro KiCad quando non hai gia il footprint.
* **`jlcparts` (yaqwsx)** (opzionale): database/web di ricerca delle parti JLC,
  utile per controlli rapidi fuori da KiCad.

## 2. Procedura tipo

1. Disegna lo schematico in KiCad, assegnando i footprint.
2. Per ogni componente compila il campo **`LCSC`** con il codice `Cxxxxxx`
   (il plugin lo cerca e lo scrive per te). La bozza in [../bom/](../bom/)
   elenca tutte le parti con MPN e package come punto di partenza.
3. Apri il plugin JLCPCB dal PCB editor: per ogni riga vedi stock e tipo
   (Basic/Extended). Risolvi gli out-of-stock e, dove possibile, preferisci
   parti Basic per ridurre costo e fee.
4. Genera i file di fabbricazione dal plugin. Finiscono in `production/`
   (cartella ignorata dal versioning, vedi `.gitignore`).
5. Carica lo zip su JLCPCB, attiva "PCB Assembly", verifica il posizionamento
   nel visualizzatore (rotazioni/offset dei componenti) e conferma.

## 3. Regole pratiche per non sbagliare

* **Basic vs Extended:** i Basic non hanno feeder fee e sono sempre a
  magazzino. Resistenze, MLCC e MOSFET comuni (AO3400, SS34, ecc.) di solito lo
  sono. Gli Extended (es. BQ25798, BME280, modulo ESP32-C6) costano una fee per
  feeder: accettabili se risolvono problemi, ma tienili monitorati.
* **Stock al momento dell'ordine:** lo stock cambia. Verifica sempre dal plugin
  poco prima di ordinare, non mesi prima.
* **Lato assemblato:** assemblare un solo lato costa meno. Se puoi, tieni i
  componenti da assemblare tutti sullo stesso lato.
* **Rotazioni/offset CPL:** alcuni footprint hanno orientamento diverso tra
  KiCad e la libreria JLC. Il plugin gestisce molte correzioni, ma controlla i
  componenti polarizzati (diodi, IC, connettori) nel visualizzatore JLC.
* **Footprint = parte reale:** quando importi da LCSC con `easyeda2kicad`,
  verifica che il footprint corrisponda al package del codice LCSC scelto.

## 4. Nota sulla verifica automatica

Oltre al plugin, se vuoi un controllo scriptabile della BOM contro lo stock
JLC (utile in CI o prima di un ordine), si puo incrociare un CSV della BOM con
il database `jlcparts`. Non e prioritario: il plugin in KiCad copre il caso
d'uso quotidiano. Da aggiungere piu avanti se serve.
