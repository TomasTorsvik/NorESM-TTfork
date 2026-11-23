# SST pacemaker usermods (Kuroshio)

These usermods run NorESM with an SST "pacemaker": inside a masked region the
sea surface temperature that BLOM sends to the coupler is replaced by an
externally prescribed SST, so the atmosphere and sea ice see the prescribed
SST there.

## Design

- Only the SST sent to the coupler (`So_t` in the ocean-to-coupler fields) is
  modified. BLOM's own prognostic temperature is **not** nudged or otherwise
  changed, and all other exported fields and fluxes are unchanged.
- The exported SST is `temp + pcmask*(So_t - temp)`, where `temp` is the BLOM
  surface temperature, `So_t` the prescribed SST and `pcmask` a mask between 0
  (model SST) and 1 (prescribed SST). Intermediate mask values give a smooth
  transition at the edge of the region.
- The mask and the prescribed SST are read by BLOM SourceMods
  (`SourceMods/src.blom`); the blending is done in the BLOM MCT driver
  (`SourceMods/src.drv`). See the Readme files in those directories.

## Required BLOM version

The SourceMods are BLOM v1.4.7 files with the pacemaker changes added. They
need a BLOM whose `cime_config/namelist_definition_blom.xml` defines the
`pacemaker` namelist group with `pacemaker_mask_file`, `pacemaker_SST_file`
and `pacemaker_year_align`. Currently this is the BLOM branch
`feature/pacemaker`, based on v1.4.7.

## Namelist (set in `user_nl_blom`)

All three variables belong to the namelist group `pacemaker` and must be set;
the model stops with an error otherwise.

    pacemaker_mask_file  = '/path/to/pacemaker_mask.nc'
    pacemaker_SST_file   = '/path/to/SST_file.nc'
    pacemaker_year_align = 1320

- `pacemaker_mask_file`: file with the mask variable `pc_mask`.
- `pacemaker_SST_file`: file with the daily SST variable `sst` [degC].
- `pacemaker_year_align`: the model year whose 1 January corresponds to record
  1 of the SST file (default -999 = unset).

## Input file layout

Both files must be on the BLOM model grid (`itdm` x `jtdm`).

- Mask file: variable `pc_mask` with NetCDF dimensions `(y, x)`.
- SST file: variable `sst` with NetCDF dimensions `(y, x, time)` (as shown by
  `ncdump`), one record per model day, record 1 being 1 January of model year
  `pacemaker_year_align`. The days must follow the model calendar (e.g.
  `noleap`, 365 records per year). Because `time` is the last NetCDF
  dimension it cannot be an unlimited dimension in classic-format files. SST
  values are only used where `pc_mask > 0`, so values elsewhere (land or
  outside the mask, including NaN or fill values) are ignored. Inside the mask
  the SST must be valid ocean temperatures in degC.

## Time handling

The SST record is computed from the model calendar date:

    record = (days from 1 January of pacemaker_year_align to the model date) + 1

The SST file is opened once and a new record is read only when the model date
moves to a new day, i.e. once per model day. Each read is logged in the BLOM
log as `get_So_t: read pacemaker SST record ...`. Because the record follows
the model date, the same setting works for startup, hybrid, branch and
continue runs. The model stops with an error if the model date falls outside
the records available in the SST file.

Example: with `pacemaker_year_align = 1320`, model date 1320-01-01 uses record
1, 1320-12-31 record 365 and 1321-01-01 record 366 (noleap calendar). A run
starting at model date 1325-01-01 with the same setting starts at record 1826.

## Other regions

To run a pacemaker experiment in a different region, provide a different mask
file (and an SST file covering the region) via the namelist; no code changes
are needed.
