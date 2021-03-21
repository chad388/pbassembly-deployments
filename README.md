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
$ conda create -c bioconda -n pb-assembly-0.0.8 pb-assembly=0.0.8 python=3.7.3
$ conda activate pb-assembly-0.0.8
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
$ mkdir Gambian_HG02723_CCS_HiFi_PB_Assembly_Falcon_Unzip
```

**2. Gather the paths to the HiFi (CCS) data in both CCS.fasta and CCS.fastq format:**
You will need these data paths when setting up the assembler configuration files.

For assembly of human samples, we typically generate 4 SMRT cells of data.
The CCS Q20 yield per SMRT cell is ~30-40 Gbps.
This translates into total input coverage ranging between ~40X-50X.
For the sample that this example is based upon, Gambian (HG02886), the total CCS Q20 yield is 137.4 Gbps or 45.8X coverage


**3. Prepare a CCS.fasta.fofn in the assembly directory:**
Fofn stands for file-of-file-names. In this instance, it is a text file containing the full path to the CCS.fasta file.
```
$ cd Gambian_HG02723_CCS_HiFi_PB_Assembly_Falcon_Unzip/
$ more CCS.fasta.fofn
/gscmnt/gc2758/analysis/reference_grant/Gambian_HG02723_CCS_HiFi_PB_Assembly_Falcon_Unzip/CCS_Data/CCS_FASTA/CCS.Q20.fasta
```
**4. Setup falcon (fc_run.cfg) and falcon-unzip HiFi (fc_unzip_HiFi.cfg) configuration files in the assembly directory and modify to work on your system:**

You can obtain configuration files from the Pacific Biosciences github page: https://github.com/PacificBiosciences/pb-assembly/tree/master/cfgs
The configuration files need to be modified to work on your specific cluster configuration: LSF, SGE, PBS, etc.

We modified the configuration files to run on the MGI LSF cluster. We have modified the NPROC, MB, and njobs settings below for each stage based upon the results of testing on our cluster.

**Example fc_run.cfg used for Human HiFi/CCS Assemblies:**
```
$ cd Gambian_HG02723_CCS_HiFi_PB_Assembly_Falcon_Unzip/
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
JOB_QUEUE=research-hpc
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
**Example fc_unzip_HiFi.cfg used for Human HiFi/CCS Assemblies:**
```
$ cd Gambian_HG02723_CCS_HiFi_PB_Assembly_Falcon_Unzip/
$ vi fc_unzip_HiFi.cfg
[General]
max_n_open_files = 20000
njobs=225
NPROC=4
MB=50000

[Unzip]
fastq=/gscmnt/gc2758/analysis/reference_grant/Gambian_HG02723_CCS_HiFi_PB_Assembly_Falcon_Unzip/CCS_Data/CCS_FASTQ/CCS.Q20.fastq
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
**5. Setup enviroment in terminal for running pb-assembly:**

Open a terminal on your computer.
SSH into a virtual-workstation on compute0.
Pulldown the pbassembly:0.0.6 docker in an interactive shell. 

```
$ ssh -Y virtual-workstation3.gsc.wustl.edu
$ LSF_DOCKER_PRESERVE_ENVIRONMENT=false bsub -q docker-interactive -Is -a 'docker(halllab/pbassembly:0.0.6)' /bin/bash
```
Use conda to select the version of pb-assembly that you want to use. 
You can verify that you have the correct version selected by checking one of the executables.
Below I am verifying the path to the fc_run.py script.

```
(base) ctomlins@blade18-1-15:/gscmnt/gc2758/analysis/reference_grant/Gambian_HG02723_CCS_HiFi_PB_Assembly_Falcon_Unzip$ conda activate pb-assembly-0.0.6
(pb-assembly-0.0.6) ctomlins@blade18-1-15:/gscmnt/gc2758/analysis/reference_grant/Gambian_HG02723_CCS_HiFi_PB_Assembly_Falcon_Unzip$ which fc_run.py
/gscmnt/gc2134/finishing/pb-assembly/.conda/envs/pb-assembly-0.0.6/bin/fc_run.py
```

**6. Launch pb-assembly fc_run.py step:**
Use the terminal that you setup in the preceeding step.
Launch the job from the assembly directory.

```
$ cd Gambian_HG02723_CCS_HiFi_PB_Assembly_Falcon_Unzip/

