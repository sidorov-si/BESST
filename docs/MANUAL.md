BESST
======

INSTALLATION
----------------
See docs/INSTALL.md.

TROUBLESHOOTING Q&A
------------------

######  What aligner should I use?
BESST requires only a sorted and indexed bamfile -- your favourite aligner can be used. However, we have had the best experience with BWA-mem using default parameters on most data used in our evaluations.

######  What aligner options are good/bad?
BESST only work with uniquely aligned reads for now (under development to consider probability distribution over read placements). The uniqueness is detected in the sorted BAM file by looking at the flag. Therefore, it is NOT suitable to specify any parameter that makes the aligner output several (suboptimal) alignments of a read, and reporting the read as mapped to multiple locations. Bowtie's "-k <int>" (with int other than 1) is an example of a parameter that is not compatible with BESST. 

######  My mate pair libraries contains paired-end contamination, what should i do?
Good! BESST can handle paired-end contamination and results can even be improved it these are not removed before alignment. Removal of adapter sequence and trimming of reads (e.g. chimeric reads due to adapter in the middle of one read in the read pair are still good to do using you favourite tool (the effect of this has not been investigated). So, BESST will have better performance when scaffolding with all reads classified as mate-pairs, paired-ends and "unknown" ( definition from the trimmer [NxTrim](https://github.com/sequencing/NxTrim) ) are left in the read files. Simply leave them as they are in the original fast(a/q) files so that BESST can infer the original orientation and contamine rate from the alignments. (To see that a sound distribution and rate is estimated, see next question)


######  BESST does not scaffold anything, what is going on?

A lot of debugging can be done immediately in "< outputfolder >/BESST_output/Statistics.txt" file where outputfolder is specified with -o when running BESST. BESST outputs statistics of insert size distribution(s) (mate-pair and PE-contamination) as well as coverage statistics and how many read pair are used for scaffolding (for each library). Below is an example of the beginning of a file. Make sure library distribution, contamine distribution and rate looks ok, as well as coverage statistics. If this does not resolve issues or you are unsure about how to proceed, please mail **ksahlin@kth.se**.

```sh
PASS 1


Mean before filtering : 2471.05005763
Std_est  before filtering:  1621.22128565
Mean converged: 2454.01278202
Std_est converged:  1438.84572664
Contamine mean before filtering : 383.841809945
Contamine stddev before filtering:  541.063248334
Contamine mean converged: 371.132937665
Contamine std_est converged:  105.713828422

LIBRARY STATISTICS
Mean of library set to: 2454.01278202
Standard deviation of library set to:  1438.84572664
MP library PE contamination:
Contamine rate (rev comp oriented) estimated to:  0.228445050646
lib contamine mean (avg fragmentation size):  371.132937665
lib contamine stddev:  105.713828422
Number of contamined reads used for this calculation:  106857.0
-T (library insert size threshold) set to:  8209.39568857
-k set to (Scaffolding with contigs larger than):  8209.39568857
Number of links required to create an edge:  5
Read length set to:  100.38
Relative weight of dominating link set to (default=3):  3

Time elapsed for getting libmetrics, iteration 0: 41.3993628025

Parsing BAM file...
L50:  37 N50:  42455 Initial contig assembly length:  4588376
Nr of contigs that was singeled out due to length constraints 78
Time initializing BESST objects:  0.00225305557251
Nr of contigs/scaffolds included in scaffolding: 130
Total time elapsed for initializing Graph:  0.00791215896606
Reading bam file and creating scaffold graph...
ELAPSED reading file: 30.2451779842
NR OF FISHY READ LINKS:  0
Number of USEFUL READS (reads mapping to different contigs uniquly):  434284
Number of non unique reads (at least one read non-unique in read pair) that maps to different contigs (filtered out from scaffolding):  29267
Reads with too large insert size from "USEFUL READS" (filtered out):  325897
Number of duplicated reads indicated and removed:  4890
Mean coverage before filtering out extreme observations =  69.01403104
Std dev of coverage before filtering out extreme observations=  61.5185426061
Mean coverage after filtering =  51.1057077906
Std coverage after filtering =  17.4310506587
Length of longest contig in calc of coverage:  106467
Length of shortest contig in calc of coverage:  8270
```



OBS:
----
#### Common pitfall: ####

If --orientation is not specified, BESST assumes that all libraries was aligned in fr orientation.
(In versions less than 1.2 BESST cannot parse rf orientations. Thus, BESST requires reads to be mapped in FR mode, i.e. --->  <---, matepairs thus need to be reverse complemented.)


#### Time and memory requirements: ####
Version 1.3 and later have implemented several major improvements in runtime and memory requirements. These fixes has the most effect on large fragmented assemblies with hundereds of thousands to millions of contigs.

#### Bam files and mapping  ####
BESST requires sorted and indexed bamfiles as input. Any read aligner + samtools can be used to obtain such files. Read pairs needs to be aligned in paired read mode. BESST provides a script (https://github.com/ksahlin/BESST/blob/master/scripts/reads_to_ctg_map.py) for obtaining sorted and indexed bamfiles with BWA-mem or BWA-sampe in one go. An example call for mapping with this script is

```sh
python reads_to_ctg_map.py /path/to/lib1_A.fq /path/to/lib1_A.fq /path/to/contigs.fasta --threads N
```
where N is an integer specifying how many threads BWA-mem should use. --nomem can be specified to the above call to use BWA-sampe as the paired read alignment pipeline.
 
INPUT:
------
Required arguments:

* -c < path to a contig file >

* -f < path to bamfiles >  (increasing order of insert size)

* -o < path to location for the output >

Highly reccomended argument:

* --orientation < fr/rf one for each library >

EXAMPLE RUN:
-----------
For scaffolding with one PE and one MP library:
```sh
runBESST -c /path/to/contigfile.fa -f /path/to/file1.bam /path/to/file2.bam -o /path/to/output --orientation fr rf
```
If the mate pair library was reversed complemented before it was aligned, '--orientation fr fr' should be specified.

Optional arguments:
-------------------

The following arguments are computed internally / set by BESST. It is however good to specify mean and standard deviation if your assembly is very fragmented compared to the library insert size (not enough large contains to compute library statistics on).

#### Library parameters: ####

* -m < the means of the insert sizes of the library, one integer number for each library > (integer numbers)

* -s < standard deviation of the libraries, one integer number for each library> (integer numbers)
 
* -T < Thresholds that are some upper levels that you think not too many PE/MP will have a longer insert size than (in the end of the mode) > (integer numbers) 

* -r < Mean read length for each of the libraries > (integer number) 

* -z <ints> Coverage cutoff for repeat classification ( e.g. -z 100 says that contigs with coverages over 100 will be discarded from scaffolding). Integer numbers, one for each library) 

* -d Check for sequencing duplicates and count only one of them (when computing nr of links) if they are occurring.

#### Contig parameters: ####

* -k <ints> Minimum contig size to be seen as a "large contigs" (for statistical scoring only). One number for each library.

* --filter_contigs <int> Remove contigs smaller than this value from all scaffolding. These contigs are not even incuded in the outpus of BESST.


#### Algorithm parameters: ####

* --iter <int> Maximum number of iterations in BFS search for paths between scaffolds.

* --max_extensions <int> Maximum number of extensions performed between contig/scaffold ends.

* -e <int> The least amount of witness links that is needed to create a link edge in graph (one for each library)

* -y Extend scaffolds with smaller contigs (default on).

* --no_score Do not perform statistical scoring, only run path search between contigs.

* --score_cutoff <float> Only consider paths with score > score_cutoff (default 1.5)



#### Under construction / proven unstable ####

* -g < Haplotype detection function on or off > ( default = 0 (off) <0 or 1> )

* -a < Maximum length difference ratio for merging of haplotypic regions> (float nr)
 
* -b < Nr of standard deviations over mean/2 of coverage to allow for clasification of haplotype> (integer value) 

* -q < optinal flag > Parallellize work load of path finder module in case of multiple processors available using multiprocessing library for pyhton.


NOTE:
-------

1. Definition of insert size: BESST assumes the following definitions: 
  * insert size = fragment length. 
  * For PE: 
```
   s                    t
   ------>      <-------
```
from s to t, that is, insertsize = readlen1 + gap + readlen2. So when mean and/or threshold is supplied, it should be mean and threshold of this distance.

2. Mapping reads: If you want to map a mate pair library, you will need to map them as paired end library i.e. forward-reverse mode. All read pair libraries should be in this order.

3. Order of scaffolding: It is crucial for the algorithm that you give the libraries in increasing order of the insert size.


