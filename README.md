# Dartmouth CQB-tRAX Pipeline
Pipeline dapted from the Lowe Lab (UCSC) for analyses of tRNA-derived small RNAs (tDRs), mature tRNAs, and inference of RNA modification from high-throughput sequencing. 


# Table of Contents
- [Introduction](#introduction)
- [Summary](#summary)
- [Directories](#directories)
- [Files](#files)
- [Implementation](#implementation)

## Introduction
The CQB-tRAX pipeline is adapted from the original [tRAX pipeline](https://github.com/UCSC-LoweLab/tRAX) from the Lowe Lab (UCSC) to be compatible with the [Dartmouth Discovery HPC](https://rc.dartmouth.edu/discoveryhpc/). This pipeline supports the analysis of tDRs, mature-tRNAs and RNA modification for human (hg19, hg38), mouse (mm10), rat(rn6), yeast(sacCer3), and fly(dm6) genomes. This repository contains example data from the [GSE149989](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE150076) (SRA accession SRP260297). The 4 samples included are subsetted to the first 500 reads per file. Required software can be installed using a [conda environment](https://docs.conda.io/en/latest/) with the enrionment file `config.yaml`

## Summary
The CQB-tRAX pipeline consists of three major steps implemented by 3 SLURM job scripts:
1. Database building
2. Adapter trimming with [CutAdapt](https://cutadapt.readthedocs.io/en/stable/) or [SeqPrep](https://github.com/jstjohn/SeqPrep)
3. Mapping with [Bowtie2](https://bowtie-bio.sourceforge.net/bowtie2/index.shtml) and analysis with custom python and R scripts

To run this pipeline you will need raw fastq files (either single or paired-end), a file outlining sample names (runfile.txt), a file outlining replicate information (repfie.txt) and a file specifying comparisons for diferntial expression analysis (design.txt)


## Directories
To get started with the tRAX pipeline, clone this repository into your working directory.

``` bash
# Run from within your working directory
git clone https://github.com/Dartmouth-Data-Analytics-Core/DAC-tRAX
```

After successful repo cloning, you should have a folder called `code`, `example_data`, and three bash scripts
1. `001_DATABASE_GENERATION.bash`
2. `002_TRIMMING.bash`
3. `003_MAPPING.bash`

Create a directory called `samples` in your working directory
``` bash
# Run from within your working directory
mkdir samples
```

The `samples` directory will hold your raw sequencing files (either paired-end or single-end) in fastq.gz format
as well as three required files needed for the pipeline to run (generated by the user.)


## Files
Three files that help guide the code need to be created and placed into your `samples` directory. 
These files should be white-space separated (one single space) and contain no headers. They should have the extension .txt. Failure to format these files in this manner will result in pipeline failure. 

1. A run file `runfile.txt`
   
This file assigns a sample name to each sequencing file you have. If your data is paired-end, this file will be comprised of three fields: field 1 should be sample name of your choice, field 2 will be the exact filename of the forward sequencing file, and field 3 will be the eaxct filename of the reverse sequencing file. For single-end data, this file will be two files where field 1 is a sample name of your choosing and field 2 is the exact filename of the sequencing file. 

An example of how `runfile.txt` should be formatted for single-end data:
```plaintext
sample1 sample1_1.fastq.gz
sample2 sample2_1.fastq.gz
sample3 sample3_1.fastq.gz
sample4 sample4_1.fastq.gz
```

An example of how `runfile.txt` should be formatted for paired-end data
```plaintext
sample1 sample1_1.fastq.gz sample1_2.fastq.gz
sample2 sample2_1.fastq.gz sample2_2.fastq.gz
sample3 sample3_1.fastq.gz sample3_2.fastq.gz
sample4 sample4_1.fastq.gz sample4_2.fastq.gz
```
2. A replicate file `repfile.txt`

This is a three column file for both paired and single-end data. Field 1 indicates a replicate ID, field 2 indicates an experimental group, and field 3 is the exact file name of your trimmed fastq files. 

An example of how `repfile.txt` should be formatted:
```plaintext
WT1 WT sample1_trimmed.fastq.gz
WT2 WT sample2_trimmed.fastq.gz
KO1 KO sample3_trimmed.fastq.gz
KO2 KO sample4_trimmed.fastq.gz
```
3. A design file `design.txt`
   
A simple 2 field text file where field 1 is the experimental group to be compared against field 2 for DESeq (field 1 = comparison numerator, field 2 = comparison denominator)

An example of how `design.txt` file should be formatted:
```plaintext
KO WT
```
These files can be generated via your terminal using nano, or externally in a text editor.
To edit these files with nano:

```bash
# Run from within your sample directory
nano runfile.txt
```
A blank text editor window will open. You can now add your text according to the required formatting. Once finished, to close and save the file, hit control + X, and hit RETURN.

# Implementation
## Database building
The first step of the tRAX pipeline is building a tRNA database using files from gtRNAdb and genome assemblies. Currently, the tRAX pipeline supports human (hg19, hg19mito, mg38, hg38mito), mouse (mm10, mm10mito), rat (rn6), yeast(sacCer3), and fly (dm6). To run this script, you will need to edit two variables (required) as well as edit SBATCH header parameters (optional.)

Parameters you can opt to change: Within the SBATCH headers, you can provide a job name, update your email and email notification preferences to notify you when a job starts, ends, or fails. 

You will find teo variables directly underneat the SBATCH headers called `WORKINGDIR` and `GENOME`. These variables must be defined within quotations. `WORKINGDIR` is the absolute path to your working directory. `GENOME` is the organism code of which your data is. DO NOT add additional whitespace between the variable name, the equal sign, and the path. Genome names are CASE SENSITIVE and should be lowercase, with the exception of sacCer3.

Example of a CORRECT variable assignment: 
```bash
WORKINGDIR="/path/to/my/workingdir/"
GENOME="hg38"
```

Example of an INCORRECT variable assignment
```bash
WORKINGDIR = path/to/my/working/dir
WORKINGDIR = "path/to/my/working/dir"
WORKINGDIR=/path/to/my/working/dir
```
Once these edits are complete, you can run the script as a SLURM job on Discovery by executing the following command from within your wokring directory
```bash
sbatch 001_MAKE_DATABASE.bash
```

This script will generate an additional subdirectory in your wokring directory named as `<genome>_db`
Within this directory, you will see organism specific files and tRNA-genome files.
Below is a list of outputs generated from the script.
1. `genes.gtf` - gtf file for your organims
2. `genome.fa` - fasta file of genome assembly
3. `genome.fa.fai` - fasts file index for genome assembly
4. `hg38-tRNAs.fa` - fasta file of tRNAs for genome pulled from gtRNAdb
5. `hg38-tRNAs.bed` - bed file of tRNAs
6. `hg38-tRNAs-detailed.ss` - gtRNAdb file
7. `hg38-tRNAs-detailed.out` - gtRNAdb file
8. `hg38-tRNAs-confidence-set.ss` - gtRNAdb file
9. `hg38-tRNAs-confidence-set.out` - gtRNAdb file
10. `hg38-tRNAs_name_map.txt` - names of tRNAs
11. `hg38-mature-tRNAs.fa` - fasta file of mature tRNAs
12. `hg38-filtered-tRNAs.fa` - fasta file of filtered tRNAs
13. `db-trnatable.txt` - table of tRNA anticodon information
14. `db-trnaloci.stk` - Stockholm alignment of tRNA loci in database
15. `db-trnaloci.bed` - Bed file of tRNA loci in database
16. `db-tRNAgenome.fa` - Fasta file of tRNA genome
17. `db-trnaalign.stk` - Stockholm alignment of tRNAs in database
18. `db-maturetRNAs.fa` - Fasta file of mature tRNAs in database
19. `db-maturetRNAs.bed` - Bed file of mature tRNAs in database
20. `db-locusnum.txt` - Locus numbers
21. `db-dbinfo.txt` - Database info
22. `db-alignnum.txt` - Alignment number
23. `db-tRNAgenome.1.bt2l` - Bowtie 2 index file (will have multiple of these files)

## Adapter trimming
The second part of the tRAX pipeline is adapter trimming. This step is executed with the script `002_TRIMMING.bash` which again requires some minor editing from the user before running. 
The following variables must be defined: 

`WORKINGDIR`
`RUNFILE`
`LAYOUT`

`WORKINGDIR` is once again the absolute path to your wokring directory. 
`RUNFILE` is the name of your `runfile.txt` file (case sensitive, must be exactly the same as you have it named)
`LAYOUT` is either `"SE"` for single-end data or `"PE"` for paired-end data (case sensitive, please use all caps)

Once these edits are complete, you can run the script as a SLURM job on Discovery by executing the following command from within your wokring directory
```bash
sbatch 002_TRIMMING.bash
```
This script will generate a subdirectory within your samples directory called `Trimming_Diagnostics` which includes the following files:
1. `runname_log.txt`: output log containing statistics generated by CutAdapt or SeqPrep
2. `runname_manifest.txt`: List of sample names and trimmed/merged FASTQ files

For single-end reads, the following files are created:
1. `samplename_trim.fastq.gz`: FASTQ file containing trimmed sequencing reads to be used for downstream analysis
2. `runname_ca.pdf`: histogram summarizing the amount of trimmed, untrimmed, and discarded reads.
3. `runnames_ca.txt`: Data source for the histogram in `runname_ca.pdf`

For Paired-end reads, the following files are created:
1. `samplename_merged.fastq.gz`: FASTQ file containing trimmed and merged sequencing reads to be used downstream.
2. `samplename_right.fastq.gz`: FASTQ file containing Read 1 that do not merge
3. `samplename_left.fastq.gz`: FASTQ file containing Read 2 that do not merge
4. `runname_sp.pdf`: Histogram summarizing the amount of merged, unmerged, and discarded reads
5. `runname_sp.txt`: Data source for the histogram in `runname_sp.pdf`

Additional parameters can be added for more complex trimming operations. Such options include:

`--runname` = runname (required)
Name used for output files

`--runfile` = runfile (required)
Tab or whitespace-delimited file with sample and FASTQ file information (described below)

`--firadapter` = FIRADAPTER (optional)
3' adapter sequence used in single end reads or forward primer/adapter sequence used in paired end reads
Default value is AGATCGGAAGAGCACACGTC

`--secadapter` = SECADAPTER (optional)
Reverse primer/adapter sequence used in paired end reads
Default value is GATCGTCGGACTGTAGAACTC

`--minlength` = MINLENGTH (optional)
Minimum length of sequencing reads to be retained after trimming and/or merging
Default value is 15

`--singleend`
Flag to specify single end reads

`--cores` = CORES (optional)
Number of cores to be used
Default value is the number of available cores on the system

`--umilength` = length (optional)
Length of UMI
Default value is 0

`--umithreeprime` (optional)
Flag to indicate that UMI is located at the 3' end of the reads instead of 5' end. It will have no effect if --umilength is 0.

To edit parameters not included in the SBATCH script, the python call can be found on lines 89-101. 

## Mapping and analysis
The third and final step of the tRAX pipeline involves mapping your trimmed reads to the tRNA genome and analyzing tRNA counts through DESeq2 (`003_MAPPING.bash`).
This script requires the most editing to run. Please fill out variables with caution, taking care to notice spaces, capitalization, and spelling to ensure a smooth run.


`WORKINGDIR` is once again the absolute path to your working directory.

`DBNAME` is the name of your database directory you created in the first script (i.e `<genome>_db`)

`EXPERIMENTNAME` is a prefix for all output files. Avoid using spaces and hypens. Instead use underscores or camelCase

`REPLICATES` is the name of your `repfile.txt` file

`COMPARISONS` is the name of your `design.txt` file

Once these edits are complete, you can run the script as a SLURM job on Discovery by executing the following command from within your wokring directory
```bash
sbatch 003_MAPPING.bash
```
NOTE: there are additional parameters which can be added to the script call.  If additional parameters must be added, the python call is located on lines 104-112 of the 003_MAPPING.bash script. Additional parameters are specified below:

Options

`--experimentname` = expname (required)
Experiment name that will be used for results

`--databasename` = dbname (required)
Directory and name of reference database generated by maketrnadb.py

`--ensemblgtf` = genes.gtf.gz (required)
Non-tRNA annotation file in Ensembl GTF file format

`--samplefile` = samplefile.txt (required)
Tab or whitespace-delimited file with sample and pre-trimmed FASTQ file information (described below)

`--exppairs` = samplespairs.txt (optional but recommended)
Tab or whitespace-delimited file with sample pairs for differential expression comparison (described below)
If not provided, all vs all pairwise comparison will be performed.

`--bedfile` = file.bed (optional)
Other non-tRNA gene annotations in BED file format
Multiple files can be specified.

`--minnontrnasize` = SIZE (optional)
Minimum read length to be retained for non-tRNA genes
Defaults value is 20

`--nofrag`
Flag to skip tRNA fragment determination
Design for full length tRNA sequencing experiments

`--lazyremap`
Skip read mapping step when BAM files already exist

`--makehub`
Convert BAM files by interpolating tRNA read alignments to the reference genome and create read coverage tracks in track hub to be used in UCSC Genome Browser

`--olddeseq`
Use DESeq1 instead of DESeq2

`--dumpother`
Output read alignments that do not have an associated gene feature into BAM files (files end in "nofeat.bam")

`--cores`
Number of processing cores to be used (default = 8)

To add additional parameters: edit the python call by appending a space followed by \
then return a newline and add the flag you desire. An example of this can be seen below. 

```bash
python "$CODEDIR"/03_processsamples.py \
--experimentname="$EXPERIMENTNAME" \
--databasename="$DATABASEPATH" \
--ensemblgtf="$GTFPATH" \
--samplefile="$REPFILE" \
--exppairs="$PAIRFILE" \
--bamdir="$BAMDIR" \
--nofrag \
--lazyremap \
--olddeseq
```
This script generates a wide number of outputs and therefore manipulates directories to clean them up and organize them. You will notice in your working directory (upon successful script execution) the addition of a subdirectory named `Outputs`. Within `Outputs` you will find a subdirectory named after your `EXPERIMENTNAME`. This subdirectory contains multiple other subdirectories with multiple files in each. 

Outputs:

`coverage_plots` (folder):
Contains a coverage plot for each amino acid as well as a combined coverage plot.
    
`DESeq_Results` (folder):
Contains multiple differential expression figures (PCAs, Volcanos, scatter plots)
Contains a subdirectory `tabular_data` for numerical data pulled from DESeq2 such as padj vals, fold changes, normalized read counts, dispersion values, and median counts. 
    
`endpoint_plots` (folder):
Contains endpoint plots for each amino acid
    
`Mapping_Diagnostics` (folder):
Contains figures and numerical data related to mapping 
    
`mismatch` (folder):
Contains mismatch information.
Contains a `five_prime`, `three_prime`, `deletions`, and `mismatches` subdirectories for misincorporation estimates. These subdirectories contain heatmaps and numerical data and figures. 
    
`pretRNAs` (folder):
Contains figures and numerical data for pre-tRNA coverages
    
`unique` (folder):
Contains only unique coverages for each amino acid as well as counts
    
`tRNA_tails` (folder):
Contains information and figures regarding the C, CC, CCA ending percentages.
    
`Full_relative_summed_tRNA_mismatches.png` (file):
A heatmap showing summed mismatches in one condition relative to the other for all iso-decoders grouped by amino acid family. 



