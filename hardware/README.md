# hardware

Progetto KiCad della scheda HiGrow 2.0 (impostato su KiCad 10).

La base del progetto e gia presente: schematico e PCB **vuoti**, validati con
`kicad-cli`, pronti da popolare.

```
hardware/
  HiGrow2.kicad_pro    # progetto (presente)
  HiGrow2.kicad_sch    # schematico vuoto (presente)
  HiGrow2.kicad_pcb    # PCB vuoto, 2 strati (presente)
  HiGrow2.kicad_prl    # preferenze locali, generate da KiCad (non versionate)
  sym-lib-table        # tabelle librerie locali (da aggiungere se servono)
  fp-lib-table
  libs/                # simboli/footprint custom (es. picchetto, gold-fingers)
```

Apri `HiGrow2.kicad_pro` con KiCad per iniziare lo schematico.

Convenzioni:

* I componenti vanno annotati con il campo `LCSC` (codice `Cxxxxxx`) per
  l'assembly JLCPCB. La verifica di stock e la generazione dei file di
  fabbricazione si fanno dal plugin KiCad, vedi
  [../docs/jlcpcb-kicad-workflow.md](../docs/jlcpcb-kicad-workflow.md).
* La distinta base di riferimento e in [../bom/](../bom/).
* Footprint custom (piastra capacitiva del picchetto, pad EC gold-fingers,
  socket batteria) vanno in `libs/` e versionati col progetto.
