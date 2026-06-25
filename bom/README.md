# bom

Distinta base (BOM) di riferimento per HiGrow 2.0.

* [bom_draft.md](bom_draft.md): BOM leggibile, raggruppata per blocco
  funzionale, allineata alla sezione 3 del [readme](../readme.md).
* [bom_draft.csv](bom_draft.csv): stessa BOM in formato tabellare, comodo per
  import/confronto.

## Stato dei codici LCSC

La colonna `LCSC` e volutamente **TBD**: i codici e lo stock JLCPCB vanno
assegnati e verificati direttamente in KiCad con il plugin
`kicad-jlcpcb-tools`, che mostra disponibilita e flag Basic/Extended in tempo
reale (vedi [../docs/jlcpcb-kicad-workflow.md](../docs/jlcpcb-kicad-workflow.md)).
Non riportiamo qui codici LCSC "a memoria" per non rischiare riferimenti
sbagliati o stock non piu validi.

La colonna `Tipo previsto` e solo una stima (Basic / Extended / Passivo) per
orientare le scelte: va confermata in KiCad.

I valori marcati "da datasheet" (passivi di contorno del BQ25798, rete TS,
feedback del boost) si fissano sul reference design TI in fase di schematico.
