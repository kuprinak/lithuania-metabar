## Metabarcoding analysis of fungi associated with roots of three plant species:  _Diphasiastrum complanatum_, _Pinus sylvestris_ and _Vaccinium myrtillus_
#### Results are published here: https://doi.org/10.1016/j.rhisph.2025.101053

> [!TIP]
> To view any HTML file, please, download it

### List of files:

* Bioinformatic steps in bash.md - script for sequence data processing using Trimmomatic, VSEARCH and FUNGuild.
* Raw_Counts.txt - row number of reads per OTU in each sample (output from VSEARCH).
* Sample_Info.txt - information about the samples including technical replicates: sample name ("Sample"), sample type ("Species"), sampling site ("Location").
* Sample_Info_noR.txt - information about the samples excluding technical replicates: sample name ("Sample"), sample type ("Species"), sampling site ("Location"), part of the forest ("Forest"), age of the _Pinus sylvestris_ at the location ("Age", years), Height of the _Pinus sylvestris_ at the location ("Height", m), the diameter of the _Diphasiastrum complanatum_ colony ("Diph_colony, m).
* Sequences_names.xlsx - information about the names of the sequences submitted to NCBI (PRJNA1185013 - trimmed reads).
* Taxonomy.txt - assignment of OTUs to fungi taxa (UNITE database v9.0), trophic modes and functional guilds (FUNGuild v1.1 database).
* MultiQC_raw_reads.html - MultiQC report on the quality of the raw reads.
* MultiQC_trimmed_reads.html - MultiQC report on the quality of the trimmed reads.
* Data_analysis_in_R.html - data analysis in R with the output.

## Used programs:

#### Python packages:

* [Trimmomatic](http://www.usadellab.org/cms/index.php?page=trimmomatic) v0.33 - trimming of adapters, low-quality bases and removing of low-quality reads
* [VSEARCH](https://peerj.com/articles/2584/) v2.15 - merging of forward and reverse reads, quality filtering of reads, dereplication of reads across the samples and removal of singletons, pooling of the samples, denoising of the sequences, identification and removal of chimaera sequences using the UCHIME algorithm, applying both _de novo_ and reference-based detections, clustering of resulting sequences to OTUs, taxonomical assignment of OTUs
* [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) v0.12.0 and [MultiQC](https://seqera.io/multiqc/) v1.14 -  quality reports for raw and trimmed reads
#### R packages:

* [DESeq2](https://genomebiology.biomedcentral.com/articles/10.1186/s13059-014-0550-8) v1.46 - normalization of read counts through variance stabilizing transformation
* [phyloseq](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0061217) v1.50 - Principle Coordinate Analysis
* [microeco](https://academic.oup.com/femsec/article/doi/10.1093/femsec/fiaa255/6041020?login=true) v1.11.0 - calculation of taxonomic abundance per sample
* [metacoder](https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1005404) v0.3.7 - pairwise comparison of taxonomic abundances
* [iNEXT](https://besjournals.onlinelibrary.wiley.com/doi/10.1111/2041-210X.12613) v3.0.1 - calculation and visualisation of species diversity indices (Hill numbers) per tample type
* [rstatix](https://github.com/kassambara/rstatix) v0.7.2 - Kruskal-Wallis test with Dunn's post-hoc test for the abundancy of trophic modes and guilds among sample types 


## Workflow:

```mermaid
flowchart TB
    A@{shape: procs, label: "Illumina raw reads"} --> B([Trimmomatic]);
    A --> Fa;
    Fa --> Mu([MultiQC]);
    B --> C@{shape: procs, label: "Trimmed reads (NCBI: PRJNA1185013)"};
    C --> Fa([FastQC]);
    Mu --> Mur@{shape: procs, label: "MultiQC_raw_reads.html, MultiQC_trimmed_reads.html"}; 
    C --> V([VSEARCH]);
    V --> T([UNITE v9.0 database]);
    T --> CV@{shape: procs, label: "Raw_Counts.txt"};
    T --> Ta@{shape: procs, label: "Taxonomy.txt"};
    Ta --> F([FUNGuild v.1.1 database]);
    F --> Gu@{shape: procs, label: "Guilds and trophic modes of taxa"};
    Gu --> rs([rstatix])
    CV --> D([DESeq2]);
    D --> N@{shape: procs, label: "Normalized and filtered read counts"};
    Ta --> Mi;
    Ta --> Me
    Ta --> Ph
    N --> Ph([phyloseq])
     N --> Mi([microeco])
    N --> Me([metacoder])
    Ta --> iN([iNEXT]);

```





