# Testing the implementation of Metadetection and Cell-Based Coadds on Abell 360 LSSTComCam data

```{abstract}
(Full author list to be determined)

The purpose of this technote is to test the technical quality of LSSTComCam commissioning data, specifically the Rubin_SV_38_7 field, by utilizing cell-based coadds and Metadetection by measuring the tangential and cross weak lensing shear profiles of the massive cluster Abell 360 (called A360 throughout the technote). The process entails generating the cell-based coadds for Metadetection to run on, identifying and removing cluster member galaxies, applying quality cuts, and calibrating the shear measurements. Once a shear profile is generated, validation is the bulk of the remaining analysis.

Cell-based coadds and Metadetection are both currently in the process of being implemented within the LSST Science Pipelines at the time of this technote. There is substantial technical value in attempting a difficult measurement prior to full implementation. Measuring the tangential shear around A360 will showcase the current abilities of these algorithms, as well as highlight where work is still needed.

As seen from the resulting shear profile of A360, the cell-based coadds and Metadetection are able to work in tandem to produce a shear catalog. The shear profile performs best in radial bins further away from the cluster center (beyond ~ 2 Mpc), which may be due to high occurances of blending near the cluster center.

This technote is one part of a series studying A360 in order to both stress test the commissioning camera  and demonstrate the technical capabilities of the Vera Rubin Observatory. We study the quality of the PSF modeling and impact it can have on cluster WL in {cite:p}`SITCOMTN-161`, implementation of cell-based coadds and subsequent use for Metadetect {cite:p}`Sheldon_2023` in this technote, photometric calibration in (in prep), source selection and photometric redshifts in {cite:p}`SITCOMTN-163`, use of Anacal {cite:p}`Li_2023` to produce a cluster shear profile in {cite:p}`SITCOMTN-164`, and background subtraction in this field and Fornax in {cite:p}`SITCOMTN-165`.
```

## Cell-Based Coadds Input

At the time of this analysis, cell-based coadds are not a part of the default LSST Science Pipelines and must be generated independently. The input images and catalogs used to generated the cell-based coadds and other analyses in this technote are from the LSST DRP1 ({cite:p}`RTN-095good`, {cite:p}`SITCOMTN-149`), focusing on images taken on the Rubin LSSTComCam {cite:p}`ComCam`. The patches and tracts used are those that fully or partially fall within 0.5 degrees of the Brightest Cluster Galaxy of A360 at RA, DEC of 37.865017, 6.982205. For more specific information on cell-based coadds in a shear context, see {cite:p}`cell_slides` for introductory material and {cite:p}`lsst_coadd` for impacts of cell-based coadds on shear measurements. The collection for the cell-based coadds is `u/mgorsuch/ComCam_Cells/a360/corr_noise_cells/20250822T224002Z`, which includes the coadds for the *g*, *r*, and *i*-bands. The cell-based coadds are stored as patch-sized coadds, divided into 484 cell regions (22 by 22 cells). Individual cells have both inner and outer boundary boxes, which are 150 by 150 and 250 by 250 pixels, respectively.

As development for the cell-based coadds progressed, some tasks were rerun on top of previous tasks to reduce processing time. For transparency, the exact `pipetask` commands, pipeline files, and package versions used will be outlined in the next subsection. As for clarity, the condensed versions of the `pipetask` command and pipeline file will be shown immediately below, though these may not be exactly equivalent to what was actually run.

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
            numberOfNoiseRealizations : 1
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
            num_noise_realizations : 1
```

There are a few reasons why there are additional tasks on top of the cell-based coaddition task, `assembleCellCoadd`. The primary input images for the cell-based coadds are warped images using the `makeDirectWarp` task. For cell-based coadds, the `doPreWarpInterpolation` configuration needs to be set to `True` manually, as the default pipeline setting is False. This `doPreWarpInterpolation` config is used to properly propagate the mask plane from the warps to the cell-based coadds. The `makePsfMatchedWarp` and `assembleDeepCoadd` tasks are needed to generate artifact masks, which are required inputs for the cell-based coaddition task to run.

Metadetection utilizes both inner and outer boundaries of cells. However, the implementation used for this analysis does not yet remove duplicate objects detected from overlaps between cells, patches, and tracts. This means that the output catalogs will contain duplicate objects that need to be manually removed. Objects that are beyond the inner region of their assigned cell will be simply removed. Overlap between patches within the same tract is 2 cells wide, and duplicates are removed by removing objects found in the outer ring of cells of each patch. There is also significant overlap between tracts. Patches in tract 10463 that fully overlap with tract 10464 are ignored. After removing the fully overlapping patches, there’s still a 4 cell wide overlap on one side between tract 10463 and 10464. Overlapping cells are again removed.

```{figure} _static/3_band_image_distribution.png
:name: image_dist

