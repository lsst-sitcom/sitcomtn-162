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

These three figures show the input image distribution for the patches, each composed of 484 cells, around A360 in the g, r, and i-bands. The red squares outline the inner patch boundaries, where the 2 cell overlap is visible. The three missing patches are due to processing errors when running Metadetection, though they do not significantly overlap with the 0.5 degree radius around the BCG (cyan circle). The colorbar is scaled to the minimum and maximum number of inputs images across the three bands (1 and 16, respectively).
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

The default setting for measuring object shapes is `wmom` (weighted moments), used throughout this technote. Each shape measurement is a weighted average of the second moments using the three bands, g, r, and i. The weights for averaging across bands come from the inverse variance of the image. In a similar vein, the flux measurements are the zero moment of each object. Both the zero and second moments are weighted by a Gaussian with a FWHM of 1.2 arcseconds.

The Metadetection shear catalog for this technote was run on the `w_2025_34` weekly stack version of the Pipeline. As for packages, [`metadetect`](https://github.com/lsst/metadetect/tree/lsst-dev) used the `lsst-dev` branch, the ['cell_coadds'](https://github.com/lsst/cell_coadds/tree/tickets/DM-43585) package again used the `tickets/DM-43585` branch, and the [`drp_tasks`](https://github.com/lsst/drp_tasks/tree/u/mirarenee/meta_test/python/lsst/drp/tasks) package used a custom branch `u/mirarenee/meta_test`. The `drp_tasks` user branch was needed to add a few minor changes in order to run Metadetection on more recent pipeline stacks.

The pipetask command used to generate the Metadetection catalog is found below:

```
pipetask run -j 4 --register-dataset-types \
-b /repo/main \
-i refcats,u/mgorsuch/ComCam_Cells/a360/corr_noise_cells \
-o u/$USER/metadetect/a360_3_band/noise \
-p /sdf/group/rubin/user/mgorsuch/notebooks/metadetect/comcam_pipeline.yaml \
-c metadetectionShear:shape_fitter='wmom' \
-d "skymap='lsst_cells_v1'"
```

The associated `comcam_pipeline.yaml` file used for defining the tasks is outline below:

```yaml
description: Pipeline for running metadetection on DC2
instrument: lsst.obs.lsst.LsstComCam

tasks:
    metadetectionShear:
        class: lsst.drp.tasks.metadetection_shear.MetadetectionShearTask
        config:
            required_bands : ["g", "r", "i"]
            connections.ref_cat: the_monster_20250219
            shape_fitter: "wmom"
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

A series of color-magnitude plots with progressive cuts is used to identify the red sequence (RS) galaxies. Each cut is applied to the entire catalog, though only the non-sheared catalog is shown in the color-magnitude plots. Previous visual inspection showed that while there is some variation in  between shear type catalogs, the variation is minimal and random enough that applying the same cuts should be sufficient for this analysis.

The catalog is first cut to galaxies less than 0.1 degree away from the BCG to focus on galaxies that are more likely to be cluster members. The red sequence cluster members are identified in a line of objects with relatively consistent color across a range of magnitudes, with the line being more apparent in the smaller sample of galaxies. This line of galaxies is highlighted with orange points, with the upper and lower limits shown in red. The same visual inspection is done again for the larger sample of galaxies, those within 0.5 degrees of the BCG. Within the larger sample, the objects identified within the limits of either the g-r or r-i RS galaxy limits are cut, leaving a sample of galaxies for measuring weak lensing.

The RS identification is limited to galaxies with `wmom_band_mag_g` < 23 for the *g*-*r* colors, and `wmom_band_mag_r` < 23 for the *r*-*i* colors. While RS galaxies may be fainter than this, it becomes more difficult to distinguish between RS and background galaxies. Since unsheared foreground galaxies will dilute the signal, but not bias it, a small contamination is allowed to keep the many source galaxies that may otherwise be cut. Any remaining bright galaxies (those with `wmom_band_mag_i` < 20) are removed below as described in the selection cut section.

```{figure} _static/color-magnitude-0-1-orange.png
:name: color-magnitude-0-1-orange

Color-magnitude diagram cut to 0.1 degrees within the BCG. Objects that are included within the cut are highlighted in orange.
```

```{figure} _static/color-magnitude-0-5-orange.png
:name: color-magnitude-0-5-orange

Color-magnitude diagram cut to 0.5 degrees within the BCG. Objects that are included within the cut are highlighted in orange.
```

```{figure} _static/color-magnitude-0-5-removed.png
:name: color-magnitude-0-5-removed

Color-magnitude diagram cut to 0.5 degrees within the BCG. Objects that are included within the cut are removed.
```

After applying the 0.5 degree cut, but prior to removing RS objects, there are 595377 object rows. After applying the RS cuts, there are 258449, with 51633 (~20%) of which are the non-sheared catalog.

## Selection Cuts

After Metadetection flags are applied and red sequence galaxies are identified and removed, additional selection cuts are applied. These cuts are primarily based on {cite:p}`yamamoto`, though are customized in several cases to better fit the data from LSSTComCam. A detailed outline of the cuts used and object removed can be seen in {numref}`selection_cuts`. In numerical terms, the number of objects remaining after selection cuts is 78500, with 15819 (20%) in the non-sheared catalog.

The Yamamoto cuts describe a size ratio cut, defined as the size of the object squared divided by size of the PSF squared, or $T^{gauss}/T^{gauss}_{PSF}$. This is used as a star-galaxy cut. For the Yamamoto measurements, these sizes are measured for the pre-PSF objects, so stars will hover around 0 for this ratio. Meanwhile, the weighted moments measurements with Metadetection for T and T_PSF are both measured after the reconvolution step, and will be slightly larger; stars will hover closer to 1. The Metadetection paper {cite:p}`Sheldon_2023` uses a cut of 1.2 for this size ratio, and indicates that while the inclusion of stars might introduce a bias, it’s quite small.

Based on {numref}`obj_T_vs_s2n`, the stars identified around `wmom_T_ratio` = 1 appear to fall consistently below `wmom_T_ratio` = 1.1. This value is chosen instead for the `wmom_T_ratio` cut, since the inclusion of extra galaxies in the weak lensing sample outweighs the potential inclusion of some low S/N stars.

The magnitude cuts also deviate slightly from {cite:p}`yamamoto`. These cuts are based on the estimated limiting magnitude of each band, as seen from {numref}`obj-mags`.

:::{table} Each cut applied to the Metadetection catalog after the RS cuts. Note that the number of rows removed is for each individual cut, so cuts may overlap with other cuts.
:name: selection_cuts
:widths: auto

| Selection Cut                         | Rows Removed | Fraction Removed |
| :------------------------------------ | -----------: | ---------------: |
| `wmom_T_ratio` > 1.1                    | 111628        | 43.2%            |
| `wmom_s2n` > 10                         | 68771         | 26.6%            |
| `wmom_T` < 20                           | 0            | 0.0%             |
| `mfrac` < 0.1                           | 0            | 0.0%             |
| `wmom_band_mag_g` < 25.7                | 93957        | 36.3%            |
| `wmom_band_mag_r` < 25.4                | 52843        | 20.4%            |
| `wmom_band_mag_i` < 25.1                | 57102        | 22.1%            |
| `wmom_band_mag_i` > 20 (brightness cut) | 5240         | 2.0%             |
| `wmom_color_mag_g-r` (abs. value) < 5   | 1007         | 0.4%             |
| `wmom_color_mag_r-i` (abs. value) < 5   | 134          | 0.1%             |
| `wmom_T` < 0.425 - 2.0*`wmom_T_err`       | 21058         | 8.1%             |
| `wmom_T` * `wmom_T_err` < 0.006           | 2243          | 0.9%             |
:::

For reference against another catalog, it's useful to look at the number of objects found in the HSM catalog ({cite:p}`HSM1`, {cite:p}`HSM2`) used in {cite:p}`SITCOMTN-161` within the same 0.5 degree radius as used in this technote. The HSM catalog first reads in 183791 objects prior to any cuts. After RS cuts, there are 104257 objects. Finally, after selection cuts, the final HSM source galaxy sample is 23531 objects. With the final `Metedetection` source galaxy sample catalog at 15819 for non-sheared objects, HSM is producing about 1.5x as many objects for the final shear catalog. However, it should also be noted that prior to RS cuts, Metadetection produces a comparable number of objects as HSM (193522 vs. 183791). This final difference is less surprising, then, as the selection cuts are between technotes differ due to the nature of separate catalogs. Still, the discrepancy is high enough that further cross-checks between the two catalogs would be beneficial.

The final source densities for Metadetection and HSM catalogs from these values, using the same 0.5 degree radius, are ~5.6 $\text{arcmin}^{-2}$ and ~8.3 $\text{arcmin}^{-2}$, respectively.

## Shear Calibration

Understanding the relationship between the shear applied to an object and the effect of that shear on measuring the object’s shape is a critical step to calibrating the catalog’s shear measurements. The main purpose of Metadetection’s 5 sub-catalogs is to calculate the linear response matrix, R.

The main assumption is that the weak lensing shear signal is small enough that we can Taylor expand the measured galaxy ellipticity about a zero shear signal. The first term goes to 0 in the limit of a large enough sample of galaxies where the shape noise averages out. The relation between the measured galaxy ellipticity and the resulting shear signal is controlled by R, the linear response of the ellipticity to an applied shear. Within the Metadetection framework, the components of R can be calculated by taking the mean measured galaxy ellipticities of the 4 artificially shear object catalogs. The magnitude of the applied shear in each catalog is 0.01, to make $\Delta\gamma_j$ a total of 0.02 for each component of R. Once R is calculated, it’s applied to the mean non-sheared object ellipticities to produce the calibrated shear.

$$
\begin{align}
\mathbf{e}&\approx\mathbf{e}|_{\gamma=0}+ \frac{\partial\mathbf{e}}{\partial\mathbf{\gamma}}\bigg\rvert_{\gamma=0} \\
\mathbf{e}&\approx\cancel{\langle\mathbf{e}\rangle|_{\gamma=0}}+\langle\mathbf{R}\mathbf{\gamma}\rangle \\
\langle\mathbf{\gamma}\rangle&\approx\langle\mathbf{R}^{-1}\rangle\langle\mathbf{e}\rangle \\
\langle R_{ij}\rangle&=\frac{\langle e_i^+\rangle-\langle e_i^-\rangle}{\Delta\gamma_j}
\end{align}
$$ (eqn1)

In the specific case of cluster lensing, we are more interested in the tangential and cross shears around the cluster, split into radial bins. The response matrix R is first calculated on all galaxy ellipticities measurements from the four sheared catalogs, after all applied cuts, but prior to binning in order to improve the uncertainty of R. Then for each bin, the mean tangential and cross ellipticities are calculated from the uncalibrated galaxy ellipticity measurements in the non-sheared catalog. The response matrix R is then applied to the mean tangential and cross ellipticities to produce the calibrated tangential and cross shears for the bin.

### Shear Results

The resulting shear profile is shown below in {numref}`shear-final`. Each bin is calculated from the calibrated shapes of galaxies in six bins that range from 0.32 Mpc to 6.39 Mpc, split evenly in $\log_{10}$ space. The x-axis shear profile points are calculated from the mean Mpc distance of each object for each bin. The error bars for all shear profiles are bootstrapped samples of each radial bin with 95% confidence levels, which is then calibrated with R. From smallest radial separation to largest, the number of galaxies in each bin are 68, 222, 511, 1388, 3401, and 10184 galaxies. Despite shear around clusters typically being fairly noisy, there is a visible upward trend in the tangential shear, with the cross shear typically hovering around 0. The exception is the bin closest to the BCG, which may be due to high blending. This region is also particularly sensitive to the binning process with the low galaxy count.

The Mpc distances are assuming a cluster redshift of z=0.22 {cite:p}`a360_z`.

```{figure} _static/shear-final.png
:name: shear-final

The reduced shear profile around A360 for both tangential and cross shear measurements, using the cuts described throughout the technote. Both measured profiles have 95% confidence intervals.
```

The theoretical shear profile is produced using Cluster Lensing Mass Modeling (CLMM) code {cite:p}`clmm`. This profile is purely for a rough reference, and is not fit to the calibrated shear data. The profile is using an NFW halo with an estimated cluster mass of 4e14 solar masses ({cite:p}`a360_mass`) and a concentration of 4. The source redshift distribution is based off of the DESC Science Requirements Document (SRD, {cite:p}`desc-srd`) Y10 N(z).

:::{table} Values of the R components used to calibrate the shape measurements. The values are taken after all cuts are applied to the source galaxy sample.
:widths: auto

|                   | Galaxies < 0.5 degrees |
| :---------------- | ---------------------: |
| R_11              | 0.2101                 |
| R_22              | 0.1932                 |
| R_11_err          | 0.000689               |
| R_22_err          | 0.000704               |
| \| R_11 - R_22 \| | 0.0169                 |
:::

## Validation & Testing

This section contains a non-exhaustive list of notes and figures on characterizing the Metadetection output catalog and the effects from various cuts.

### Object Size and S/N Cuts

The measured object size compared to the signal-to-noise ratio is a simple cut to differentiate stars and galaxies.

```{figure} _static/obj_T_vs_s2n.png
:name: obj_T_vs_s2n

The relationship between the object size ratio and the S/N of each object before and after selection cuts (both with red sequence galaxies removed). The red line is a visual reference to see what objects are removed by the 1.1 object size ratio cut, though other objects may be removed due to additional cuts. Stars are expected to fall near an object ratio of 1, which is seen clearly for high S/N objects. The remaining objects have the line of stars removed, though some low S/N stars may survive the cut, as those tend to have higher size uncertainties as seen in {cite:p}`yamamoto`.
```

### False Detections

{cite:p}`yamamoto` introduces a few cuts for spurious detections. The first is an upper limit of `wmom_T` as a function of `wmom_T_err`, while the second cut is the product of `wmom_T` x `wmom_T_err`. These focus on areas around bright stars and spurious detections within cluster fields, respectively. Plots in {numref}`junk1` and {numref}`junk2` were used to get an initial idea of where the data lie. Visual inspection of these objects showed that the majority are around bright stars, spurious detections, and small galaxies with close, undetected neighbors. Still, these cuts remove potentially viable galaxies from the sample, and don’t find every spurious detection. The values used in these cuts come from minimizing the number of spurious detections and maximizing the number of viable galaxies kept in the final catalog using visual inspection.

```{figure} _static/junk1.png
:name: junk1

Distribution of data points before and after cuts. The red lines indicate where the relevant junk cut removes data. The large cluster of points prior to applying all cuts in the left plot is mostly stars removed from the star-galaxy cut, and is not present in the plot on the right.
```

```{figure} _static/junk2.png
:name: junk2

The distributions of data points before and after cuts were used for the purpose of estimating an intial value for the `wmom_T` x `wmom_T_err` cut. It should be noted that while the majority of objects that are removed with this cut are spurious, all of these objects identified in this cut are already removed by the other cuts.
```

### Object Distributions

Plotting the distribution of objects on the sky is a simple but effective way to discover inconsistencies within the data. For example, the leftmost plot in {numref}`object-distribution` shows an overdensity of objects detected that aligns with where patches overlap, unexpected after removing exact duplicates based on RA and DEC coordinates. This duplication is addressed, as seen in the middle and right-most plots in {numref}`object-distribution`.

```{figure} _static/object-distribution-before-after.png
:name: object-distribution

Object distributions of the non-sheared catalog at various points during cuts. Left: Galaxy distributions prior to any cuts. There is a clear overdensity that overlaps between patches. Middle: the distribution after the duplicates and red sequence galaxies have been removed. Right: the object distribution after all cuts have been applied.
```

## Final Remarks

From what was able to be achieved within this technote, the combination of cell-based coadds and Metadetection have the necessary infrastructure needed to successfully run end-to end within the LSST Science Pipelines infrastructure and produce a shape catalog. With this shape catalog, a tangential shear profile of A360 was created. Radial bins beyond ~2 Mpc tend to perform the best, while the inner bins struggle to find a clear signal; this is not surprising within a cluster environment.

As for technical details, it should be noted that many configuration settings are not yet available to the pipeline infrastructure through pipeline `.yaml` files. The main motivation for custom branches was to enable changes (e.g. changing `wmom` to `pgauss`) for testing purposes.

There are still plenty of tasks that can done next. It might be beneficial to incorporate alternative deblending algorithms to potentially improve the performance of the shear profile within the inner radial bins near the cluster center. Additionally, running detection tasks on the cell-based coadds would be helpful for comparing objects detected in the Metadetection catalog, as well as obtain PSF measurements for reserved stars in order to do a more thorough $\rho$-statistics analysis. More technical investigations should be done to understand why object coordinates differ between patches with the same WCS information, as well as investigate the source of extremely large S/N values and lack of objects with a S/N below 10.

## References

```{bibliography}
```