A hifiasm fork for metagenome assembly using Hifi reads.

## Getting Started
```sh
# Install hifiasm-meta (g++ and zlib required)
git clone https://github.com/xfengnefx/hifiasm-meta.git
cd hifiasm-meta && make

# Run
hifiasm_meta -t32 -o asm reads.fq.gz 2>asm.log
hifiasm_meta -t32 --force-rs -o asm reads.fq.gz 2>asm.log  # if the dataset has high redundancy
```
A test dataset and the assembled results are available at [zenodo](https://zenodo.org/record/5908204). 
This is downsampled from SRR13128014 (zymoBIOMICS D6331 mock community), containing (only the) 5 E.coli strains.
Hifiasm-meta r57 takes roughly 5 minutes and a peak memory of 18GB. 

## About this fork

Hifiasm\_meta comes with a read selection module, which enables the assembly of dataset of high redundancy without compromising overall assembly quality, and meta-centric graph cleaning modules.
In post-assembly stage, hifiasm\_meta traverses the primay assembly graph
and try to rescue some genome bins that would be overlooked by traditional binners.
Currently hifiasm\_meta does not take bining info.

## Output files

Contig graph: asm.p\_ctg\*.gfa and asm.a\_ctg\*.gfa

Raw unitig graph: asm.r\_utg\*.gfa

Cleaned unitig graph: asm.p\_utg\*.gfa 

Contig name format: `^s[0-9]+\.[uc]tg[0-9]{6}[lc]`, where the `s[0-9]+` is a disconnected subgraph label of the contig. It might be useful to be able to quickly checking whether two contigs are in the same disconnected subgraph (i.e. haplotype that wasn't assembled in to a single contig, tangled haplotypes).


## Special Notes

Based on the limited available test data, real datasets are unlikely to require read selection; mock datasets, however, might need it.

Bin file is one-way compatible with the stable hifiasm for now: stable hifiasm can use hifiasm\_meta's bin file, but not vice versa. Meta needs to store extra info from overlap & error correction step.

## Switches

See also README\_ha.md, the stable hifiasm doc.

```
# General options
-o              Prefix of output files [hifiasm_meta.asm]. 
                For detailed description of all assembly graphs, 
                 see above or manpage.
-B	        	Use bin files under a different prefix than the 
                 one specified by -o.
-t              Number of CPU threads used by hifiasm\_meta (default: 1).
-h              Show help information and exit. Returns 0.
--version       Show version number and exit. Returns 0.

# Read selection options
-S              Enable read selection.
                If enabled, hifiasm_meta will estimate the total number of 
                 read overlaps. If the estimation seems within acceptable, 
                 no read will be dropped; otherwise, reads will be dropped 
                 from the most redundant ones until the criteria are satisfied.
--force-rs      Force read selection. Read will be dropped according to the 
                 runtime kmer frequency threshold described below.
--lowq-10       Runtime 10% quantile kmer frequency threshold.
                Lower value means less reads kept, if read selection is triggered. [50]
--lowq-5        Runtime 5% quantile kmer frequency threshold.
                Lower value means less reads kept, if read selection is triggered. [50]
--lowq-3        Runtime 3% quantile kmer frequency threshold.
                Lower value means less reads kept, if read selection is triggered. [disabled]

# Error correction options
-k              K-mer length [51]. This option must be less than 64.
-w              Minimizer window size [51].
-f              Number of bits for bloom filter; 0 to disable [37]. 
                This bloom filter is used to filter out singleton k-mers 
                 when counting all k-mers. 
-r              Rounds of haplotype-aware error corrections [3]. 
                This option affects all outputs of hifiasm\_meta.
--min-hist-cnt  When analyzing the k-mer spectrum, ignore counts below INT [5].

# Assembly options
-z              Length of adapters that should be removed [0]. 
                This option remove INT bases from both ends of each read.
-i              Ignore error corrected reads and overlaps saved in bin files.

# Debugging options
--dbg-gfa       Use extra bin files to speed up the debugging of graph cleaning.
                If set and the extra bin files do not already exist, 
                 assembly runs normally (i.e. from scratch or resume from bin files) 
                 and writes the extra bin files.
                If set and bin files as well as extra bin files are present, 
                 assembly will resume from raw unitig graph stage.
--dump-all-ovlp Dump all overlaps ever calculated during the final overlaping. 
--write-paf     Dump overlaps, produces 2 files, one contains the intra-haplotype or unphased overlaps, the other contains inter-haplotype overlaps. If coverage is very high, this might not be the full set of overlaps.
--write-ec      Dump error corrected reads.
-e              Ban assembly, i.e. terminate before generating string graph. 
```


## Preliminary results (r49)


We evaluated hifiasm-meta on the following public datasets:

|                  | accession   | #bases (Gb) | N50 read<br>length (kb)| Median read QV | Sample description                               |
|------------------|-------------|-------------|-----------------------|----------------|--------------------------------------------------|
| ATCC        | SRR11606871 | 59.2        | 12.0                  | 36             | Mock, ATCC MSA-1003                     |
| zymoBIOMICS | SRR13128014 | 18.0        | 10.6                  | 40             | Mock, ZymoBIOMICS D6331                 |
| sheepA           | SRR10963010 | 51.9        | 14.3                  | 25             | Sheep gut microbiome                             |
| sheepB           | SRR14289618 | 206.4       | 11.8                  | N/A*            | Sheep gut microbiome                             |
| humanO1          | SRR15275213 | 18.5        | 11.4                  | 40             | Human gut, pool of 4 omnivore samples |
| humanO2          | SRR15275212 | 15.5        | 10.3                  | 41             | Human gut, pool of 4 omnivore samples |
| humanV1          | SRR15275211 | 18.8        | 11.0                  | 39             | Human gut, pool of 4 vegan samples    |
| humanV2          | SRR15275210 | 15.2        | 9.6                   | 40             | Human gut, pool of 4 vegan samples    |
| chicken          | SRR15214153 | 33.6        | 17.6                  | 30             | Chicken gut microbiome                           |

*Base quality was not available for this dataset.

In the empirical datasets, we evaluated assemblies with [checkM](https://github.com/Ecogenomics/CheckM). Following the convention,
we define near-complete as having at more than 90\% checkM completeness score and less than 5\% contamination score.
High-quality is defined as >70\% complete and <10\% contaminated.
Medium-quality is defined as >50\% complete and QS>50, 
where QS (quality score) is given by `completeness-(5*contamination)`.
Binning was performed with metabat2. 
Additionally, we split out any >1Mb circles from genome bins and let them form bins on themselves.

|             | >1Mb circular contigs | >1Mb circular contigs,<br>near-complete | Near-complete MAGs | High-quality MAGs | Medium-quality MAGs |
|-------------|-----------------------|--------------------------------------|--------------------|-------------------|---------------------|
| sheepA      | 139                   | 125                                  | 186                | 42                | 33                  |
| sheepB      | 245                   | 219                                  | 377                | 55               | 47                 |
| chicken     | 69                    | 57                                   | 87                 | 20                | 15                  |
| humanO1     | 33                    | 27                                   | 53                 | 20                | 19                  |
| humanO2     | 26                    | 23                                   | 48                 | 17                | 16                  |
| humanV1     | 38                    | 33                                   | 73                 | 23                | 15                  |
| humanV2     | 34                    | 27                                   | 53                 | 22                | 17                  |
| humanPooled | 75                    | 62                                   | 109                | 39                | 41                  |


A [Bandage](https://github.com/rrwick/Bandage) plot of sheepA's primary contig graph (screenshot omitted some small unconnected contigs at the bottom):

<p align="center">
  <img src="https://user-images.githubusercontent.com/79302051/137414543-e1eed925-6fa8-49e8-9ae6-b7d0d1353cd2.png"/>
</p>

ATCC contained 20 species and zymoBIOMICS contained 21 strains of 17 species.
Hifiasm-meta recovered 14 out of 15 abundant (0.18\%-18\%) species in ATCC as single complete contigs. 
The other 5 rare species had insufficient coverage to be fully assembled. 
The challenge of the zymoBIOMICS dataset is its mixture of 5 _E.coli_ strains (8\% abundance each).
Hifiasm-meta assembled strain B766 into a complete circular contig, 
strain B3008 into 2 contigs and the rest as fragmented contigs.

The two mock datasets were assembled with `--force-rs -A`, the rest used default. Performance on 48 threads (`-t48`):

|             | Wall clock (h) | PeakRSS (Gb) |
|-------------|------------|---------|
| ATCC        | 22         | 323     |
| zymoBIOMICS | 5.3        | 131     |
| sheepA      | 17.8       | 208     |
| sheepB      | 214        | 724     |
| chicken     | 15.8       | 201     |
| humanO1     | 3          | 70      |
| humanO2     | 2.3        | 69      |
| humanV1     | 3.4        | 76      |
| humanV2     | 2.2        | 62      |
| humanPooled | 18         | 224     |