These three figures show the input image distribution for the patches, each composed of 484 cells, around A360 in the g, r, and i-bands. The red squares outline the inner patch boundaries, where the 2 cell overlap is visible. The three missing patches are due to processing errors when running Metadetection, though they do not significantly overlap with the 0.5 degree radius around the BCG (cyan circle). Additionally, the 0.3 degree radius (orange circle) contains no missing patches; this radius contains the upper limit of the data used for the shear profile after binning is applied. The colorbar is scaled to the minimum and maximum number of inputs images across the three bands (1 and 16, respectively).
```

```{figure} _static/3_band_psf_e_distribution.png
:name: ellip_dist

PSF ellipticity modulus distribution with one PSF realization per cell for the patches around A360 in the *g*, *r*, and *i*-bands. The red squares outline the inner patch boundaries. The *i*-band plot is in good agreement with a similar plot in {cite:p}`SITCOMTN-161`. The mean ellipticities are 0.0563, 0.0689, and 0.0987 for the *g*, *r*, and *i*-bands, respectively. The colorbar is scaled from 0 to the highest value across the 3 bands, which is about 0.255.
```

Note that for cell-based coadds, there is a single PSF model for each cell, realized at the center of the cell. The distribution of PSF ellipticities are seen in {numref}`ellip_dist`.

### Exact commands used for generating cell-based coadds

The exact commands used to generate the cell-based coadds become quite long and are collected here for transparency and to avoid cluttering other sections.

The two following `pipetask` commands were used to initially generate the cell-based coadds. These collections will also contain the `psf_matched_warp`s and artifact masks needed for later commands. The list is tracts is only shown in the first command, though this list is the same for all commands. These commands below were run on the `w_2025_17` weekly stack version of the Pipeline, with the exception of a few user branches in ['drp_tasks'](https://github.com/lsst/drp_tasks/tree/u/mirarenee/no_ap_corr) and ['cell_coadds'](https://github.com/lsst/cell_coadds/tree/u/mirarenee/no_ap_corr), with both under the branch `u/mirarenee/no_ap_corr`. The pipeline file used was nearly the same as the one listed above, with the only exceptions being the `numberOfNoiseRealizations` and `num_noise_realizations`, which were not yet utilized.

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
AND (band='i' OR band='r') AND skymap='lsst_cells_v1'"
```

```
pipetask run -j 4 --register-dataset-types  \
-b /repo/main \
-i LSSTComCam/runs/DRP/DP1/w_2025_17/DM-50530 \
-o u/$USER/ComCam_Cells/a360_g \
-p /sdf/group/rubin/user/mgorsuch/ComCam/pipeline.yaml \
-d "tract=10463 ... AND (band='g') AND skymap='lsst_cells_v1'"
```

The `direct_warps` were then recreated in order to generate noise realizations that go through the warping process. These were again run on the weekly stack `w_2025_17` and the `u/mirarenee/no_ap_corr` branches for the `drp_tasks` and `cell_coadds` packages.

```
pipetask run -j 4 --register-dataset-types  \
-b /repo/main \
-i LSSTComCam/runs/DRP/DP1/w_2025_17/DM-50530,u/$USER/ComCam_Cells/a360,u/$USER/ComCam_Cells/a360_g \
-o u/$USER/ComCam_Cells/a360/corr_noise \
-p /sdf/group/rubin/user/mgorsuch/ComCam/pipeline-warp-cell.yaml \
-d "tract=10463 ... AND (band='i' OR band='r' OR band='g') AND skymap='lsst_cells_v1'"
```

