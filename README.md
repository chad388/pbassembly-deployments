# PacBIO PB-Assembly Deployment Configurations

https://github.com/PacificBiosciences/pbbioconda/wiki/Release-notes

These are configurations including dockerfiles and deployment scripts for running PacBio's pb-assembly at MGI and on Google Cloud. PB Assembly is availble via bioconda. Install is easy, and the resulting install size is about 1.8 Gb. See building pb-asssembly section below.

# Running on MGI

Network Location: /gscmnt/gc2134/finishing/pb-assembly
Versions Available: v0.0.6 (ptython 3), v0.0.2
Dockers: halllab/pbassembly:0.0.6, halllab/pbassembly:0.0.2

**Start an interactive job using the hall lab pb-assembly docker. Do not preserve the environment for this session. The environment established in the docker container is essential.**
```
$ LSF_DOCKER_PRESERVE_ENVIRONMENT=false bsub -q docker-interactive -Is -a 'docker(halllab/pbassembly:0.0.6)' /bin/bash
```
**Activate the pb-assembly environment. The available environments can be listed, and the default is "base," as seen in the command line.**
```
(base) ctomlins@blade18-1-16:/gscuser/ctomlins$ conda info --env
# conda environments:
#
pb-assembly-0.0.6        /gscmnt/gc2134/finishing/pb-assembly/.conda/envs/pb-assembly-0.0.6
base                  *  /opt/conda

(base) ctomlins@blade18-1-16:/gscuser/ctomlins$ conda activate pb-assembly-0.0.6
(pb-assembly-0.0.6) ctomlins@blade18-1-16:/gscuser/ctomlins$ 
```
**Launch jobs to LSF. Here the environment is needed to be preserved, do not include the LSF_DOCKER_PRESERVE_ENVIRONMENT=false environment variable. Below is an example of running falcon unzip.**
```
(pb-assembly-0.0.6) ctomlins@blade18-1-16:/gscuser/ctomlins$ bsub -q research-hpc -a 'docker(halllab/pbassembly:0.0.6)' fc_unzip.py fc_unzip.cfg

```
# Building PB Assembly for MGI
To have an install of pb-assembly for general use, follow these instructions. One key is to set HOME to a network location. A good docker to use is `halllab/miniconda3:latest`, but the docker must have `conda` installed.

```
$ LSF_DOCKER_PRESERVE_ENVIRONMENT=false bsub -q docker-interactive -Is -a 'docker(halllab/miniconda3:latest)' /bin/bash
$ HOME=/gscmnt/gc2134/finishing/pb-assembly; export HOME
$ mkdir -p "${HOME}"
$ cd "${HOME}"
$ conda init bash
$ source /etc/profile
$ conda create -c bioconda -n pb-assembly-0.0.6 pb-assembly=0.0.6 python=3.7.3
$ conda activate pb-assembly-0.0.6
```
# Add another version. If you use -n instead of -p the executables will be installed to /opt/conda/envs within the container.
```
$ HOME=/gscmnt/gc2134/finishing/pb-assembly; export HOME
$ conda create -p $HOME/.conda/envs/pb-assembly-0.0.8 -c bioconda pb-assembly=0.0.8 python=3.7
$ conda activate pb-assembly-0.0.8
```
# Setting Up And Launching a PB-Assembly
These are the steps that are used to launch pb-assembly on the MGI compute0 platform

**Create an assembly directory**

```
$ mkdir Gambian_HG02886_CCS_HiFi_PB_Assembly_Falcon_Unzip
```

**Prepare read data directories and copy in CCS.fasta and CCS.fastq files:**
PacBio HiFi data is available in CCS.bam, CCS.fasta, and CCS.fastq formats. By default, the data is filtered to only include reads with a quality value of Q20 or greater.
Q20 equates to allowing 1 error per 100 basepairs or 99% base accuracy. For pb-assembly we need the data in both CCS.fasta and CCS.fastq formats.
```
$ cd Gambian_HG02886_CCS_HiFi_PB_Assembly_Falcon_Unzip
$ mkdir CCS_Data
$ cd CCS_Data
$ mkdir CCS_FASTA CCS_FASTQ
$ cp CCS.fasta CCS_FASTA/
$ cp CCS.fastq CCS_FASTQ/
```

