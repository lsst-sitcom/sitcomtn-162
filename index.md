# Testing the implementation of Metadetection and Cell-Based Coadds on Abell 360 ComCam data

```{abstract}
The purpose of this technote is to test the technical quality of ComCam commissioning data, specifically the Rubin_SV_38_7 field, by utilizing cell-based coadds and Metadetection by measuring the tangential shear profile, and the cross shear profile, of the massive cluster Abell 360 (called A360 throughout the technote). The process entails generating the cell-based coadds for Metadetection to run on, identifying and removing cluster member galaxies, applying quality cuts, and calibrating the shear measurements. Once a shear profile is generated, validation is the bulk of the remaining analysis.

Cell-based coadds and Metadetection are both currently in the process of being implemented within the LSST Science Pipelines at the time of this technote. There is quite a bit of technical value in attempting a difficult measurement prior to full implementation. Measuring the tangential shear around A360 will showcase the current abilities of these algorithms, as well as highlight where work is still needed.
```

## Cell-Based Coadds Input

At the time of this analysis, cell-based coadds are not a part of the default LSST Science Pipeline and must be generated independently. The equivalent of the pipetask command below was run on the w_2025_17 weekly stack version of the Pipeline, along with customized branches in `drp_tasks` and `cell_coadds` using the branch `u/mirarenee/no_ap_corr`. The patches and tracts are those that fully or partially fall within 0.5 degrees of the Brightest Cluster Galaxy of A360 at RA, DEC of 37.865017, 6.982205.

```
pipetask run -j 4 --register-dataset-types  \
-b /repo/main \
-i LSSTComCam/runs/DRP/DP1/w_2025_17/DM-50530 \
-o u/$USER/ComCam_Cells/a360 \
-p /sdf/group/rubin/user/mgorsuch/ComCam/pipeline.yaml \
-d "((tract=10463 AND patch IN (30..34,40..44,50..54,60..64,70..74,80..84,90..94)) \
OR (tract=10464 AND patch IN (37..39,47..49,57..59,67..69,77..79,87..89,97..99)) \
OR (tract=10704 AND patch IN (0..5)) \
OR (tract=10705 AND patch IN (8, 9))) \
AND (band='g' OR band='r' OR band='i') AND skymap='lsst_cells_v1'"
```

The `pipeline.yaml` file used is shown below:

```yaml
description: A simple pipeline to test development of cell-based coadds in ComCam

instrument: lsst.obs.lsst.LsstComCam

tasks:
    makeDirectWarp:
        class: lsst.drp.tasks.make_direct_warp.MakeDirectWarpTask
        config:
            connections.calexp_list: preliminary_visit_image
            connections.visit_summary: visit_summary
            connections.warp: direct_warp
            connections.masked_fraction_warp: direct_warp_masked_fraction
            doWarpMaskedFraction : true
            doPreWarpInterpolation : true
    makePsfMatchedWarp:
        class: lsst.drp.tasks.make_psf_matched_warp.MakePsfMatchedWarpTask
        config:
            connections.direct_warp: direct_warp
            connections.psf_matched_warp: psf_matched_warp
    assembleDeepCoadd:
        class: lsst.drp.tasks.assemble_coadd.CompareWarpAssembleCoaddTask
        config:
            connections.inputWarps: direct_warp
            connections.psfMatchedWarps: psf_matched_warp
            doWriteArtifactMasks : true
    assembleCellCoadd:
        class: lsst.drp.tasks.assemble_cell_coadd.AssembleCellCoaddTask
        config:
            connections.inputWarps: direct_warp
            connections.visitSummaryList: visit_summary
```

There are a few reasons why there are additional tasks on top of the cell-based coaddition task, `assembleCellCoadd`. The primary input images for the cell-based coadds are warped images using the `makeDirectWarp` task. For cell-based coadds, the `doPreWarpInterpolation` configuration needs to be set to `True` manually, as the default pipeline setting is False. This `doPreWarpInterpolation` config is used to properly propagate the mask plane to the from the warps to the cell-based coadds. The `makePsfMatchedWarp` and `assembleDeepCoadd` tasks are needed to generate artifact masks, which are required inputs for the cell-based coaddition task to run.

The collections for the cell-based coadds are stored in `u/mgorsuch/a360_cell_coadd` for *r*- and *i*-bands, and `u/mgorsuch/a360_cell_coadd_g` for *g*-band (separate collections due to not running g-band initially).

Cell-based coadds are stored as patch-sized coadds, divided into 484 cell regions (22 by 22 cells). Individual cells have both inner and outer boundary boxes, which are 150 by 150 and 250 by 250 pixels, respectively.

