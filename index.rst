############################
DRP Resource Usage Estimates
############################

.. abstract::

Using job profiling and dataset size information from intermittent DRP runs at USDF, task dependencies inferred from LSSTCam DRP pipeline definitions, and observing sequences from consdb or from a opsim simulation, the computing resource usage (disk space and CPU time) are estimated for DP2 and DR1 processings.


Methodology
===========
The "intermittent" DRP runs are performed at USDF approximately bi-weekly.  These runs are used to exercise new development in the Rubin science pipelines in order to uncover potential issues with running the pipelines at scale.  In addition to testing the algorithmic and middleware changes, these runs are also useful for tracking pipeline performance.  For the DRP resource usage estimates, we track the per-job memory usage, cpu run times, and wall times as extracted from the metadata that are generated for every instance of the various pipeline tasks.

For producing resource usage estimates for a potential DRP processing campaign, it is sufficient for most tasks to use typical values for the cpu time and memory requirements since they are not strongly dependent on their inputs.  However, the resource usage for a number of tasks does depend strongly on the input data.  In particular, several tasks that process coadd-related data products depend strongly on the number of visits that the coadd comprises.   Similarly, tasks that measure per-source properties on images will have resource needs that scale as the local object density on the sky.  For determining the cpu time and memory scaling with number of visits per coadd, we extract the expected numbers of input visits from the QuantumGraph.  The local stellar density is the most relevant quantity for source measurement scaling, however, at this time we don't account for that dependence.

The observation sequence for a potential DRP campaign can be obtained either from real observations by querying consdb or from simulated observing cadences contained in an opsim db file.  The key features of a pointing that determine the structure of the QG are the individual CCD overlaps with the skymap patches.  Thos overlaps determine the number of warps per patch and hence the visit depth for a given coadd.  In order to calculate those overlaps, for consdb observations, we use the per CCD ``s_region`` info, which describes CCD area projected on the sky, from consdb, while for opsim db observations, we use the approximate wcs obtained with the ``lsst.obs.base.createInitialSkyWcsFromBoresight`` function to obtain the equivalent CCD sky projection.  Since we're doing an approximate calculation anyway, we simply use the ``skymap.findTractPatchList`` function to obtain the CCD-patch overlaps.

We determine the pipeline tasks, their dependencies, and the dataset types from the pipeline yaml, e.g., `drp_pipe/LSSTCam/DRP.yaml` for the full pipeline.  We then infer the number of instances of each task and their associated dataIds from the task dimensions in the pipeline definition and the overlap information.

The overall CPU time resource estimates are the predicted CPU run times weighted by the expected number of cores required to do the processing.  This number is inferred from the predicted memory usage, assuming a fixed amount of memory per core.  For running at USDF, we nominally have 4 GB per core, so a job that requires 1 CPU minute and 5 GB of memory would have weighted CPU time of 2 CPU minutes.

For disk storage estimates, we sample the dataset sizes for a recent run to obtain an average dataset size.  To obtain the predicted storage for each dataset type, we multiply the number of corresponding task instances by those measured average dataset sizes.

Modeling CPU Times and Memory Usage
===================================
For tasks that do not have significant scaling dependence, we simply use the mean values for the per-job CPU times and memory usage.  Figure 1 below shows those distributions for the ``isr``, ``calibrateImage``, ``makeDirectWarp``, and ``makePsfMatchedWarp`` from the DM-52836/``w_2025_41`` DRP processing, with the mean values used in the CPU and memory calculations indicated.

**Figure 1**: CPU time and memory usage for ``isr``, ``calibrateImage``, ``makeDirectWarp``, and ``makePsfMatchedWarp`` tasks in DM-52836.  The red dashed lines indicate the mean values that are used in the projected estimates.

.. grid:: 1 2 2 2
   :gutter: 2

   .. grid-item::
      :columns: 6

      .. image:: /_static/DM-52836_isr_resource_usage.png
         :alt: fig_1a

   .. grid-item::
      :columns: 6

      .. image:: /_static/DM-52836_calibrateImage_resource_usage.png
         :alt: fig_1b

   .. grid-item::
      :columns: 6

      .. image:: /_static/DM-52836_makeDirectWarp_resource_usage.png
         :alt: fig_1c

   .. grid-item::
      :columns: 6

      .. image:: /_static/DM-52836_makePsfMatchedWarp_resource_usage.png
         :alt: fig_1d


