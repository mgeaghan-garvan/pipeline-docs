# Running GATK-SV

This is documentation for running the GATK-SV pipeline on Terra.

GATK-SV runs in one of two main modes: batch and single-sample. Batch mode is intended for cohorts larger than 100 samples, and won't work for cohorts less than 10 samples. For small cohorts, single-sample mode should be used on each sample individually. Single-sample mode requires a reference panel, which will ideally be created using a cohort as similar as possible to the target sample. A standard reference panel is hosted by the Broad Institute in their public Google Cloud resource bucket based on samples from the 1000 Genomes project and can be used if no other relevant reference panel is available.

The full GATK-SV documentation can be found in the [gatk-sv github repository](https://github.com/broadinstitute/gatk-sv), and further documentation for specifically running the pipeline on Terra can be found in the `resources` sub-directory of this repository ([batch mode](resources/terra/cohort/cohort_mode_workspace_dashboard.md), [single-sample mode](resources/terra/single/single_sample_workspace_dashboard.md)).

## Terra workspace setup

Follow the instructions in the [Terra docs](../terra/README.md) to create a Terra workspace for running GATK-SV.

### Building default inputs

The GATK-SV pipeline is large and complex, and as such the Broad Institute have provided a script to generate default inputs to the workflow. This script requires the Python package `jinja2` to run. To run:
- Go to the directory inputs/values and copy the file `google_cloud.json` and give it a unique name, e.g. `google_cloud.my.json`. This file needs to contain the Google Cloud and/or Terra project details for running the workflow. For examples:
    ``` json
    {
        "google_project_id": "some-google-cloud-project",
        "terra_billing_project_id": "some-terra-billing-project"
    }
    ```
- In the repository's root directory, run:
    ``` bash
    ./scripts/input/build_default_inputs.sh -d . -c google_cloud.my  # replace `google_cloud.my` with whatever prefix was given to the copy of the Google Cloud JSON file
    ```
- This will generate several example input files for both the batch and single-sample modes. The inputs (input and output JSON configuration files, as well as the default workspace definition TSV file and sample table TSV file) can be used to run the workflow both on Terra and on a custom Cromwell server. These inputs are based on the 1000 Genomes samples. For the Terra inputs, they can mostly be used as-is due to Terra's separation of workflow from sample information; for the inputs intended for use on a custom Cromwell server, these inputs will need to be adapted for the cohort of interest.
  - Example files generated by these scripts are provided under the `resources` sub-directory in this repository.
  - The cohort inputs are available under `inputs/build/ref_panel_1kg`
    - Terra inputs are in the sub-directory `terra`, while inputs for a custom Cromwell server are under `test`
      - Terra input JSON files are within `terra/workflow_configurations`, and the sub-directory `terra/workflow_configurations/output_configurations` contains a few customised output JSON files.
      - The samples.tsv file specifies all the sample IDs, the locations of their BAM and gVCF files, and whether to allow the workflow to access Google Storage files that are specified as "requester pays".
      - The sample_set_membership.tsv file specifies sets of samples that are to be processed all in a single job simultaneously.
    - The Terra workspace.tsv file will need to be changed for one field: `cohort_ped_file` needs to point to a PED file specific to the cohort of interest. The PED format is described [here](https://gatk.broadinstitute.org/hc/en-us/articles/360035531972-PED-Pedigree-format).
      - **IMPORTANT:** When constructing the PED file, ensure that the entries in the second column (individual ID) exactly match the entries in the sample_id column in the Terra data tables (samples.tsv and sample_set_membership.tsv).
  - The single-sample inputs are available under `inputs/build/NA12878`
    - Again, Terra inputs are in the sub-directory `terra`, while inputs for a custom Cromwell server are under `test`
    - Similar to the cohort mode, the samples.tsv file specifies all the sample IDs to be individually processed, the locations of their BAM and gVCF files, and whether to allow the workflow to access Google Storage files that are specified as "requester pays".
    - The Terra workspace.tsv file will need to be changed for single-sample mode as well, namely in specifying both the VCF file for the reference panel (`ref_panel_vcf`) and the PED file for the reference panel (`ref_panel_ped_file`).

### Adding the GATK-SV workflow to the Terra workspace

All of the GATK-SV workflows are currently hosted through Dockstore and originate from Shyam's github account. The branch is terra_test_2 which is a fork of the Broad master branch at commit 7505396 (https://github.com/broadinstitute/gatk-sv/commit/7505396171365b84e26962241be43828b67d88b8) with a dockstore configuration file added. These workflows can be added to the Terra workspace by following the steps in the [Terra docs](../terra/README.md).

The single-sample pipeline consists of a [single workflow](https://dockstore.org/my-workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineSingleSample.wdl).

It is possible to run the batch pipeline with a single workflow ([GATKSVPipelineBatch](https://dockstore.org/my-workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch)) that runs all the necessary sub-modules, but it isn't recommend for Terra, and the inputs for this workflow aren't produced by the build scripts. In part, this is due to the large number of parameters required for the entire workflow, which Terra doesn't handle very well. It can also be easier to diagnose and manage failures of runs when the workflow is run as separate sub-modules. Additionally, there are quality control steps (stages 02 and 07b) where manual inspection of results and selection of an outlier cutoff parameter is recommended for further processing. As such, for discussion of working in Terra, this documentaiton will only cover running the individual sub-modules.

There are currently 12 main modules to the pipeline, which are listed here in the order to be run:
- [01_GatherSampleEvidence](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_01_GatherSampleEvidence:terra_test_2)
  - By default this does not use the MELT software due to licencing restrictions. We have our own MELT docker (v2.2.2) [here](australia-southeast1-docker.pkg.dev/pb-dev-312200/gatk-sv/melt:v2.2.2). This can be supplied to the `GatherSampleEvidence:melt_docker` parameter to enable MELT. **However**, we have run into errors running the MELT-enabled workflow, which have been solved by disabling the fast algorithm in the Picard tools `CollectWgsMetrics` sub-task. This option isn't exposed in the `terra_test_2` branch, so another branch was created for this purpose: [terra_test_2_melt](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_01_GatherSampleEvidence:terra_test_2)
- [02_EvidenceQC](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_02_EvidenceQC:terra_test_2)
- [03_TrainGCNV](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_03_TrainGCNV:terra_test_2)
- [04_GatherBatchEvidence](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_04_GatherBatchEvidence:terra_test_2)
- [05_ClusterBatch](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_05_ClusterBatch:terra_test_2)
- [06_GenerateBatchMetrics](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_06_GenerateBatchMetrics:terra_test_2)
- 07_FilterBatch. This consists of three sub-modules. A single-workflow version exists, and is hosted [here](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_07_FilterBatch:terra_test_2), but it is recommended to run the sub-modules individually:
  - [07a_FilterBatchSites](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_07a_FilterBatchSites:terra_test_2)
  - [07b_PlotSVCountsPerSample](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_07b_PlotSVCountsPerSample:terra_test_2)
  - [07c_FilterBatchSamples](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_07c_FilterBatchSamples:terra_test_2)
- [08_MergeBatchSites](https://dockstore.org/my-workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_08_MergeBatchSites)
  - This is only required when multiple batches need to be merged into a single cohort.
- The remaining four workflows have two versions: one for a single batch, and one for a cohort made up of multiple batches that have been merged with module 08:
  - [09_GenotypeBatch](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_09_GenotypeBatch:terra_test_2)
    - Single-batch version: [09_SingleBatch_GenotypeBatch](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_09_SingleBatch_GenotypeBatch:terra_test_2)
  - [10_RegenotypeCNVs](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_10_RegenotypeCNVs:terra_test_2)
    - Single-batch version: [10_SingleBatch_RegenotypeCNVs](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_10_SingleBatch_RegenotypeCNVs:terra_test_2)
  - [11_MakeCohortVcf](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_11_MakeCohortVcf:terra_test_2)
    - Single-batch version: [11_SingleBatch_MakeCohortVcf](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_11_SingleBatch_MakeCohortVcf:terra_test_2)
  - [12_AnnotateVcf](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_12_AnnotateVcf:terra_test_2)
    - Single-batch version: [12_SingleBatch_AnnotateVcf](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_12_SingleBatch_AnnotateVcf:terra_test_2)
- There is also an additional workflow, [extra_FilterOutlierSamples](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_extra_FilterOutlierSamples:terra_test_2), which isn't part of the official pipeline, but may be useful for outlier filtration. The GATK-SV documentation recommends running 07b_PlotSVCountsPerSample first, reconfigured with the single VCF intended to be filtered, so as to enable setting the IQR cutoff.

### Workflow outputs

The default outputs can be used by going to the "Outputs" tab on the workflow page on Terra, and clicking "Use defaults" in the header row of the "Attribute" column.

## Running the workflow (Terra)

The following instructions assume that the GATK-SV workflow is to be run on the Terra platform, as this simplifies the process of configuring the workflow. For very large cohorts, it may be necessary to run this pipeline on a custom Cromwell server due to the limits of Terra, however we haven't tested the workflow in this manner to date.

### Stage 01: Gather Sample Evidence

This stage is run on each sample individually. Simply select all the samples to be processed by selecting the "Run workflow(s) with inputs defined by data table" option, selecting "sample" under the "Select root entity type" dropdown box, clicking the "SELECT DATA" button, and selecting all desired samples. If running MELT, supply a MELT docker image to the `GatherSampleEvidence:melt_docker` parameter. As mentioned above, it may also be useful to use the default algorithm (as opposed to the default fast algorithm) for the Picard tools CollectWgsMetrics sub-task. To do this, use the `terra_test_2_melt` branch of the workflow and set `GatherSampleEvidence:use_fast_algorithm` to `false`.

When everything is configured, run the workflow.

### Stage 02: Evidence QC

This stage is run at the sample-set level rather than the sample level in Terra. A single sample-set should represent a batch of up to 500 samples. Simply select "sample_set" under the "Select root entity type" dropdown box, then click the "SELECT DATA" button and select the sample set for the current batch. Then run the workflow.

After this step has run, it is a good idea to do some manual inspection of the quality control results. The important files to look at are:

`<BATCH_NAME>_WGD_scores.txt` - dosage scores, which should be centred around 0. PCR- samples tend to be slightly lower than 0, PCR+ samples slightly higher.
`<BATCH_NAME>_ploidy_plots.tar` - ploidy plots archive, which contains sex assignments (`ploidy_est/sample_sex_assignments.txt.gz`). Check this file for any discrepancies and update the PED file if necessary. Also check each `ploidy_est/estimated_CN_per_bin.all_samples.chr*.png` plot for any potential autosomal aneuploidies and remove the outlying samples.
`<BATCH_NAME>.<SV_CALLER>.QC.outlier.low` - table of low-call outliers for each SV caller. The SV callers will be `Manta`, `Wham`, and optionally `Melt`.
`<BATCH_NAME>.<SV_CALLER>.QC.outlier.high` - table of high-call outliers.

At this stage, a new sample-set should be created reflecting the outlier-removed batch. This will be used for the following workflows.

### Stage 03: Train gCNV

Simply select the new outlier-removed sample-set created following stage 02 and start the run.

### Stage 04: Gather Batch Evidence

Run using the same sample set as stage 03. **Important:** ensure that the PED file is properly formatted and that the sample IDs recorded in it match the sample IDs in the Terra data table.

### Stage 05: Cluster Batch

Run using the same sample set as in stage 03 and stage 04.

### Stage 06: Filter Batch

Run using the same sample set as in stages 03-05.

#### 07a: Filter Batch Sites

Run using the same sample set as in stages 03-06.

#### 07b: Plot SV Counts Per Sample

Run using the same sample set as in stages 03-07a. Optionally, the parameter `N_IQR_cutoff` can be changed from the default value of `6`. This parameter specifies the number of interquartile ranges below and above quartile 1 (Q1) and quartile 3 (Q3), respectively, to use as an outlier threshold. For example, at the default value of `6`, the upper threshold of SV counts per sample to be used as an outlier threshold will be `Q3 + (6 * IQR)`; the lower threshold will be `Q1 - (6 * IQR)`. This stage of the pipeline can be re-run with new values for `N_IQR_cutoff` if necessary. The plots generated by this stage can then be used to select the final value for `N_IQR_cutoff` to be used in the following stage (07c).

#### 07c: Filter Batch Samples

Run using the same sample set as in stages 03-07a, and setting the `N_IQR_cutoff` to an appropriate value based on the plots generated by stage 07b and the needs of the project.

### Stage 08: Merge Batch Sites

This only needs to be run for multi-batch cohorts. If this is the case, it should be run on a "sample-set-set" - a set defining all of the batches in the cohort. This meta-set of batches is created by selecting `sample_set` as the root entity type and selecting all batches that form the cohort.

### Stages 09-12

For the remaining stages, there are two modes: single-batch mode and multi-batch mode. For cohorts only comprised of a single batch of samples, the single-batch versions of the workflows (`*_SingleBatch`) should be used, while the multi-batch versions should be used otherwise.

### Stage 09: Genotype Batch

Run using the same sample set as in stages 03-07c.

### Stage 10: Regenotype CNVs

For multi-batch cohorts, run using the same "sample-set-set" defined in stage 08. Otherwise run using the same sample set as in stages 03-09 and use the `*_SingleBatch` single batch version of the workflow.

### Stage 11: Make Cohort VCF

For multi-batch cohorts, run using the same "sample-set-set" defined in stage 08. Otherwise run using the same sample set as in stages 03-10 and use the `*_SingleBatch` single batch version of the workflow.

### Stage 12: Annotate VCF

For multi-batch cohorts, run using the same "sample-set-set" defined in stage 08. Otherwise run using the same sample set as in stages 03-11 and use the `*_SingleBatch` single batch version of the workflow.

## Expected time and cost

The following table lists approximate times and costs for running each stage with based on a recent pilot run of a cohort of 179 samples where the inputs were GRCh-38-aligned CRAM files with typical coverages of ~30x and sizes of ~30GB.

| Stage | Notes | Approx. Time (hours) | Approx. Cost Per Job (AUD) | Approx. Cost Per Sample (AUD) |
| ----- | ----- | -------------------- | -------------------------- | ----------------------------- |
| 01 | Run at sample level, without MELT | 13 | $1.65 | $1.65 |
| 01 | Run at sample level, with MELT | 19 | $3.15 | $3.15 |
| 02 | Run at batch level - 179 samples | 3 | $2.15 | $0.01 |
| 03 | Run at batch level - 167 samples (outliers removed) | 7 | $30 | $0.18 |
| 04 | Run at batch level - 167 samples (outliers removed) | 12 | $130 | $0.78 |
| 05 | Run at batch level - 167 samples (outliers removed) | 1 | $0.20 | < $0.01 |
| 06 | Run at batch level - 167 samples (outliers removed) | 6 | $8.30 | $0.05 |
| 07a | Run at batch level - 167 samples (outliers removed) | 1 | < $0.01 | < $0.01 |
| 07b | Run at batch level - 167 samples (outliers removed) | 1 | $0.01 | < $0.01 |
| 07c | Run at batch level - 167 samples (outliers removed) | 1 | $0.02 | < $0.01 |
| 08 | Not run in our pilot project (only run for multi-batch cohorts, at cohort level) | NA | NA | NA |
| 09 | Run at batch level - single-batch version - 167 samples (outliers removed) | 4 | $0.53 | < $0.01 |
| 10 | Run at cohort level (one batch only) - single-batch version - 167 samples (outliers removed) | 2 | $0.06 | < $0.01 |
| 11 | Run at cohort level (one batch only) - single-batch version - 167 samples (outliers removed) | 7 | $4.62 | $0.03 |
| 12 | Run at cohort level (one batch only) - single-batch version - 167 samples (outliers removed) | 2 | $0.78 | < $0.01 |