**Split up CCS.fasta files into ~400 MB chunks:**
The assembly building step of pb-assembly requires CCS.fasta as input. The initial pre-assembly stage of pb-assembly is parallelized. You can distribute jobs across multiple cluster nodes by splitting up the CCS.fasta file into multiple chunks. We use an in-house developed script to split the CCS.fasta into ~400 MB chunks, which is the approximate size suggested by Pacific Biosciences.

```
$ cd Gambian_HG02886_CCS_HiFi_PB_Assembly_Falcon_Unzip/CCS_FASTA/
$ split_fasta.pl CCS.fasta 
```
Output is CCS_1.fasta, CCS_2.fasta, CCS_3.fasta.......

**Prepare a CCS.fasta.fofn in the assembly directory:**
Fofn stands for file-of-file-names. In this instance, it is a text file containing the full path to each of the CCS fasta chunk files that were created during the preceeding step.
```
$ cd Gambian_HG02886_CCS_HiFi_PB_Assembly_Falcon_Unzip/
$ more CCS.fasta.fofn
/gscmnt/gc2758/analysis/reference_grant/Gambian_HG02886_CCS_HiFi_PB_Assembly_Falcon_Unzip/CCS_Data/CCS_FASTA/m64043_200414_165107.Q20_0.fasta
/gscmnt/gc2758/analysis/reference_grant/Gambian_HG02886_CCS_HiFi_PB_Assembly_Falcon_Unzip/CCS_Data/CCS_FASTA/m64043_200414_165107.Q20_1.fasta
/gscmnt/gc2758/analysis/reference_grant/Gambian_HG02886_CCS_HiFi_PB_Assembly_Falcon_Unzip/CCS_Data/CCS_FASTA/m64043_200414_165107.Q20_2.fasta
/gscmnt/gc2758/analysis/reference_grant/Gambian_HG02886_CCS_HiFi_PB_Assembly_Falcon_Unzip/CCS_Data/CCS_FASTA/m64043_200414_165107.Q20_3.fasta
/gscmnt/gc2758/analysis/reference_grant/Gambian_HG02886_CCS_HiFi_PB_Assembly_Falcon_Unzip/CCS_Data/CCS_FASTA/m64043_200413_165107.Q20_4.fasta
/gscmnt/gc2758/analysis/reference_grant/Gambian_HG02886_CCS_HiFi_PB_Assembly_Falcon_Unzip/CCS_Data/CCS_FASTA/m64043_200413_165107.Q20_5.fasta
/gscmnt/gc2758/analysis/reference_grant/Gambian_HG02886_CCS_HiFi_PB_Assembly_Falcon_Unzip/CCS_Data/CCS_FASTA/m64043_200414_165107.Q20_6.fasta
```


**Setup falcon (fc_run.cfg) and falcon-unzip HiFi (fc_unzip_HiFi.cfg) configuration files in the assembly directory and modify to work on your system:**

You can obtain configuration files from the Pacific Biosciences github page: https://github.com/PacificBiosciences/pb-assembly/tree/master/cfgs
The configuration files need to be modified to work on your specific cluster configuration: LSF, SGE, PBS, etc.


Prepare data for assembly
PacBio HiFi data is available in CCS.bam, CCS.fasta, and CCS.fastq formats. By default, the data is filtered to only include reads with a quality value of Q20 or greater.
Q20 equates to allowing 1 error per 100 basepairs or 99% base accuracy. For pb-assembly we need the data in both CCS.fasta and CCS.fastq formats. 

**Prepare read data directories**
```


