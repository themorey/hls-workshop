# Genomics Workshop - NextFlow, Docker and Bioconda on Azure


## Pre-requisites
1. Verify data
    - In the Maarifa POC cluster the data files have been downloaded to `/shared/genomics_workshop_data`
    - If needed, it can be downloaded from https://aka.ms/genomicsworkshop/sample_data as genomics_workshop_data.tar
        - The default NFS export mount in the cluster is `/shared`
        - Expand the `genomics_workshop_data.tar` into `/shared` so that the data is in it shows as `/shared/genomics_workshop_data`
	
        ```
        cd /shared
        wget -O genomics_workshop_data.tar https://aka.ms/genomicsworkshop/sample_data
        tar xvf genomics_workshop_data.tar
        rm genomics_workshop_data.tar
        ```

2. A CycleCloud installation

    - CycleCloud is already configured in the Maarifa POC environment (refer to Cyclecloud onbarding document)
    - Storage options include:
        - Azure Files container mounted to `/work` (Persistent Storage)
        - Shared NFS directory mounted to `/shared` (Persisten Storage)
        - Local SSD on each compute node mounted to `/mnt` (Ephemeral Storage)
    - A custom image was created that includes `miniconda2`, `docker` and `singularity`
    - A conda environment named `shared_env` was created with the tools required for this exercise

3. Connect to the Cyclecloud Slurm cluster

## Preamble

 1. This cluster has a couple of cluster-init projects defined that does the following:
    - Mounts a shared Azure Files container to `/work`
    - Cyclecloud provisioned user accounts are added to the `docker` group for rootless Docker access
 2. The exercises illustrate the following:
    - How to access packages using Conda. 
    - How to run docker images.
    - How to do the the above through nextflow
    - How to use nextflow with the Slurm executor

	
## Exercise 1: Datasets 

 1. SSH into the Cluster headnode (ie. ssh username@10.0.3.133)
 2. The `/shared/genomics_workshop_data/` directory is pre-loaded for the exercises
 3. Verify that directory has the human genome index and sample Fastqs
    - bowtie2_index/grch38/
    - NA12787/

 4. Verify `/shared/genomics_workshop_data/` directory also contains two example nextflow files (and nextflow config files)
    - bowtie2.local.nf
    - nextflow.config
    - bowtie2.container.nf
    - nextflow.slurm.config

 5. If the data is not there is can be downloaded and extracted from https://aka.ms/genomicsworkshop/sample_data 

## Exercise 2: Bowtie in Conda Environment using Slurm

1. NOTE: steps 1 & 2 was already built into the image and is provided for Reference only (ie. if you want to create your own environment with these tools)

2. Install conda. This is done for the user account, so run the following as the user:

    ```
    # Get the installer
    wget https://repo.anaconda.com/miniconda/Miniconda2-latest-Linux-x86_64.sh

    # Run the installer
    bash Miniconda2-latest-Linux-x86_64.sh  -b

    # Set up the path and environment
    echo "source ~/miniconda2/etc/profile.d/conda.sh" >> ~/.bashrc
    echo "export PATH=~/miniconda2/bin:$PATH" >> ~/.bashrc
    source ~/miniconda2/etc/profile.d/conda.sh
    export PATH=~/miniconda2/bin:$PATH
    ```

2. Install tools:

    ```
    # Set up the conda "channels"
    conda config --add channels bioconda
    conda config --add channels conda-forge

    # install bioinformatics tools
    conda install -y bowtie2 samtools bcftools htop glances nextflow
    ```

