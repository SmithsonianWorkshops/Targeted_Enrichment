# Running phyluce on Hydra
---

Tutorial modified from Brant Faircloth's https://phyluce.readthedocs.io/en/latest/tutorials/tutorial-1.html for Phyluce 1.7. It has been tailored to the Smithsonian HPC cluster, Hydra, by the SIBG bioinformatics group.

You will generate 17 job files for Hydra during this tutorial that encompass these major steps:  
1. Get the data  
2. Count the read data  
3. Clean the read data  
4. Assemble the data  
5. Assembly QC  
6. Finding UCE loci  
7. Extracting UCE loci  
8. Exploding the monolithic FASTA file  
9. Aligning UCE loci  
10. Alignment cleaning  
11. Final data matrices  
12. Preparing data for RAxML and ExaML  

Here, we will process raw Illumina UCE data for 4 taxa that were enriched with the 5000 UCE Tetrapod probe set:

1. *Mus musculus* (PE100)
2. *Anolis carolinensis* (PE100)
3. *Alligator mississippiensis* (PE150)
4. *Gallus gallus* (PE250)

### 1. Get the data and setup directories
First we will get the tutorial data.

* log in to Hydra (If you need assistance with Hydra, check the [support wiki](https://confluence.si.edu/display/HPC/High+Performance+Computing)).
* change to your directory  
    + hint: `cd /pool/genomics/USERNAME` where *USERNAME* is your Hydra login name
* create a project directory  
```mkdir uce-tutorial```
* change to that directory  
```cd uce-tutorial```
* make subdirectories for job files and log files
```mkdir job_files log_files```

If you are running this tutorial after the workshop, see the end of the document for instructions in order to download the data using wget. Because there are so many of us working on the login node at once today, we will copy the data from ```/data/genomics/tutorial_data```

* make a directory to hold the raw data  
```mkdir raw-fastq```
* change to the directory we just created  
```cd raw-fastq```
* copy the data to your working directory using ```cp```. The data are here: ```/data/genomics/tutorial_data/fastq.zip```  
* unzip the fastq data  
```unzip fastq.zip```
* delete the zip file  
```rm fastq.zip```

##### How many files do you see in ```raw-fastq```?

### 2. Count the read data

This step is not required to process UCEs, but it allows you to count the number of reads for each taxon. This step will be completed using standard Unix tools: echo, gunzip, wc (word count), awk and we will need our first job file.

* **JOB FILE #1:** Counting read data (it is best practice to use a job file, even for a trivial task like this).
    + hint: use the QSub Generator: https://hydra-adm01.si.edu/tools/QSubGen/
    		+ *Remember Chrome works best with this.*
    + **CPU time:** short *(we will be using short for all job files in this tutorial)*
    + **memory:** 2GB
    + **PE:** serial
    + **shell:** sh *(use for all job files in the tutorial)*
    + **modules:** none
    + **command:**
    ```
    cd ../raw-fastq
    for i in *R1*.fastq.gz;
    do echo $i;
    gunzip -c $i | wc -l | awk '{print $1/4}';
    done
    ```
    + **job name:** countreads *(or name of your choice)*
    + **log file name:** ../log_files/countreads.log *(The log file is being directed into the `log_files` directory)
    + **change to cwd:** Checked *(keep checked for all job files)*
    + **join stderr & stdout** Checked *(Keep checked for all job files)*
    + hint: upload your job file using ```scp``` from your local machine into the `job_files` directory. See [here](https://confluence.si.edu/display/HPC/Disk+Space+and+Disk+Usage) and [here](https://confluence.si.edu/display/HPC/Transferring+files+to+or+from+Hydra) on the Hydra wiki for more info.
    + hint: cd into the `job_files` directory and submit the job on Hydra using ```qsub```

Here is a sample job file:  
```
    # /bin/sh
    # ----------------Parameters---------------------- #
    #$ -S /bin/sh
    #$ -q sThC.q
    #$ -l mres=2G,h_data=2G,h_vmem=2G
    #$ -cwd
    #$ -j y
    #$ -N countreads
    #$ -o ../log_files/countreads.log
    #
    # ----------------Modules------------------------- #
    #
    # ----------------Your Commands------------------- #
    #
    echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
    #
    cd ../raw-fastq
    for i in *R1*.fastq.gz;
    do echo $i;
    gunzip -c $i | wc -l | awk '{print $1/4}';
    done
    #
    echo = `date` job $JOB_NAME done
```  

The log file, `countreads.log`, will be in your log_files directory. Here's what should be in the log after your job completes:

```
Alligator_mississippiensis_GGAGCTATGG_L001_R1.fastq.gz
1750000
Anolis_carolinensis_GGCGAAGGTT_L001_R1.fastq.gz
1874362
Gallus_gallus_TTCTCCTTCA_L001_R1.fastq.gz
376559
Mus_musculus_CTACAACGGC_L001_R1.fastq.gz
1298196
```

### 3. Clean the read data
These data are raw, so we need to trim adapters and low quality reads before assembly. There are many tools for this. We have modified phyluce's illumiprocessor to use Trim Galore! instead of Trimmomatic. Trim Galore! has the needed functionality but behaves better on Hydra (i.e. does not use Java).

* Now change back to the ```uce-tutorial``` directory with the ```cd ..``` command.

* You will first need to copy a configuration file called `illumiprocessor.conf` from `/data/genomics/tutorial_data` to your `uce-tutorial` directory.
	+ hint: use ```cp```.
	+ Make sure you've changed to the ```uce-tutorial``` directory from ```raw-fastq``` with the command ```cd ..``` before copying the file.

* Note that in most of the steps below there is an ```--output``` argument which is the name of an output file or directory that is created by the command. When you are working on your own analyses with real data consider the naming so that the contents will be well described.

* **JOB FILE #2:** illumiprocessor  
    + hint: your raw reads are in ```raw-fastq``` but you want to run illumiprocessor from ```uce-tutorial```
    + **PE:** multi-thread 2 *this will use two CPUs on a single compute node*
    + **memory:** 2 GB
    + **modules:** bioinformatics/illumiprocessor
    + **command:** (Note: the arguments should start on the same line as the command. The '\' in the arguments allows them to span multiple lines.)
        ```
        cd ..
        illumiprocessor \
          --input raw-fastq \
          --output clean-fastq \
          --config illumiprocessor.conf \
          --paired \
          --cores $NSLOTS
        ```
    + **job name:** illumiprocessor *Use the name of the phyluce command as the job name for all the remaining job files*
    + **log file name:** `../log_files/illumiprocessor.log`


Here is a sample job file:
```
	# /bin/sh
	# ----------------Parameters---------------------- #
	#$ -S /bin/sh
	#$ -pe mthread 2
	#$ -q sThC.q
	#$ -l mres=4G,h_data=2G,h_vmem=2G
	#$ -cwd
	#$ -j y
	#$ -N illumiprocessor
	#$ -o ../log_files/illumiprocessor.log
	#
	# ----------------Modules------------------------- #
	module load bioinformatics/illumiprocessor
	#
	# ----------------Your Commands------------------- #
	#
	echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
	echo + NSLOTS = $NSLOTS
	#
	cd ..
	illumiprocessor \
	--input raw-fastq \
	--output clean-fastq \
	--config illumiprocessor.conf \
	--paired \
	--cores $NSLOTS
	#
	echo = `date` job $JOB_NAME done
```

* Submit and check the log file (located in `log_files`) and the directory ```clean-fastq``` for results. Your cleaned files will be in ```clean-fastq/<species_code>/split-adapter-quality-trimmed```
* *Note: this job will take 5-10 minutes to run. You can use this time to read ahead and check the ```clean-fastq``` directory,*
* **Troubleshooting:** With some Phyluce steps if you need to restart a command you will get an error that the output directory already exists. You will see a line like this in your log file: ```[WARNING] Output directory exists, REMOVE [Y/n]? Traceback (most recent call last):...```
    * Remove the offending directory and re-submit with `qsub`.
* It's a good idea to check your output files. You can use this command to list the contents of all the split-adapter-quality-trimmed directories in clean-fastq/:

```
$ ls -lh clean-fastq/*/split-adapter-quality-trimmed

clean-fastq/alligator_mississippiensis/split-adapter-quality-trimmed:
total 290624
-rw-rw-r-- 1 user user 143M Jul 12 10:50 alligator_mississippiensis-READ1.fastq.gz
-rw-rw-r-- 1 user user 139M Jul 12 10:50 alligator_mississippiensis-READ2.fastq.gz
-rw-rw-r-- 1 user user 2.8M Jul 12 10:50 alligator_mississippiensis-READ-singleton.fastq.gz
.
.
.
```

You should see three files per species code: `-READ1.fastq.gz` `-READ2.fastq.gz` `-READ-singleton.fastq.gz`. The size in bytes of each file is listed, these should all be larger than 0 (which would be an empty file). In the above example, the file sizes are 143MB, 139MB and 2.8MB.

### 4. Assemble the data
We will use Spades to assemble the data into contigs. There will be a separate Spades run for each sample in your dataset. This is the most time consuming computer intensive portion of the pipeline. For today's tutorial, we will not have time to complete the assemblies.

Note: Prior to Phyluce 1.7 Trinity was supported as an assembler. This is no longer supported by Phyluce 1.7.

* Copy the directory with completed assemblies: ```/data/genomics/tutorial_data/spades-assemblies``` to your ```uce-tutorial``` directory.  
* hint: use ```cp -r```.  
* Find the contigs!
* **Skip to the next section (#5).** If you have extra time now, feel free to make the Spades job file but wait to submit it until later.

* Running the Spades assembly (TO TRY AT A LATER DATE):  
    + You need a configuration file to run Spades within phyluce. Create a file called ```assembly.conf``` in ```uce-tutorial```.
    + hint: change directory to ```uce-tutorial``` and use ```nano``` to create the file  
    + The file has one line for each sample starting with the sample ID and then the full path to the directory containing the cleaned reads.
    + contents of the new assembly.conf file (hint: you will need to change `USERNAME` to your Hydra user name):  
    ```
    [samples]
    alligator_mississippiensis:/pool/genomics/USERNAME/uce-tutorial/clean-fastq/alligator_mississippiensis/split-adapter-quality-trimmed/
    anolis_carolinensis:/pool/genomics/USERNAME/uce-tutorial/clean-fastq/anolis_carolinensis/split-adapter-quality-trimmed/
    gallus_gallus:/pool/genomics/USERNAME/uce-tutorial/clean-fastq/gallus_gallus/split-adapter-quality-trimmed/
    mus_musculus:/pool/genomics/USERNAME/uce-tutorial/clean-fastq/mus_musculus/split-adapter-quality-trimmed/
    ```  

* **JOB FILE #3:** Spades
    + **CPU time** short or medium (depending on number of samples)
    + **PE:** mthread 12
    + **memory:** 6 GB (must have more than 40 GB total for this - here we are specifying 12 X 6 = 72 GB total RAM)
    + **modules:** bioinformatics/phyluce/1.7.1
    + **commands:**  ``````  
        + **arguments:**  
        ```
        cd ..
        phyluce_assembly_assemblo_spades \
          --conf assembly.conf \
          --output spades-assemblies \
          --cores $NSLOTS \
          --memory 72
        ```
TODO: add note about `72`
TODO: add note about manually specifying to phyluce 1.7.1

Here is a sample job file:
```
  # /bin/sh
  # ----------------Parameters---------------------- #
  #$ -S /bin/sh
  #$ -pe mthread 12
  #$ -q sThC.q
  #$ -l mres=72G,h_data=6G,h_vmem=6G
  #$ -cwd
  #$ -j y
  #$ -N phyluce_assembly_assemblo_spades
  #$ -o ../log_files/phyluce_assembly_assemblo_spades.log
  #
  # ----------------Modules------------------------- #
  module load bioinformatics/phyluce/1.7.0
  #
  # ----------------Your Commands------------------- #
  #
  echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
  echo + NSLOTS = $NSLOTS
  #
  cd ..
  phyluce_assembly_assemblo_spades \
    --conf assembly.conf \
    --output spades-assemblies \
    --cores $NSLOTS \
    --memory 72
  #
  echo = `date` job $JOB_NAME done
```

### 5. Assembly QC
Let's check to see how well the assemblies worked.

* Make sure your current working directory is ```uce-tutorial```

* **JOB FILE #4:** Get FASTA lengths
    + **PE:** serial  
    + **memory:** 6 GB  
    + **modules:** bioinformatics/phyluce/1.7.1
    + **command:**  
    ```
    for i in spades-assemblies/contigs/*.fasta;
    do phyluce_assembly_get_fasta_lengths --input $i --csv;
    done
    ```

* Check the log file for output similar to below (header not included):  
```
samples,contigs,total bp,mean length,95 CI length,min length,max length,median legnth,contigs >1kb   
alligator_mississippiensis.contigs.fasta,48745,11542663,236.79686121653504,1.267636202104886,56,7554,207.0,2020
anolis_carolinensis.contigs.fasta,3608,1011309,280.29628603104214,5.080088180025299,56,4994,142.0,38
gallus_gallus.contigs.fasta,13593,7386751,543.4231589788861,2.641667797243644,56,5441,409.0,1608
mus_musculus.contigs.fasta,11663,1758513,150.77707279430678,2.1298863180657035,56,4551,70.0,225
```

### 6. Finding UCE loci
Now we want to run ```lastz``` to match contigs to the UCE probe set and to remove duplicates. The search matches are stored in a sqlite database.

* sqlite is a database system that phyluce uses to store information about the contigs such as presence/absence. Phyluce scripts access this data and advanced users can access this database directly.

* [lastz](https://github.com/lastz/lastz) is a pairwise aligner that originally written to make whole chromosome alignments. It is used in several parts of the Phyluce's pipelines including this one of aligning probe sequence to contigs.

* Before we locate UCE loci (in other words, match your contigs to the UCE probes), you need to get the probe set used for the enrichments:   
	+ copy ```uce-5k-probes.fasta``` from ```/data/genomics/tutorial_data``` to your ```uce-tutorial``` directory

* **JOB FILE #5:** Match contigs to probes:
    + **memory:** 6 GB  
    + **modules:** bioinformatics/phyluce/1.7.1
    + **command:** ```phyluce_assembly_match_contigs_to_probes```
        + **arguments:**
        ```
        --contigs spades-assemblies/contigs \
        --probes uce-5k-probes.fasta \
        --output uce-search-results
        ```  
* The directory ```uce-search-results``` is created with the results.
    * *If you need to resubmit this command, delete this directory first or you'll receive an error message in your log file.*
* Look at the log file, ```phyluce_assembly_match_contigs_to_probes.log```, to see how many unique and duplicate matches and how many loci were removed for each taxon.
* Here is an example output:

```
alligator_mississippiensis: 4587 (9.41%) uniques of 48745 contigs, 0 dupe probe matches, 64 UCE loci removed for matching multiple contigs, 344 contigs removed for matching multiple UCE loci
```  
* To reduce the likelihood of paralogous loci being included, Phyluce removes any loci for a sample where there is more than contig that matches.

* If you execute `ls uce-search-results` you'll see that there is a `.lastz` file for each taxon, this is a text file with the raw lastz results that phyluce reads to create a database `probe.matches.sqlite` that has all the taxa and the UCE matched for each taxon. Phyluce will read that database in subsequent steps, but for more advanced work you can also view it manually using these instructions: https://phyluce.readthedocs.io/en/latest/daily-use/daily-use-3-uce-processing.html#database

### 7. Extracting UCE loci
Now that we have located UCE loci, we need to determine which taxa we want in our analysis, create a list of those taxa, and also a list of which UCE loci we enriched in each taxon (the “data matrix configuration file”). We will then use this list to extract FASTA data for each taxon for each UCE locus.

* First, we need to decide which taxa we want in our “taxon set." Create a configuration file.

    + hint: use ```nano``` to create a file called ```taxon-set.conf``` in ```uce-tutorial``` listing the taxa you want to include like so:

```
[all]
alligator_mississippiensis
anolis_carolinensis
gallus_gallus
mus_musculus
```
    + In a large study, you may have different taxa sets that use different subsets of the full taxa set.

* Now that we have the list of taxa we will be using, we have phyluce query the database created in the last step `uce-search-results/probe.matches.sqlite` and output every UCE locus found in any of our taxa. At this point it doesn't matter if the locus was only found in one taxon or all four. In a subsequent step (`phyluce_align_get_only_loci_with_min_taxa`) we will filter loci that have a minimum number of taxa.
    + Make sure you current directory is ```uce-tutorial```
    + ```mkdir -p taxon-sets/all```
        + We will put the analysis for this taxon set in the subdirectory `taxon-sets/all`
        + Why `mkdir -p`? The `-p` stands for "parents." The `mkdir` command will create the parent folder (`taxon-sets`) if it doesn't exist already.

* **JOB FILE #6:** Get match counts:
    + **PE:** serial
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.7.1
    + **command:** ```phyluce_assembly_get_match_counts```
        + **arguments:**
        ```
        --locus-db uce-search-results/probe.matches.sqlite \
        --taxon-list-config taxon-set.conf \
        --taxon-group 'all' \
        --incomplete-matrix \
        --output taxon-sets/all/all-taxa-incomplete.conf
        ```  

* Submit the job and check in ```taxon-sets/all``` to see if the file ```all-taxa-incomplete.conf``` is there.

* Now, that we have the taxa and loci list listed in `all-taxa-incomplete.conf`, we need to get the actual sequence data. This is still stored in the Spades contig files that the assembler created for us. The `probe.matches.sqlite` database has the mapping of a UCE name for a taxa to its contig name from the assembler.
* Change to the ```taxon-sets/all``` directory: if you're not there already ```cd taxon-sets/all```
* Make a log directory to hold our log files: ```mkdir log```

* **JOB FILE #7:** get FASTA data for taxa in our taxon set
    + **PE:** serial
    + **memory:** 6 GB
    + **modules:** bioinformatics/phyluce/1.7.1  
    + **command:** ```phyluce_assembly_get_fastas_from_match_counts```  
        + **arguments:**   
        ```
        --contigs ../../spades-assemblies/contigs \
        --locus-db ../../uce-search-results/probe.matches.sqlite \
        --match-count-output all-taxa-incomplete.conf \
        --output all-taxa-incomplete.fasta \
        --incomplete-matrix all-taxa-incomplete.incomplete \
        --log-path log
        ```  

* The extracted FASTA data this command creates are in a monolithic FASTA file (all loci for all organisms) named ```all-taxa-incomplete.fasta``` - **find it!**
    + Phyluce has changed the fasta headers from the original Spades output to a nicely formatted version with the taxon code and UCE ID.

### 8. Exploding the monolithic FASTA file (OPTIONAL- these checks are not required to proceed with the pipeline)
One way we will be looking at our sequence data is by each taxon separately. We take the large, monolithic, fasta file `all-taxa-incomplete.fasta` and "explode" it into a separate fasta file for each taxon.

* Make sure your current working directory is ```uce-tutorial/taxon-sets/all```

* **JOB FILE #8:** explode the monolithic FASTA file
    + **PE:** serial
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.7.1  
    + **command:** ```phyluce_assembly_explode_get_fastas_file```  
        + **arguments:**  
        ```
        --input all-taxa-incomplete.fasta \
        --output exploded-fastas \
        --by-taxon
        ```
* ```phyluce_assembly_explode_get_fastas_file``` created a directory ```exploded-fastas```, take a look at the fasta files.

* Make sure your current working directory is ```uce-tutorial/taxon-sets/all```
* **JOB FILE #9:** get summary stats on the FASTAs
    + **PE:** serial
    + **memory:** 6 GB
    + **modules:** bioinformatics/phyluce/1.7.1  
    + **command:**
    ```
    for i in exploded-fastas/*.fasta;
    do phyluce_assembly_get_fasta_lengths --input $i --csv;
    done
    ```

* Check your log file for output like this (note the values may not be identical to this example and there will not be a header line):

```
samples,contigs,total bp,mean length,95 CI length,min length,max length,median legnth,contigs >1kb
alligator-mississippiensis.unaligned.fasta,4071,2695452,662.110537951,3.29387421188,224,2579,674.0,165
anolis-carolinensis.unaligned.fasta,685,387996,566.417518248,9.17982752242,224,1039,537.0,6
gallus-gallus.unaligned.fasta,3885,2773870,713.994851995,3.80263831483,224,1594,729.0,443
mus-musculus.unaligned.fasta,729,516360,708.312757202,10.3885688238,224,1150,809.0,101
```


### 9. Aligning UCE loci

In the next step we will align each locus using MAFFT. Following this we will look at Phyluce's different aligning trimming options.

We will be using the ```phyluce_align_seqcap_align``` script. It aligns each locus and by default also does edge trimming (described later).
We're going to disable that trimming by adding the `--no-trim` option. On large datasets aligning can be computationaly intensive. By disabling
trimming at this point you can try different trimming options without having to rerun the mafft alignments.


* make sure you are in the correct directory: ```uce-tutorial/taxon-sets/all```

* **JOB FILE #10:** align WITHOUT edge-trimming
    + **PE:** mthread 4
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.7.1  
    + **command:** ```phyluce_align_seqcap_align```  
        + **arguments:**  
        ```
        --fasta all-taxa-incomplete.fasta \
        --output mafft-fasta-no-trim \
        --taxa 4 \
        --aligner mafft \
        --no-trim \
        --cores $NSLOTS \
        --incomplete-matrix \
        --output-format fasta \
        --log-path log
        ```  

* output alignments are in ```uce-tutorial/taxon-sets/all/mafft-fasta-no-trim```

* We can output summary stats for these alignments by running the following program:

* **JOB FILE #11:** get alignment summary data
    + **PE:** serial
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.7.1  
    + **command:** ```phyluce_align_get_align_summary_data```  
        + **arguments:**  
        ```
        --alignments mafft-nexus-edge-trimmed \
        --cores $NSLOTS \
        --log-path log
        ```
* **Inspect the summary information that will be in the log file.**

* make sure we are in the correct directory: ```uce-tutorial/taxon-sets/all```  

Trimming:
* **Edge trimming:** The starting, unaligned, contigs are different lengths as a result of differnces in the sequence capture, sequencing and assembly processes (as well as taxonomic differences). The edge trimmer will lop off jagged ends of sequences where few taxa have data to leave a core region that is well represented by taxa. The Phyluce script `phyluce_align_get_trimmed_alignments_from_untrimmed` will do this for our unaligned sequences. It uses several parameters to adjust how much of the edges will be trimmed as well as a minimum length of the resulting alignment is required to retain the alignment in the study. We will be using the program defaults, but these parameters can be adjust to be more or less conservative in trimming.

* **Internal trimming:** This process removes sites from an alignment that are gaps for many of the taxa. It is called internal trimming Because
the sites removed can be from anywhere in the alignment not just the ends. This process is helpful for loci that contain introns with highly
variable lengths and sequence diversity. The Phyluce scirpt `phyluce_align_get_gblocks_trimmed_alignments_from_untrimmed` runs the external program Gblocks on our alignments. We will again be using the default trimming parameters, but these can be tuned to be more or less conservative in the trimming.

* **JOB FILE #12:** Edge trim the aligned sequences
    + **PE:** multi-thread 4
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.7.1  
    + **command:** ```phyluce_align_get_trimmed_alignments_from_untrimmed```  
       + **arguments:**
       ```
       ```

* When you look at the log, the ```.``` values that you see represent loci that were aligned and successfully trimmed. Any ```X``` values that you see represent loci that were removed because trimming reduced their length to effectively nothing.

* output alignments are in ```uce-tutorial/taxon-sets/all/...```

* Now, we are going to trim these loci using Gblocks.

* **JOB FILE #13:** internal trim with Gblocks
    + **PE:** multi-thread 2
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.7.1  
    + **command:** ```phyluce_align_get_gblocks_trimmed_alignments_from_untrimmed```  
        + **arguments:**  
        ```
        --alignments mafft-nexus-internal-trimmed \
        --output mafft-nexus-internal-trimmed-gblocks \
        --cores $NSLOTS \
        --log log
        ```

* As above, when you look at the log, the ```.``` values that you see represent loci that were aligned and successfully trimmed. Any ```X``` values that you see represent loci that were removed because trimming reduced their length to effectively nothing.

* output alignments are in ```uce-tutorial/taxon-sets/all/mafft-nexus-internal-trimmed-gblocks```

* We can output summary stats for these alignments by running the following program:

* **JOB FILE #14:** get alignment summary data
    + **PE:** serial
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.7.1  
    + **command:** ```phyluce_align_get_align_summary_data```  
        + **arguments:**  
        ```
        --alignments mafft-nexus-internal-trimmed-gblocks \
        --cores $NSLOTS \
        --log-path log
        ```
* **Inspect the summary information that will be in the log file.**

### 10. Alignment cleaning
Each alignment now contains the locus name along with the taxon name. This is not what we want downstream, so we need to clean our alignments. For the remainder of this tutorial, we will work with the Gblocks trimmed alignments, so we will clean those alignments:

* Make sure we are in the correct directory: ```uce-tutorial/taxon-sets/all```

* **JOB FILE #15:** clean alignments
    + **PE:** multi-thread 2
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.7.1  
    + **command:** ```phyluce_align_remove_locus_name_from_nexus_lines```  
        + **arguments:**  
        ```
        --alignments mafft-nexus-internal-trimmed-gblocks \
        --output mafft-nexus-internal-trimmed-gblocks-clean \
        --cores $NSLOTS \
        --log-path log
        ```   

* Now, if you look at the alignments, you will see that the locus names are removed. We’re ready to generate our final data matrices.

### 11. Final data matrices
To create a 75% data matrix (i.e. 25% or less missing), run the following. Notice that the integer following –taxa is the total number of organisms in the study.

* Make sure we are in the correct directory: ```uce-tutorial/taxon-sets/all```

* **JOB FILE #16:** create a 75% data matrix
    + **PE:** multi-thread 2
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.7.1  
    + **command:** ```phyluce_align_get_only_loci_with_min_taxa```  
        + **arguments:**  
        ```
        --alignments mafft-nexus-internal-trimmed-gblocks-clean \
        --taxa 4 \
        --percent 0.75 \
        --output mafft-nexus-internal-trimmed-gblocks-clean-75p \
        --cores $NSLOTS \
        --log-path log
        ```

* Output alignments are put in ```uce-tutorial/taxon-sets/all/mafft-nexus-internal-trimmed-gblocks-clean-75p```

### 12. Preparing data for RAxML and ExaML
Here we will formatting our 75p data matrix into a phylip file for RAxML or ExaML.

* Make sure you are in the correct directory: ```uce-tutorial/taxon-sets/all```  

* **JOB FILE #17:** generate a phylip file
    + **PE:** serial
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.7.1  
    + **command:** ```phyluce_align_format_nexus_files_for_raxml```  
        + **arguments:**  
        ```
        --alignments mafft-nexus-internal-trimmed-gblocks-clean-75p \
        --output mafft-nexus-internal-trimmed-gblocks-clean-75p-raxml \
        --charsets \
        --log-path log
        ```

* The directory ```mafft-nexus-internal-trimmed-gblocks-clean-75p-raxml``` now contains the alignment in phylip format ready for RAxML or ExaML.


### Addendum
*Downloading tutorial data from figshare*
If you are working on this tutorial and don't have access to pre-downloaded raw fastq files on Hydra, you can download them from figshare with this command:

```
wget -O fastq.zip https://ndownloader.figshare.com/articles/1284521/versions/1
unzip fastq.zip
rm fastq.zip
```

In the fastq files we downloaded for you we renamed the fastq files so they will work with Trim Galore! To rename your newly downloaded files use these commands:
```
mv Alligator_mississippiensis_GGAGCTATGG_L001_R1_001.fastq.gz Alligator_mississippiensis_GGAGCTATGG_L001_R1.fastq.gz
mv Alligator_mississippiensis_GGAGCTATGG_L001_R2_001.fastq.gz Alligator_mississippiensis_GGAGCTATGG_L001_R2.fastq.gz
mv Anolis_carolinensis_GGCGAAGGTT_L001_R1_001.fastq.gz Anolis_carolinensis_GGCGAAGGTT_L001_R1.fastq.gz
mv Anolis_carolinensis_GGCGAAGGTT_L001_R2_001.fastq.gz Anolis_carolinensis_GGCGAAGGTT_L001_R2.fastq.gz
mv Gallus_gallus_TTCTCCTTCA_L001_R1_001.fastq.gz Gallus_gallus_TTCTCCTTCA_L001_R1.fastq.gz
mv Gallus_gallus_TTCTCCTTCA_L001_R2_001.fastq.gz Gallus_gallus_TTCTCCTTCA_L001_R2.fastq.gz
mv Mus_musculus_CTACAACGGC_L001_R1_001.fastq.gz Mus_musculus_CTACAACGGC_L001_R1.fastq.gz
mv Mus_musculus_CTACAACGGC_L001_R2_001.fastq.gz Mus_musculus_CTACAACGGC_L001_R2.fastq.gz
```