As noted, the resource usage for some tasks in the coadd and variability-related processing stages have strong dependence on the number of visits.  In order to model that dependence, we partition the data in bins of numbers of visits, then fit a line to the median values in those bins.  Some of these tasks have strong dependence on numbers of visits for both CPU time and memory, and some just have a strong dependence on just one or the other.  In Figure 2, we show the fitted lines for ``assembleDeepCoadd``, where a # visit-dependence is modeled for both CPU time and memory usage, and for ``measureObjectForced`` where just the CPU time shows a strong dependence on the # visits, whereas for the memory we use a constant value, in this case, the mean value of 1.5 GB.

**Figure 2**: CPU time and memory usage for ``assembleDeepCoadd`` and ``measureObjectForced`` for DM-52836.

.. grid:: 1 2 2 2
   :gutter: 2

   .. grid-item::
      :columns: 6

      .. image:: /_static/DM-52836_assembleDeepCoadd_resource_usage.png
         :alt: fig_2a

   .. grid-item::
      :columns: 6

      .. image:: /_static/DM-52836_measureObjectForced_resource_usage.png
         :alt: fig_2b


Comparison of Projected and Actual Resource Usage
=================================================
Here we use a recent intermittent DRP to compare our predicted resource estimates with the actual values.  As inputs to our calculation, we use the metadata and mean storage sizes for the actual DRP run being considered, DM-52836.

In Table 1, we compare the predicted number of jobs from our overlaps-based calcuation with the numbers computed in the QuantumGraph generation step for a representative subset of tasks.  For tasks including and downstream of ``makeDirectWarp``, our overlaps-based calculation over-predicts the numbers of jobs at the ~24% level.

**Table 1**: Number of jobs for selected tasks for DM-52836

.. table:: Number of jobs per task for DM-52836
   :widths: grid

   +----------------------------+-----------+----------+
   | task                       | predicted | actual   |
   +============================+===========+==========+
   | isr                        | 868438    | 854501   |
   +----------------------------+-----------+----------+
   | consolidateVisitSummary    | 4798      | 4721     |
   +----------------------------+-----------+----------+
   | associateIsolatedStar      | 383       | 393      |
   +----------------------------+-----------+----------+
   | makeDirectWarp             | 2593360   | 2115198  |
   +----------------------------+-----------+----------+
   | assembleDeepCoadd          | 153261    | 123369   |
   +----------------------------+-----------+----------+
   | makeHealSparsePropertyMaps | 2019      | 1634     |
   +----------------------------+-----------+----------+
   | mergeObjectDetection       | 29184     | 26788    |
   +----------------------------+-----------+----------+
   | forcedPhotObjectDetector   | 1477196   | 1185206  |
   +----------------------------+-----------+----------+


In Table 2, we compare the predicted CPU time estimates per task with the actual CPU times as found from the task metadata files.

**Table 2**: Predicted and actual CPU times for top 95% of tasks in DM-52836

