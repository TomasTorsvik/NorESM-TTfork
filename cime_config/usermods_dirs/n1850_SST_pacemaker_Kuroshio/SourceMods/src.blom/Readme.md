Small modifications were made to the BLOM source code to enable reading and initialization of the pacemaker mask variable (pcmask) from an external NetCDF file.
These updates allow BLOM to include region-specific SST forcing (overiding the BLOM SST) directly during grid initialization.

Files Modified:

1. mod_grid.F90

  Added declaration of the new variable pcmask to the grid module.

  Ensures the mask is initialized together with other grid variables in inivar_grid().

2. geoenv_file.F

  Updated to read the pacemaker_mask variable from the pacemaker_mask.nc file.

  Integrated pcmask loading alongside other grid fields.

  Ensures the mask is available globally after grid initialization.

Description of pcmask:

  Variable name: pcmask

  Source file: pacemaker_mask.nc

The mask has 1.0 in the core of the pacemaker region to 0.0 at the edges of the region (smooth tapering).

Current configuration:

The mask is set for the Kuroshio region in the North Pacific.

File names (namelist):

  The mask file and the external SST file are given by the namelist group
  `pacemaker` (read in geoenv_file.F from the BLOM namelist file), e.g. in
  user_nl_blom:

    pacemaker_mask_file = '/path/to/pacemaker_mask.nc'
    pacemaker_SST_file  = '/path/to/SST_file.nc'

  Both must be set, otherwise the model stops with an error. The variable names
  in the files are still fixed: `pc_mask` (mask) and `sst` (SST).
