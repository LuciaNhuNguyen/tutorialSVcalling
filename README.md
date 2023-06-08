# STRUCTURAL VARIANT (SV) CALLING TUTORIAL FOR CLINICAL APPLICATION
---

## Table of contents
- [Calling Structural Variant (SV) and Clinical Application](#variant-annotation---gatk4-funcotator)
  - [Table of contents](#table-of-contents)
  - [SV process](#sv-process)
    - [1. Directory/Data preparation](#1-directorydata-preparation)
    - [2. Upstream analysis](#2-upstream-dna-analysis)
    - [3. Structural variant calling ](#3-structural-variant-calling)
    - [4. Querying VCF files](#4-querying-vcf-files)
    - [5. Visual inspection of SVs](#5-visual-inspection-of-svs)
    - [6. Annotation](#6-annotation)
  - [References](#reference)

## Tools
- Minimap2 to map short read (>100bp) and long reads (PacBio and Nanopore)
- GATK to pre-process data
- Samtools to manipulate BAM files
- Delly to call SVs using short/long reads
- Bcftools to manipulate VCF files with SV calls
- Samplot to visualise and inspect SVs
- SnpEff to annotate SVs

## SV process
```mermaid
flowchart TD
    subgraph "SV CALLING"
    A("Data preparation + Upstream analysis")--> B[[SV Calling]] --> C{{"Querying VCF files <br> Visual inspection of SVs"}} --> D(["Annotation"])
    style A fill:#34495e,stroke:#333,stroke-width:4px,color:#fff
    style B fill:#228B22,stroke:#333,stroke-width:4px,color:#ADD8E6
    style C fill:#767076,stroke:#F9F2F4,stroke-width:2px,color:#fff
    end
```

### **1. Directory/Data preparation**
```bash
# Create output directory
mkdir reference
mkdir raw_data
mkdir output
```

```bash
# Download raw data for BAM files of short reads
Downloading 
From: https://drive.google.com/uc?id=1T0aUdNBS-2NhmFe0ZPDzyIu0YxWheNLo
To: subset.sorted.normal.chr5.bam
Downloading
From: https://drive.google.com/uc?id=1taZrKPkRvsEZnpTunHFzELcQemLqJ04B
To: subset.sorted.cancer.chr5.bam
```

```bash
# Download raw data for CRAM files of long reads
Downloading
From: https://drive.google.com/file/d/11Ycnf1lrDxzwmTV0OmEbcEbomq9URV2R
To: SRR22508184.HG005.PacBio.remove_dup.cram (1.51GB)
```

```bash
# Download reference file for Human genome GRCh38 
wget "https://hgdownload-test.gi.ucsc.edu/goldenPath/hg38/bigZips/analysisSet/hg38.analysisSet.fa.gz" -O reference/hg38.fa.gz
gzip -d reference/hg38.fa.gz
bgzip reference/hg38.fa
samtools faidx reference/hg38.fa.gz

# Download human genome GRCh38. Chromosome 5
wget "https://www.ncbi.nlm.nih.gov/sviewer/viewer.fcgi?db=nuccore&report=fasta&id=CM000667.2" -O hg38.chr5.fa.gz
gzip -d reference/hg38.chr5.fa.gz
bgzip reference/hg38.chr5.fa
samtools faidx reference/hg38.chr5.fa.gz
```

### **2. Upstream DNA analysis**
Minimap2 is a powerful sequence alignment application that can compare DNA or mRNA sequences to a huge reference library. Typical use cases include: (1) mapping PacBio or Oxford Nanopore genomic reads to the human genome; (2) finding overlaps between long reads with error rate up to ~15%; (3) splice-aware alignment of PacBio Iso-Seq or Nanopore cDNA or Direct RNA reads against a reference genome; (4) aligning Illumina single- or paired-end reads; (5) assembly-to-assembly alignment; (6) full-genome alignment between two closely related species with divergence below ~15%.

Minimap2 is 10x quicker than common long-read mappers such as BLASR, BWA-MEM, NGMLR, and GMAP for 10kb noisy read sequences. On simulated long reads, it is more accurate and generates physiologically relevant alignment suitable for downstream analysis. Minimap2 is 3x faster than BWA-MEM and Bowtie2 for >100bp Illumina short reads and twice (2x) as accurate on simulated data. The minimap2 article or the preprint include detailed evaluations.

```bash
# Install Minimap2
git clone https://github.com/lh3/minimap2
cd minimap2 && make

# Or
conda install -c bioconda minimap2
```
#### Index reference genome before running minimap2
```bash
./minimap2 -d reference/hg38.mmi reference/hg38.fa.gz
```

#### Map long reads
```bash
./minimap2 -ax map-hifi --MD reference/hg38.mmi raw_data/SRR22508184.HG005.PacBio.trimmed.fastq.gz > raw_data/SRR22508184.HG005.PacBio.aln.sam
```
Explanation:
```bash
./minimap2 -ax map-ont ref.mmi ont.fq.gz > aln.sam         # Oxford Nanopore genomic reads
./minimap2 -ax map-hifi ref.mmi pacbio-ccs.fq.gz > aln.sam # PacBio HiFi/CCS genomic reads (v2.19 or later)
./minimap2 -ax asm20 ref.mmi pacbio-ccs.fq.gz > aln.sam    # PacBio HiFi/CCS genomic reads (v2.18 or earlier)
./minimap2 -ax sr ref.mmi read1.fa read2.fa > aln.sam      # short genomic paired-end reads
# man page for detailed command line options
man ./minimap2.1
```

#### Convert SAM to BAM
```bash
samtools view -Sb raw_data/SRR22508184.HG005.PacBio.aln.sam > raw_data/SRR22508184.HG005.PacBio.aln.bam
```

#### Sort the BAM file by coordinate
```bash
gatk SortSam \
--INPUT raw_data/SRR22508184.HG005.PacBio.aln.bam \
--OUTPUT raw_data/SRR22508184.HG005.PacBio.sorted.bam \
--SORT_ORDER coordinate
```

#### Remove duplicated reads
```bash
gatk MarkDuplicates \
--INPUT raw_data/SRR22508184.HG005.PacBio.sorted.bam \
--OUTPUT raw_data/SRR22508184.HG005.PacBio.remove_dup.bam \
--METRICS_FILE raw_data/SRR22508184.HG005.PacBio.remove_dup.metrics2 \
--OPTICAL_DUPLICATE_PIXEL_DISTANCE 2500 \
--CREATE_INDEX true \
--REMOVE_DUPLICATES true
```

### **3. Structural variant calling**
Delly is an integrated structural variation (SV) prediction approach that can detect, genotype, and visualize deletions, tandem duplications, inversions, and translocations in short-read and long-read massively parallel sequencing data at single-nucleotide resolution. It detects and delineates genomic rearrangements throughout the genome using paired-ends, split-reads, and read-depth.
```bash
# Install Delly
git clone --recursive https://github.com/dellytools/delly.git
cd delly/
make all

# Or
conda install -c bioconda delly

# Download the exclusion file containing genomic regions to be excluded from analysis, such as repetitive or problematic regions.
wget "https://github.com/dellytools/delly/blob/c65628b8ce83db90c180fa0ce6b52a5debac6926/excludeTemplates/human.hg38.excl.tsv" -O reference/human.hg38.excl.tsv
```

#### 3.1 SV calling for long reads
```bash
samtools index raw_data/SRR22508184.HG005.PacBio.remove_dup.cram

delly lr -y pb -g reference/hg38.fa.gz -o output/sv_longread.bcf raw_data/SRR22508184.HG005.PacBio.remove_dup.cram
```
Explanation:
```bash
delly lr -y ont -g hg38.fa -o delly.bcf input.bam # Oxford Nanopore genomic reads

delly lr -y pb -g hg38.fa -o delly.bcf input.bam # PacBio HiFi/CCS genomic reads
```
*Note*:
Filtering of called SV is also affected by the specific circumstance of the sample presented. We may use bcftools to filter the vcf files based on allele frequency, SV length, and SVs with the number of supporting reads to remove typically erroneous variations.

#### 3.2 Somatic SV calling
```bash
samtools index raw_data/subset.sorted.normal.chr5.bam

samtools index raw_data/subset.sorted.cancer.chr5.bam

delly call -q 20 -x reference/human.hg38.excl.tsv -g reference/hg38.chr5.fa.gz -o output/sv_somatic.bcf raw_data/subset.sorted.cancer.chr5.bam raw_data/subset.sorted.normal.chr5.bam
```
Explanation:
- `q 20` : only reads with a mapping quality of at least 20 will be considered for variant calling.
- `g fasta`: the reference genome file
- `x reference.excl`: indicates the exclusion file containing genomic regions to be excluded from analysis, such as repetitive or problematic regions.
- `o sv.bcf`: output BCF/VCF file
- `tumor.bam` 
- `control.bam`

*Note*:
Filtering of called SV is also affected by the specific circumstance of the sample presented. We may use bcftools to filter the vcf files based on allele frequency, SV length, and SVs with the number of supporting reads to remove typically erroneous variations.

#### 3.3 Germline SV calling
```bash
delly call -q 20 -x reference/human.hg38.excl.tsv -g reference/hg38.chr5.fa.gz -o output/sv_germline.bcf raw_data/subset.sorted.cancer.chr5.bam
```
*Note*:
Filtering of called SV is also affected by the specific circumstance of the sample presented. We may use bcftools to filter the vcf files based on allele frequency, SV length, and SVs with the number of supporting reads to remove typically erroneous variations.

### **4. Querying VCF files**
Bcftools offers many possibilities to query and reformat SV calls. For instance, to output a table with the chromosome, start, end, identifier and genotype of each SV we can use:
```bash
bcftools query -f "%CHROM\t%POS\t%INFO/END\t%ID[\t%GT]\n" output/sv_somatic.bcf | head
```

```bash
bcftools view output/sv_somatic.bcf | grep "^#" -A 2
```

Calculate the number of structural variations in the BCF file
```bash
bcftools view -H output/sv_somatic.bcf | column -t | less -S

bcftools view -H output/sv_somatic.bcf | awk '{print $8}' | wc -l

bcftools view -H output/sv_longread.bcf | awk '{print $3}' | wc -l
```

Calculate the number of Deletion (DEL), Duplication (DUP), Inversion (INV), Translocation (BND), Insertion (INS)  
```bash
bcftools view -H output/sv_somatic.bcf | awk '{print $5}' | sort | uniq -c

bcftools view -H output/sv_somatic.bcf | awk '{print $3}' | grep -o "^DEL" | wc -l

bcftools view -H output/sv_somatic.bcf | awk '{print $3}' | grep -o "^DUP" | wc -l

bcftools view -H output/sv_somatic.bcf | awk '{print $3}' | grep -o "^INV" | wc -l

bcftools view -H output/sv_somatic.bcf | awk '{print $3}' | grep -o "^BND" | wc -l

bcftools view -H output/sv_somatic.bcf | awk '{print $3}' | grep -o "^INS" | wc -l 
```
### **5. Visual inspection of SVs**
IGV is great for interactive browsing, but for huge numbers of SVs, command-line programs like Samplot can plot many SVs in batch. Samplot is a bioconda package that can be installed using the conda package management. https://github.com/ryanlayer/samplot

```bash
# Install Samplot
conda install -c bioconda samplot 
```

```bash
samplot plot \
    -n normal cancer HGOO5.PacBio \
    -b raw_data/subset.sorted.normal.chr5.bam \
       raw_data/subset.sorted.cancer.chr5.bam\
       raw_data/SRR22508184.HG005.PacBio.remove_dup.cram \
    -o region.png \
    -c chr5 \
    -s 34287816 \
    -e 34288320 \
    -t DEL
```
Explanation 
- `n`: The names to be shown for each sample in the plot
- `b`: The BAM/CRAM files of the samples (space-delimited)
- `o`: The name of the output file containing the plot
- `c`: The chromosome of the region of interest
- `s`: The start location of the region of interest
- `e`: The end location of the region of interest
- `t`: The type of the variant of interest

### **6. Annotation**
SnpEff is a toolset for genetic variation annotation and functional impact prediction. It annotates and predicts the impact of genetic variations (such as amino acid alterations) on genes and proteins. https://pcingola.github.io/SnpEff/

```bash
# Install SnpEff 
# Go to home dir
cd

# Download latest version
wget https://snpeff.blob.core.windows.net/versions/snpEff_latest_core.zip

# Unzip file
unzip snpEff_latest_core.zip

# Download Clinvar databases
wget https://ftp.ncbi.nlm.nih.gov/pub/clinvar/vcf_GRCh38/clinvar.vcf.gz -O hg38.clinvar.vcf.gz
gzip -d hg38.clinvar.vcf.gz
bgzip hg38.clinvar.vcf && tabix -p vcf hg38.clinvar.vcf.gz
```

```bash
# Pre-process of BCF/VCF files
bcftools view output/sv_somatic.bcf > /dev/null # check errors
bcftools view output/sv_somatic.bcf > output/sv_somatic.vcf
```
 
```bash
# Annotate with SnpSift and Clinvar
java -jar $HOME/DNA_softwares/snpEff/SnpSift.jar annotate $HOME/DNA_softwares/snpEff/data/GRCh38.99/hg38.clinvar.vcf.gz output/sv_somatic.vcf > output/sv_somatic.clinvar.ann.vcf

gatk VariantsToTable -V output/sv_somatic.clinvar.ann.vcf -F CHROM -F POS -F TYPE -F ID -F ALLELEID -F CLNDN -F CLNSIG -F CLNSIGCONF -F CLNSIGINCL -F CLNVC -F GENEINFO -GF AD -GF GQ -GF GT -O output/sv_somatic.clinvar.ann.csv
```

## **Reference**

Belyeu, J.R., Chowdhury, M., Brown, J. et al. Samplot: a platform for structural variant visual validation and automated filtering. Genome Biol 22, 161 (2021). https://doi.org/10.1186/s13059-021-02380-5. 

Cingolani, P., Patel, V. M., Coon, M., Nguyen, T., Land, S. J., Ruden, D. M., & Lu, X. (2012). Using Drosophila melanogaster as a model for genotoxic chemical mutational studies with a new program, SnpSift. Frontiers in genetics, 3, 35. https://doi.org/10.3389%2Ffgene.2012.00035

Danecek, P., Bonfield, J. K., Liddle, J., Marshall, J., Ohan, V., Pollard, M. O., ... & Li, H. (2021). Twelve years of SAMtools and BCFtools. Gigascience, 10(2), giab008. https://doi.org/10.1093/gigascience/giab008.

Li, H. (2018). Minimap2: pairwise alignment for nucleotide sequences. Bioinformatics, 34(18), 3094-3100. https://doi.org/10.1093/bioinformatics/bty191.

Rausch, T., Zichner, T., Schlattl, A., Stütz, A. M., Benes, V., & Korbel, J. O. (2012). DELLY: structural variant discovery by integrated paired-end and split-read analysis. Bioinformatics, 28(18), i333-i339. https://doi.org/10.1093/bioinformatics/bts378.

Van der Auwera, G. A., & O'Connor, B. D. (2020). Genomics in the cloud: using Docker, GATK, and WDL in Terra. O'Reilly Media.