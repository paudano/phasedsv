Installing Phased-SV.

1. Requirements:

Building:
  gcc >= 7.0
  CMAKE >= 1.10
	ninja >= 1.8
	swig >= 3.0.5

Running:
  samtools >= 1.7
	bedtools >= 2.27
	bioawk >= 20110810
  Canu >= 1.5
	ucsc genome browser tools
	vt >= 0.57


These should all be accessible in your path.

2. Make binaries. The full tree and dependencies will be built by typing

`make` in the phasedsv directory.  This takes some time, and necessary
modules will be downloaded. 

It may be necessary to configure your build. In particular, if the g++ >= 7.0 is not the default directory, set the environment variables:

CC=/path/to/gcc
CXX=/path/to/g++



