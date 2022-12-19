# Running DRAGEN-GATK

This is a work-in-progress SOP/documentation for running the DRAGEN-GATK pipeline on Terra.

## Terra workspace setup

Follow the instructions in the [Terra docs](../terra/README.md) to create a Terra workspace for running DRAGEN-GATK.

### Data table setup

DRAGEN-GATK only requires a sample table; no workspace data table is required. The sample table is a TSV file, containing one row per sample with tab-delimited fields. It requires only a few columns, which differ depending on whether you are running the pipeline from FASTQs or re-processing BAM/CRAM files.

To run from FASTQs, the sample table needs the following columns:

| Column | Description |
| ------ | ----------- |
| entity:sample_id | External sample ID |
| R1 | Google Cloud Storage path to FASTQ file for read 1 |
| R2 | Google Cloud Storage path to FASTQ file for read 2 |
| RGID | Read group ID. Typically we use the format '\<flowcell ID\>.\<flowcell lane\>.\<sample ID\>', where 'sample ID' is the internal sample ID rather than the external sample name. |
| RGPL | Platform, e.g. 'ILLUMINA' |
| RGPU | Platform unit. Typically we use the format '\<flowcell ID\>.\<flowcell lane\>'. |
| RGLB | Library ID |
| RGCN | Sequencing centre, e.g. 'KCCG' |
| zone | Google Cloud zone to run the analysis in, typically 'australia-southeast1-a' |

For re-processing a sample from BAMs or CRAMs, only the following columns are required:

| Column | Description |
| ------ | ----------- |
| entity:sample_id | External sample ID |
| inputBam | BAM file to be reprocessed. Alternative to 'inputCram'. |
| inputCram | CRAM file to be re-processed. Alternative to 'inputBam'. |
| zone | Google Cloud zone to run the analysis in, typically 'australia-southeast1-a' |

**Note:** Internal and external sample IDs, along with library IDs, are recorded in the KCCG's MetricsDB runs database.

### Adding the DRAGEN-GATK workflow to the Terra workspace

The DRAGEN-GATK workflow is part of the WARP (WDL Analysis Research Pipelines) repository published by the Broad Institute. It is executed by running the single-sample whole-genome germline alignment and variant calling pipeline with the parameter `dragen_functional_equivalence_mode` set to `true`. The version of the WARP repository we are currently using is hosted on Shyam's GitHub on branch [australian_zone_dockers](https://github.com/shyamrav/warp/tree/australian_zone_dockers), which contains a Dockstore configuration file. The two workflows we currently use from this repository relevant to DRAGEN-GATK are registered and tracked through Dockstore:
- [WholeGenomeFromFastq](https://dockstore.org/workflows/github.com/shyamrav/warp/WholeGenomeFromFastq:australian_zone_dockers) is the pipeline to run alignment and variant calling starting from FASTQ files.
  - There is [another version of this pipeline](https://dockstore.org/my-workflows/github.com/mgeaghan-garvan/warp/WholeGenomeFromFastq), which originates from Michael's GitHub account on the branch `fastqtosam_dynamic_disk`, which was created to fix an issue where larger FASTQs would run out of disk space due to the `additional_disk` parameter not being used for the FastqToSam task (instead it was originally hard-coded at 400GB).
- [WholeGenomeReprocessing](https://dockstore.org/workflows/github.com/shyamrav/warp/WholeGenomeReprocessing:australian_zone_dockers) is the pipeline to re-process BAM or CRAM files.

To add these workflows to the Terra workspace, follow the steps in the [Terra docs](../terra/README.md).

### Workflow inputs

Example input JSON files for these workflows can be found here:
- [WholeGenomeFromFastq](../dragen-gatk/resources/json/inputs.fastq.json)
  - [WholeGenomeFromFastq - with additional disk](../dragen-gatk/resources/json/inputs.fastq_add_disk.json)
- [WholeGenomeReprocessing](../dragen-gatk/resources/json/inputs.bam.json)

### Workflow outputs

The default outputs can be used by going to the "Outputs" tab on the workflow page on Terra, and clicking "Use defaults" in the header row of the "Attribute" column.

## Running the workflow

- On the workflow page on Terra, select "Run workflow(s) with inputs defined by data table".
- Under "Step 1", select "sample" from the drop-down menu.
- Under "Step 2", click the "Select Data" button, then select all samples from the sample table that you wish to process. Click "OK" to confirm the selection.
- Click "Run Analysis", give the run a description, and begin the run.

## Expected time and cost

For a typical sample, consisting of paired-end data and around 30X coverage, a run can be expected to last between 50 and 100 hours and cost between $15 and $30 per sample.
