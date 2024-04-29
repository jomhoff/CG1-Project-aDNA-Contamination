# **Workflow**

## **Introduction**

This workflow is designed to accession Sequence Read Archives (SRAs) of target species from target studies on NCBI, then run the SRAs through Derrick Wood's [Kraken2](https://github.com/DerrickWood/kraken2.git), which is a taxonomic sequence classifier that assigns taxonomic labels to DNA sequences. The goal of this workflow is to 
identify potential biological contaminants in SRAs. For more information on Kraken2, visit the [Kraken2 Manual](docs/MANUAL.markdown).

## **Sourcing SRAs**

To find a project with the SRAs of the desired species:
1. Search the genus on [BioProject](https://www.ncbi.nlm.nih.gov/bioproject?cmd=Retrieve&dopt=Overview&list_uids=192.) 
2. Open up the project page and Google it (or any citations listed) to try to find the original citation it is paired with. 

To get a list of SRA accessions:
1. If the year is correct, click on its SRA experiments (should be under 'Number of Links')
2. Click Send results to Run selector 
3. Under Download, click Accession List. This will send all the SRAs associated with the BioProject to a .txt file
4. Upload the .txt file to the computational server in an accessible location

## **Kraken2**

Installing Kraken2
```
conda create -n xx #create a conda environment, replace xx with name
conda install bioconda::kraken2
```

Building a Kraken database
```
cd /path/to/where/you/want/kraken/database
kraken2-build --db Silva --special silva --clean  #use --clean to save lots of storage
```
### **Ok, let's use Kraken now**
**Prefetch command**: this command takes the path to all of the SRA files and preps the SRAs for running with Kraken2.
- Written by M. Forcellati, adapted by J. Hoffman

```
# Accession all of the SRA files
user="" #add your user. Ex: user="jhoffman1"
ID="" #add your specimen ID
downloaddir=/nas5/$user/CG1/raw_reads/

# List of every pair-end read file on NCBI SRA referred to mammoth genus
filelist=/nas5/$user/CG1/scripts/$ID.txt

# Change number for every "chunk" of sample analyzed
	#replace x with number of samples
for sample in {1..x};
do
        specimen=$(sed "${sample}q;d" $filelist | awk '{print $1 }')
        mkdir -p $downloaddir/${specimen}

        cd $downloaddir

        prefetch $specimen --output-directory $downloaddir
done

```

**Full Pipeline**: this is the pipeline for running Kraken2 on the SRAs
- Written by M. Forcellati, adapted by J. Hoffman

```
#!/bin/bash
#PBS -q batch
#PBS -S /bin/bash
#PBS -l ncpus=24
#PBS -l mem=32G
#PBS -l walltime=20:00:00
#PBS -o /nas5/user/CG1/outerr/fullYOURFIlE.out
#PBS -e /nas5/user/CG1/outerr/fullYOURFIlE.err
#PBS -J 1-x

######################
# Run full pipeline. #
######################


##############
# Initialize #
##############
#enter your user. Ex: "jhoffman1"
user=""
ID=""

#activate conda
conda activate /nas5/$user/miniconda3/envs/CG1
# List of files is necessary for the interactive job sample specificity
	#replace with directory to your filelist
filelist=/nas5/$user/CG1/scripts/$ID.txt

# Directory for outputting trimmed reads [ temporary ]
	#change to your output directory
OutputDirTrim=/nas5/$user/CG1/trimmed_reads

# Source of Prefetched reads
	#change directory for the output from the PrefetchCommand.sh 
DataDir=/nas5/$user/CG1/raw_reads

# Initialize your bash profile.
source ~/.bash_profile

# kraken out
	#change to your own output 
OUTDIR=/nas5/$user/CG1/analysis/kraken_run

# final kraken out - change to project folder when done
	#change this to your project folder (the SRA you're working on)
FINALOUT=/nas5/$user/CG1/results/$ID

##############################
# fasterqdump and TrimGalore #
 # Get specimen name
        specimen=$(sed "${PBS_ARRAY_INDEX}q;d" $filelist | awk '{print $1 }')
        cd $DataDir/$specimen

        # Run fasterqdump to get full SRA
        fasterq-dump $DataDir/$specimen

        # Make an output directory
         mkdir -p $OutputDirTrim/$specimen

        # Run trimming
          trim_galore --quality 20 --paired --retain_unpaired --gzip *_1* *_2* --output_dir $OutputDirTrim/$specimen

##########
# kraken #
##########

mkdir -p $OUTDIR/$specimen

cd $OutputDirTrim/$specimen

kraken2 --db $DBNAME  --threads 24 --output $OUTDIR/$specimen/${specimen}.out.txt  --report-minimizer-data --use-names --paired --gzip-compressed --classified-out $OUTDIR/$specimen/${specimen}#.fq *_val_1.fq.gz *_val_2.fq.gz

# Output final file
cp $OUTDIR/$specimen/${specimen}.classified.out.txt ${FINALOUT}/${specimen}.classified.out.txt

cp $OUTDIR/$specimen/${specimen}.out.txt ${FINALOUT}/${specimen}.out.txt


# Clean up - run once this code actually works.
rm -rif $DataDir/$specimen
rm -rif $OutputDirTrim/$specimen
rm -rif $TRIMDIR/${specimen}
rm -rif $OUTDIR/${specimen}

~
```

## **Visualizing Results**





