# hifi_trimmer

## Installation

```
git clone git@github.com:prototaxites/hifi-trimmer.git
cd hifi-trimmer
pip install .
```

## Usage

```
Usage: hifi_trimmer process_blast [OPTIONS] BLASTOUT ADAPTER_YAML

  Processes the input blastout file according to the adapter yaml key.

  BLASTOUT: tabular file resulting from BLAST with -outfmt "6 std qlen". If
  the qlen column is missing, lengths can be calculated by passing the --bam
  option.

  ADAPTER_YAML: yaml file contaning a list with the following fields per
  adapters:

  - name: (name of adapter. can be a regular expression)
    discard_middle: True/False (discard read if adapter found in middle)
    discard_end: True/False (discard read if adapter found in end)
    trim_end: True/False (trim read if adapter found in end)
    middle_pident: int (minimum pident requred to identify adapter in middle of read)
    middle_length: int (minimum match length required to identify adapter in middle of read)
    end_pident: int (minimum pident requred to identify adapter in end window)
    end_length: int (minimum match length requred to identify adapter in end window)

  Output: By default, writes BED to standard output. This can be redirected
  with the -o/--output option.

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
                                  adapter specifications to TSV
  --no-summary BOOLEAN            Skip writing a summary TSV with the number
                                  of hits for each adapter
  --help                          Show this message and exit.```

```
Usage: hifi_trimmer filter_bam [OPTIONS] BAM BED OUTFILE

  Filter the reads stored in a BAM file using the appropriate BED file
  produced by blastout_to_bed and write to a bgzipped fasta file.

  BAM: BAM file in which to filter reads BED: BED file describing regions of
  the read set to exclude. OUTFILE: File to write the filtered reads to
  (bgzipped).

Options:
  --help  Show this message and exit.
```

## Example

First, BLAST your reads against an adapter database:

```
blast -query <(samtools fasta /path/to/bam) -db /path/to/adapter/blast/db -reward 1 -penalty -5 -gapopen 3 -gapextend 3 -dust no -soft_masking true -evalue 700 -searchsp 1750000000000 -outfmt "6 std qlen" | bgzip > blastout.gz
```

To create a BED file, you then need to create a YAML file describing the actions to take for each adapter:

```
- adapter: "^NGB00972"     // regular expression matching adapter names
  discard_middle: True     // discard read if adapter is found in the middle
  discard_end: False       // discard read if adapter found at end
  trim_end: True           // trim read if adapter is found at end (overridden by discard choice)
  middle_pident: 95        // minimum percent identity for a match in the middle of the read
  middle_length: 44        // minimum match length for a match in the middle of the read
  end_pident: 90           // minimum percent identity for a match at the end of the read
  end_length: 18           // minimum match length for a match at the end of the read
```

Then run `blastout_to_bed` to generate a BED file (you only need to specify the BAM file if the blastout doesn't have read lengths in column 13):

```
hifi_trimmer blastout_to_bed --bam /path/to/bam /path/to/blastout.gz /path/to/yaml --output /path/to/bed
```

Then filter the bam file using the BED file: 

```
hifi_trimmer filter_bam_to_fasta /path/to/bed /path/to/bam /path/to/final/fasta.gz
```