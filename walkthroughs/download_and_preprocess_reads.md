### Download and preprocess reads
This walkthrough is for downloading reads from NCBI and running *VEBA's* `preprocess.py` module to decontaminate either metagenomic or metatranscriptomic reads.  Skip step 3 if you already have reads (e.g., a new project):

#### 1. Activate the preprocess environment 
Since *VEBA* has so many capabilities I have partitioned different walkthroughs into modules and environments for those modules.  In this case, we are going to use the `VEBA-preprocess_env`.  For assembly, we will use `VEBA-assembly_env`, etc.  This is to increase our flexibility in updating dependencies.
```
conda activate VEBA-preprocess_env
```

If you want to either remove human contamination or count ribosomal reads then make sure VEBA database in your path.  If it's not there then `export VEBA_DATABASE=/path/to/database/`.  If you just want to quality trim/remove adapters then you don't need the databases.

```
echo $VEBA_DATABASE
/expanse/projects/jcl110/db/veba/v1.0 
# ^_^ Yours will be different obviously #
```

#### 2. Make a list of identifiers and create necessary directories
*VEBA* is built on structured data so if you're downloading from NCBI SRA then this should be really easy.  If not, you'll just have to explicitly get each forward read, reverse read, and name.  Let's just assume you're downloading from SRA for the sake of this walkthrough.

Here are the SRA identifiers for `PRJNA777294` which is the Plastisphere dataset from the paper:

```
mkdir -p logs/
mkdir -p Fastq/
cat identifiers.list
```


<details open>
<summary>Output:</summary>

```
SRR17458603
SRR17458604
SRR17458605
SRR17458606
SRR17458607
SRR17458608
SRR17458609
SRR17458610
SRR17458611
SRR17458612
SRR17458613
SRR17458614
SRR17458615
SRR17458616
SRR17458617
SRR17458618
SRR17458619
SRR17458620
SRR17458621
SRR17458622
SRR17458623
SRR17458624
SRR17458625
SRR17458626
SRR17458627
SRR17458628
SRR17458629
SRR17458630
SRR17458631
SRR17458632
SRR17458633
SRR17458634
SRR17458635
SRR17458636
SRR17458637
SRR17458638
SRR17458639
SRR17458640
SRR17458641
SRR17458642
SRR17458643
SRR17458644
SRR17458645
SRR17458646
```

</details>

#### 3. Create a directory for the raw Fastq reads and download the sequences from NCBI

**Note:** `kingfisher` is not officially part of the `VEBA` suite and is only provided for convenience.

```
# Set the threads you want to use.  Here we are using 4.
N_JOBS=4

# Go into fastq directory temporarily
cd Fastq/

# Iterate through the identifier list and download each read set
for ID in $(cat ../identifiers.list);
	do kingfisher get -r $ID -m prefetch -t ${N_JOBS}
	done
	
# Gzip the fastq	
pigz -p ${N_JOBS} *.fastq

# Get back out to main directory
cd ..
```

Common `kingfisher` errors: 

* If you get an error related to `prefetch` try changing the `-m` argument (e.g., `-m aws-http`): 

```
(VEBA-preprocess_env) [jespinoz@exp-15-01 Fastq]$ for ID in $(cat ../identifiers.list); do kingfisher get -r $ID -m prefetch; done
09/18/2022 04:04:25 PM INFO: Attempting download method prefetch ..
09/18/2022 04:04:25 PM WARNING: Method prefetch failed: Error was: Command prefetch -o SRR4114636.sra SRR4114636 returned non-zero exit status 127.
STDERR was: b'bash: prefetch: command not found\n'STDOUT was: b''
09/18/2022 04:04:25 PM WARNING: Method prefetch failed
Traceback (most recent call last):
  File "/expanse/projects/jcl110/anaconda3/envs/VEBA-preprocess_env/bin/kingfisher", line 261, in <module>
    main()
  File "/expanse/projects/jcl110/anaconda3/envs/VEBA-preprocess_env/bin/kingfisher", line 241, in main
    extraction_threads = args.extraction_threads,
  File "/expanse/projects/jcl110/anaconda3/envs/VEBA-preprocess_env/lib/python3.7/site-packages/kingfisher/__init__.py", line 234, in download_and_extract
    raise Exception("No more specified download methods, cannot continue")
Exception: No more specified download methods, cannot continue
```

* If SRA-Tools didn't install correctly, you may get this error when converting .sra to .fastq[.gz] files.  If so, just reinstall `sra-tools` via `conda install -c bioconda sra-tools --force-reinstall` in your `VEBA-preprocess_env` environment: 

```
STDERR was: b'bash: fasterq-dump: command not found\n'STDOUT was: b''
```

#### 4. Perform quality/adapter trimming, remove human contamination, and count the ribosomal reads but don't remove them.

Here we are going to count the reads for the human contamination and ribosomal reads but not keep them.  If you wanted to keep the human reads to do some type of human-based study then do `--retain_contaminated_reads 1`.  If you wanted to just do read trimming and not remove any contamination or count anything then you would just leave out the `-x` and `-k` arguments.  For example, if you were using this to process some human reads.  


