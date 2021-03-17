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
(base) ebelter@blade18-1-16:/gscuser/ebelter$ conda info --env
# conda environments:
#
pb-assembly-0.0.6        /gscmnt/gc2134/finishing/pb-assembly/.conda/envs/pb-assembly-0.0.6
base                  *  /opt/conda

(base) ebelter@blade18-1-16:/gscuser/ebelter$ conda activate pb-assembly-0.0.6
(pb-assembly-0.0.6) ebelter@blade18-1-16:/gscuser/ebelter$ 
```
**Launch jobs to LSF. Here the environment is needed to be preserved, do not include the LSF_DOCKER_PRESERVE_ENVIRONMENT=false environment variable. Below is an example of running falcon unzip.**
```
(pb-assembly-0.0.6) ebelter@blade18-1-16:/gscuser/ebelter$ bsub -q research-hpc -a 'docker(halllab/pbassembly:0.0.6)' fc_unzip.py fcunzip.cfg

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




