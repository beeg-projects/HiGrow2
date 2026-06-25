# hardware

Progetto KiCad della scheda HiGrow 2.0.

Il progetto va creato qui dentro con nome `HiGrow2` (KiCad 8 o 9), in modo da
ottenere:

```
hardware/
  HiGrow2.kicad_pro    # progetto
  HiGrow2.kicad_sch    # schematico
  HiGrow2.kicad_pcb    # layout PCB
  HiGrow2.kicad_prl    # preferenze locali (puo essere ignorato dal versioning)
  sym-lib-table        # tabelle librerie (se servono librerie locali)
  fp-lib-table
  libs/                # eventuali simboli/footprint custom (es. picchetto, gold-fingers)
```

Convenzioni:

* I componenti vanno annotati con il campo `LCSC` (codice `Cxxxxxx`) per
  l'assembly JLCPCB. La verifica di stock e la generazione dei file di
  fabbricazione si fanno dal plugin KiCad, vedi
  [../docs/jlcpcb-kicad-workflow.md](../docs/jlcpcb-kicad-workflow.md).
* La distinta base di riferimento e in [../bom/](../bom/).
* Footprint custom (piastra capacitiva del picchetto, pad EC gold-fingers,
  socket batteria) vanno in `libs/` e versionati col progetto.
