# Running GATK-SV

This is a work-in-progress SOP/documentation for running the GATK-SV pipeline on Terra.

GATK-SV runs in one of two main modes: batch and single-sample. Batch mode is intended for cohorts larger than 100 samples, and won't work for cohorts less than 10 samples. For small cohorts, single-sample mode should be used on each sample individually. Single-sample mode requires a reference panel, which will ideally be created using a cohort as similar as possible to the target sample. A standard reference panel is hosted by the Broad Institute in their public Google Cloud resource bucket based on samples from the 1000 Genomes project and can be used if no other relevant reference panel is available.

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
  - The cohort inputs are available under `inputs/build/ref_panel_1kg`
    - Terra inputs are in the sub-directory `terra`, while inputs for a custom Cromwell server are under `test`
      - Terra input JSON files are within `terra/workflow_configurations`, and the sub-directory `terra/workflow_configurations/output_configurations` contains a few customised output JSON files.
    - The Terra workspace.tsv file will need to be changed for one field: `cohort_ped_file` needs to point to a PED file specific to the cohort of interest. The PED format is described [here](https://gatk.broadinstitute.org/hc/en-us/articles/360035531972-PED-Pedigree-format).
  - The single-sample inputs are available under `inputs/build/NA12878`
    - Again, Terra inputs are in the sub-directory `terra`, while inputs for a custom Cromwell server are under `test`
    - The Terra workspace.tsv file will need to be changed for single-sample mode as well, namely in specifying both the VCF file for the reference panel (`ref_panel_vcf`) and the PED file for the reference panel (`ref_panel_ped_file`).

### Adding the GATK-SV workflow to the Terra workspace

All of the GATK-SV workflows are currently hosted through Dockstore and originate from Shyam's github account. The branch is terra_test_2 which is a fork of the Broad master branch at commit 7505396 (https://github.com/broadinstitute/gatk-sv/commit/7505396171365b84e26962241be43828b67d88b8) with a dockstore configuration file added. These workflows can be added to the Terra workspace by following the steps in the [Terra docs](../terra/README.md).

The single-sample pipeline consists of a [single workflow](https://dockstore.org/my-workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineSingleSample.wdl).

It is possible to run the batch pipeline with a single workflow ([GATKSVPipelineBatch](https://dockstore.org/my-workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch)) that runs all the necessary sub-modules, but it isn't recommend for Terra, and the inputs for this workflow aren't produced by the build scripts. In part, this is due to the large number of parameters required for the entire workflow, which Terra doesn't handle very well. It can also be easier to diagnose and manage failures of runs when the workflow is run as separate sub-modules. Additionally, there is a step (module 07b) where manual inspection of results and selection of an outlier cutoff parameter is recommended for further processing. As such, for discussion of working in Terra, this documentaiton will only cover running the individual sub-modules.

There are currently 12 main modules to the pipeline, which are listed here in the order to be run:
- [01_GatherSampleEvidence](https://dockstore.org/workflows/github.com/shyamrav/gatk-sv/GATKSVPipelineBatch_01_GatherSampleEvidence:terra_test_2)
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

## Running the workflow

-- WIP --

## Expected time and cost

-- WIP --
