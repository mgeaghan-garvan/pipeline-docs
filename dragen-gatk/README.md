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
| zone | Google Cloud zone to run the analysis in, typically 'australia-southeast1-a |

For re-processing a sample from BAMs or CRAMs, only the following columns are required:

| Column | Description |
| ------ | ----------- |
| entity:sample_id | External sample ID |
| inputBam | BAM file to be reprocessed. Alternative to 'inputCram'. |
| inputCram | CRAM file to be re-processed. Alternative to 'inputBam'. |
| zone | Google Cloud zone to run the analysis in, typically 'australia-southeast1-a |

**Note:** Internal and external sample IDs, along with library IDs, are recorded in the KCCG's MetricsDB runs database.

### Adding the DRAGEN-GATK workflow to the Terra workspace