The `pipeline-warp-cell.yaml` file used is below:
```yaml
description: A simple pipeline to test development of cell-based coadds in ComCam. This assumes that there is a collection already available containing the artifact masks created using MakePsfMatchedWarpTask and CompareWarpAssembleCoaddTask.

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
            numberOfNoiseRealizations : 1
    assembleCellCoadd:
        class: lsst.drp.tasks.assemble_cell_coadd.AssembleCellCoaddTask
        config:
            connections.inputWarps: direct_warp
            connections.visitSummaryList: visit_summary
```

 The `pipetask` command below was used to incorporate the noise realization implementation in the cell-based coadds. The command was run on the `w_2025_34` weekly stack version of the Pipeline, with the exception of a few ticket branches in ['drp_tasks'](https://github.com/lsst/drp_tasks/tree/tickets/DM-43585) and ['cell_coadds'](https://github.com/lsst/cell_coadds/tree/tickets/DM-43585), with both under the branch `tickets/DM-43585`.

 ```
pipetask run -j 4 --register-dataset-types  \
-b /repo/main \
-i u/$USER/ComCam_Cells/a360/corr_noise \
-o u/$USER/ComCam_Cells/a360/corr_noise_cells \
-p /sdf/group/rubin/user/mgorsuch/ComCam/pipeline-ap.yaml \
-d "tract=10463 ... AND (band='i' OR band='r' OR band='g') AND skymap='lsst_cells_v1'"
```

The `pipeline-ap.yaml` used is found below:
```yaml
description: A simple pipeline to test development of cell-based coadds in ComCam

instrument: lsst.obs.lsst.LsstComCam

tasks:
    assembleCellCoadd:
        class: lsst.drp.tasks.assemble_cell_coadd.AssembleCellCoaddTask
        config:
            connections.inputWarps: direct_warp
            connections.visitSummaryList: visit_summary
            num_noise_realizations : 1
```

## Running Metadetection

Metadetection ({cite:p}`meta_hm`, {cite:p}`Sheldon_2017`, {cite:p}`Sheldon_2020`, and most recently {cite:p}`Sheldon_2023`) is a shear calibration software focused on an empirical approach of artificially shearing images of galaxies to measure the response calibration matrix R, which is then applied to the unsheared images to calibrate their shear measurements. Metadetection is the sequel software to the original Metacalibration. The primary difference between the two is that while Metacalibration measures the shear response of individual objects for calibration, Metadetection is designed to detect and measure after the applied shear, resulting in 5 catalogs of shear types (non-sheared, in the plus/minus $g_1$ direction, and in the plus/minus $g_2$ direction). The main consequence of this is that since detection is shear-dependent, as seen in {cite:p}`Sheldon_2020`, the 5 Metadetection catalogs do not have necessarily the same objects, and cannot be matched to each other; shear is instead calibrated using the mean shape values.

Metadetection is currently being integrated into the LSST Science Pipelines as a pipeline task to fully utilize the cell-based coaddition based tasks in the pipeline structure. Since this is a work-in-progress, the packages used here are not finalized.

Detection is done on an inverse variance weighted average coadd of the three individual band coadds. As for measurement, the algorithm used here is `gauss`, a forward model that jointly fits each object across the three bands, *g*, *r*, and *i*. These measurements are done pre-PSF, so that the flux measurements are less sensitive to changes in the PSF between filters. This feature will be useful for the color-based red sequence cuts.

The Metadetection shear catalog for this technote was run on the `w_2025_34` weekly stack version of the Pipeline. As for packages, [`metadetect`](https://github.com/lsst/metadetect/tree/lsst-dev) used the `lsst-dev` branch, the ['cell_coadds'](https://github.com/lsst/cell_coadds/tree/tickets/DM-43585) package again used the `tickets/DM-43585` branch, and the [`drp_tasks`](https://github.com/lsst/drp_tasks/tree/u/mirarenee/meta_test/python/lsst/drp/tasks) package used a custom branch `u/mirarenee/meta_test`. The `drp_tasks` user branch was needed to add a few minor changes in order to run Metadetection on more recent pipeline stacks.

The pipetask command used to generate the Metadetection catalog is found below:

```
pipetask run -j 4 --register-dataset-types \
-b /repo/main \
-i refcats,u/mgorsuch/ComCam_Cells/a360/corr_noise_cells \
-o u/$USER/metadetect/a360_3_band/noise \
-p /sdf/group/rubin/user/mgorsuch/notebooks/metadetect/comcam_pipeline.yaml \
-c metadetectionShear:shape_fitter='gauss' \
-d "skymap='lsst_cells_v1'"
```

The associated `comcam_pipeline.yaml` file used for defining the tasks is outline below:

```yaml
description: Pipeline for running metadetection on DP1
instrument: lsst.obs.lsst.LsstComCam

tasks:
    metadetectionShear:
        class: lsst.drp.tasks.metadetection_shear.MetadetectionShearTask
        config:
            required_bands : ["g", "r", "i"]
            connections.ref_cat: the_monster_20250219
            shape_fitter: "gauss"
            python: |
                config.ref_loader.filterMap = {'lsst_'+band: 'monster_ComCam_%s' % (band) for band in 'ugrizy'}
```

An important note is that, for this analysis, a single noise image is associated for each cell within Metadetection. A noise image is generated for each warp that is also subjected to the warping process, which is needed to accurately capture correlated noise ({cite:p}`Sheldon_2017`, {cite:p}`Sheldon_2020`). The noise distribution is randomly pulled from a Gaussian distribution centered at 0 with a variance equal to the median variance of the input image.

After removing duplicated objects from patch overlap, there are a total of 1174579 object rows across the five shear type catalogs. After removing objects flagged by Metadetection (i.e. objects cut due to measurement failures), there are 967436 objects, about 17.6% of the initial catalog. The number of flagged objects is quite high, and requires further investigation.

## Red Sequence Galaxy Identification

Cluster member galaxies of the lensing cluster structure will not have a lensing signal (at least not from the cluster itself). Due to this, these galaxies need to be identified and removed from the lensing sample to avoid diluting the shear signal ({cite:p}`signal_dilution`). These lensing galaxies are primarily identified through visual inspection using color-magnitude plots across three different bands in this technote. Other methods, like photometric redshifts, may also be used, though using only three available bands limits this approach.

```{figure} _static/object-magnitudes.png
:name: obj-mags

The distribution of object magnitudes, from Metadetection flux measurements, in each of the bands used. This distribution is after duplicates are removed and Metadetection flagged objects are cut, though prior to red sequence galaxy identification and selection cuts. The red vertical lines are the current magnitude cuts at the limiting magnitude, determined by eye. These magnitude cuts are applied after the RS galaxy cut.
```

A series of color-magnitude plots with progressive cuts is used to visually identify the red sequence (RS) galaxies. Each cut is applied to the entire catalog, though only the non-sheared catalog is shown in the color-magnitude plots. Previous visual inspection showed that while there is some variation in-between shear type catalogs, the variation is minimal and random enough that applying the same cuts across all catalogs should be sufficient for this analysis.

The catalog is first cut to galaxies less than 0.1 degree away from the BCG to focus on galaxies that are more likely to be cluster members. The red sequence cluster members are identified in a line of objects with relatively consistent color across a range of magnitudes, with the line being more apparent in the smaller sample of galaxies. This line of galaxies is highlighted with orange points, with the upper and lower limits shown in red. The same visual inspection is done again for the larger sample of galaxies, those within 0.5 degrees of the BCG.

RS galaxies are selected if they satisfy one of the two requirements in all 3 color-magnitude diagrams: the data point falls within the range identified during visual inspection, **or** the 2-$\sigma$ error bars for the color measurement intersect the RS range. The error bar requirement is meant to capture potential RS galaxies that may fall out of the selection region due to noisy measurements, particularly at the fainter end. The RS identification is then also limited to galaxies with `gauss_band_mag_r` < 24. While RS galaxies may be fainter than this, it becomes more difficult to distinguish between RS and background galaxies. Since unsheared foreground galaxies will dilute the signal, but not bias it, a small contamination is allowed to keep the many source galaxies that may otherwise be cut. Any remaining bright galaxies (those with `gauss_band_mag_i` < 20) are removed below as described in the selection cut section.

Once the RS galaxies are identified, they are removed from the sample of galaxies within 0.5 degrees of the BCG, which the becomes the sample that goes on the further selection cuts.

```{figure} _static/01-RS-all_cuts.png
:name: color-magnitude-0-1-orange

Color-magnitude diagram cut to 0.1 degrees within the BCG. Objects that are included within the cut are highlighted in orange.
```

```{figure} _static/05-RS-all_cuts.png
:name: color-magnitude-0-5-orange

Color-magnitude diagram cut to 0.5 degrees within the BCG. Objects that are included within the cut are highlighted in orange.
```

```{figure} _static/05-RS-removed.png
:name: color-magnitude-0-5-removed

Color-magnitude diagram cut to 0.5 degrees within the BCG. Objects that are included within the cut are removed.
```

After applying the 0.5 degree cut, but prior to removing RS objects, there are 595377 object rows. After applying the RS cuts, there are 258449, with 51633 (~20%) of which are the non-sheared catalog.

## Selection Cuts

After Metadetection flags are applied and red sequence galaxies are identified and removed, additional selection cuts are applied. These cuts are primarily based on {cite:p}`yamamoto`, though are customized in several cases to better fit the data from LSSTComCam. A detailed outline of the cuts used and object removed can be seen in {numref}`selection_cuts`. In numerical terms, the number of objects remaining after selection cuts is 78500, with 15819 (20%) in the non-sheared catalog.

The Yamamoto cuts describe a size ratio cut, defined as the size of the object squared divided by size of the PSF squared, or $T^{gauss}/T^{gauss}_{PSF}$. This is used as a star-galaxy cut. For the Yamamoto measurements and the measurements here, these sizes are measured for the pre-PSF objects, so stars will hover around 0 for this ratio. Based on {numref}`obj_T_vs_s2n`, the stars identified around `gauss_T_ratio` = 0 appear to fall consistently below `gauss_T_ratio` = 0.2. This smaller value is chosen instead for the `gauss_T_ratio` cut, since the inclusion of extra galaxies in the weak lensing sample outweighs the potential inclusion of some low S/N stars.

There are a few additional differences from the {cite:p}`yamamoto` cuts. The magnitude cut is based off of {numref}`mag-cut-hist2` and {cite:p}`SITCOMTN-161`, while the junk cuts and size cuts are specific to DES and don't seem to appear to affect LSSTComCam data. Additionally, masks for bright objects, galactic cirrus, SFD dust, and areas with low numbers of exposures are applied. Mask details can be found in (in prep technote).

:::{table} Each cut applied to the Metadetection catalog after the RS cuts. Note that the number of rows removed is for each individual cut, so cuts may overlap with other cuts.
:name: selection_cuts
:widths: auto

| Selection Cut                            | Rows Removed | Fraction Removed |
| :--------------------------------------- | -----------: | ---------------: |
| `gauss_T_ratio` > 1.05                   | 76863        | 37.5%            |
| `gauss_s2n` > 10                         | 30335        | 14.8%            |
| `mfrac` < 0.1                            | 0            | 0.0%             |
| `gauss_band_mag_i` < 23.5                | 74987        | 36.6%            |
| `gauss_band_mag_i` > 20 (brightness cut) | 11161        | 5.5%             |
| `gauss_color_mag_g-r` (abs. value) < 5   | 355          | 0.2%             |
| `gauss_color_mag_r-i` (abs. value) < 5   | 49           | 0.02%            |
| `gauss_color_mag_g-i` (abs. value) < 5   | 574          | 0.3%             |
:::

As a visual summary of the cuts done to produce the sample, it's helpful to plot the distribution of magnitudes, in a similar vein as Figure 2 in {cite:p}`mag_cut`. While one goal of the magnitude cut is to reduce the impact of the SNR cut (discussed more in the photo-z section), which would align closer to a cut closer to `i_mag` < 24.1, a cut of 24.0 is chosen to be consistent with {cite:p}`SITCOMTN-161`. A description of each label can be found in {numref}`mag_dist_nums`, with the numbers in the non-sheared catalog after each cut is applied. The cuts listed for {numref}`mag-cut-hist2` and {numref}`mag_dist_nums` are not in the same order that the cuts are applied within the shear profile analysis, but are rather used to understand the significance of the SNR cut.

```{figure} _static/mag-cut-hist2.png
:name: mag-cut-hist2

Magnitude distributions of the *i*-band after various cuts done to produce the weak lensing sample. Left: All cuts are applied in the order described above. Right: The magnitude cut is not applied to show where the SNR cut becomes significant at fainter magnitudes and begins to remove objects. The magnitude cut is instead represented by the vertical line.
```

:::{table} The number of objects in each of the magnitude histograms in {cite:p}`mag_cut` and the descriptions of the cuts used. While the table {numref}`selection_cuts` shows the total number of rows removed, this table is meant to show the additive effect as each cut is applied. All objects are within 0.5 degrees of the BCG.
:name: mag_dist_nums
:widths: auto

| Cut Name             | Description                                         | Rows in non-sheared catalog |
| :------------------- | :-------------------------------------------------- | --------------------------: |
| **All detections**   | Detections that have a finite *i*-band magnitude    | 178698                      |
| **Good detections**  | Flagged and duplicate objects removed               | 54228                       |
| **Masks**            | Masks are applied                                   | 42806                       |
| **WL Cuts**          | Selection cuts                                      | 39735                       |
| **Star-galaxy**      | Size ratio cut to remove stars from the sample      | 25001                       |
| **Magnitude Cut**    | Magnitude where SNR begins to remove objects        | 18645                       |
| **SNR Cut**          | Objects with SNR < 10 are removed                   | 18566                       |
| **RS Cuts**          | Red sequence galaxies are identified and removed    | 17651                       |
:::

For reference against another catalog, it's useful to look at the number of objects found in the HSM catalog ({cite:p}`HSM1`, {cite:p}`HSM2`) used in {cite:p}`SITCOMTN-161` within the same 0.5 degree radius as used in this technote. The HSM catalog first reads in 175383 objects prior to any cuts, and then 12852 objects after all cuts are applied. This is a similar ballpark as the final Metadetect catalog size of 17651, especially considering that the selection and quality cuts of two catalogs differ due to the nature of separate catalogs.

The global source density for Metadetection is 8.14 $\pm$ 0.1 $\text{arcmin}^{-2}$, comparable to the 7-8 $\text{arcmin}^{-2}$ range found using the HSM catalog in {cite:p}`SITCOMTN-161`. The density for the Metadetection catalog does trend a bit higher, towards the range of 8-9 $\text{arcmin}^{-2}$, though this is likely due to the fact that this HSM estimate does not yet account for the area affected by masks.

```{figure} _static/gal_den_profile.png
:name: galaxy-den-profile

The galaxy density within each radial bin used for the shear profile. Error bars are one standard deviation Poisson shot noise. Area that has been masked is removed from the calculation.
```

## Photo-Z Data

In order to model a shear profile to compare to the observed data, an $N(z)$ estimate will be needed. With the utilities available in the Cluster Lensing Mass Modeling (CLMM) code {cite:p}`clmm`, there is a default source redshift distribution based off of the DESC Science Requirements Document (SRD, {cite:p}`desc-srd`) Y10 N(z). However, this is not very representative of the data that's being used here, which is comparatively much shallower. Instead, the initial photo-z catalog used here is created with DNF ({cite:p}`DNF_pz`) on DP1 data, produced in the same manner as described in {cite:p}`SITCOMTN-163`.

The objects in the Metadetection catalogs are then matched to the objects in the DP1 catalogs by nearest neighbor, based on RA/DEC coordinates and limited to matches within 1 arcsecond. The matching is only needed for the non-sheared catalog, which is the catalog used for the $N(z)$ estimate. Objects are then additionally cut from the matched catalogs if they fall below a redshift estimate of 0.37, based on the cut used in {cite:p}`SITCOMTN-163`. The $N(z)$ generated by the method above is ***not*** used for the shear profile generated by the data, only for the test model used for reference. Future works with Metadetection data should run photo-z algorithms on each of the shear profile catalogs individually to fully capture the additional selection effects within the response.

Once the matched catalogs are created, and using a standard cosmology ($\Omega_m=0.3$, $h=0.7$), the statistics of the lensing efficiency can be calculated. The CLMM package has tools for calculating these using the photo-z point estimate of each galaxy, with the theory based on {cite:p}`beta_theory`, which will be summarized below.

First off is the geometric lensing efficiency $\beta$, which is the ratio between $D_{ls}$ and $D_s$, the angular diameter distances between the lens and and the source, and the observer to the source, respectively. $\beta$ is not allowed to go below zero, as sources in front cluster will have no contribution to the lensing signal. The $\beta_s$ term then is the ratio of $\beta$'s of a source at a specific redshift, and a test source at infinite redshift (set to $z=1000$ by default).

$$
\begin{align}
\beta &= \text{max}\left( 0, \frac{D_{ls}}{D_{s}} \right) \\
\beta_s(z_i) &= \frac{\beta(z_i)}{\beta_{\infty}}
\end{align}
$$

From the matched source galaxy sample, the statistics of these $\beta$ parameters can be calculated. Weights can be implemented, though this analysis simply sets the weights to 1 to produce a simple mean.

$$
\begin{align}
\left<\beta_s\right> &= \frac{\Sigma\beta_s(z_i)w_i}{\Sigma w_i} \\
\left<\beta_s^2\right> &= \frac{\Sigma\beta_s^2(z_i)w_i}{\Sigma w_i} \\
\end{align}
$$

With the mean $\beta_s$ statistics a curve predicting the shape of the reduced shear as a function of $R$, the projected radial distance from the BCG in Mpc, is approximately given by

$$
\begin{align}
g(R) &\approx \left(\frac{\left<\beta_s\right>\gamma_{\infty}(R)}{1-\left<\beta_s\right>\kappa_{\infty}(R)}\right)
\left[
1+\left(\frac{\left<\beta_s^2\right>}{\left<\beta_s^2\right>}-1\right)\left<\beta_s\right>\kappa_{\infty}(R)
\right].
\end{align}
$$
The terms $\gamma_{\infty}$ and $\kappa_{\infty}$ represent the shear and convergence of a test source at infinite redshift, with the $\left<\beta_s\right>$ and $\left<\beta_s^2\right>$ terms modulating the theoretical reduced shear based on the ensemble redshift information from the matched source catalog.


## Shear Calibration

Understanding the relationship between the shear applied to an object and the effect of that shear on measuring the object’s shape is a critical step to calibrating the catalog’s shear measurements. The main purpose of Metadetection’s 5 sub-catalogs is to calculate the linear response matrix, R.

The main assumption is that the weak lensing shear signal is small enough that we can Taylor expand the measured galaxy ellipticity about a zero shear signal. The first term goes to 0 in the limit of a large enough sample of galaxies where the shape noise averages out. The relation between the measured galaxy ellipticity and the resulting shear signal is controlled by R, the linear response of the ellipticity to an applied shear. Within the Metadetection framework, the components of R can be calculated by taking the mean measured galaxy ellipticities of the 4 artificially shear object catalogs. The magnitude of the applied shear in each catalog is 0.01, to make $\Delta\gamma_j$ a total of 0.02 for each component of R. Once R is calculated, it’s applied to the mean non-sheared object ellipticities to produce the calibrated reduced shear.

$$
\begin{align}
\mathbf{e}&\approx\mathbf{e}|_{\gamma=0}+ \frac{\partial\mathbf{e}}{\partial\mathbf{\gamma}}\bigg\rvert_{\gamma=0} \\
\mathbf{e}&\approx\cancel{\langle\mathbf{e}\rangle|_{\gamma=0}}+\langle\mathbf{R}\mathbf{\gamma}\rangle \\
\langle\mathbf{\gamma}\rangle&\approx\langle\mathbf{R}^{-1}\rangle\langle\mathbf{e}\rangle \\
\langle R_{ij}\rangle&=\frac{\langle e_i^+\rangle-\langle e_i^-\rangle}{\Delta\gamma_j}
\end{align}
$$ (eqn1)

In the specific case of cluster lensing, we are more interested in the tangential and cross shears around the cluster, split into radial bins. The response matrix R is first calculated on all galaxy ellipticities measurements from the four sheared catalogs, after all applied cuts, but prior to binning in order to improve the uncertainty of R. The calibration is then applied to the galaxies in the non-sheared catalog, producing the reduced shear. Using the utilities in CLMM, the mean tangential and cross terms are calculated from the calibrated shears for each radial bin.

A note on coordinate systems: Both the Metadetection $g_1$ and $g_2$ shapes and the CLMM code used here utilize the "Euclidean" definition described in section 5.1 of {cite:p}`galsim`.

### Shear Results

The resulting shear profile is shown below in {numref}`shear-final`. Each bin is calculated from the calibrated shapes of galaxies in five bins that range from 0.71 Mpc to 3.75 Mpc, split evenly in $\log_{10}$ space. The range for the binning is based off of {cite:p}`binning`. The error bars for the bins are 1 standard error (standard deviation / $\sqrt(N)$). From smallest radial separation to largest, the number of galaxies in each bin are 245, 433, 974, 1810, and 3119 galaxies. Despite shear around clusters typically being fairly noisy, there is a visible upward trend in the tangential shear, with the cross shear typically hovering around 0.

The Mpc distances are assuming a cluster redshift of z=0.22 ({cite:p}`a360_z`).

```{figure} _static/shear-final.png
:name: shear-final

The reduced shear profile around A360 for both tangential and cross shear measurements, using the cuts described throughout the technote. The dotted green line represents a reference NFW model. Error bars are 1 standard error. Detection significance for the tangential and cross shears are 3.73 and 0.02 sigma, respectively.
```

The theoretical shear profile is produced using CLMM. This profile is purely for a rough reference, and is not fit to the calibrated shear data. The profile is using an NFW halo with an estimated cluster mass of 6e14 solar masses ({cite:p}`a360_mass`) and a concentration of 3.5. The source redshift distribution is based off of the mean $\beta_s$ statistics described in the photo-z section above.

:::{table} Values of the R components used to calibrate the shape measurements. The values are taken after all cuts are applied to the source galaxy sample.
:widths: auto

|                   | Galaxies < 0.5 degrees |
| :---------------- | ---------------------: |
| R_11              | 0.6393                 |
| R_22              | 0.5841                 |
| R_11_err          | 0.00253                |
| R_22_err          | 0.00256                |
| \| R_11 - R_22 \| | 0.0552                 |
:::

## Validation & Testing

This section contains a non-exhaustive list of notes and figures on characterizing the Metadetection output catalog and the effects from various cuts.

### Object Size and S/N Cuts

The measured object size compared to the signal-to-noise ratio is a simple cut to differentiate stars and galaxies.

```{figure} _static/obj_T_vs_s2n.png
:name: obj_T_vs_s2n

The relationship between the object size ratio and the S/N of each object before and after selection cuts (both with red sequence galaxies removed). The red line is a visual reference to see what objects are removed by the 1.1 object size ratio cut, though other objects may be removed due to additional cuts. Stars are expected to fall near an object ratio of 1, which is seen clearly for high S/N objects. The remaining objects have the line of stars removed, though some low S/N stars may survive the cut, as those tend to have higher size uncertainties as seen in {cite:p}`yamamoto`.
```

### Object Distributions

Plotting the distribution of objects on the sky is a simple but effective way to discover inconsistencies within the data. For example, the leftmost plot in {numref}`object-distribution` shows an overdensity of objects detected that aligns with where patches overlap, unexpected after removing exact duplicates based on RA and DEC coordinates. This duplication is addressed, as seen in the middle and right-most plots in {numref}`object-distribution`.

```{figure} _static/object-distribution-before-after.png
:name: object-distribution

Object distributions of the non-sheared catalog at various points during cuts. Left: Galaxy distributions prior to any cuts. There is a clear overdensity that overlaps between patches. Middle: the distribution after the duplicates and red sequence galaxies have been removed. Right: the object distribution after all cuts have been applied.
```

## Final Remarks

From what was able to be achieved within this technote, the combination of cell-based coadds and Metadetection have the necessary infrastructure needed to successfully run end-to end within the LSST Science Pipelines infrastructure and produce a shape catalog. With this shape catalog, a tangential shear profile of A360 was created. Radial bins beyond ~2 Mpc tend to perform the best, while the inner bins struggle to find a clear signal; this is not surprising within a cluster environment.

As for technical details, it should be noted that many configuration settings are not yet available to the pipeline infrastructure through pipeline `.yaml` files. The main motivation for custom branches was to enable changes (e.g. changing `wmom` to `gauss`) for testing purposes.

There are still plenty of tasks that can done next. It might be beneficial to incorporate alternative deblending algorithms to potentially improve the performance of the shear profile within the inner radial bins near the cluster center. Additionally, running detection tasks on the cell-based coadds would be helpful for comparing objects detected in the Metadetection catalog, as well as obtain PSF measurements for reserved stars in order to do a more thorough $\rho$-statistics analysis. More technical investigations should be done to understand why object coordinates differ between patches with the same WCS information, as well as investigate the source of extremely large S/N values and lack of objects with a S/N below 10.

## References

```{bibliography}
```