* ⚠️ If your host is not human then you will need to use a different contamination reference.  See item #22 in the [FAQ](https://github.com/jolespin/veba/blob/main/FAQ.md).

* ⚠️ As of 2022.10.18 *VEBA* has switched from using the "GRCh38 no alt analysis set" to the "CHM13v2.0 telomore-to-telomere" build for human.  If you've installed *VEBA* before this date or are using `v1.0.0` release from [Espinoza et al. 2022](https://bmcbioinformatics.biomedcentral.com/articles/10.1186/s12859-022-04973-8) then you can update with the following code:

```
conda activate VEBA-database_env
wget -v -P ${VEBA_DATABASE} https://genome-idx.s3.amazonaws.com/bt/chm13v2.0.zip
unzip -d ${VEBA_DATABASE}/Contamination/ ${VEBA_DATABASE}/chm13v2.0.zip
rm -rf ${VEBA_DATABASE}/chm13v2.0.zip

# Use this if you want to remove the previous GRCh38 index
rm -rf ${VEBA_DATABASE}/Contamination/grch38/
```

Continuing with the tutorial...just make note of the human index here and swap out GRCh38 for CHM13v2.0 if you decided to update:

```
N_JOBS=4

# Human Bowtie2 index
HUMAN_INDEX=${VEBA_DATABASE}/Contamination/grch38/GCA_000001405.15_GRCh38_no_alt_analysis_set.fna.bowtie_index

# or use this if you have updated from GRCh38 to CHM13v2.0
# HUMAN_INDEX=${VEBA_DATABASE}/Contamination/chm13v2.0/chm13v2.0

# Ribosomal k-mer fasta
RIBOSOMAL_KMERS=${VEBA_DATABASE}/Contamination/kmers/ribokmers.fa.gz

# Iterate through identifiers and run preprocessing
for ID in  $(cat identifiers.list); do

	# Get forward and reverse reads
	R1=Fastq/${ID}_1.fastq.gz
	R2=Fastq/${ID}_2.fastq.gz
	
	# Get a name to use for log files
	N=preprocessing__${ID}
	
	# Remove any preexisting log file
	rm -f logs/${N}.*
	
	# Set up the command (use source from base environment instead of conda because of the `init` issues)
	CMD="source activate VEBA-preprocess_env && preprocess.py -n ${ID} -1 ${R1} -2 ${R2} -p ${N_JOBS} -x ${HUMAN_INDEX} -k ${RIBOSOMAL_KMERS} --retain_contaminated_reads 0 --retain_kmer_hits 0 --retain_non_kmer_hits 0"
	
	# If you have SunGrid engine, do something like this:
	# qsub -o logs/${N}.o -e logs/${N}.e -cwd -N ${N} -j y -pe threaded ${N_JOBS} "${CMD}"
	
	# If you have SLURM engine, do something like this:
	# sbatch -J ${N} -N 1 -c ${N_JOBS} --ntasks-per-node=1 -o logs/${N}.o -e logs/${N}.e --export=ALL -t 12:00:00 --mem=20G --wrap="${CMD}"
	
	done
```
Note: `preprocess.py` is a wrapper around `fastq_preprocessor` which takes in 0 and 1 as False and True, respectively.  The reasoning for this is that I was able to keep the prefix `retain` while setting defaults easier.

It creates the following directory structure where each sample is it's own subdirectory.  Makes globbing much easier:


```
ls veba_output/preprocess/

SRR17458603  SRR17458606  SRR17458609  SRR17458612  SRR17458615  SRR17458618  SRR17458621  SRR17458624  SRR17458627  SRR17458630  SRR17458633  SRR17458636  SRR17458639  SRR17458642  SRR17458645
SRR17458604  SRR17458607  SRR17458610  SRR17458613  SRR17458616  SRR17458619  SRR17458622  SRR17458625  SRR17458628  SRR17458631  SRR17458634  SRR17458637  SRR17458640  SRR17458643  SRR17458646
SRR17458605  SRR17458608  SRR17458611  SRR17458614  SRR17458617  SRR17458620  SRR17458623  SRR17458626  SRR17458629  SRR17458632  SRR17458635  SRR17458638  SRR17458641  SRR17458644
```

The main files in each of the output directories are the following: 

* cleaned_1.fastq.gz - Cleaned and trimmed fastq file (forward)
* cleaned_2.fastq.gz - Cleaned and trimmed fastq file (reverse)
* seqkit_stats.concatenated.tsv - Concatenated read statistics for all intermediate steps (e.g., fastp, bowtie2 removal of contaminated reads if provided, bbduk.sh removal of contaminated reads if provided)

#### 5. If you want some read statistics then merge the dataframes.

```
concatenate_dataframes.py  veba_output/preprocess/*/output/seqkit_stats.concatenated.tsv
```

#### 6. All the reads trimmed and decontaminated reads will be here:
```
veba_output/preprocess/*/output/cleaned_1.fastq.gz
veba_output/preprocess/*/output/cleaned_2.fastq.gz
```

#### Next steps:

Now it's time to assemble these reads, recover genomes from the assemblies, and map reads to produce counts tables.  Please see the next walkthroughs.