$ ls -lrt | awk '{print $9}'
CCS_Data/
CCS.fasta.fofn
fc_run.cfg
fc_unzip_HiFi.cfg

$ bsub -oo pb_run.log -R "rusage[mem=20000] span[hosts=1]" -q research-hpc -a 'docker(halllab/pbassembly:0.0.6)' fc_run fc_run.cfg
```
This stage of the assembly process took 44 hours or ~1.83 days when running on the data for sample HG02723.

The main output from this stage is a collapsed haplotype assembly of primary contigs in fasta format:
**p_ctg.fasta**

**HG02723 p_ctg.fasta Assembly Statistics**
```
 #CONTIGS   1,868           
  LENGTH    2,893,463,624 bp     
  AVG       1,548,963 bp       
  N50       19,307,067 bp
  LARGEST   80,703,395  bp      
  Contigs > 1M: 236 ( 2,723,997,977 bp ) 94.1%
  Contigs 250K--1M: 170 ( 83,143,418 bp ) 2.9%
  Contigs 100K--250K: 272 ( 41,971,624 bp ) 1.5%
  Contigs 10K--100K: 1189 ( 44,342,245 bp ) 1.5%
  Contigs 5K--10K: 1 ( 8,360 bp ) 0.0003%
  Contigs 2K--5K: 0 ( 0 bp ) 0%
  Contigs 0--2K: 0 ( 0 bp ) 0%
  ```


**8. Launch pb-assembly fc_unzip.py step**
Follow the same procedure that was used in step 6 above to setup your environment for running pb-assembly.
```
$ cd Gambian_HG02723_CCS_HiFi_PB_Assembly_Falcon_Unzip/
$ bsub -oo pb_unzip.log -R "rusage[mem=20000] span[hosts=1]" -q research-hpc -a 'docker(halllab/pbassembly:0.0.6)' fc_unzip.py --target=ccs fc_unzip_HiFi.cfg
```

This stage of the assembly process took 121 hours or ~5.05 days to run from start to finish.

The main outputs from this stage are a set of polished primary contigs and associated haplotigs in fasta format:  **polished_p_ctgs.fasta & polished_h_ctgs.fasta**

**HG02723 polished p_ctg.fasta Assembly Statistics**
```
 #CONTIGS   1,738           
  LENGTH    2,886,709,800 bp
  AVG       1,660,937 bp        
  N50       20,615,266 bp
  LARGEST   80,725,402 bp       
  Contigs > 1M: 236 ( 2,723,682,636 bp ) 94.4%
  Contigs 250K--1M: 163 ( 80,254,051 bp ) 2.8%
  Contigs 100K--250K: 264 ( 40,874,654 bp ) 1.4%
  Contigs 10K--100K: 1075 ( 41,898,459 bp ) 1.5%
  Contigs 5K--10K: 0 ( 0 bp ) 0%
  Contigs 2K--5K: 0 ( 0 bp ) 0%
  Contigs 0--2K: 0 ( 0 bp ) 0%
  ```
  **HG02723 polished h_ctg.fasta Assembly Statistics**
  ```
  #CONTIGS  16,176          
  LENGTH    2,511,021,594 bp     
  AVG       155,231 bp         
  N50       333,721 bp
  LARGEST   2,086,659 bp        
  Contigs > 1M: 141 ( 170,229,548 bp ) 6.8%
  Contigs 250K--1M: 3112 ( 1,384,195,888 bp ) 55.1%
  Contigs 100K--250K: 3414 ( 560,922,333 bp ) 22.3%
  Contigs 10K--100K: 9508 ( 395,663,969 bp ) 15.8%
  Contigs 5K--10K: 1 ( 9,856 bp ) 0.0004%
  Contigs 2K--5K: 0 ( 0 bp ) 0%
  Contigs 0--2K: 0 ( 0 bp ) 0%
  ```

**9. Launch pb-assembly fc_phase.py step**

Setup Falcon-Phase config file
```
$ more fc_phase.cfg
[General]
#target=minced