Metadetection utilizes both inner and outer boundaries of cells, and does not duplicate objects from this overlap. However, there is also overlap between patches and tracts that Metadetection does not address, and this leads to duplicate objects within the object tables. Overlap between patches within the same tract is 2 cells wide, and duplicates are removed by removing objects found in the outer ring of cells of each patch. There is also significant overlap between tracts. Patches in tract 10463 that fully overlap with tract 10464 are ignored. After removing the fully overlapping patches, there’s still a 4 cell wide overlap on one side between tract 10463 and 10464. Overlapping cells are again removed. Currently, the WCS information is not enough to remove exact duplicates based on RA and DEC. Why this is the case requires further investigation.

[Plot for Input Image Distribution]

[Plot for PSF ellipticity Distribution]

Note that for cell-based coadds, there is a single PSF model for each cell, realized at the center of the cell. The distribution of PSF ellipticities are seen in Fig. [...]

## Running Metadetection

...

The Metadetection shear catalog for this technote was run on the w_2025_17 weekly stack version of the Pipeline, along with customized branches in `drp_tasks` and `metadetect` using the branch `u/mirarenee/meta_test`, since a few minor changes were needed to run Metadetection on more recent pipeline stacks.

The pipetask command used to generate the Metadetection catalog is found below:

```
pipetask run -j 4 --register-dataset-types \
-b /repo/main \
-i refcats,u/mgorsuch/a360_cell_coadd,u/mgorsuch/a360_cell_coadd_g \
-o u/$USER/metadetect/a360_3_band \
-p /sdf/group/rubin/user/mgorsuch/notebooks/metadetect/comcam_pipeline.yaml \
-d "skymap='lsst_cells_v1'"
```

The associated `comcam_pipeline.yaml` file used for defining the tasks is outline below:

```yaml
description: Pipeline for running metadetection on ComCam
instrument: lsst.obs.lsst.LsstComCam

tasks:
    metadetectionShear:
        class: lsst.drp.tasks.metadetection_shear.MetadetectionShearTask
        config:
            connections.ref_cat: the_monster_20250219
            required_bands : ["g", "r", "i"]
            python: |
               from metadetect.lsst.configs import get_config as get_mdet_config
               mdet_config = get_mdet_config()
               mdet_config['metacal']['types']=['noshear', '1p', '1m', '2p', '2m']

               config.ref_loader.filterMap = {'lsst_'+band: 'monster_ComCam_%s' % (band) for band in 'ugrizy'}
```

Currently, with the custom branches, the only python line needed in the pipeline file is the filter map. The custom branches have a workaround in place for changing Metadetection specific configs for the time being, though the additional python lines should be sufficient for changing those configs through the pipeline file in the future.

An important note is that, for this analysis, a single noise image is generated for each cell within Metadetection. The noise image is generated from a Gaussian distribution with a standard deviation equal to the median variance of the image. It’s possible to produce multiple noise images prior to the warping process in order to better account for noise correlation across pixels, though this extra warping is computationally expensive and skipped for now.

The output catalogs contain useful information on object detection and measurement. There are a total of 5 “sub-catalogs”, one being the non-sheared objects, two with objects sheared in the plus/minus g_1 direction, and the final two catalogs sheared in the plus/minus g_2 direction. Each of these sub-catalogs retains information on the location, shape, and flux of each detected object. Since shearing objects affects detection, each catalog has a slightly differing number of objects detected. These catalogs cannot be directly compared, but can act as independent realizations.

...

## Red Sequence Galaxy Identification

...

A series of color-magnitude plots with progressive cuts is used to identify the red sequence (RS) galaxies. Each cut is applied to the entire catalog, though only the non-sheared catalog is shown in the color-magnitude plots. Previous visual inspection showed that while there is some variation in  between shear type catalogs, the variation is minimal and random enough that applying the same cuts should be sufficient for this analysis.

...

## Selection Cuts

...

## Shear Calibration

Understanding the relationship between the shear applied to an object and the effect of that shear on measuring the object’s shape is a critical step to calibrating the catalog’s shear measurements. The main purpose of Metadetection’s 5 sub-catalogs is to calculate the linear response matrix, R.

The main assumption is that the weak lensing shear signal is small enough that we can Taylor expand the measured galaxy ellipticity about a zero shear signal. The first term goes to 0 in the limit of a large enough sample of galaxies where the shape noise averages out. The relation between the measured galaxy ellipticity and the resulting shear signal is controlled by R, the linear response of the ellipticity to an applied shear. Within the Metadetection framework, the components of R can be calculated by taking the mean measured galaxy ellipticities of the 4 artificially shear object catalogs. The magnitude of the applied shear in each catalog is 0.01, to make $\Delta\gamma_j$ a total of 0.02 for each component of R. Once R is calculated, it’s applied to the mean non-sheared object ellipticities to produce the calibrated shear.

Calibration equations...

### Shear Results

..

## Validation & Testing

..

### Object Size and S/N Cuts

..

### False Detections

..