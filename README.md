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
# Setting Up And Launching a PB-Assembly for MGI
These are the steps that are used to launch pb-assembly on the MGI compute0 platform

**1. Create an assembly directory**

```
$ mkdir Gambian_HG02886_CCS_HiFi_PB_Assembly_Falcon_Unzip
```

**2. Prepare read data directories and copy in CCS.fasta and CCS.fastq files:**
PacBio HiFi data is available in CCS.bam, CCS.fasta, and CCS.fastq formats. By default, the data is filtered to only include reads with a quality value of Q20 or greater.
Q20 equates to allowing 1 error per 100 basepairs or 99% base accuracy. For pb-assembly we need the data in both CCS.fasta and CCS.fastq formats.
```
$ cd Gambian_HG02886_CCS_HiFi_PB_Assembly_Falcon_Unzip
$ mkdir CCS_Data
$ cd CCS_Data
$ mkdir CCS_FASTA CCS_FASTQ
$ cp CCS.Q20.fasta CCS_FASTA/
$ cp CCS.Q20.fastq CCS_FASTQ/
```

**3. Split up CCS.fasta files into ~400 MB chunks:**
The assembly building step of pb-assembly requires CCS.fasta as input. The initial pre-assembly stage of pb-assembly is parallelized. You can distribute jobs across multiple cluster nodes by splitting up the CCS.fasta file into multiple chunks. We use an in-house developed script to split the CCS.fasta into ~400 MB chunks, which is the approximate size suggested by Pacific Biosciences.

```
$ cd Gambian_HG02886_CCS_HiFi_PB_Assembly_Falcon_Unzip/CCS_FASTA/
$ split_fasta.pl CCS.fasta 
```
Output is CCS_1.fasta, CCS_2.fasta, CCS_3.fasta.......

**4. Prepare a CCS.fasta.fofn in the assembly directory:**
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
**5. Setup falcon (fc_run.cfg) and falcon-unzip HiFi (fc_unzip_HiFi.cfg) configuration files in the assembly directory and modify to work on your system:**

You can obtain configuration files from the Pacific Biosciences github page: https://github.com/PacificBiosciences/pb-assembly/tree/master/cfgs
The configuration files need to be modified to work on your specific cluster configuration: LSF, SGE, PBS, etc.

We modified the configuration files to run on the MGI LSF cluster. We have modified the NPROC, MB, and njobs settings below for each stage based upon the results of testing on our cluster.

Example fc_run.cfg used for Human HiFi/CCS Assemblies:
```
$ cd Gambian_HG02886_CCS_HiFi_PB_Assembly_Falcon_Unzip/
$ vi fc_run.cfg
[General]
input_fofn=CCS.fasta.fofn
input_type=preads
pa_DBdust_option=
pa_fasta_filter_option=streamed-median
target=assembly
skip_checks=False
LA4Falcon_preload=false

#### Data Partitioning
pa_DBsplit_option=-x500 -s400
ovlp_DBsplit_option=-s400

#### Repeat Masking
pa_HPCTANmask_option=
#no-op repmask param set
pa_REPmask_code=0,300;0,300;0,300

####Pre-assembly
length_cutoff=5000
pa_HPCdaligner_option=-v -B128 -M24
pa_daligner_option= -k18 -e0.75 -l1200 -h256 -w8 -s100
falcon_sense_option=--output-multi --min-idt 0.70 --min-cov 4 --max-n-read 200
falcon_sense_greedy=False

####Pread overlapping
ovlp_HPCdaligner_option=-v -B128 -M24
ovlp_daligner_option=-k24 -e.92 -l1800 -h600 -s100

####Final Assembly
length_cutoff_pr=5000
overlap_filtering_setting=--max-diff 100 --max-cov 100 --min-cov 2 --ignore-indels
fc_ovlp_to_graph_option=

[job.defaults]
job_type=lsf
#pwatcher_type=blocking
pwatcher_type=blocking
JOB_QUEUE=pacbio-sequel
MB=40000
NPROC=6
njobs=100
submit = bsub -q ${JOB_QUEUE} -J ${JOB_NAME} -M ${MB}000 -o ${JOB_STDOUT} -e ${JOB_STDERR} -K -R 'rusage[mem=${MB}]' -n ${NPROC} -a 'docker(halllab/pbassembly:0.0.6)' bash ${JOB_SCRIPT}

[job.step.da]
NPROC=4
MB=32768
njobs=100
[job.step.la]
NPROC=4
MB=32768
njobs=100
[job.step.cns]
NPROC=8
MB=65536
njobs=100
[job.step.pda]
NPROC=4
MB=32768
njobs=100
[job.step.pla]
NPROC=4
MB=32768
njobs=100
[job.step.asm]
NPROC=16
MB=210000
njobs=1
```
Example fc_unzip_HiFi.cfg used for Human HiFi/CCS Assemblies:
```
$ cd Gambian_HG02886_CCS_HiFi_PB_Assembly_Falcon_Unzip/
$ vi fc_unzip_HiFi.cfg
[General]
max_n_open_files = 20000
njobs=225
NPROC=4
MB=50000

[Unzip]
fastq=/gscmnt/gc2758/analysis/reference_grant/Gambian_HG02886_CCS_HiFi_PB_Assembly_Falcon_Unzip/CCS_Data/CCS_FASTQ/CCS.Q20.fastq
input_fofn=CCS.fasta.fofn

[job.defaults]
pwatcher_type = fs_based
job_type = local
submit = bsub -q ${JOB_QUEUE} -M 50000000 -J ${JOB_NAME} -o ${JOB_STDOUT} -e ${JOB_STDERR} -R 'rusage[mem=50000]' -n 4 -a 'docker(halllab/pbassembly:latest)' bash ${JOB_SCRIPT}
JOB_QUEUE = research-hpc
NPROC=4
MB=50000
njobs=100

[job.high]
njobs=10
NPROC=4
MB=100000

[job.highmem]
njobs=1
MB=210000
NPROC=4
```
**6. Setup enviroment in terminal for running pb-assembly**
This involves interactively logging into docker container that was setup for pb-assembly

```
$ ssh -Y virtual-workstation3.gsc.wustl.edu
$ 
