# Overview

_Note: this page is actively under development and represents developmental features_

This page describes the additional modules dedicated to detecting and quantifying mtDNA deletions from sequencing data. We've most thoroughly evaluated our computational approach using the mscATAC-seq approach, but the general approach can be applied in other settings, such as exome sequencing data. Overall, three main execution steps are required: 1) deletion detection; 2) single-cell preparation (optional but suggested); 3) quantification of deletion heteroplasmy.

## Step 1. Detect deletions in bulk .bam file

The first step is to identify deletions. The `mgatk-del-find` executable takes in a `.bam` file (presumably containing many single-cells for a single-cell assay) and returns the most likely positions that are participating in a deletion. An example of this functionality is available in `tests` from this repository. Execute the command like so:

```
mgatk-del-find -i barcode/pearson_bci.bam
```

This will produce three output files in the same file path as the original `.bam` file, including two plots and one raw data table. First, let's look at the output data table:

`head barcode/pearson_bci.clip.tsv`

```
position	coverage	clip_count
16569	289	286
1	235	201
13095	526	192
6073	450	178
3571	391	74
10934	276	72
301	383	67
13753	644	53
296	504	48
```

This file shows the number of reads that cover each position as well as the number of reads that are clipped at the particular position, sorted by the number of clipped reads. Unsurprisingly, positions 1 and 16569 come near the top, which is to be expected as the mtDNA is circular and is linked at these two positions. While this table is good for parsing and downstream analyses, we include a summary visualization in the `pearson_bci.clipped_viz.pdf` file to rapidly determine potential mtDNA deletion points:

![clips](https://user-images.githubusercontent.com/16229089/70869377-52812580-1f58-11ea-8e23-9bc88df72b81.png)

Here, the plot shows the positions of the mtDNA genome along the x-axis and the number of clipped reads where a clip occurs at the particular position. The top ten most clipped loci are shown with the position annotated (excluding the circular genome junction in the d-loop). After evaluating several different datasets for putative mtDNA deletions, were observed recurrent positions that are nominated, which are shown in grey. In other words, we've identified several positions that correspond to a "filter list" that we de-emphasize with this color scheme. From this visualization, we can see that `6073-13095` is the most likely deletion for this sample. 

We can verify that the purported clips correspond to a loss of coverage in the putative deleted area by looking at the `pearson_bci.coverage_viz.pdf` file:

![coverage](https://user-images.githubusercontent.com/16229089/70869376-52812580-1f58-11ea-95ed-71e17819f7fa.png)

Overall, these images strongly implicate the `chrM:6073-13095` region as an mtDNA deletion in this sample. 

<br>

## Step 2. Process mtDNA sequencing data using mgatk (optional)

To quantify heteroplasmy in individual samples/cells for specific deletions, we utilize the `mgatk-del` module, which is shown in **Step 3**. The input that module is a folder of quality-controlled `.bam` files, where it is assumed that one cell is present in each `.bam` file, and we further recommend removing PCR duplicates. If your sample comes from a 10x mscATAC-seq run, we generate these files as an intermediate, which can be accessible using the regular `mgatk` execution (see here in the wiki for more information). Critically, one must throw the `-qc` or `--keep-qc-bams` flag to save these intermediate files as they are ordinary deleted in the standard execution. An example execution is shown below: 

```
mgatk bcall -i barcode/pearson_bci.bam -o pearson_bci -n pearson_bci -c 4 \
     -g rCRS -qc -bt CB -b barcode/pearson_barcodes.tsv 
```

In brief, this produces all of the standard output for an `mgatk` execution but further results in a folder (`pearson_bci/qc_bam`) that can be used as the input 

For clarity, this step is optional as one may prepare `.bam` files in other workflows, but for genotyping with `mscATAC-seq` assay, we recommend executing the `mgatk` command similar to that shown above. 

<br>

## Step 3. Genotype single cells with mgatk-del

Finally, after the mtDNA deletions are identified (**Step 1**) and the input files are pre-processed (**Step 2**), we can execute `mgatk-del`. In brief, the tool requires an input (`-i`) of a folder of `.bam` files and the left (`-lc`) and right (`-rc`) coordinates of putative mtDNA deletions for quantification. For our example on this page, we recommend the following execution:

```
mgatk-del -i pearson_bci/qc_bam -n mainDel -o pearson_bci -lc 6073 -rc 13095
```

By specifying the output folder (`pearson_bci`) to be the same as what was declared in **Step 2**, the `.deletion_heteroplasmy.tsv` file will be present in the `/final/` folder with the other genotyping files. We recommend this for convenience in organization, but the `-o` argument can be any output folder destination. 

Here, `mgatk-del` produces only a single output file, which is shown here:

```
cat pearson_bci/final/mainDel.deletion_heteroplasmy.tsv 
cell_id	heteroplasmy	reads_del	reads_wt	reads_all	deletion
CTAACTTAGAGCCACA-1	40.91	72	104	176	del6073-13095
GCCTAGGCAGTTCGGC-1	45.83	66	78	144	del6073-13095
CACCACTAGGAGGCGA-1	34.17	68	131	199	del6073-13095
```

Each line shows a unique cell/deletion combination and includes all relevant information about the deletion heteroplasmy, including the number of reads supporting a "wild-type" allele and the number supporting a deletion allele. The heteroplasmy is defined as the ratio of `reads_del` to `reads_all` multiplied by `100%`. 

We can also specify the quantification of multiple deletions by supplying multiple coordinates to the `-lc` and `-rc` flags like so:

```
mgatk-del -i pearson_bci/qc_bam -n wNonsense -o pearson_bci \
     -lc 6073,6000 -rc 13095,14000
```

This is particularly useful for complex single-cell experiment settings or when evaluating many different potential deletions. We can then examine the output:

```
cat pearson_bci/final/wNonsense.deletion_heteroplasmy.tsv 
cell_id	heteroplasmy	reads_del	reads_wt	reads_all	deletion
CTAACTTAGAGCCACA-1	40.91	72	104	176	del6073-13095
CTAACTTAGAGCCACA-1	0	0	175	175	del6000-14000
GCCTAGGCAGTTCGGC-1	45.83	66	78	144	del6073-13095
GCCTAGGCAGTTCGGC-1	0	0	169	169	del6000-14000
CACCACTAGGAGGCGA-1	34.17	68	131	199	del6073-13095
CACCACTAGGAGGCGA-1	0	0	218	218	del6000-14000
```

Again, we see that each line is a unique cell/deletion combination with all appropriate summary statistics. 

Here, we verify that the heteroplasmy is identically zero for all cells for the `del6000-14000`, which is expected as these positions had no evidence of contributing to a deletion from the analyses upstream. We emphasize that the deletions are parsed from `-lc` and `-rc` such that the order of the arguments for both parameters correspond to one deletion (*e.g.* a chrM:6073-14000 deletion was not considered since 6073 was first in the `-lc` comma-separated string but 14000 was second in the `-rc` string). Thus, a parameterization of `-lc 6073,6073 -rc 13095,14000` would be required to compute both of the chrM:6073-13095 and chrM:6073-14000 heteroplasmies.  

### Authors
- Caleb Lareau
- Jeffrey Verboon

<br>
