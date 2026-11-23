# BLOM MCT driver SourceMods (src.drv) for the SST pacemaker

These files are the BLOM v1.4.7 versions of `drivers/mct/ocn_comp_mct.F90` and
`drivers/mct/sumsbuff_mct.F` with only the pacemaker changes. They require a
BLOM with the `pacemaker` namelist group in
`cime_config/namelist_definition_blom.xml` (currently branch
`feature/pacemaker`, based on v1.4.7) and the `src.blom` SourceMods of these
usermods.

## Design

Only the SST sent to the coupler is overridden. BLOM's own temperature is not
nudged, and no other exported field or flux is changed. In the masked region
the atmosphere and sea ice therefore see the prescribed SST, while the ocean
itself evolves freely.

## sumsbuff_mct.F

When accumulating the SST in the send buffer, the model surface temperature
`temp` is blended with the prescribed SST `So_t` using the mask `pcmask`:

    SST sent to coupler = temp + pcmask*(So_t - temp)

With `pcmask = 0` this is the model SST, with `pcmask = 1` the prescribed SST.

## ocn_comp_mct.F90

Adds the private subroutine `get_So_t`, called before the initial export in
`ocn_init_mct` and after every `blom_step` in `ocn_run_mct`, and closes the SST
file in `ocn_final_mct`.

`get_So_t`:

- Computes the required record from the model calendar date on every task:
  `record = daynum_diff(calendar, 1 January of pacemaker_year_align, date) + 1`.
  BLOM advances `date` once per model day, so the record changes once per
  model day.
- Returns immediately if that record is already loaded.
- On first use, opens `pacemaker_SST_file` once on the master task, looks up
  variable `sst`, checks its grid size and reads the length of its time
  dimension, which is then broadcast to all tasks.
- Stops with an error if the record is outside 1 to the number of records.
- Reads the record on the master task, distributes it to all tasks as `So_t`,
  and writes one log line (`get_So_t: read pacemaker SST record ...`).

Because the record follows the model date, startup, hybrid, branch and
continue runs need no special treatment.

## SST file layout

Variable `sst` [degC] with NetCDF dimensions `(y, x, time)` on the BLOM model
grid, one record per model day following the model calendar, record 1 being 1
January of model year `pacemaker_year_align`. The `time` dimension cannot be
unlimited in classic-format files because it is the last NetCDF dimension.
