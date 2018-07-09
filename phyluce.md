Running phyluce on Hydra
---

Tutorial modified from Brant Faircloth's http://phyluce.readthedocs.org/en/latest/tutorial-one.html. It has been tailored to the Smithsonian HPC cluster, Hydra, by the SIBG bioinformatics group.

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


### 1. Get the data  
First we will get the tutorial data.

* log in to Hydra (If you need assistance with Hydra, check the [support wiki](https://confluence.si.edu/display/HPC/High+Performance+Computing)).
* change to your directory  
    + hint: `cd /pool/genomics/USERNAME` where *USERNAME* is your Hydra login name
* create a project directory  
```mkdir uce-tutorial```
* change to that directory  
```cd uce-tutorial```

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

This step is not required to process UCEs, but it allows you to count the number of reads for each taxon. Here we are using Unix tools echo, gunzip, wc (word count), awk and we will need our first job file.

* **JOB FILE #1:** Counting read data (it is best practice to use a job file, even for a trivial task like this).
    + hint: use the QSub Generator: https://hydra-4.si.edu/tools/QSubGen
    		+ *Remember Chrome works best with this and to accept the security warning message*
    + **CPU time:** short *(we will be using short for all job files in this tutorial)*
    + **memory:** 2GB
    + **PE:** serial
    + **shell:** sh *(use for all job files in the tutorial)*
    + **modules:** none
    + **command:** 
    ```
    for i in *R1*.fastq.gz;
    do echo $i;
    gunzip -c $i | wc -l | awk '{print $1/4}';
    done
    ```
    + **job name:** countreads *(or name of your choice)*
    + **log file name:** countreads.log
    + **change to cwd:** Checked *(keep checked for all job files)*
    + **join stderr & stdout** Checked *(Keep checked for all job files)*
    + hint: upload your job file using ```scp``` from your local machine into the `raw-fastq` directory. See [here](https://confluence.si.edu/display/HPC/Disk+Space+and+Disk+Usage) and [here](https://confluence.si.edu/display/HPC/Transferring+files+to+or+from+Hydra) on the Hydra wiki for more info.
    + hint: submit the job on Hydra using ```qsub```

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
    #$ -o countreads.log
    #
    # ----------------Modules------------------------- #
    #
    # ----------------Your Commands------------------- #
    #
    echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
    #
    for i in *R1*.fastq.gz;
    do echo $i;
    gunzip -c $i | wc -l | awk '{print $1/4}';
    done
    #
    echo = `date` job $JOB_NAME done
```  

Here's what should be in the log file after your job completes:

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
    + **PE:** multi-thread 2 
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.5_tg
    + **command:** `illumiprocessor`
    	+ **arguments:**  (Note: the arguments should start on the same line as the command. The '\' in the arguments allows them to span multiple lines.)
        ```
        --input raw-fastq \
        --output clean-fastq \
        --config illumiprocessor.conf \
        --paired \
        --cores $NSLOTS
        ```
    + **job name:** illumiprocessor *Use the name of the phyluce command as the job name for all the remaining job files*


Here is a sample job file:
```
	# /bin/sh
	# ----------------Parameters---------------------- #
	#$ -S /bin/sh
	#$ -pe mthread 2
	#$ -q sThC.q
	#$ -l mres=2G,h_data=2G,h_vmem=2G
	#$ -cwd
	#$ -j y
	#$ -N illumiprocessor
	#$ -o illumiprocessor.log
	#
	# ----------------Modules------------------------- #
	module load bioinformatics/phyluce/1.5_tg
	#
	# ----------------Your Commands------------------- #
	#
	echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
	echo + NSLOTS = $NSLOTS
	#
	illumiprocessor --input raw-fastq \
	--output clean-fastq \
	--config illumiprocessor.conf \
	--paired \
	--cores $NSLOTS
	#
	echo = `date` job $JOB_NAME done
```

* Submit and check the log file and ```clean-fastq``` for results. Your cleaned files will be in ```clean-fastq/split-adapter-quality-trimmed```
* *Note: this job will take about 10 minutes to run. You can use this time to read ahead and check the ```clean-fastq``` directory,* 
* **Troubleshooting:** With some Phyluce steps if you need to restart a command you will get an error that the output directory already exists. You will see a line like this in your log file: ```[WARNING] Output directory exists, REMOVE [Y/n]? Traceback (most recent call last):...```
    * Remove the offending directory and re-submit with `qsub`.


### 4. Assemble the data
We will use Trinity to assemble the data into contigs. There will be a separate Trinity run for each sample in your dataset. This is the most time consuming computer intensive portion of the pipeline. For today's tutorial, we will not have time to complete the assemblies.  

* Copy the directory with completed assemblies: ```/data/genomics/tutorial_data/trinity-assemblies``` to your ```uce-tutorial``` directory.  
* hint: use ```cp -r```.  
* Find the contigs!
* **Skip to the next section (#5).** If you have extra time now, feel free to make the Trinity job file but wait to submit it until later.

* Running the Trinity assembly (TO TRY AT A LATER DATE):  
    + You need a configuration file to run Trinity within phyluce. Create a file called ```assembly.conf``` in ```uce-tutorial```.
    + hint: change directory to ```uce-tutorial``` and use ```nano``` to create the file  
    + The file has one line for each sample starting with the sample ID and then the full path to the directory containing the cleaned reads.
    + contents of the new assembly.conf file (hint: you will need to change YOUR-PATH to your directory):  
    ```
    [samples] 
    alligator_mississippiensis:/YOUR-PATH/clean-fastq/alligator_mississippiensis/split-adapter-quality-trimmed/
    anolis_carolinensis:/YOUR-PATH/clean-fastq/anolis_carolinensis/split-adapter-quality-trimmed/
    gallus_gallus:/YOUR-PATH/clean-fastq/gallus_gallus/split-adapter-quality-trimmed/
    mus_musculus:/YOUR-PATH/clean-fastq/mus_musculus/split-adapter-quality-trimmed/
    ```  

* **JOB FILE #3:** Trinity
    + **CPU time** short or medium (depending on number of samples)
    + **PE:** mthread 2
    + **memory:** 6 GB (must have more than 8 GB total for this - here we are specifying 6 X 2 = 12 GB total RAM)
    + **modules:** bioinformatics/phyluce/1.5_tg
    + **commands:**  ```phyluce_assembly_assemblo_trinity```  
        + **arguments:**  
        ```
        --conf assembly.conf \
        --output trinity-assemblies \
        --cores $NSLOTS
        ```

### 5. Assembly QC
Let's check to see how well the assemblies worked.

* Make sure your current working directory is ```uce-tutorial```

* **JOB FILE #4:** Get FASTA lengths
    + **PE:** serial  
    + **memory:** 2 GB  
    + **modules:** bioinformatics/phyluce/1.5_tg
    + **command:**  
    ```
    for i in trinity-assemblies/contigs/*.fasta;
    do phyluce_assembly_get_fasta_lengths --input $i --csv;
    done
    ```
   
* Check the log file for output similar to below (header not included):  
```
samples,contigs,total bp,mean length,95 CI length,min length,max length,median legnth,contigs >1kb   
alligator_mississippiensis.contigs.fasta,6220,3615052,581.19807074,4.06558418147,224,11060,574.0,228
anolis_carolinensis.contigs.fasta,2367,1024996,433.035910435,5.55683628344,224,4363,319.0,32
gallus_gallus.contigs.fasta,16875,7088986,420.088059259,1.92888733993,224,6321,298.0,612
mus_musculus.contigs.fasta,1882,996123,529.289585547,8.06646682863,224,6047,370.0,137
```

### 6. Finding UCE loci
Now we want to run ```lastz``` to match contigs to the UCE probe set and to remove duplicates.The search matches are stored in a sqlite database.

* sqlite is a database system that phyluce uses to store information about the contigs such as presence/absence. Phyluce scripts access this data and advanced users can access this database directly.

* Before we locate UCE loci (in other words, match your contigs to the UCE probes), you need to get the probe set used for the enrichments:   
	+ copy ```uce-5k-probes.fasta``` from ```/data/genomics/tutorial_data``` to your ```uce-tutorial``` directory 

* **JOB FILE #5:** Match contigs to probes:
    + **PE:** mthread 2  
    + **memory:** 2 GB  
    + **modules:** bioinformatics/phyluce/1.5_tg
    + **command:** ```phyluce_assembly_match_contigs_to_probes```
        + **arguments:** 
        ```
        --contigs trinity-assemblies/contigs \
        --probes uce-5k-probes.fasta \
        --output uce-search-results
        ```  
* The directory ```uce-search-results``` is created with the results.
    * *If you need to resubmit this command, delete this directory first or you'll receive an error message in your log file.*
* Look at the log file, ```phyluce_assembly_match_contigs_to_probes.log```, to see how many unique and duplicate matches and how many loci were removed for each taxon.
* Here is an example output:

```
alligator_mississippiensis:
    4071 (65.45%) uniques of 6220 contigs
    0 dupe probe matches
    227 UCE loci removed for matching multiple contigs
    21 contigs removed for matching multiple UCE loci
```  
* To reduce the likelihood of paralogous loci being included, Phyluce removes any loci for a sample where there is more than contig that matches.

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

* Now that we have this file created, we do the following to create the initial list of loci for each taxon:
    + Make sure you current directory is ```uce-tutorial```
    + ```mkdir -p taxon-sets/all```

* **JOB FILE #6:** Get match counts:
    + **PE:** serial
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.5_tg
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


* Now, we need to extract FASTA data that correspond to the loci in ```all-taxa-incomplete.conf```:  
* Change to the ```taxon-sets/all``` directory: if you're not there already ```cd taxon-sets/all```
* Make a log directory to hold our log files: ```mkdir log```

* **JOB FILE #7:** get FASTA data for taxa in our taxon set
    + **PE:** serial
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.5_tg  
    + **command:** ```phyluce_assembly_get_fastas_from_match_counts```  
        + **arguments:**   
        ```
        --contigs ../../trinity-assemblies/contigs \
        --locus-db ../../uce-search-results/probe.matches.sqlite \
        --match-count-output all-taxa-incomplete.conf \
        --output all-taxa-incomplete.fasta \
        --incomplete-matrix all-taxa-incomplete.incomplete \
        --log-path log
        ```  
    
* The extracted FASTA data this command creates are in a monolithic FASTA file (all data for all organisms) named ```all-taxa-incomplete.fasta``` - **find it!**

### 8. Exploding the monolithic FASTA file
We can "explode" the monolithic fasta file into a file of UCE loci that we have enriched by taxon in order to get individual statistics on UCE assemblies for a given taxon.  

* Make sure your current working directory is ```uce-tutorial/taxon-sets/all```

* **JOB FILE #8:** explode the monolithic FASTA file
    + **PE:** serial
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.5_tg  
    + **command:** ```phyluce_assembly_explode_get_fastas_file```  
        + **arguments:**  
        ```
        --input all-taxa-incomplete.fasta \
        --output-dir exploded-fastas \
        --by-taxon
        ```
* ```phyluce_assembly_explode_get_fastas_file``` created a directory ```exploded-fastas```, take a look at the fasta files.

* Make sure your current working directory is ```uce-tutorial/taxon-sets/all```
* **JOB FILE #9:** get summary stats on the FASTAs
    + **PE:** serial
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.5_tg  
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

When you align UCE loci, you can either leave them as-is, without trimming, edge trim, or internal trim. See the PHYLUCE docs for more details about why you might choose one over the other. By default, edge-trimming is turned on. We will use MAFFT.

* make sure you are in the correct directory: ```uce-tutorial/taxon-sets/all```

* **JOB FILE #10:** align with edge-trimming
    + **PE:** mthread 4
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.5_tg  
    + **command:** ```phyluce_align_seqcap_align```  
        + **arguments:**  
        ```
        --fasta all-taxa-incomplete.fasta \
        --output mafft-nexus-edge-trimmed \
        --taxa 4 \
        --aligner mafft \
        --cores $NSLOTS \
        --incomplete-matrix \
        --log-path log
        ```  
 
* When you look at the log, the ```.``` values that you see represent loci that were aligned and successfully trimmed. Any ```X``` values that you see represent loci that were removed because trimming reduced their length to effectively nothing.

* output alignments are in ```uce-tutorial/taxon-sets/all/mafft-nexus-edge-trimmed```

* We can output summary stats for these alignments by running the following program:

* **JOB FILE #11:** get alignment summary data
    + **PE:** serial
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.5_tg  
    + **command:** ```phyluce_align_get_align_summary_data```  
        + **arguments:**  
        ```
        --alignments mafft-nexus-edge-trimmed \
        --cores $NSLOTS \
        --log-path log
        ```
* **Inspect the summary information that will be in the log file.**

* Now, let’s do the same thing, but run internal trimming. We will do that by turning off trimming –no-trim and outputting FASTA formatted alignments with –output-format fasta.

* make sure we are in the correct directory: ```uce-tutorial/taxon-sets/all```  

* **JOB FILE #12:** align with no-trim and output FASTA
    + **PE:** multi-thread 4
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.5_tg  
    + **command:** ```phyluce_align_seqcap_align```  
       + **arguments:**
       ```
       --fasta all-taxa-incomplete.fasta \
       --output mafft-nexus-internal-trimmed \
       --taxa 4 \
       --aligner mafft \
       --cores $NSLOTS \
       --incomplete-matrix \
       --output-format fasta \
       --no-trim \
       --log-path log
       ```

* output alignments are in ```uce-tutorial/taxon-sets/all/mafft-nexus-internal-trimmed```

* Now, we are going to trim these loci using Gblocks.

* **JOB FILE #13:** internal trim with Gblocks
    + **PE:** multi-thread 2
    + **memory:** 2 GB
    + **modules:** bioinformatics/phyluce/1.5_tg  
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
    + **modules:** bioinformatics/phyluce/1.5_tg  
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
    + **modules:** bioinformatics/phyluce/1.5_tg  
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
    + **modules:** bioinformatics/phyluce/1.5_tg  
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
    + **modules:** bioinformatics/phyluce/1.5_tg  
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
