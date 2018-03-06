EMA: An aligner for barcoded short-read sequencing data
-------------------------------------------------------

EMA uses a latent variable model to align barcoded short-reads (such as those produced by [10x Genomics](https://www.10xgenomics.com)' sequencing platform). More information is available in [our paper](https://www.biorxiv.org/content/early/2017/11/16/220236). Full [experimental setup](https://github.com/arshajii/ema-paper-data/blob/master/experiments.ipynb) is [available here](https://github.com/arshajii/ema-paper-data/blob/master/experiments.ipynb).

### Download and Compile
In a nutshell:

```
$ git clone --recursive https://github.com/arshajii/ema
$ cd ema
$ make
```

The `--recursive` flag is needed because EMA uses BWA's C API.

### Usage
Input FASTQs must first be preprocessed with `ema preproc` then sorted with `ema sort` (see below for more details).

```
usage: ema <preproc|sort|count|align|help> [options]

preproc: preprocess barcoded FASTQ files
  -1 <FASTQ1 path>: first FASTQ file [required]
  -2 <FASTQ2 path>: second FASTQ file [none]
  -w <whitelist path>: barcode whitelist [required]
  -n <num buckets>: number of barcode buckets to make [20]
  -c <counts file>: preexisting barcode counts [none]

sort: sort preprocessed FASTQs by barcode
  -1 <FASTQ1 path>: first FASTQ file [required]
  -2 <FASTQ2 path>: second FASTQ file [required]

count: performs preliminary barcode count
  -1 <FASTQ1 path>: first FASTQ file [required]
  -w <whitelist path>: barcode whitelist [required]
  -i: indicates FASTQ is interleaved
  -o <output file>: output file [stdout]

align: choose best alignments based on barcodes
  -1 <FASTQ1 path>: first (preprocessed and sorted) FASTQ file [required]
  -2 <FASTQ2 path>: second (preprocessed and sorted) FASTQ file [required]
  -s <EMA-FASTQ path>: specify special FASTQ path [none]
  -x: multi-input mode; takes input files after flags and spawns a thread for each
  -r <FASTA path>: indexed reference [required]
  -o <SAM file>: output SAM file [stdout]
  -R <RG string>: full read group string (e.g. $'@RG\tID:foo\tSM:bar') [none]
  -d: apply fragment read density optimization
  -t <threads>: set number of threads [1]

help: print this help message
```

### Preprocessing

**Note: A much faster preprocessor has been implemented. The method below still works, but will be substantially slower. Details are below**

For large data sets, preprocessing can most easily be done as follows (assuming interleaved FASTQs):

```
$ cat data/*.fastq | ema count -1 - -w /path/to/whitelist.txt -i -o counts_file
$ cat data/*.fastq | ema preproc -1 - -w /path/to/whitelist.txt -c counts_file
```

Then the FASTQs in each bucket (except the no-barcode bucket) must be sorted with `ema sort`.

#### Faster preprocessing 
(Coming soon.)

### Parallelism

EMA supports several modes of parallelism:

1. Single-file parallelism: multiple worker threads work on a single FASTQ (specified with `-t <threads>`).
2. Multi-file parallelism: multiple FASTQs can be specified after the flags, and each will be assigned a thread (enabled with `-x`).
3. Multi-process parallelism: multiple EMA instances can be spawned to work on individual FASTQs.

Options 1 and 2 are advantageous because they use only a single copy of the data structures needed in alignment (i.e. those of BWA), but have a slight overhead due to multithreading.

The different options can be combined: for example, specifying `-t T` _and_ `-x` will spawn `T` threads to work on each input FASTQ.

Option 3 can be enabled by running a separate instance of EMA for each barcode bucket produced during preprocessing (this can also be used to sort the FASTQs with `ema sort`). A script using [GNU Parallel](https://www.gnu.org/software/parallel/) is provided in [`utils`](util/) and can be run as follows (in the same directory as the `bucket` folders):

```
$ EMAPATH=/path/to/ema/executable PICARDPATH=/path/to/picard/jar ./ema_wrapper.sh
```

### Output

EMA outputs a standard SAM file with several additional tags:

- `XG`: alignment probability
- `XC`: cloud identifier
- `XA`: alternate high-probability alignments
