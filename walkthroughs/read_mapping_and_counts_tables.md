### Read mapping and counts tables
This walkthrough goes through read mapping and generating counts tables.

What you'll end up with at the end of this are counts tables at the contig, MAG, SLC, ORF, and SSO levels.

It is assumed you've completed either the [end-to-end metagenomics](end-to-end_metagenomics.md) or the [recovering viruses from metatranscriptomics](recovering_viruses_from_metatranscriptomics.md) walkthroughs.

We focus on global read mapping in this tutorial but local is similar. 

_____________________________________________________

#### Steps:

1. Create a global index of genomes and their gene models
2. Map reads to global reference and create base counts tables
3. Merge the counts tables for all the samples

_____________________________________________________


#### 1. Create a global index of genomes and their gene models

Here we are going to concatenate all of the binned contigs (i.e., MAGs) and their respective gene models (i.e., GFF files) then index using `Bowtie2`.

**Conda Environment:** `conda activate VEBA-mapping_env`


```
# Set the number of threads
N_JOBS=16

# Set a useful name for log files
N=index-global
rm -f logs/${N}.*

# Get list to all genomes
ls veba_output/binning/*/*/output/genomes/*.fa > veba_output/misc/all_genomes.list

GENOMES=veba_output/misc/all_genomes.list

# Get list to all gene models
ls veba_output/binning/*/*/output/genomes/*.gff > veba_output/misc/all_genomes.gene_models.list

GENE_MODELS=veba_output/misc/all_genomes.gene_models.list

# Set up command
CMD="source activate VEBA-mapping_env && index.py -r ${GENOMES} -g ${GENE_MODELS} -o veba_output/index/global/ -p ${N_JOBS}"

# Either run this command or use SunGridEnginge/SLURM
```

The following output files will produced: 

* reference.fa.gz - Concatenated reference fasta
* reference.fa.gz.\*.bt2 - Bowtie2 index of reference fasta
* reference.gff - Concatenated gene models
* reference.saf - SAF format for reference


#### 2. Map reads to global reference and create base counts tables

Here we are map all of the reads to the global reference and create base counts tables for contigs and ORFs using *featureCounts*.

**Conda Environment:** `conda activate VEBA-mapping_env`

```
# Set a lower number of threads since we are running for each sample
N_JOBS=2

# Get the index directory
INDEX_DIRECTORY=veba_output/index/global/output/

for ID in $(cat identifiers.list); do
	N="mapping-global__${ID}";
	rm -f logs/${N}.*
	
	# Get the forward and reverse reads
	R1=veba_output/preprocess/${ID}/output/cleaned_1.fastq.gz
	R2=veba_output/preprocess/${ID}/output/cleaned_2.fastq.gz
	
	# Specify the output directory
	OUT_DIR=veba_output/mapping/global/${ID}
	
	# Set up command	
	CMD="source activate VEBA-mapping_env && mapping.py -1 ${R1} -2 ${R2} -n ${ID} -o ${OUT_DIR} -p ${N_JOBS} -x ${INDEX_DIRECTORY}"
	
	# Either run this command or use SunGridEnginge/SLURM

	done
```

The following output files will produced for each sample: 

* counts.orfs.tsv.gz - ORF-level counts table
* counts.scaffolds.tsv.gz - Contig-level counts table
* mapped.sorted.bam - Sorted BAM file
* unmapped_1.fastq.gz - Unmapped reads (forward)
* unmapped_2.fastq.gz - Unmapped reads (reverse)

#### 3. Merge the counts tables for all the samples:
We have individual counts vectors for ORFs and contigs per sample.  Now we need to merge the contig counts and ORF counts separately.  While we are at it, let's aggregate the contig counts into MAGs and SLCs then aggregate the ORF counts into SSOs.  We are going to use the `merge_contig_mapping.py` and `merge_orf_mapping.py` scripts installed with *VEBA*.  If for some reason they aren't installed, then download them here:
https://github.com/jolespin/veba/tree/main/src/scripts

```
# Get mapping directory
MAPPING_DIRECTORY=veba_output/mapping/global

# Set output directory (this is default)
OUT_DIR=veba_output/counts

# Concatenate all of the scaffolds to bins from all of the domains
cat veba_output/binning/*/*/output/scaffolds_to_bins.tsv > veba_output/misc/all_genomes.scaffolds_to_bins.tsv

SCAFFOLDS_TO_BINS=veba_output/misc/all_genomes.scaffolds_to_bins.tsv

# Concatenate all of the clusters from all of the domains
cat veba_output/clusters/*/clusters.tsv > veba_output/misc/all_genomes.clusters.tsv

ClUSTERS=veba_output/misc/all_genomes.clusters.tsv

# Concatenate all of the orthogroups from all of the domains
cat veba_output/cluster/*/output/proteins_to_orthogroups.tsv > veba_output/misc/all_genomes.orthogroups.tsv

ORTHOGROUPS=veba_output/misc/all_genomes.orthogroups.tsv


# Merge contig-level counts
merge_contig_mapping.py -m ${MAPPING_DIRECTORY} -c ${CLUSTERS}  -i ${SCAFFOLDS_TO_BINS} -o ${OUT_DIR}

# Merge ORF-level counts
merge_orf_mapping.py -m ${MAPPING_DIRECTORY} -c ${ORTHOGROUPS}  -i ${SCAFFOLDS_TO_BINS} -o ${OUT_DIR}
```

The following output files will produced: 

* scaffold_to_mag.tsv - Identifier mapping [id_contig]<tab>[id_mag]
* mag_to_slc.tsv - Identifier mapping [id_mag]<tab>[id_slc]
* orf_to_orthogroup.tsv - Identifier mapping [id_orf]<tab>[id_sso]
* X_contigs.tsv.gz - Counts tables gzipped and tab-delimited (samples, contigs)
* X_mags.tsv.gz - Counts tables gzipped and tab-delimited (samples, MAGs)
* X_orfs.tsv.gz - Counts tables gzipped and tab-delimited (samples, ORFs)
* X_orthogroups.tsv.gz - Counts tables gzipped and tab-delimited (samples, SSOs)
* X_slcs.tsv.gz - Counts tables gzipped and tab-delimited (samples, SLCs)

#### Next steps:

Now it's time to analyze the data using compositional data analysis (CoDA).  

##### Recommended reading for CoDA:

* Thomas P Quinn, Ionas Erb, Mark F Richardson, Tamsyn M Crowley, Understanding sequencing data as compositions: an outlook and review, Bioinformatics, Volume 34, Issue 16, 15 August 2018, Pages 2870–2878, https://doi.org/10.1093/bioinformatics/bty175
* Thomas P Quinn, Ionas Erb, Greg Gloor, Cedric Notredame, Mark F Richardson, Tamsyn M Crowley, A field guide for the compositional analysis of any-omics data, GigaScience, Volume 8, Issue 9, September 2019, giz107, https://doi.org/10.1093/gigascience/giz107
* Espinoza, J.L., Shah, N., Singh, S., Nelson, K.E. and Dupont, C.L. (2020), Applications of weighted association networks applied to compositional data in biology. Environ Microbiol, 22: 3020-3038. https://doi.org/10.1111/1462-2920.15091