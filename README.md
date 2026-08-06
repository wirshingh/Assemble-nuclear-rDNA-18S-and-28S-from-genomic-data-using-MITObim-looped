# Assemble nuclear rDNA (18S and 28S) from genomic data with BBmap and MITObim using a loop
This workflow is in two parts. The first part will convert trimmed forward and reverse reads into single interleaved files uisng BBmap. The second part will use the interleaved data and a user provided seed to assemble nuclear ribosomal sequences using MITObim, rename the final contig files with sample IDs and save them to a new directory. 

## Part 1
## Convert trimmed reads into interleaved format using BBmap
Link to BBmap site - https://sourceforge.net/projects/bbmap/
### File preparation
The trimmed reads file names must end in "_R1_PE_trimmed.fastq.gz" and "_R2_PE_trimmed.fastq.gz" for the script to work. The script will utilize whatever text is in front of either "_R1(orR2)_PE_trimmed.fastq.gz" as the sample name. Alternatively, if your trimmed reads file names end with different text, the job file may also be modified accordingly. 

### Run BBmap on Hydra 
Before running the script, under CONFIGURATION add the following information: 

1. SAMPLEDIR_TRM=

   After the "=", paste the full path to the trimmed sequences directory. 
   Verify that your trimmed reads file names end with the appropriate text.

5. SAMPLEDIR_BASE=

   After the "=", paste the full path the base directory. This is where the results will go.

Save the job file below as "bbmap_interleaved.job", and run the job below on Hydra (qsub bbmap_interleaved.job).

When complete, the interleaved sequence files will be in a directory called "interleaved_sequences".

```
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 2
#$ -q sThC.q
#$ -l mres=6G,h_data=3G,h_vmem=3G
#$ -cwd
#$ -j y
#$ -N bbmap_interleaved
#$ -o bbmap_interleaved.log
#
# ----------------Modules------------------------- #
module load bioinformatics/bbmap
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
#

#============================================================================
# PART 1 - CONFIGURATION
#============================================================================
# Create variable by copying full path to trimmed sequences
SAMPLEDIR_TRM="path to trimmed reads"

# Create variable by copying full path to the base directory. This is where the results will go.
SAMPLEDIR_BASE="path to base directory"

#============================================================================
# PART 2 - CREATE OUTPUT DIRECTORIES
#============================================================================
# In the base directory create a directory where the interleaved sequences will be placed
mkdir -p ${SAMPLEDIR_BASE}/interleaved_sequences

#============================================================================
# PART 3 - RUN BBMAP IN A LOOP
#============================================================================
# Use loop to generate a sample names for each sample and run BBmap 
for GETSAMPLENAME in ${SAMPLEDIR_TRM}/*_R1_PE_trimmed.fastq.gz
do
SAMPLENAME=$(basename "$GETSAMPLENAME" _R1_PE_trimmed.fastq.gz)

reformat.sh \
in1="${SAMPLEDIR_TRM}/${SAMPLENAME}_R1_PE_trimmed.fastq.gz" \
in2="${SAMPLEDIR_TRM}/${SAMPLENAME}_R2_PE_trimmed.fastq.gz" \
out="${SAMPLEDIR_BASE}/interleaved_sequences/${SAMPLENAME}_interleaved.fastq.gz"
done

#
echo = `date` job $JOB_NAME done

```

## Part 2
## Use interleaved sequence data created in Part 1 to assemble nuclear rDNA using MITObim in a loop
link to MITObim - https://github.com/chrishah/MITObim
### File Preparation
Create a fasta file (.fasta) with the seed that will be used to assemble the gene of interest. For nuclear ribosomal genes, partial or complete 18S or 28S sequences may be used. Depending on the taxon, using either 18S or 28S, MITObim may assemble the entire nuclear ribosomal operon, or may just assemble the single gene from the seed. 
 
### Run MITObim on Hydra
#### NOTE: The following commands in the job file must be modified before submitting to Hydra.

Under CONFIGURATION add the following information:

1. SAMPLEDIR_INT=

   Paste the full path to the directory that contains the interleaved sequences.

2. SAMPLEDIR_BASE=

   Paste the full path the project's base directory. This is where the results will go.

3. SEED=

   Paste the full path to the .fasta file that will be the seed (e.g., 18S or 28S sequence)

