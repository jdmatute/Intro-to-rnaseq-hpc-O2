---
title: "Automating an RNA-Seq workflow"
author: "Bob Freeman, Meeta Mistry, Radhika Khetani"
date: "Tuesday, August 22, 2017"
---

## Learning Objectives:

* Automate a workflow by grouping a series of sequential commands into a script
* Modify and submit the workflow script to the cluster

## Automating the analysis path from Sequence reads to Count matrix

Once you have optimized all the tools and parameters using a single sample (likely using an interactive session), you can write a script to run the whole workflow on all the samples in parallel.

This will ensure that you run every sample with the exact same parameters, and will enable you to keep track of all the tools and their versions. In addition, the script is like a lab notebook; in the future, you (or your colleagues) can go back and check the workflow for methods, which enables efficiency and reproducibility.

Before we start with the script, let's check how many cores our interactive session has by using `squeue`. 

```bash
$ squeue -u eCommonsID
```

We need to have an interactive session with 6 cores, if you already have one you are set. If you have a session with fewer cores then `exit` out of your current interactive session and start a new one with `-n 6`.

```bash
$ srun --pty -p interactive -t 0-12:00 -n 6 --mem 8G --reservation=hbc bash
```

### More Flexibility with variables

We can write a shell script that will run on a specific file, but to make it more flexible and efficient we would prefer that it lets us give it an input fastq file when we run the script. To be able to provide an input to any shell script, we need to use **Positional Parameters**.

For example, we can refer to the components of the following command as numbered variables **within** the actual script:

```bash
# * DO NOT RUN *
sh  run_rnaseq.sh  input.fq  input.gtf  12
```

`$0` => run_rnaseq.sh

`$1` => input.fq

`$2` => input.gtf

`$3` => 12

The variables $1, $2, $3,...$9 and so on are **positional parameters** in the context of the shell script, and can be used within the script to refer to the files/number specified on the command line. Basically, the script is written with the expectation that $1 will be a fastq file and $2 will be a GTF file, and so on.

*There can be virtually unlimited numbers of inputs to a shell script, but it is wise to only have a few inputs to avoid errors and confusion when running a script that used positional parameters.*

