# hifi_trimmer

`hifi_trimmer` is a command-line tool for filtering and trimming extraneous adapter hits 
from a HiFi read set using a BLAST search against a fasta file of adapter sequences. It is 
designed to be highly configurable, with per-adapter settings to determine actions if 
the adapter is found at the ends of a read or in the middle. To improve reproducibility,
the primary output of the tool is a BED file that describes the region of each read to be
excluded. The tool also includes a command to filter the reads to disk using the produced
BED file.

Although there is no explicit option to set the number of threads, both the polars backend
for BLAST file processing and the bgzip backend for writing filtered FASTA files are able
to use additional CPUs available to increase processing and compression speed.

## Installation

```
git clone git@github.com:sanger-tol/hifi-trimmer.git
cd hifi-trimmer
pip install .
```

## Usage

To process a blast output TSV to a BED file:

```
Usage: hifi_trimmer process_blast [OPTIONS] BLASTOUT ADAPTER_YAML

  Processes the input blastout file according to the adapter yaml key.

  BLASTOUT: tabular file resulting from a BLAST query of a readset against a
  BLAST database of adapter sequences, run with -outfmt "6 std qlen". If the
  qlen column is missing, lengths can be calculated by passing the --bam
  option.

  ADAPTER_YAML: yaml file containing a list with the following fields per
  adapter:

  - name: (name of adapter. can be a regular expression)
    discard_middle: True/False (discard read if adapter found in middle)
    discard_end: True/False (discard read if adapter found in end)
    trim_end: True/False (trim read if adapter found in end)
    middle_pident: int (minimum pident requred to identify adapter in middle of read)
    middle_length: int (minimum match length required to identify adapter in middle of read)
    end_pident: int (minimum pident requred to identify adapter in end window)
    end_length: int (minimum match length requred to identify adapter in end window)

  Output: By default, writes BED to [prefix].bed, and a summary file with
  counts of adapter hits detected and filtered to [prefix].summary.

Options:
  -p, --prefix TEXT               Output prefix for results. Defaults to the
                                  basename of the blastout if not provided.
  -b, --bam FILENAME              If blastout file has no read length field, a
                                  BAM file of reads to get read lengths
  -ml, --min_length_after_trimming INTEGER
                                  Minumum length of a read after trimming the
                                  ends in order not to be discarded
  -el, --end_length INTEGER       Window size at either end of the read to be
                                  considered as 'ends' for searching
  -hf, --hits                     Write the hits identified using the given
                                  adapter specifications to TSV. The format is
                                  standard BLAST outfmt 6 with the following
                                  extra columns: read_length, discard (bool),
                                  trim_l (bool), trim_r (bool)
  --no-summary BOOLEAN            Skip writing a summary TSV with the number
                                  of hits for each adapter
  --help                          Show this message and exit.
```

To filter a bam file using the BED file:

```
Usage: hifi_trimmer filter_bam [OPTIONS] BAM BED OUTFILE

  Filter the reads stored in a BAM file using the appropriate BED file
  produced by blastout_to_bed and write to a bgzipped fasta file.

  BAM: BAM file in which to filter reads

  BED: BED file describing regions of the read set to exclude.

  OUTFILE: File to write the filtered reads to (bgzipped).

Options:
  --help  Show this message and exit.
```

## Example

First, BLAST your reads against an adapter database:

```
blastn -query <(samtools fasta /path/to/bam) \
  -db /path/to/adapter/blast/db \
  -reward 1 -penalty -5 -gapopen 3 -gapextend 3 \
  -dust no -soft_masking true -evalue 700 \
  -searchsp 1750000000000 \
  -outfmt "6 std qlen" |\
  bgzip > blastout.gz
```

To create a BED file, you then need to create a YAML file describing the actions to take for each adapter. The adapter name
can be a regular expression, but note that each adapter name in the BLAST file must match only one entry in
the YAML.

```
- adapter: "^NGB00972"  // regular expression matching adapter names
  discard_middle: True  // discard read if adapter is found in the middle
  discard_end: False    // discard read if adapter found at end
  trim_end: True        // trim read if adapter is found at end (overridden by discard choice)
  middle_pident: 95     // minimum percent identity for a match in the middle of the read
  middle_length: 44     // minimum match length for a match in the middle of the read
  end_pident: 90        // minimum percent identity for a match at the end of the read
  end_length: 18        // minimum match length for a match at the end of the read
```

Then run `blastout_to_bed` to generate a BED file:

```
hifi_trimmer process_blast /path/to/blastout.gz /path/to/yaml
```

Then filter the bam file using the BED file: 

```
hifi_trimmer filter_bam /path/to/bam /path/to/bed /path/to/final/fasta.gz
```