3. Run a Slurm bowtie job with the conda tools

    
    Copy the sample job script named `bowtie-slurm.sh` to your home directory:
    ```
    cp /shared/genomics_workshop_data/bowtie-slurm.sh ~
    ```

    More details on Slurm job scripts: [SchedMD](https://slurm.schedmd.com/sbatch.html)
    Slurm sbatch examples and directives: [UB-CCR](https://ubccr.freshdesk.com/support/solutions/articles/5000688140-submitting-a-slurm-job-script)

    Submit the jobs as follows from the Slurm head node:
    ```
    $ sbatch bowtie-slurm.sh
    Submitted batch job 19

    $ squeue --job 19
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
                19       htc   bowtie jerry.mo CF       0:20      1 htc-1
    ```

    Output for this job will be written to `~/slurm-19.out` and a successful result will look as follows:
    ```
    100000 reads; of these:
    100000 (100.00%) were paired; of these:
    6616 (6.62%) aligned concordantly 0 times
    63233 (63.23%) aligned concordantly exactly 1 time
    30151 (30.15%) aligned concordantly >1 times
    ----
    6616 pairs aligned concordantly 0 times; of these:
      525 (7.94%) aligned discordantly 1 time
    ----
    6091 pairs aligned 0 times concordantly or discordantly; of these:
      12182 mates make up the pairs; of these:
        8324 (68.33%) aligned 0 times
        2141 (17.58%) aligned exactly 1 time
        1717 (14.09%) aligned >1 times
    95.84% overall alignment rate
    [mpileup] 1 samples in 1 input files
    [mpileup] maximum number of reads per input file set to -d 250
    ```

## Exercise 3: Bowtie in Docker Container on Slurm
1. Run the same set of bowtie and samtools commands, but using Docker containers

    ```
    # view the docker images on this VM
    docker image list

    # Run bowtie2 again, but now using the container instead of the conda install binary

    cd ~/genomics-workshop
    mkdir docker_results
    cp /shared/genomics_workshop_data/slurm-job-template.sh docker-slurm.sh
    
    # change the job name to docker-bowtie
    sed -i 's|bowtie|docker-bowtie|g' docker-slurm.sh

    # add the Docker run command to the job script
    cat << EOF >> docker-slurm.sh
    docker run -u $(id -u ${USER}):$(id -g ${USER})  -v ~/genomics-workshop/docker_results:/results -v /shared/genomics_workshop_data:/data biocontainers/bowtie2:v2.3.1_cv1  bowtie2 -x bowtie2_index/grch38/grch38_1kgmaj -1 /data/NA12787/SRR622461_1.fastq.gz -2 /data/NA12787/SRR622461_2.fastq.gz  -u 100000 -S /results/exercise3.sam

    docker run -u $(id -u ${USER}):$(id -g ${USER})  -v ~/genomics-workshop/docker_results:/results -v /shared/genomics_workshop_data:/data biocontainers/samtools:v1.7.0_cv4 /bin/bash -c "samtools view -bS /results/exercise3.sam | samtools sort -o /results/exercise3.bam"
    EOF

    # submit the script using sbatch command
    sbatch docker-slurm.sh
    ```
    Monitor the job and output as in Exercise 2

## Exercise 4: Nextflow with docker, on Slurm
- Finally, use Nextflow with the Slurm scheduler by specifying the execution engine in the config file. 
- Using the NF flow file, the docker tasks are submitted to the Slurm compute nodes. 

```
    conda activate shared_env

    nextflow -c /shared/genomics_workshop_data/nextflow.slurm.config run /shared/genomics_workshop_data/bowtie2.container.nf

- Verify that the job is in queue by using the `squeue` command
- Notice also that the autoscaler kicks in and scales up a compute node.

- Successful output will look like this:
```
N E X T F L O W  ~  version 20.07.1
Launching `/shared/genomics_workshop_data/bowtie2.container.nf` [cheesy_easley] - revision: e62788e7d0

==========================================================
 Microsoft Azure Genomics Workshop 
==========================================================
Data Dir                       : /shared/genomics_workshop_data
reads                          : /shared/genomics_workshop_data/NA12787/SRR622461_{1,2}.fastq.gz
Number of reads to process:    : 100000
bowtie2_index                  : /shared/genomics_workshop_data/bowtie2_index/grch38/grch38_1kgmaj
Resultdir                      : ~/genomics-workshop/results
Results prefix                 : exercise4
executor >  slurm (3)
[44/cd0c1f] process > runBowtie2 (1) [100%] 1 of 1 ✔
[45/bc92fa] process > samToBam (1)   [100%] 1 of 1 ✔
[7a/0384bc] process > bamToVCF (1)   [100%] 1 of 1 ✔
Completed at: 15-Oct-2020 19:02:49
Duration    : 5m 26s
CPU hours   : (a few seconds)
Succeeded   : 3
```