.. table:: Predicted and actual CPU times in DM-52836
   :widths: grid

   +--------------------------------+------------+------------+------------+------------+
   | task                           | predicted  | actual     | ratio      | cumulative |
   |                                |            |            |            | fraction   |
   +================================+============+============+============+============+
   | measureObjectForced            |    2.9e+08 |    3.3e+08 |       1.12 |       0.21 |
   +--------------------------------+------------+------------+------------+------------+
   | measureObjectUnforced          |    1.2e+08 |    2.0e+08 |       1.62 |       0.34 |
   +--------------------------------+------------+------------+------------+------------+
   | calibrateImage                 |    1.4e+08 |    1.4e+08 |       0.98 |       0.43 |
   +--------------------------------+------------+------------+------------+------------+
   | forcedPhotObjectDetector       |    1.2e+08 |    9.8e+07 |       0.80 |       0.49 |
   +--------------------------------+------------+------------+------------+------------+
   | isr                            |    8.0e+07 |    7.9e+07 |       0.98 |       0.54 |
   +--------------------------------+------------+------------+------------+------------+
   | makePsfMatchedWarp             |    9.6e+07 |    7.8e+07 |       0.82 |       0.59 |
   +--------------------------------+------------+------------+------------+------------+
   | makeDirectWarp                 |    9.4e+07 |    7.6e+07 |       0.82 |       0.64 |
   +--------------------------------+------------+------------+------------+------------+
   | reprocessVisitImage            |    7.3e+07 |    6.3e+07 |       0.87 |       0.68 |
   +--------------------------------+------------+------------+------------+------------+
   | fitDeepCoaddPsfGaussians       |    4.8e+07 |    6.1e+07 |       1.26 |       0.72 |
   +--------------------------------+------------+------------+------------+------------+
   | assembleDeepCoadd              |    6.8e+07 |    5.3e+07 |       0.77 |       0.75 |
   +--------------------------------+------------+------------+------------+------------+
   | refitPsfModelDetector          |    3.2e+07 |    4.4e+07 |       1.36 |       0.78 |
   +--------------------------------+------------+------------+------------+------------+
   | subtractImages                 |    5.5e+07 |    4.3e+07 |       0.78 |       0.81 |
   +--------------------------------+------------+------------+------------+------------+
   | assembleCellCoadd              |    4.0e+07 |    3.6e+07 |       0.90 |       0.83 |
   +--------------------------------+------------+------------+------------+------------+
   | detectAndMeasureDiaSource      |    4.3e+07 |    3.4e+07 |       0.78 |       0.85 |
   +--------------------------------+------------+------------+------------+------------+
   | rewarpTemplate                 |    3.8e+07 |    3.0e+07 |       0.79 |       0.87 |
   +--------------------------------+------------+------------+------------+------------+
   | deblendCoaddFootprints         |    3.1e+07 |    2.6e+07 |       0.85 |       0.89 |
   +--------------------------------+------------+------------+------------+------------+
   | deconvolve                     |    3.4e+07 |    2.6e+07 |       0.77 |       0.91 |
   +--------------------------------+------------+------------+------------+------------+
   | fitDeblendedObjectsExp         |    2.2e+07 |    2.4e+07 |       1.10 |       0.92 |
   +--------------------------------+------------+------------+------------+------------+
   | forcedPhotDiaObjectDetector    |    2.5e+07 |    2.0e+07 |       0.80 |       0.94 |
   +--------------------------------+------------+------------+------------+------------+
   | fitDeblendedObjectsSersic      |    1.7e+07 |    1.8e+07 |       1.11 |       0.95 |
   +--------------------------------+------------+------------+------------+------------+



Wall Time Considerations
========================
The measured wall times for some jobs can be substantially larger than the CPU times, e.g., for jobs that are affected by slow I/O.  Since we have used job CPU times instead of wall times, it's useful to have comparisons of the typical wall and cpu times for the each task.  In Figure 3, we show box plots of the distributions of wall-to-cpu time ratios for each task, sorted by integrated wall time for DM-52836.


Predictions for DP2
===================
These estimates use the DRP.yaml pipeline definition in ``w_2025_41``, and are based on resource usage data from DM-52836.   The input observations are a selection of science observations using LSSTCam, extracted from the consolidated database (consdb).  The query used for the selection is shown below.  For computing the number of datasets per dataset type and information such as the number of visits per patch per band, the ``lsst_cells_v1`` skymap was used.

Storage by Dataset Type
^^^^^^^^^^^^^^^^^^^^^^^

CPU Time by Task
^^^^^^^^^^^^^^^^

Data Selection
^^^^^^^^^^^^^^


Predictions for DR1
===================
The inputs used for these calculations are the Y1 observations from the baseline v5.0 simulations, specifically, from the ``baseline_v5.0.0_10yrs.db`` opsim db file.  As with the DP2 estimates, the DRP.yaml pipeline definition in ``w_2025_43``, the resource usage data from DM-52836 and the ``lsst_cells_v1`` skymap were used.


Storage by Dataset Type
^^^^^^^^^^^^^^^^^^^^^^^

CPU Time by Task
^^^^^^^^^^^^^^^^

Data Selection
^^^^^^^^^^^^^^


See the `Documenteer documentation <https://documenteer.lsst.io/technotes/index.html>`_ for tips on how to write and configure your new technote.
