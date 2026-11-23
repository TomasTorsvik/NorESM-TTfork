# BLOM SourceMods (src.blom) for the SST pacemaker

These files are the BLOM v1.4.7 versions of `phy/mod_grid.F90` and
`phy/geoenv_file.F` with only the pacemaker additions. They require a BLOM
with the `pacemaker` namelist group in `cime_config/namelist_definition_blom.xml`
(currently branch `feature/pacemaker`, based on v1.4.7). See the top-level
`Readme.md` of these usermods for the overall design: only the SST sent to the
coupler is overridden; BLOM's own temperature is not nudged.

## mod_grid.F90

Adds to the grid module:

- `pcmask`: pacemaker mask [0-1] on the local tile.
- `So_t`: prescribed pacemaker SST [degC] on the local tile (filled by the
  driver, see `src.drv`).
- `pacemaker_mask_file`, `pacemaker_SST_file`: file names (default `'unset'`).
- `pacemaker_year_align`: model year whose 1 January corresponds to record 1
  of the SST file (default -999 = unset).

`pcmask` and `So_t` are initialised together with the other grid arrays in
`inivar_grid`.

## geoenv_file.F

- Reads the namelist group `pacemaker` from `ocn_in` on the master task:

      pacemaker_mask_file  = '/path/to/pacemaker_mask.nc'
      pacemaker_SST_file   = '/path/to/SST_file.nc'
      pacemaker_year_align = 1320

  Set these in `user_nl_blom`. All three must be set, otherwise the model
  stops with an error.
- Reads the mask variable `pc_mask` (NetCDF dimensions `(y, x)`, model grid)
  from `pacemaker_mask_file` and distributes it to all tasks as `pcmask`.
- Broadcasts `pacemaker_year_align` to all tasks.

The mask is 1 in the core of the pacemaker region and tapers smoothly to 0 at
its edges. The current mask file covers the Kuroshio region in the North
Pacific; other regions only need a different mask file.