[job.defaults]
NPROC=2
njobs=200
MB=40000
pwatcher_type=blocking
job_type=lsf
JOB_QUEUE=ccdg
submit = bsub -q ${JOB_QUEUE} -M 50000000 -J ${JOB_NAME} -o ${JOB_STDOUT} -e ${JOB_STDERR} -R 'rusage[mem=50000]' -n 6 -K -a 'docker(halllab/pbassembly:0.0.6)' bash ${JOB_SCRIPT}


[Phase]
cns_p_ctg_fasta = ./4-polish/polished_p_ctgs.fasta
cns_h_ctg_fasta = ./4-polish/polished_h_ctgs.fasta
reads_1=/gscmnt/gc2758/analysis/reference_grant/HiC_Data_Ed_Green_Lab/HG02723_Gambian/All_R1.fastq.gz
reads_2=/gscmnt/gc2758/analysis/reference_grant/HiC_Data_Ed_Green_Lab/HG02723_Gambian/All_R2.fastq.gz
min_aln_len=3000
iterations=10000000
enzyme="GATC,GAATC,GATTC,GAGTC,GACTC"
output_format=pseudohap
```

Setup pb-assembly environment in terminal (Step 5 above) and launch command as LSF job:

```
$ bsub -oo pb_phase.log -R "rusage[mem=20000] span[hosts=1]" -q research-hpc -a 'docker(halllab/pbassembly:0.0.6)' fc_phase.py fc_phase.cfg
```

This stage took approximately 96 hours or 4 days to run from start to finish.

The output from this stage are two sets of phased haplotigs in fasta format: **phased.0.fasta & phased.1.fasta**

**HG02723 phased.0.fasta Assembly Statistics**
```
  #CONTIGS  1,780           
  LENGTH    2,890,378,344 bp
  AVG       1,623,808 bp        
  N50       20,629,600 bp
  LARGEST   80,753,658 bp       
  Contigs > 1M: 236 ( 2,725,163,374 bp ) 94.3%
  Contigs 250K--1M: 164 ( 80,715,540 bp ) 2.8%
  Contigs 100K--250K: 269 ( 41,853,621 bp ) 1.4%
  Contigs 10K--100K: 1111 ( 42,645,809 bp ) 1.5%
  Contigs 5K--10K: 0 ( 0 bp ) 0%
  Contigs 2K--5K: 0 ( 0 bp ) 0%
  Contigs 0--2K: 0 ( 0 bp ) 0%
```
**HG02723 phased.1.fasta Assembly Statistics**
```
  #CONTIGS  1,780
  LENGTH    2,889,791,364 bp
  AVG       1,623,478 bp
  N50       20,622,770 bp
  LARGEST   80,746,993 bp
  Contigs > 1M: 236 ( 2,725,408,095 bp ) 94.3%
  Contigs 250K--1M: 163 ( 80,242,266 bp ) 2.8%
  Contigs 100K--250K: 268 ( 41,399,963 bp ) 1.4%
  Contigs 10K--100K: 1113 ( 42,741,040 bp ) 1.5%
  Contigs 5K--10K: 0 ( 0 bp ) 0%
  Contigs 2K--5K: 0 ( 0 bp ) 0%
  Contigs 0--2K: 0 ( 0 bp ) 0%
  ```