> [This is an example of a simple script that used the concept of positional parameters and the associated variables](http://steve-parker.org/sh/eg/var3.sh.txt). You should try this script out after the class to get a better handle on positional parameters for shell scripting.

Let's use this new concept in the script we are writing. We want the first positional parameter ($1) to be the name of our fastq file. We could just use the variable `$1` throughout the script to refer to the fastq file, but this variable name is not intuitive, so we want to create a new variable called `fq` and copy the contents of `$1` into it.

```bash
#!/bin/bash/

# initialize a variable with an intuitive name to store the name of the input fastq file

fq=$1
```

> When we set up variables we do not use the `$` before it, but when we *use the variable*, we always have to have the `$` before it. >
>
> For example: 
>
> initializing the `fq` variable => `fq=$1`
>
> using the `fq` variable => `fastqc $fq`

To ensure that all the output files from the workflow are properly named with sample IDs we should extract the "base name" (or sample ID) from the name of the input file.

```
# grab base of filename for naming outputs

base=$(basename $fq .subset.fq)
echo "Sample name is $base"           
```

> **Remember `basename`?**
>
> 1. the `basename` command: this command takes a path or a name and trims away all the information before the last `\` and if you specify the string to clear away at the end, it will do that as well. In this case, if the variable `$fq` contains the path *"~/unix_workshop/rnaseq/raw_data/Mov10_oe_1.subset.fq"*, `basename $fq .subset.fq` will output "Mov10_oe_1".
> 2. to assign this value to the `base` variable, we place the `basename...` command in parentheses and put a `$` outside. This syntax is necessary for assigning the output of a command to a variable.

Next we want to specify how many cores the script should use to run the analysis. This provides us with an easy way to modify the script to run with more or fewer cores without have to replace the number within all commands where cores are specified.

```
# specify the number of cores to use

cores=6
```
Next we'll initialize 2 more variables named `genome` and `gtf`, these will contain the paths to where the reference files are stored. This makes it easier to modify the script for when you want to use a different genome, i.e. you'll just have to change the contents of these variable at the beginning of the script.

```
# directory with genome reference FASTA and index files + name of the gene annotation file

genome=/n/groups/hbctraining/unix_workshop_other/reference_STAR/
gtf=~/unix_workshop/rnaseq/reference_data/chr1-hg19_genes.gtf
```

We'll create output directories, but with the `-p` option. This will make sure that `mkdir` will create the directory only if it does not exist, and it won't throw an error if it does exist.

```
# make all of the output directories
# The -p option means mkdir will create the whole path if it 
# does not exist and refrain from complaining if it does exist

mkdir -p ~/unix_workshop/rnaseq/results/fastqc/
mkdir -p ~/unix_workshop/rnaseq/results/STAR
mkdir -p ~/unix_workshop/rnaseq/results/counts
```

Now that we have already created our output directories, we can now specify variables with the path to those directories both for convenience but also to make it easier to see what is going on in a long command.

```
# set up output filenames and locations

fastqc_out=~/unix_workshop/rnaseq/results/fastqc/
align_out=~/unix_workshop/rnaseq/results/STAR/${base}_
counts_input_bam=~/unix_workshop/rnaseq/results/STAR/${base}_Aligned.sortedByCoord.out.bam
counts=~/unix_workshop/rnaseq/results/counts/${base}_featurecounts.txt
```

### Keeping track of tool versions

All of our variables are now staged. Next, let's make sure all the modules are loaded. This is also a good way to keep track of the versions of tools that you are using in the script:

```
# set up the software environment

module load fastqc/0.11.5
module load gcc/6.2.0  
module load star/2.5.2b
module load samtools/1.3.1
PATH=/n/app/bcbio/tools/bin:$PATH 	# for using featureCounts if not already in $PATH
```

### Preparing for future debugging

In the script, it is a good idea to use `echo` for debugging. `echo` basically displays the string of characters specified within the quotations. When you have strategically place `echo` commands specifying what stage of the analysis is next, in case of failure you can determine the last `echo` statement displayed to troubleshoot the script.

```
echo "Processing file $fq"
```

> You can also use `set -x`:
>
> `set -x` is a debugging tool that will make bash display the command before executing it. In case of an issue with the commands in the shell script, this type of debugging lets you quickly pinpoint the step that is throwing an error. Often, tools will display the error that caused the program to stop running, so keep this in mind for times when you are running into issues where this is not available.
> You can turn this functionality off by saying `set +x`

### Running the tools

```
# Run FastQC and move output to the appropriate folder
fastqc $fq

# Run STAR
STAR --runThreadN $cores --genomeDir $genome --readFilesIn $fq --outFileNamePrefix $align_out --outFilterMultimapNmax 10 --outSAMstrandField intronMotif --outReadsUnmapped Fastx --outSAMtype BAM SortedByCoordinate --outSAMunmapped Within --outSAMattributes NH HI NM MD AS

# Create BAM index
samtools index $counts_input_bam

# Count mapped reads
featureCounts -T $cores -s 2 -a $gtf -o $counts $counts_input_bam
```

### Last addition to the script

It is best practice to have the script **usage** specified at the top any script. This should have information such that when your future self, or a co-worker, uses the script they know what it will do and what input(s) are needed. For our script, we should have the following lines of comments right at the top after `#!/bin/bash/`:

```
# This script takes a fastq file of RNA-Seq data, runs FastQC and outputs a counts file for it.
# USAGE: sh rnaseq_analysis_on_allfiles.sh <name of fastq file>
```

It is okay to specify this everything else is set up, since you will have most clarity about the script only once it is fully done.

### Saving and running script

To transfer the contents of the script to O2, you can copy and paste the contents into a new file using `vim`. 

```bash
$ cd ~/unix_workshop/rnaseq/scripts/

$ vim rnaseq_analysis_on_input_file.sh 
```
> *Alternatively, you can save the script on your computer and transfer it to `~/unix_workshop/rnaseq/scripts/` using FileZilla.*


We should all have an interactive session with 6 cores, so we can run the script as follows:

```bash
$ sh rnaseq_analysis_on_input_file.sh ~/unix_workshop/rnaseq/raw_data/Mov10_oe_1.subset.fq
```

## Running the script to submit jobs in parallel to the SLURM scheduler

The above script will run in an interactive session **one file at a time**. But the whole point of writing this script was to run it on all files at once. How do you think we can do this?

To run the above script **"in serial"** for all of the files on a worker node via the job scheduler, we can create a separate submission script that will need 2 components:

1. **SLURM directives** at the **beginning** of the script. This is so that the scheduler knows what resources we need in order to run our job on the compute node(s).
2. a `for` loop that iterates through and runs the above script for all the fastq files.

Below is what this second script would look like **\[DO NOT RUN THIS\]**:

```
#!/bin/bash

#SBATCH -p medium 		# partition name
#SBATCH -t 0-2:00 		# hours:minutes runlimit after which job will be killed
#SBATCH -n 6 		# number of cores requested -- this needs to be greater than or equal to the number of cores you plan to use to run your job
#SBATCH --job-name STAR_mov10 		# Job name
#SBATCH -o %j.out			# File to which standard out will be written
#SBATCH -e %j.err 		# File to which standard err will be written

# this `for` loop, will take the fastq files as input and run the script for all of them one after the other. 
for fq in ~/unix_workshop/rnaseq/raw_data/*.fq
do
  echo "running analysis on $fq"
  rnaseq_analysis_on_input_file.sh $fq
done
```

**But we don't want to run the analysis on these 6 samples one after the other!** We want to run them "in parallel" as 6 separate jobs. 

**Note:** If you create and run the above script, or something similar to it, i.e. with SLURM directives at the top, you should give the script name `.run` or `.slurm` as the extension. This will make it obvious that it is meant to submit jobs to the SLURM scheduler. 

***
**Exercise**

How would you run the above script?

***

## Parallelizing the analysis for efficiency

Parallelization will save you a lot of time with real (large) datasets. To parallelize our analysis, we will still need to write a second script that will call the original script we just wrote. We will still use a `for` loop, but we will be creating a regular shell script and we will be specifying the SLURM directives differently. 

Use `vim` to start a new shell script called `rnaseq_analysis_on_allfiles-for_slurm.sh`: 

```bash
$ vim rnaseq_analysis_on_allfiles_for-slurm.sh
```

This script loops through the same files as in the previous (demo) script, but the command being submitted within the `for` loop is `sbatch` with SLURM directives specified on the same line:

```bash
#! /bin/bash

for fq in ~/unix_workshop/rnaseq/raw_data/*.fq
do

sbatch -p short -t 0-2:00 -n 6 --job-name rnaseq-workflow --wrap="sh ~/unix_workshop/rnaseq/scripts/rnaseq_analysis_on_input_file.sh $fq"
sleep 1	# wait 1 second between each job submission
  
done
```
> Please note that after the `sbatch` directives the command `sh ~/unix_workshop/rnaseq/scripts/rnaseq_analysis_on_input_file.sh $fq` is in quotes.

What you should see on the output of your screen would be the jobIDs that are returned from the scheduler for each of the jobs that your script submitted.

You can use `squeue -u eCommonsID` to check progress.

Don't forget about the `scancel` command, should something go wrong and you need to cancel your jobs.

> **NOTE:** All job schedulers are similar, but not the same. Once you understand how one works, you can transition to another one without too much trouble. They all have their pros and cons which are considered by the system administrators when picking one for a given HPC environment. 

#### Generating a Count Matrix

The above script will generate separate count files for each sample. Hence, after the script has run, you will have to merge them using `paste` and do some cleanup using `cut` to generate a full count matrix wherein the first column is gene names and the rest of the columns are gene counts in each sample (as shown below).

<img src="../img/count_matrix.png" width=500>

Alternatively, you could remove featureCounts from the original script, and run it after all the jobs finish generating BAM files.

---
*This lesson has been developed by members of the teaching team at the [Harvard Chan Bioinformatics Core (HBC)](http://bioinformatics.sph.harvard.edu/). These are open access materials distributed under the terms of the [Creative Commons Attribution license](https://creativecommons.org/licenses/by/4.0/) (CC BY 4.0), which permits unrestricted use, distribution, and reproduction in any medium, provided the original author and source are credited.*

* *The materials used in this lesson was derived from work that is Copyright © Data Carpentry (http://datacarpentry.org/). 
All Data Carpentry instructional material is made available under the [Creative Commons Attribution license](https://creativecommons.org/licenses/by/4.0/) (CC BY 4.0).*