4. ITERATIONS=

   Enter the number of iterations that will be used for each assembly. The default is set to "4".

   #### Note: The number of iterations will effect the size of your final contig and the time used for assembly. Usually, 4 iterations is sufficient for complete 18S or 28S. However, more or fewer iterations can be used depending on the taxon, data and user needs.

Part 3 of the job will copy, and rename with Sample IDs, all final fasta contings (e.g., 18S and/or 28S) and log files from their output directories to a directory named 'mitobim_final_renamed_contigs_and_logs'.

All of the raw final results for each sample will be a separate directory labeled "mitobim_results".

To run the MITObim job - after completing the CONFIGURATION section, save the modified job below as "mitobim_loop.job" and submit the job on Hydra (qsub mitobim_loop.job).


```
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 6
#$ -q mThC.q
#$ -l mres=24G,h_data=4G,h_vmem=4G
#$ -cwd
#$ -j y
#$ -N mitobim_loop.job
#$ -o mitobim.log
#
# ----------------Modules------------------------- #
module load ~/modulefiles/miniconda
source activate mitobim
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#

#============================================================================
# CONFIGURATION
#============================================================================

# Set sample directory path to interleaved sequences
SAMPLEDIR_INT="full path to interleaved sequences"

# Set sample directory to the base directory. Where results will go.
SAMPLEDIR_BASE="full path to base directory"

# Set path to the seed file in .fasta format
SEED="full path to seed file"

# Set number of iterations. Default number is set to 4.
ITERATIONS="4"

#============================================================================
# PART 1 - CREATE OUTPUT DIRECTORIES
#============================================================================

# Create directory where results results will be copied
mkdir -p ${SAMPLEDIR_BASE}/mitobim_results

# Create directory for final renamed contigs and log files
mkdir -p ${SAMPLEDIR_BASE}/mitobim_final_renamed_contigs_and_logs 

#============================================================================
# PART 2 - RUN MITOBIM IN A LOOP AND RENAME OUTPUT FILES
#============================================================================

# Use loop to generate sample names for each sample  
for GETSAMPLENAME in ${SAMPLEDIR_INT}/*_interleaved.fastq.gz
do
SAMPLENAME=$(basename "$GETSAMPLENAME" _interleaved.fastq.gz)

# Create sample-specific directories in the mitobim_results directory
mkdir -p "${SAMPLEDIR_BASE}/mitobim_results/${SAMPLENAME}_mitobim"
cd "${SAMPLEDIR_BASE}/mitobim_results/${SAMPLENAME}_mitobim" || exit

# Prints the name of each file/directory being processed
echo "Processing file: ${SAMPLENAME}"

# Run mitobim from sample-specific directories in a loop
MITObim.pl \
-sample "${SAMPLENAME}" \
-ref mitobim \
--readpool "${SAMPLEDIR_INT}"/${SAMPLENAME}_interleaved.fastq.gz \
--quick "${SEED}" \
--end "${ITERATIONS}" --pair --clean &> "log_${SAMPLENAME}"

#============================================================================
# PART 3 - RENAME OUTPUT FILES
#============================================================================
   
# Copies the final contig and log files from the mitobim run and pastes them in new directory
cp ${SAMPLEDIR_BASE}/mitobim_results/${SAMPLENAME}_mitobim/iteration*/*_noIUPAC.fasta ${SAMPLEDIR_BASE}/mitobim_final_renamed_contigs_and_logs
cp ${SAMPLEDIR_BASE}/mitobim_results/${SAMPLENAME}_mitobim/log_* ${SAMPLEDIR_BASE}/mitobim_final_renamed_contigs_and_logs

# Iterate over files in the directory and obtain the filename
for mitobim_internalrename in ${SAMPLEDIR_BASE}/mitobim_final_renamed_contigs_and_logs/"${SAMPLENAME}"*; do
    # Extract filename without extension
    filename=$(basename "$mitobim_internalrename" _noIUPAC.fasta)

    # Perform internal rename substitution and deletion using awk
    awk -v fname="$filename" -F '>' '{if (NF > 1) print $1 ">" fname; else print}' "$mitobim_internalrename" > tmpfile && mv tmpfile "$mitobim_internalrename"
	done
done

echo "Final renamed contigs and logs should be in directory named 'mitobim_final_renamed_contigs_and_logs'"
echo "Done"

#
echo = `date` job $JOB_NAME done


```


