# Dillinger
Phased-SV
=========

Local assembly based SV detection using single-molecule sequencing reads
and a phased SNV VCF file. Sample data to run this is available at

http://ftp.1000genomes.ebi.ac.uk/vol1/ftp/data_collections/hgsv_sv_discovery/working/20180521_PhasedSVSampleData/

Summary
-------

This software pipeline performs SV calling with four main steps:
1. Local assembly of haplotype-partitioned reads.
2. Merging of local assemblies into a reference-guided assembly.
3. Mapping merged assemblies to the reference.
4. Filtering SVs by read-support of breakpoints.


For installation, please refer to Install.md


Running PhasedSV
----------------

1. Data generation.
Phased-SV assumes you have BAM files of reads aligned to the
reference, and a vcf file of phased SNVs. A minimum of 40X works
best. To run a test on chromosome 22, you can download data listed in
SampleData.txt. If you are generating bams from scratch, you can
reference the README.md in pbsamstream for the commands to generate
correctly formatted bam files.
2. Configuration.

   2.1.  Setup python environment. It is easiest to ensure
    compatibility with python modules if virtual environments are
    used.  A utility script is provided to create and populate the
    virtualenv with the required python modules for running phasedsv.
    You can create the module using
 ```source setup_virtualenv.sh```

   2.2. `local_assembly/Configure.mak`:  This sets up variables
  used in the make files that run local assemblies. They need to point
  to the reference (indexed by samtools faidx and blasr sawriter), and
  the canu installation. The value of "READ_SOURCE" should be set to
  "HGSVG_BAM" if running PacBio RSII alignments, and anything else (or
  not set) otherwise. The template
  `local_assembly/Configure.mak.template` may be used  to create
  `local_assembly/Configure.mak`. 
  
   2.3. Create a BAM fofn.  This should be a file of complete paths to
	the bam or bams if the alignments are split into multiple bams. If
	the parents are being used for a trio assembly, there should be
	similar file for reads from each parent.
	
3. Run local assemblies.
  3.1. Assembly parameter file
Create a parameter file describing the source data for the
file, with key=value format for assigning variables in BASH. This
requires the following keys: 
   `REF`: The full path to the reference reads are aligned to. 
  `BAMS`: The path to the bam file of file names. This is one line per
	   bam, with the full path to the file. 
  `VCF`: The VCF file that has the phased SNVs. 
  `SAMPLE`: The name of the sample that is being assembled (the sample ID
	     in the VCF file).
  `DEST` : The top level directory where the assemblies, alignments, and records will go.
  An example file is given in assembly_parameters.template.

   3.2 Defining regions.
 Define the regions that will be assembled. These can be copied
 from `hgsvg/regions/Windows.60kb-span.20kbp-stride.txt`.  In general
 they are in the format chrom.start-end. 
 3.3 Trio assembly.
	   If you are running a trio assembly, you need to generate bed
	   files that contain the regions which the parental reads may be
	   unambiguously assigned.  The example below uses the phased vcf
	   for the Puerto Rican family.
`hgsvg/phasing/DetermineInheritance.py  --vcf data/rgn1.vcf.gz --child HG00733 --fa HG00731 --mo HG00732 --faBed fa.bed --moBed mo.bed`

   3.3 Run local assemblies. This is a computationally intensive task,
 and is best ran on a cluster. For every line in the regions file, a
 local assembly must be generated. For single-sample assemblies, the
 command is RunTiledAssembly.sh. There is a helper script that will
 generate grid commands for SGE and SLURM in `local_assembly/grid_scripts/ConfigureGridScripts.py`:
```
usage: ConfigureGridScripts.py [-h] --regions REGIONS --params PARAMS
                               [--runTrio] --grid {sge,uge,slurm}
                               [--base BASE] [--conc CONC] [--config CONFIG]

Prepare submission scripts for sge, uge, or slurm cluster management systems.

optional arguments:
  -h, --help            show this help message and exit
  --regions REGIONS     Full path to regions.
  --params PARAMS       Path to parameter file
  --runTrio             Prepare commands for trio assembly.
  --grid {sge,uge,slurm}
                        Grid type
  --base BASE           base name for grid commands
  --conc CONC           Number of concurrent jobs
  --config CONFIG       Extra configuration parameters for job submission
```
   For example, with the test data on SGE:
`local_assembly/grid_scripts/ConfigureGridScripts.py  --regions PhasedSVTestData/regions.txt  --params assembly_parameters.trio --runTrio --grid sge --base asm --conc 50`

After this, you can submit jobs running `source ./asm.submit.sh`

4. Calling variation.
First you will need to configure the directory where the varint
calling snakefiles run. The base directory will be in the value of the
DEST directory of the assembly_parameters file, which by default is
`asm`.
Calling variation happens in two steps: (1) stitching and aligning
local assemblies, and (2) filtering variants from local assembly
alignments using the snakefiles `hgsvg/stitching/Stitching.snakefile` and
`hgsvg/stitching/SVQC.snakefile`.

   4.1 Setup configuration script `phasedsv.json`. This json is used by
both snakefiles.
There is a template in `hsvg/phasedsv.json.template`. The values that
should be specified are:
 `ref`: path to reference reads are aligned to.
 `sample`: the name of the sample, used in plotting.
 `bams`: the path to the child bams fofn.
 `cov_cutoff` minimal coverage for read-back SV filtering. The default
 of 3 was determined empirically for 40-fold sequence coverage.
 `inversions` a path to a file of inversions detected in this
 sample, or a blank file.
 `tr_cluster_size` Condense clusters of SVs in tandem repeats that
 have this value or greater number of SVs.

   4.2. Setup grid configuration `grid.json`.
   This json file gives the parameters for submitting jobs to the
   cluster. Two tempaltes are available:
   files: `hgsvg/grid.json.sge` and `hgsvg/grid.json.slurm`.

   If you have SGE or SLURM, copy the corresponding file to `grid.json`
   in the working directory.

   4.3. Run the stitching and SVQC snakefiles.
   `snakemake -p -s hgsvg/stitching/Stitching.snakefile`
   `snakemake -p -s hgsvg/stitching/SVQC.snakefile`

	 If you want to distribut this on a cluster, use -j and --cluster
	 "{params.grid_opts}", along with any necessary --jobscript
	 parameters.   
