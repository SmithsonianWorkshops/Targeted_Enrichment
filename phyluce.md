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


###1. Get the data  
First we will get the tutorial data.

* log in to Hydra
* change to your directory  
    + hint: `cd /pool/genomics/USERNAME` where *USERNAME* is your Hydra login name
* create a project directory  
```mkdir uce-tutorial```
* change to that directory  
```cd uce-tutorial```
* If you are running this tutorial after the workshop, see the [end of the document](###Addendum) for instructions in order to download the data using wget. Because there are so many of us working on the login node at once today, we will copy the data from ```/pool/genomics/tutorial_data```
* make a directory to hold the raw data  
```mkdir raw-fastq```
* change to the directory we just created  
```cd raw-fastq```
* copy the data to your working directory using ```cp```. The data are here: ```/pool/genomics/tutorial_data/fastq.zip```  
* unzip the fastq data  
```unzip fastq.zip```
* delete the zip file  
```rm fastq.zip```

#####How many files do you see in ```raw-fastq```?

###2. Count the read data

This step is not required to process UCEs, but it allows you to count the number of reads for each taxon. Here we are using Unix tools echo, gunzip, wc (word count), awk and we will need our first job file.

* **JOB FILE #1:** Counting read data (it is best practice to use a job file, even for a trivial task like this).
    + hint: use the QSub Generator: https://hydra-3.si.edu/tools/QSubGen
    		+ **Remember Chrome works best with this and to accept the security warning message*
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
    + hint: upload your job file using ```scp``` from your local machine into the `raw-fastq` directory
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

###3. Clean the read data
These data are raw, so we need to trim adapters and low quality reads before assembly. There are many tools for this. We have modified phyluce's illumiprocessor to use Trim Galore! instead of Trimmomatic. Trim Galore! has the needed functionality but behaves better on Hydra (i.e. does not use Java). 

* You will first need to copy a configuration file called `illumiprocessor.conf` from `/pool/genomics/tutorial_data` to your `uce-tutorial` directory.
	+ hint: use ```cp```.
	+ Make sure you've changed from the ```uce-tutorial``` directory to ```raw-fastq``` with the command ```cd ..``` before copying the file.


* **JOB FILE #2:** illumiprocessor  
    + hint: your raw reads are in ```raw-fastq``` but you want to run illumiprocessor from ```uce-tutorial```
    + **PE:** multi-thread 2 
    + **memory:** 2 GB
    + **modules:** phyluce_tg
    + **command:** `illumiprocessor`
    	+ **arguments:**  
        ```
        --input raw-fastq \  
        --output clean-fastq \
        --config illumiprocessor.conf \
        --paired \
        --cores $NSLOTS
        ```
    
* Check the log file and ```clean-fastq``` for results. Your cleaned files will be in ```clean-fastq/split-adapter-quality-trimmed```

###4. Assemble the data
We will use Trinity to assemble the data into contigs. There will be a separate Trinity run for each sample in your dataset. This is the most time consuming computer intensive portion of the pipeline.

* You need a configuration file to run Trinity within phyluce. Create a file called ```assembly.conf``` in ```uce-tutorial```.
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
    + **PE:** mthread 2  
    + **memory:** 6 GB (must have more than 8 GB total for this - here we are specifying 6 X 2 = 12 GB total RAM)
    + **modules:** phyluce_tg
    + **commands:**  ```phyluce_assembly_assemblo_trinity```  
        + **arguments:**  
        ```
        --conf assembly.conf \
        --output trinity-assemblies \
        --cores $NSLOTS
        ```

* Check the log file and the newly created directory ```trinity-assemblies/``` to see results.  
* Find your contigs!  

###5. Assembly QC
Let's check to see how well the assemblies worked.

* **JOB FILE #4:**
    + **PE:** serial  
    + **memory:** 1 GB  
    + **modules:** phyluce_tg
    + **command:**  
    ```
    for i in trinity-assemblies/contigs/*.fasta;
    do phyluce_assembly_get_fasta_lengths --input $i --csv;
    done
    ```
   
* Check the log file for output similar to the below (header not included):  
```
samples,contigs,total bp,mean length,95 CI length,min length,max length,median legnth,contigs >1kb   
alligator_mississippiensis.contigs.fasta,10587,5820479,549.776046094,3.5939422934,224,11285,413.0,1182   
anolis_carolinensis.contigs.fasta,2458,1067208,434.177379984,5.72662897806,224,4359,319.0,34   
gallus_gallus.contigs.fasta,19905,8841661,444.192966591,2.06136172068,224,9883,306.0,1530   
mus_musculus.contigs.fasta,2162,1126231,520.920906568,7.75103292163,224,6542,358.0,186   
```

###6. Finding UCE loci
Now we want to run ```lastz``` to match contigs to the UCE probe set and to remove duplicates.The search matches are stored in a sqlite database.

* sqlite is a database system that phyluce uses to store information about the contigs such as presence/absence. Phyluce scripts access this data and advanced users can access this database directly.

* Before we locate UCE loci (in other words, match your contigs to the UCE probes), you need to get the probe set used for the enrichments:   
	+ copy ```uce-5k-probes.fasta``` from ```/pool/genomics/tutorial_data``` to your ```uce-tutorial``` directory 

* **JOB FILE #5:** Match contigs to probes:
    + **PE:** mthread 2  
    + **memory:** 2 GB  
    + **modules:** phyluce_tg
    + **command:** ```phyluce_assembly_match_contigs_to_probes```
        + **arguments:** 
        ```
        --contigs trinity-assemblies/contigs \
        --probes uce-5k-probes.fasta \
        --output uce-search-results
        ```  
* The directory ```uce-search-results``` is created with the results.
* Look at the log file to see how many unique and duplicate matches and how many loci were removed for each taxon.    
* Here is an example output:

```
alligator_mississippiensis:
    4315 (40.76%) uniques of 10587 contigs
    0 dupe probe matches
    230 UCE loci removed for matching multiple contigs
    40 contigs removed for matching multiple UCE loci
```  
* To reduce the likelihood of paralogous loci being included, Phyluce removes any loci where there is more than contig that matches.

###7. Extracting UCE loci
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
    + **modules:** phyluce_tg
    + **command:** ```phyluce_assembly_get_match_counts```
        + **arguments:** 
        ```
        --locus-db uce-search-results/probe.matches.sqlite \
        --taxon-list-config taxon-set.conf \
        --taxon-group 'all' \
        --incomplete-matrix \
        --output taxon-sets/all/all-taxa-incomplete.conf
        ```  
    
* Check ```taxon-sets/all``` to see if the .conf file is there.  
* Now, we need to extract FASTA data that correspond to the loci in ```all-taxa-incomplete.conf```:  
* Change to the taxon-sets/all directory: ```cd taxon-sets/all```  
* Make a log directory to hold our log files: ```mkdir log```

* **JOB FILE #7:** get FASTA data for taxa in our taxon set
    + **PE:** serial
    + **memory:** 2 GB
    + **modules:** phyluce_tg  
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
    
* The extracted FASTA data are in a monolithic FASTA file (all data for all organisms) named ```all-taxa-incomplete.fasta``` - find it!

###8. Exploding the monolithic FASTA file
We can "explode" the monolithic fasta file into a file of UCE loci that we have enriched by taxon in order to get individual statistics on UCE assemblies for a given taxon.  

* Make sure your current working directory is ```uce-tutorial/taxon-sets/all```

* **JOB FILE #8:** explode the monolithic FASTA file
    + **PE:** serial
    + **memory:** 2 GB
    + **modules:** phyluce_tg  
    + **command:** ```phyluce_assembly_explode_get_fastas_file```  
        + **arguments:**  
        ```
        --input all-taxa-incomplete.fasta \
        --output-dir exploded-fastas \
        --by-taxon
        ```
* ```phyluce_assembly_explode_get_fastas_file``` created a directory ```uce-tutorial/taxon-sets/all/exploded-fastas```

* Make sure your current working directory is still ```uce-tutorial/taxon-sets/all```
* **JOB FILE #9:** get summary stats on the FASTAs
    + **PE:** serial
    + **memory:** 2 GB
    + **modules:** phyluce_tg  
    + **command:** 
    ```
    for i in exploded-fastas/*.fasta;
    do phyluce_assembly_get_fasta_lengths --input $i --csv;
    done
    ```

* Check your log file for output like this (note the values will not be identical to this example and there will not be a header line):

```
samples,contigs,total bp,mean length,95 CI length,min length,max length,median legnth,contigs >1kb
alligator-mississippiensis.unaligned.fasta,4315,3465679,803.170104287,3.80363492428,224,1794,823.0,980
anolis-carolinensis.unaligned.fasta,703,400214,569.294452347,9.16433421241,224,1061,546.0,7
gallus-gallus.unaligned.fasta,3923,3273674,834.482283966,4.26048496461,231,1864,852.0,1149
mus-musculus.unaligned.fasta,825,594352,720.426666667,9.85933217965,225,1178,823.0,139
```


###9. Aligning UCE loci

When you align UCE loci, you can either leave them as-is, without trimming, edge trim, or internal trim. See the PHYLUCE docs for more details about why you might choose one over the other. By default, edge-trimming is turned on. We will use MAFFT.

* make sure you are in the correct directory: ```uce-tutorial/taxon-sets/all```

* **JOB FILE #10:** align with edge-trimming
    + **PE:** mthread 4
    + **memory:** 2 GB
    + **modules:** phyluce_tg  
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
    + **modules:** phyluce_tg  
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
    + **modules:** phyluce_tg  
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
    + **modules:** phyluce_tg  
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
    + **modules:** phyluce_tg  
    + **command:** ```phyluce_align_get_align_summary_data```  
        + **arguments:**  
        ```
        --alignments mafft-nexus-internal-trimmed-gblocks \  
        --cores $NSLOTS \
        --log-path log
        ```

###10. Alignment cleaning
Each alignment now contains the locus name along with the taxon name. This is not what we want downstream, so we need to clean our alignments. For the remainder of this tutorial, we will work with the Gblocks trimmed alignments, so we will clean those alignments:

* Make sure we are in the correct directory: ```uce-tutorial/taxon-sets/all```

* **JOB FILE #15:** clean alignments
    + **PE:** multi-thread 2
    + **memory:** 2 GB
    + **modules:** phyluce_tg  
    + **command:** ```phyluce_align_remove_locus_name_from_nexus_lines```  
        + **arguments:**  
        ```
        --output mafft-nexus-internal-trimmed-gblocks-clean \
        --cores $NSLOTS \
        --log-path log
        ```   
  
* Now, if you look at the alignments, you will see that the locus names are removed. We’re ready to generate our final data matrices.

###11. Final data matrices
To create a 75% data matrix (i.e. 25% or less missing), run the following. Notice that the integer following –taxa is the total number of organisms in the study.

* Make sure we are in the correct directory: ```uce-tutorial/taxon-sets/all```

* **JOB FILE #16:** create a 75% data matrix
    + **PE:** multi-thread 2
    + **memory:** 2 GB
    + **modules:** phyluce_tg  
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

###12. Preparing data for RAxML and ExaML
Here we will formatting our 75p data matrix into a phylip file for RAxML or ExaML. 

* Make sure you are in the correct directory: ```uce-tutorial/taxon-sets/all```  

* **JOB FILE #17:** generate a phylip file
    + **PE:** serial
    + **memory:** 2 GB
    + **modules:** phyluce_tg  
    + **command:** ```phyluce_align_format_nexus_files_for_raxml```  
        + **arguments:**  
        ```
        --alignments mafft-nexus-internal-trimmed-gblocks-clean-75p \  
        --output mafft-nexus-internal-trimmed-gblocks-clean-75p-raxml \  
        --charsets \
        --log-path log
        ```

* The directory ```mafft-nexus-internal-trimmed-gblocks-clean-75p-raxml``` now contains the alignment in phylip format ready for RAxML or ExaML.


###Addendum
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