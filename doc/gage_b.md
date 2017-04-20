# Assemble three genomes from GAGE-B data sets by ANCHR

[TOC levels=1-3]: # " "
- [Assemble three genomes from GAGE-B data sets by ANCHR](#assemble-three-genomes-from-gage-b-data-sets-by-anchr)
- [*Rhodobacter sphaeroides* 2.4.1](#rhodobacter-sphaeroides-241)
    - [Rsph: download](#rsph-download)
    - [Rsph: combinations of different quality values and read lengths](#rsph-combinations-of-different-quality-values-and-read-lengths)
    - [Rsph: down sampling](#rsph-down-sampling)
    - [Rsph: generate super-reads](#rsph-generate-super-reads)
    - [Rsph: create anchors](#rsph-create-anchors)
    - [Rsph: results](#rsph-results)
    - [Rsph: merge anchors](#rsph-merge-anchors)
- [*Mycobacterium abscessus* 6G-0125-R](#mycobacterium-abscessus-6g-0125-r)
    - [Mabs: download](#mabs-download)
    - [Mabs: combinations of different quality values and read lengths](#mabs-combinations-of-different-quality-values-and-read-lengths)
    - [Mabs: down sampling](#mabs-down-sampling)
    - [Mabs: generate super-reads](#mabs-generate-super-reads)
    - [Mabs: create anchors](#mabs-create-anchors)
    - [Mabs: results](#mabs-results)
    - [Mabs: merge anchors](#mabs-merge-anchors)
- [*Vibrio cholerae* CP1032(5)](#vibrio-cholerae-cp10325)
    - [Vcho: download](#vcho-download)


# *Rhodobacter sphaeroides* 2.4.1

## Rsph: download

* Reference genome

    * Taxid: [272943](https://www.ncbi.nlm.nih.gov/Taxonomy/Browser/wwwtax.cgi?id=272943)
    * RefSeq assembly accession: [GCF_000012905.2](ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/006/745/GCF_000006745.1_ASM674v1/GCF_000006745.1_ASM674v1_assembly_report.txt)

```bash
mkdir -p ~/data/anchr/Rsph/1_genome
cd ~/data/anchr/Rsph/1_genome

aria2c -x 9 -s 3 -c ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/012/905/GCF_000012905.2_ASM1290v2/GCF_000012905.2_ASM1290v2_genomic.fna.gz

TAB=$'\t'
cat <<EOF > replace.tsv
NC_007493.2${TAB}1
NC_007494.2${TAB}2
NC_009007.1${TAB}A
NC_007488.2${TAB}B
NC_007489.1${TAB}C
NC_007490.2${TAB}D
NC_009008.1${TAB}E
EOF

faops replace GCF_000012905.2_ASM1290v2_genomic.fna.gz replace.tsv genome.fa

cp ~/data/anchr/paralogs/gage/Results/Rsph/Rsph.multi.fas paralogs.fas

```

* Illumina

    SRX160386, SRR522246

```bash
mkdir -p ~/data/anchr/Rsph/2_illumina
cd ~/data/anchr/Rsph/2_illumina

cat << EOF > sra_ftp.txt
ftp://ftp.sra.ebi.ac.uk/vol1/srr/SRR522/SRR522246
EOF

aria2c -x 9 -s 3 -c -i sra_ftp.txt

cat << EOF > sra_md5.txt
d3fb8d78abada2e481dd30f3b5f7293d        SRR522246
EOF

md5sum --check sra_md5.txt

fastq-dump --split-files ./SRR522246  
find . -name "*.fastq" | parallel -j 2 pigz -p 8

ln -s SRR522246_1.fastq.gz R1.fq.gz
ln -s SRR522246_2.fastq.gz R2.fq.gz
```

* GAGE-B assemblies

```bash
mkdir -p ~/data/anchr/Rsph/8_competitor
cd ~/data/anchr/Rsph/8_competitor

aria2c -x 9 -s 3 -c http://ccb.jhu.edu/gage_b/genomeAssemblies/R_sphaeroides_MiSeq.tar.gz

tar xvfz R_sphaeroides_MiSeq.tar.gz abyss_ctg.fasta
tar xvfz R_sphaeroides_MiSeq.tar.gz soap_ctg.fasta
tar xvfz R_sphaeroides_MiSeq.tar.gz spades_ctg.fasta
tar xvfz R_sphaeroides_MiSeq.tar.gz velvet_ctg.fasta

```

## Rsph: combinations of different quality values and read lengths

* qual: 20, 25, and 30
* len: 100, 120, and 140

```bash
BASE_DIR=$HOME/data/anchr/Rsph

cd ${BASE_DIR}
tally \
    --pair-by-offset --with-quality --nozip \
    -i 2_illumina/R1.fq.gz \
    -j 2_illumina/R2.fq.gz \
    -o 2_illumina/R1.uniq.fq \
    -p 2_illumina/R2.uniq.fq

parallel --no-run-if-empty -j 2 "
        pigz -p 4 2_illumina/{}.uniq.fq
    " ::: R1 R2

cd ${BASE_DIR}
parallel --no-run-if-empty -j 2 "
    scythe \
        2_illumina/{}.uniq.fq.gz \
        -q sanger \
        -a /home/wangq/.plenv/versions/5.18.4/lib/perl5/site_perl/5.18.4/auto/share/dist/App-Anchr/illumina_adapters.fa \
        --quiet \
        | pigz -p 4 -c \
        > 2_illumina/{}.scythe.fq.gz
    " ::: R1 R2

cd ${BASE_DIR}
parallel --no-run-if-empty -j 6 "
    mkdir -p 2_illumina/Q{1}L{2}
    cd 2_illumina/Q{1}L{2}
    
    if [ -e R1.fq.gz ]; then
        echo '    R1.fq.gz already presents'
        exit;
    fi

    anchr trim \
        --noscythe \
        -q {1} -l {2} \
        ../R1.scythe.fq.gz ../R2.scythe.fq.gz \
        -o stdout \
        | bash
    " ::: 20 25 30 ::: 100 120 140

```

* Stats

```bash
BASE_DIR=$HOME/data/anchr/Rsph
cd ${BASE_DIR}

printf "| %s | %s | %s | %s |\n" \
    "Name" "N50" "Sum" "#" \
    > stat.md
printf "|:--|--:|--:|--:|\n" >> stat.md

printf "| %s | %s | %s | %s |\n" \
    $(echo "Genome";   faops n50 -H -S -C 1_genome/genome.fa;) >> stat.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "Paralogs";   faops n50 -H -S -C 1_genome/paralogs.fas;) >> stat.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "Illumina"; faops n50 -H -S -C 2_illumina/R1.fq.gz 2_illumina/R2.fq.gz;) >> stat.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "PacBio";   faops n50 -H -S -C 3_pacbio/pacbio.fasta;) >> stat.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "uniq";   faops n50 -H -S -C 2_illumina/R1.uniq.fq.gz 2_illumina/R2.uniq.fq.gz;) >> stat.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "scythe";   faops n50 -H -S -C 2_illumina/R1.scythe.fq.gz 2_illumina/R2.scythe.fq.gz;) >> stat.md

for qual in 20 25 30; do
    for len in 100 120 140; do
        DIR_COUNT="${BASE_DIR}/2_illumina/Q${qual}L${len}"

        printf "| %s | %s | %s | %s |\n" \
            $(echo "Q${qual}L${len}"; faops n50 -H -S -C ${DIR_COUNT}/R1.fq.gz  ${DIR_COUNT}/R2.fq.gz;) \
            >> stat.md
    done
done

cat stat.md
```

| Name     |     N50 |        Sum |        # |
|:---------|--------:|-----------:|---------:|
| Genome   | 3188524 |    4602977 |        7 |
| Paralogs |    2337 |     147155 |       66 |
| Illumina |     251 | 4237215336 | 16881336 |
| PacBio   |         |            |          |
| uniq     |     251 | 4199507606 | 16731106 |
| scythe   |     251 | 3261298332 | 16731106 |
| Q20L100  |     149 | 1287602433 |  8847984 |
| Q20L120  |     154 |  909473257 |  5940550 |
| Q20L140  |     162 |  426006455 |  2603868 |
| Q25L100  |     139 |  952173268 |  6942908 |
| Q25L120  |     146 |  544771400 |  3729154 |
| Q25L140  |     156 |  165137858 |  1042218 |
| Q30L100  |     126 |  458552566 |  3649914 |
| Q30L120  |     135 |  140859954 |  1027960 |
| Q30L140  |     149 |   15145623 |    99638 |

## Rsph: down sampling

```bash
BASE_DIR=$HOME/data/anchr/Rsph
cd ${BASE_DIR}

# works on bash 3
ARRAY=(
    "2_illumina/Q20L100:Q20L100"
    "2_illumina/Q20L120:Q20L120"
    "2_illumina/Q20L140:Q20L140"
    "2_illumina/Q25L100:Q25L100"
    "2_illumina/Q25L120:Q25L120"
    "2_illumina/Q25L140:Q25L140"
    "2_illumina/Q30L100:Q30L100"
    "2_illumina/Q30L120:Q30L120"
    "2_illumina/Q30L140:Q30L140"
)

for group in "${ARRAY[@]}" ; do
    
    GROUP_DIR=$(group=${group} perl -e '@p = split q{:}, $ENV{group}; print $p[0];')
    GROUP_ID=$( group=${group} perl -e '@p = split q{:}, $ENV{group}; print $p[1];')
    printf "==> %s \t %s\n" "$GROUP_DIR" "$GROUP_ID"

    echo "==> Group ${GROUP_ID}"
    DIR_COUNT="${BASE_DIR}/${GROUP_ID}"
    mkdir -p ${DIR_COUNT}
    
    if [ -e ${DIR_COUNT}/R1.fq.gz ]; then
        continue     
    fi
    
    ln -s ${BASE_DIR}/${GROUP_DIR}/R1.fq.gz ${DIR_COUNT}/R1.fq.gz
    ln -s ${BASE_DIR}/${GROUP_DIR}/R2.fq.gz ${DIR_COUNT}/R2.fq.gz

done
```

## Rsph: generate super-reads

```bash
BASE_DIR=$HOME/data/anchr/Rsph
cd ${BASE_DIR}

perl -e '
    for my $n (
        qw{
        Q20L100 Q20L120 Q20L140
        Q25L100 Q25L120 Q25L140
        Q30L100 Q30L120 Q30L140
        }
        )
    {
        printf qq{%s\n}, $n;
    }
    ' \
    | parallel --no-run-if-empty -j 3 "
        echo '==> Group {}'
        
        if [ ! -d ${BASE_DIR}/{} ]; then
            echo '    directory not exists'
            exit;
        fi        

        if [ -e ${BASE_DIR}/{}/pe.cor.fa ]; then
            echo '    pe.cor.fa already presents'
            exit;
        fi

        cd ${BASE_DIR}/{}
        anchr superreads \
            R1.fq.gz R2.fq.gz \
            --nosr -p 8 \
            -o superreads.sh
        bash superreads.sh
    "

```

Clear intermediate files.

```bash
BASE_DIR=$HOME/data/anchr/Rsph

find . -type f -name "quorum_mer_db.jf"          | xargs rm
find . -type f -name "k_u_hash_0"                | xargs rm
find . -type f -name "readPositionsInSuperReads" | xargs rm
find . -type f -name "*.tmp"                     | xargs rm
find . -type f -name "pe.renamed.fastq"          | xargs rm
find . -type f -name "pe.cor.sub.fa"             | xargs rm
```

## Rsph: create anchors

```bash
BASE_DIR=$HOME/data/anchr/Rsph
cd ${BASE_DIR}

perl -e '
    for my $n (
        qw{
        Q20L100 Q20L120 Q20L140
        Q25L100 Q25L120 Q25L140
        Q30L100 Q30L120 Q30L140
        }
        )
    {
        printf qq{%s\n}, $n;
    }
    ' \
    | parallel --no-run-if-empty -j 3 "
        echo '==> Group {}'

        if [ -e ${BASE_DIR}/{}/anchor/pe.anchor.fa ]; then
            exit;
        fi

        rm -fr ${BASE_DIR}/{}/anchor
        bash ~/Scripts/cpan/App-Anchr/share/anchor.sh ${BASE_DIR}/{} 8 false
    "

```

## Rsph: results

* Stats of super-reads

```bash
BASE_DIR=$HOME/data/anchr/Rsph
cd ${BASE_DIR}

REAL_G=4602977

bash ~/Scripts/cpan/App-Anchr/share/sr_stat.sh 1 header \
    > ${BASE_DIR}/stat1.md

perl -e '
    for my $n (
        qw{
        Q20L100 Q20L120 Q20L140
        Q25L100 Q25L120 Q25L140
        Q30L100 Q30L120 Q30L140
        }
        )
    {
        printf qq{%s\n}, $n;
    }
    ' \
    | parallel -k --no-run-if-empty -j 4 "
        if [ ! -d ${BASE_DIR}/{} ]; then
            exit;
        fi

        bash ~/Scripts/cpan/App-Anchr/share/sr_stat.sh 1 ${BASE_DIR}/{} ${REAL_G}
    " >> ${BASE_DIR}/stat1.md

cat stat1.md
```

* Stats of anchors

```bash
BASE_DIR=$HOME/data/anchr/Rsph
cd ${BASE_DIR}

bash ~/Scripts/cpan/App-Anchr/share/sr_stat.sh 2 header \
    > ${BASE_DIR}/stat2.md

perl -e '
    for my $n (
        qw{
        Q20L100 Q20L120 Q20L140
        Q25L100 Q25L120 Q25L140
        Q30L100 Q30L120 Q30L140
        }
        )
    {
        printf qq{%s\n}, $n;
    }
    ' \
    | parallel -k --no-run-if-empty -j 8 "
        if [ ! -e ${BASE_DIR}/{}/anchor/pe.anchor.fa ]; then
            exit;
        fi

        bash ~/Scripts/cpan/App-Anchr/share/sr_stat.sh 2 ${BASE_DIR}/{}
    " >> ${BASE_DIR}/stat2.md

cat stat2.md
```

| Name    |   SumFq | CovFq | AvgRead | Kmer |   SumFa | Discard% | RealG |  EstG | Est/Real | SumKU | SumSR |   RunTime |
|:--------|--------:|------:|--------:|-----:|--------:|---------:|------:|------:|---------:|------:|------:|----------:|
| Q20L100 |   1.29G | 279.7 |     145 |   41 |   1.15G |  10.833% |  4.6M | 4.73M |     1.03 | 5.48M |     0 | 0:22'03'' |
| Q20L120 | 909.47M | 197.6 |     153 |   45 | 806.21M |  11.354% |  4.6M | 4.62M |     1.00 | 5.02M |     0 | 0:15'20'' |
| Q20L140 | 426.01M |  92.6 |     163 |   49 | 372.31M |  12.605% |  4.6M | 4.44M |     0.97 | 4.62M |     0 | 0:07'40'' |
| Q25L100 | 952.17M | 206.9 |     137 |   39 |  908.2M |   4.618% |  4.6M | 4.56M |     0.99 | 4.68M |     0 | 0:17'32'' |
| Q25L120 | 544.77M | 118.4 |     146 |   43 | 518.14M |   4.888% |  4.6M | 4.51M |     0.98 | 4.61M |     0 | 0:10'47'' |
| Q25L140 | 165.14M |  35.9 |     158 |   49 | 155.77M |   5.670% |  4.6M | 3.94M |     0.86 | 4.11M |     0 | 0:03'48'' |
| Q30L100 | 458.55M |  99.6 |     125 |   37 | 447.45M |   2.421% |  4.6M | 4.51M |     0.98 | 4.61M |     0 | 0:06'44'' |
| Q30L120 | 140.86M |  30.6 |     136 |   41 | 136.89M |   2.815% |  4.6M | 4.05M |     0.88 | 4.22M |     0 | 0:02'45'' |
| Q30L140 |  15.15M |   3.3 |     152 |   71 |  13.21M |  12.783% |  4.6M | 1.23M |     0.27 | 1.37M |     0 | 0:00'50'' |

| Name    | N50SRclean |   Sum |     # | N50Anchor |     Sum |    # | N50Anchor2 |    Sum |  # | N50Others |     Sum |     # |   RunTime |
|:--------|-----------:|------:|------:|----------:|--------:|-----:|-----------:|-------:|---:|----------:|--------:|------:|----------:|
| Q20L100 |       1330 | 5.48M | 17757 |      2250 |    3.2M | 1525 |          0 |      0 |  0 |       252 |   2.28M | 16232 | 0:05'32'' |
| Q20L120 |       3068 | 5.02M |  8630 |      3972 |   3.95M | 1261 |       1195 |   1.2K |  1 |       227 |   1.07M |  7368 | 0:04'42'' |
| Q20L140 |       4138 | 4.62M |  3906 |      4782 |   4.01M | 1136 |       1257 |  9.78K |  8 |       404 | 602.26K |  2762 | 0:03'30'' |
| Q25L100 |       9779 | 4.68M |  3114 |     10320 |   4.36M |  635 |          0 |      0 |  0 |       182 | 318.08K |  2479 | 0:04'08'' |
| Q25L120 |       6836 | 4.61M |  2520 |      7338 |   4.32M |  828 |          0 |      0 |  0 |       307 | 297.38K |  1692 | 0:03'28'' |
| Q25L140 |       1821 | 4.11M |  4522 |      2663 |    2.9M | 1207 |       1379 |  49.7K | 37 |       559 |   1.16M |  3278 | 0:01'25'' |
| Q30L100 |       7166 | 4.61M |  2776 |      7451 |   4.32M |  832 |          0 |      0 |  0 |       274 |  295.1K |  1944 | 0:03'49'' |
| Q30L120 |       1980 | 4.22M |  4933 |      2717 |   3.06M | 1260 |       1291 | 28.68K | 22 |       528 |   1.13M |  3651 | 0:01'56'' |
| Q30L140 |        337 | 1.37M |  4344 |      2154 | 214.55K |  103 |       1216 | 10.93K |  9 |       280 |   1.14M |  4232 | 0:00'48'' |

## Rsph: merge anchors

```bash
BASE_DIR=$HOME/data/anchr/Rsph
cd ${BASE_DIR}

# merge anchors
mkdir -p merge
anchr contained \
    Q20L100/anchor/pe.anchor.fa \
    Q20L120/anchor/pe.anchor.fa \
    Q20L140/anchor/pe.anchor.fa \
    Q25L100/anchor/pe.anchor.fa \
    Q25L120/anchor/pe.anchor.fa \
    Q25L140/anchor/pe.anchor.fa \
    Q30L100/anchor/pe.anchor.fa \
    Q30L120/anchor/pe.anchor.fa \
    Q30L140/anchor/pe.anchor.fa \
    --len 1000 --idt 0.98 --proportion 0.99999 --parallel 16 \
    -o stdout \
    | faops filter -a 1000 -l 0 stdin merge/anchor.contained.fasta
anchr orient merge/anchor.contained.fasta --len 1000 --idt 0.98 -o merge/anchor.orient.fasta
anchr merge merge/anchor.orient.fasta --len 1000 --idt 0.999 -o stdout \
    | faops filter -a 1000 -l 0 stdin merge/anchor.merge.fasta

# merge anchor2 and others
anchr contained \
    Q20L100/anchor/pe.anchor2.fa \
    Q20L120/anchor/pe.anchor2.fa \
    Q20L140/anchor/pe.anchor2.fa \
    Q25L100/anchor/pe.anchor2.fa \
    Q25L120/anchor/pe.anchor2.fa \
    Q25L140/anchor/pe.anchor2.fa \
    Q30L100/anchor/pe.anchor2.fa \
    Q30L120/anchor/pe.anchor2.fa \
    Q30L140/anchor/pe.anchor2.fa \
    Q20L100/anchor/pe.others.fa \
    Q20L120/anchor/pe.others.fa \
    Q20L140/anchor/pe.others.fa \
    Q25L100/anchor/pe.others.fa \
    Q25L120/anchor/pe.others.fa \
    Q25L140/anchor/pe.others.fa \
    Q30L100/anchor/pe.others.fa \
    Q30L120/anchor/pe.others.fa \
    Q30L140/anchor/pe.others.fa \
    --len 1000 --idt 0.98 --proportion 0.99999 --parallel 16 \
    -o stdout \
    | faops filter -a 1000 -l 0 stdin merge/others.contained.fasta
anchr orient merge/others.contained.fasta --len 1000 --idt 0.98 -o merge/others.orient.fasta
anchr merge merge/others.orient.fasta --len 1000 --idt 0.999 -o stdout \
    | faops filter -a 1000 -l 0 stdin merge/others.merge.fasta

# sort on ref
bash ~/Scripts/cpan/App-Anchr/share/sort_on_ref.sh merge/anchor.merge.fasta 1_genome/genome.fa merge/anchor.sort
nucmer -l 200 1_genome/genome.fa merge/anchor.sort.fa
mummerplot -png out.delta -p anchor.sort --large

# mummerplot files
rm *.[fr]plot
rm out.delta
rm *.gp

mv anchor.sort.png merge/

# quast
rm -fr 9_qa
quast --no-check --threads 16 \
    -R 1_genome/genome.fa \
    8_competitor/abyss_ctg.fasta \
    8_competitor/soap_ctg.fasta \
    8_competitor/spades_ctg.fasta \
    8_competitor/velvet_ctg.fasta \
    merge/anchor.merge.fasta \
    merge/others.merge.fasta \
    1_genome/paralogs.fas \
    --label "abyss,soap,spades,velvet,merge,others,paralogs" \
    -o 9_qa

```

* Clear QxxLxxx.

```bash
BASE_DIR=$HOME/data/anchr/Rsph
cd ${BASE_DIR}

rm -fr 2_illumina/Q{20,25,30}L*
rm -fr Q{20,25,30}L*
```

* Stats

```bash
BASE_DIR=$HOME/data/anchr/Rsph
cd ${BASE_DIR}

printf "| %s | %s | %s | %s |\n" \
    "Name" "N50" "Sum" "#" \
    > stat3.md
printf "|:--|--:|--:|--:|\n" >> stat3.md

printf "| %s | %s | %s | %s |\n" \
    $(echo "Genome";   faops n50 -H -S -C 1_genome/genome.fa;) >> stat3.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "Paralogs";   faops n50 -H -S -C 1_genome/paralogs.fas;) >> stat3.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "anchor.merge"; faops n50 -H -S -C merge/anchor.merge.fasta;) >> stat3.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "others.merge"; faops n50 -H -S -C merge/others.merge.fasta;) >> stat3.md

cat stat3.md
```

| Name         |     N50 |     Sum |   # |
|:-------------|--------:|--------:|----:|
| Genome       | 3188524 | 4602977 |   7 |
| Paralogs     |    2337 |  147155 |  66 |
| anchor.merge |   20406 | 4550050 | 375 |
| others.merge |    1106 |  173689 | 149 |

# *Mycobacterium abscessus* 6G-0125-R

## Mabs: download

* Reference genome

    * *Mycobacterium abscessus* ATCC 19977
        * Taxid: [561007](https://www.ncbi.nlm.nih.gov/Taxonomy/Browser/wwwtax.cgi?id=243277)
        * RefSeq assembly accession: [GCF_000069185.1](ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/006/745/GCF_000006745.1_ASM674v1/GCF_000006745.1_ASM674v1_assembly_report.txt)
    * *Mycobacterium abscessus* 6G-0125-R
        * RefSeq assembly accession: GCF_000270985.1

```bash
mkdir -p ~/data/anchr/Mabs/1_genome
cd ~/data/anchr/Mabs/1_genome

aria2c -x 9 -s 3 -c ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/069/185/GCF_000069185.1_ASM6918v1/GCF_000069185.1_ASM6918v1_genomic.fna.gz

TAB=$'\t'
cat <<EOF > replace.tsv
NC_010397.1${TAB}1
NC_010394.1${TAB}unnamed
EOF

faops replace GCF_000069185.1_ASM6918v1_genomic.fna.gz replace.tsv genome.fa

cp ~/data/anchr/paralogs/gage/Results/Mabs/Mabs.multi.fas paralogs.fas

```

* Illumina

    SRX246890, SRR768269

```bash
mkdir -p ~/data/anchr/Mabs/2_illumina
cd ~/data/anchr/Mabs/2_illumina

cat << EOF > sra_ftp.txt
ftp://ftp.sra.ebi.ac.uk/vol1/srr/SRR768/SRR768269
EOF

aria2c -x 9 -s 3 -c -i sra_ftp.txt

cat << EOF > sra_md5.txt
afcf09a85f0797ab893b05200b575b9d        SRR768269
EOF

md5sum --check sra_md5.txt

fastq-dump --split-files ./SRR768269  
find . -name "*.fastq" | parallel -j 2 pigz -p 8

ln -s SRR768269_1.fastq.gz R1.fq.gz
ln -s SRR768269_2.fastq.gz R2.fq.gz
```

* GAGE-B assemblies

```bash
mkdir -p ~/data/anchr/Mabs/8_competitor
cd ~/data/anchr/Mabs/8_competitor

aria2c -x 9 -s 3 -c http://ccb.jhu.edu/gage_b/genomeAssemblies/M_abscessus_MiSeq.tar.gz

tar xvfz M_abscessus_MiSeq.tar.gz abyss_ctg.fasta
tar xvfz M_abscessus_MiSeq.tar.gz soap_ctg.fasta
tar xvfz M_abscessus_MiSeq.tar.gz spades_ctg.fasta
tar xvfz M_abscessus_MiSeq.tar.gz velvet_ctg.fasta

```

## Mabs: combinations of different quality values and read lengths

* qual: 20, 25, and 30
* len: 100, 120, and 140

```bash
BASE_DIR=$HOME/data/anchr/Mabs

cd ${BASE_DIR}
tally \
    --pair-by-offset --with-quality --nozip \
    -i 2_illumina/R1.fq.gz \
    -j 2_illumina/R2.fq.gz \
    -o 2_illumina/R1.uniq.fq \
    -p 2_illumina/R2.uniq.fq

parallel --no-run-if-empty -j 2 "
        pigz -p 4 2_illumina/{}.uniq.fq
    " ::: R1 R2

cd ${BASE_DIR}
parallel --no-run-if-empty -j 2 "
    scythe \
        2_illumina/{}.uniq.fq.gz \
        -q sanger \
        -a /home/wangq/.plenv/versions/5.18.4/lib/perl5/site_perl/5.18.4/auto/share/dist/App-Anchr/illumina_adapters.fa \
        --quiet \
        | pigz -p 4 -c \
        > 2_illumina/{}.scythe.fq.gz
    " ::: R1 R2

cd ${BASE_DIR}
parallel --no-run-if-empty -j 6 "
    mkdir -p 2_illumina/Q{1}L{2}
    cd 2_illumina/Q{1}L{2}
    
    if [ -e R1.fq.gz ]; then
        echo '    R1.fq.gz already presents'
        exit;
    fi

    anchr trim \
        --noscythe \
        -q {1} -l {2} \
        ../R1.scythe.fq.gz ../R2.scythe.fq.gz \
        -o stdout \
        | bash
    " ::: 20 25 30 ::: 100 120 140

```

* Stats

```bash
BASE_DIR=$HOME/data/anchr/Mabs
cd ${BASE_DIR}

printf "| %s | %s | %s | %s |\n" \
    "Name" "N50" "Sum" "#" \
    > stat.md
printf "|:--|--:|--:|--:|\n" >> stat.md

printf "| %s | %s | %s | %s |\n" \
    $(echo "Genome";   faops n50 -H -S -C 1_genome/genome.fa;) >> stat.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "Paralogs";   faops n50 -H -S -C 1_genome/paralogs.fas;) >> stat.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "Illumina"; faops n50 -H -S -C 2_illumina/R1.fq.gz 2_illumina/R2.fq.gz;) >> stat.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "PacBio";   faops n50 -H -S -C 3_pacbio/pacbio.fasta;) >> stat.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "uniq";   faops n50 -H -S -C 2_illumina/R1.uniq.fq.gz 2_illumina/R2.uniq.fq.gz;) >> stat.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "scythe";   faops n50 -H -S -C 2_illumina/R1.scythe.fq.gz 2_illumina/R2.scythe.fq.gz;) >> stat.md

for qual in 20 25 30; do
    for len in 100 120 140; do
        DIR_COUNT="${BASE_DIR}/2_illumina/Q${qual}L${len}"

        printf "| %s | %s | %s | %s |\n" \
            $(echo "Q${qual}L${len}"; faops n50 -H -S -C ${DIR_COUNT}/R1.fq.gz  ${DIR_COUNT}/R2.fq.gz;) \
            >> stat.md
    done
done

cat stat.md
```

| Name     |     N50 |        Sum |       # |
|:---------|--------:|-----------:|--------:|
| Genome   | 5067172 |    5090491 |       2 |
| Paralogs |    1580 |      83364 |      53 |
| Illumina |     251 | 2194026140 | 8741140 |
| PacBio   |         |            |         |
| uniq     |     251 | 2191831898 | 8732398 |
| scythe   |     194 | 1580945973 | 8732398 |
| Q20L100  |     182 | 1116526651 | 6381972 |
| Q20L120  |     184 | 1022588693 | 5688032 |
| Q20L140  |     188 |  880254750 | 4716680 |
| Q25L100  |     177 |  926043102 | 5426906 |
| Q25L120  |     180 |  824293152 | 4678024 |
| Q25L140  |     184 |  680187695 | 3704988 |
| Q30L100  |     170 |  663451172 | 4046430 |
| Q30L120  |     174 |  556341951 | 3257720 |
| Q30L140  |     179 |  421267610 | 2353262 |

## Mabs: down sampling

```bash
BASE_DIR=$HOME/data/anchr/Mabs
cd ${BASE_DIR}

# works on bash 3
ARRAY=(
    "2_illumina/Q20L100:Q20L100"
    "2_illumina/Q20L120:Q20L120"
    "2_illumina/Q20L140:Q20L140"
    "2_illumina/Q25L100:Q25L100"
    "2_illumina/Q25L120:Q25L120"
    "2_illumina/Q25L140:Q25L140"
    "2_illumina/Q30L100:Q30L100"
    "2_illumina/Q30L120:Q30L120"
    "2_illumina/Q30L140:Q30L140"
)

for group in "${ARRAY[@]}" ; do
    
    GROUP_DIR=$(group=${group} perl -e '@p = split q{:}, $ENV{group}; print $p[0];')
    GROUP_ID=$( group=${group} perl -e '@p = split q{:}, $ENV{group}; print $p[1];')
    printf "==> %s \t %s\n" "$GROUP_DIR" "$GROUP_ID"

    echo "==> Group ${GROUP_ID}"
    DIR_COUNT="${BASE_DIR}/${GROUP_ID}"
    mkdir -p ${DIR_COUNT}
    
    if [ -e ${DIR_COUNT}/R1.fq.gz ]; then
        continue     
    fi
    
    ln -s ${BASE_DIR}/${GROUP_DIR}/R1.fq.gz ${DIR_COUNT}/R1.fq.gz
    ln -s ${BASE_DIR}/${GROUP_DIR}/R2.fq.gz ${DIR_COUNT}/R2.fq.gz

done
```

## Mabs: generate super-reads

```bash
BASE_DIR=$HOME/data/anchr/Mabs
cd ${BASE_DIR}

perl -e '
    for my $n (
        qw{
        Q20L100 Q20L120 Q20L140
        Q25L100 Q25L120 Q25L140
        Q30L100 Q30L120 Q30L140
        }
        )
    {
        printf qq{%s\n}, $n;
    }
    ' \
    | parallel --no-run-if-empty -j 3 "
        echo '==> Group {}'
        
        if [ ! -d ${BASE_DIR}/{} ]; then
            echo '    directory not exists'
            exit;
        fi        

        if [ -e ${BASE_DIR}/{}/pe.cor.fa ]; then
            echo '    pe.cor.fa already presents'
            exit;
        fi

        cd ${BASE_DIR}/{}
        anchr superreads \
            R1.fq.gz R2.fq.gz \
            --nosr -p 8 \
            -o superreads.sh
        bash superreads.sh
    "

```

Clear intermediate files.

```bash
BASE_DIR=$HOME/data/anchr/Mabs

find . -type f -name "quorum_mer_db.jf"          | xargs rm
find . -type f -name "k_u_hash_0"                | xargs rm
find . -type f -name "readPositionsInSuperReads" | xargs rm
find . -type f -name "*.tmp"                     | xargs rm
find . -type f -name "pe.renamed.fastq"          | xargs rm
find . -type f -name "pe.cor.sub.fa"             | xargs rm
```

## Mabs: create anchors

```bash
BASE_DIR=$HOME/data/anchr/Mabs
cd ${BASE_DIR}

perl -e '
    for my $n (
        qw{
        Q20L100 Q20L120 Q20L140
        Q25L100 Q25L120 Q25L140
        Q30L100 Q30L120 Q30L140
        }
        )
    {
        printf qq{%s\n}, $n;
    }
    ' \
    | parallel --no-run-if-empty -j 3 "
        echo '==> Group {}'

        if [ -e ${BASE_DIR}/{}/anchor/pe.anchor.fa ]; then
            exit;
        fi

        rm -fr ${BASE_DIR}/{}/anchor
        bash ~/Scripts/cpan/App-Anchr/share/anchor.sh ${BASE_DIR}/{} 8 false
    "

```

## Mabs: results

* Stats of super-reads

```bash
BASE_DIR=$HOME/data/anchr/Mabs
cd ${BASE_DIR}

REAL_G=5090491

bash ~/Scripts/cpan/App-Anchr/share/sr_stat.sh 1 header \
    > ${BASE_DIR}/stat1.md

perl -e '
    for my $n (
        qw{
        Q20L100 Q20L120 Q20L140
        Q25L100 Q25L120 Q25L140
        Q30L100 Q30L120 Q30L140
        }
        )
    {
        printf qq{%s\n}, $n;
    }
    ' \
    | parallel -k --no-run-if-empty -j 4 "
        if [ ! -d ${BASE_DIR}/{} ]; then
            exit;
        fi

        bash ~/Scripts/cpan/App-Anchr/share/sr_stat.sh 1 ${BASE_DIR}/{} ${REAL_G}
    " >> ${BASE_DIR}/stat1.md

cat stat1.md
```

* Stats of anchors

```bash
BASE_DIR=$HOME/data/anchr/Mabs
cd ${BASE_DIR}

bash ~/Scripts/cpan/App-Anchr/share/sr_stat.sh 2 header \
    > ${BASE_DIR}/stat2.md

perl -e '
    for my $n (
        qw{
        Q20L100 Q20L120 Q20L140
        Q25L100 Q25L120 Q25L140
        Q30L100 Q30L120 Q30L140
        }
        )
    {
        printf qq{%s\n}, $n;
    }
    ' \
    | parallel -k --no-run-if-empty -j 8 "
        if [ ! -e ${BASE_DIR}/{}/anchor/pe.anchor.fa ]; then
            exit;
        fi

        bash ~/Scripts/cpan/App-Anchr/share/sr_stat.sh 2 ${BASE_DIR}/{}
    " >> ${BASE_DIR}/stat2.md

cat stat2.md
```

| Name    |   SumFq | CovFq | AvgRead | Kmer |   SumFa | Discard% | RealG |  EstG | Est/Real | SumKU | SumSR |   RunTime |
|:--------|--------:|------:|--------:|-----:|--------:|---------:|------:|------:|---------:|------:|------:|----------:|
| Q20L100 |   1.12G | 219.3 |     184 |   49 | 888.29M |  20.442% | 5.09M | 5.73M |     1.13 |  7.8M |     0 | 0:15'18'' |
| Q20L120 |   1.02G | 200.9 |     186 |   51 |  815.6M |  20.242% | 5.09M | 5.67M |     1.11 | 7.56M |     0 | 0:14'00'' |
| Q20L140 | 880.25M | 172.9 |     189 |   53 | 696.71M |  20.851% | 5.09M | 5.43M |     1.07 | 6.04M |     0 | 0:12'06'' |
| Q25L100 | 926.04M | 181.9 |     182 |   47 | 772.97M |  16.530% | 5.09M | 5.42M |     1.06 | 5.89M |     0 | 0:12'19'' |
| Q25L120 | 824.29M | 161.9 |     185 |   49 | 688.02M |  16.532% | 5.09M | 5.39M |     1.06 | 5.81M |     0 | 0:10'58'' |
| Q25L140 | 680.19M | 133.6 |     188 |   53 | 565.92M |  16.800% | 5.09M | 5.35M |     1.05 | 5.69M |     0 | 0:09'18'' |
| Q30L100 | 663.45M | 130.3 |     179 |   45 | 574.15M |  13.460% | 5.09M | 5.34M |     1.05 |  5.6M |     0 | 0:08'06'' |
| Q30L120 | 556.34M | 109.3 |     182 |   49 | 480.51M |  13.630% | 5.09M | 5.31M |     1.04 | 5.55M |     0 | 0:07'01'' |
| Q30L140 | 421.27M |  82.8 |     186 |   53 | 361.98M |  14.073% | 5.09M | 5.24M |     1.03 | 5.45M |     0 | 0:05'44'' |

| Name    | N50SRclean |   Sum |     # | N50Anchor |    Sum |    # | N50Anchor2 |    Sum |  # | N50Others |     Sum |     # |   RunTime |
|:--------|-----------:|------:|------:|----------:|-------:|-----:|-----------:|-------:|---:|----------:|--------:|------:|----------:|
| Q20L100 |        330 |  7.8M | 40666 |      1255 | 814.3K |  629 |       1146 |  1.15K |  1 |       278 |   6.98M | 40036 | 0:03'23'' |
| Q20L120 |        385 | 7.56M | 35474 |      1292 |  1.05M |  791 |       1342 |  5.48K |  4 |       308 |    6.5M | 34679 | 0:03'16'' |
| Q20L140 |       1535 | 6.04M | 11588 |      2274 |  3.95M | 1899 |       1284 | 23.73K | 17 |       406 |   2.07M |  9672 | 0:03'27'' |
| Q25L100 |       1769 | 5.89M | 10332 |      2472 |  4.16M | 1876 |       1130 |  1.13K |  1 |       416 |   1.74M |  8455 | 0:02'31'' |
| Q25L120 |       2105 | 5.81M |  8745 |      2802 |  4.39M | 1812 |       1276 |  9.06K |  7 |       379 |   1.41M |  6926 | 0:02'32'' |
| Q25L140 |       2687 | 5.69M |  6879 |      3298 |  4.59M | 1661 |       1284 | 25.05K | 19 |       333 |   1.07M |  5199 | 0:02'05'' |
| Q30L100 |       3287 |  5.6M |  6101 |      3835 |  4.75M | 1518 |       1221 |  2.41K |  2 |       268 | 851.53K |  4581 | 0:02'10'' |
| Q30L120 |       3820 | 5.55M |  5165 |      4373 |  4.81M | 1383 |       1225 | 12.06K |  9 |       257 | 726.39K |  3773 | 0:01'48'' |
| Q30L140 |       3702 | 5.45M |  4484 |      4145 |  4.71M | 1407 |       1536 | 40.49K | 27 |       360 | 692.91K |  3050 | 0:01'31'' |

## Mabs: merge anchors

```bash
BASE_DIR=$HOME/data/anchr/Mabs
cd ${BASE_DIR}

# merge anchors
mkdir -p merge
anchr contained \
    Q20L100/anchor/pe.anchor.fa \
    Q20L120/anchor/pe.anchor.fa \
    Q20L140/anchor/pe.anchor.fa \
    Q25L100/anchor/pe.anchor.fa \
    Q25L120/anchor/pe.anchor.fa \
    Q25L140/anchor/pe.anchor.fa \
    Q30L100/anchor/pe.anchor.fa \
    Q30L120/anchor/pe.anchor.fa \
    Q30L140/anchor/pe.anchor.fa \
    --len 1000 --idt 0.98 --proportion 0.99999 --parallel 16 \
    -o stdout \
    | faops filter -a 1000 -l 0 stdin merge/anchor.contained.fasta
anchr orient merge/anchor.contained.fasta --len 1000 --idt 0.98 -o merge/anchor.orient.fasta
anchr merge merge/anchor.orient.fasta --len 1000 --idt 0.999 -o stdout \
    | faops filter -a 1000 -l 0 stdin merge/anchor.merge.fasta

# merge anchor2 and others
anchr contained \
    Q20L100/anchor/pe.anchor2.fa \
    Q20L120/anchor/pe.anchor2.fa \
    Q20L140/anchor/pe.anchor2.fa \
    Q25L100/anchor/pe.anchor2.fa \
    Q25L120/anchor/pe.anchor2.fa \
    Q25L140/anchor/pe.anchor2.fa \
    Q30L100/anchor/pe.anchor2.fa \
    Q30L120/anchor/pe.anchor2.fa \
    Q30L140/anchor/pe.anchor2.fa \
    Q20L100/anchor/pe.others.fa \
    Q20L120/anchor/pe.others.fa \
    Q20L140/anchor/pe.others.fa \
    Q25L100/anchor/pe.others.fa \
    Q25L120/anchor/pe.others.fa \
    Q25L140/anchor/pe.others.fa \
    Q30L100/anchor/pe.others.fa \
    Q30L120/anchor/pe.others.fa \
    Q30L140/anchor/pe.others.fa \
    --len 1000 --idt 0.98 --proportion 0.99999 --parallel 16 \
    -o stdout \
    | faops filter -a 1000 -l 0 stdin merge/others.contained.fasta
anchr orient merge/others.contained.fasta --len 1000 --idt 0.98 -o merge/others.orient.fasta
anchr merge merge/others.orient.fasta --len 1000 --idt 0.999 -o stdout \
    | faops filter -a 1000 -l 0 stdin merge/others.merge.fasta

# sort on ref
bash ~/Scripts/cpan/App-Anchr/share/sort_on_ref.sh merge/anchor.merge.fasta 1_genome/genome.fa merge/anchor.sort
nucmer -l 200 1_genome/genome.fa merge/anchor.sort.fa
mummerplot -png out.delta -p anchor.sort --large

# mummerplot files
rm *.[fr]plot
rm out.delta
rm *.gp

mv anchor.sort.png merge/

# quast
rm -fr 9_qa
quast --no-check --threads 16 \
    -R 1_genome/genome.fa \
    8_competitor/abyss_ctg.fasta \
    8_competitor/soap_ctg.fasta \
    8_competitor/spades_ctg.fasta \
    8_competitor/velvet_ctg.fasta \
    merge/anchor.merge.fasta \
    merge/others.merge.fasta \
    1_genome/paralogs.fas \
    --label "abyss,soap,spades,velvet,merge,others,paralogs" \
    -o 9_qa

```

* Clear QxxLxxx.

```bash
BASE_DIR=$HOME/data/anchr/Mabs
cd ${BASE_DIR}

rm -fr 2_illumina/Q{20,25,30}L*
rm -fr Q{20,25,30}L*
```

* Stats

```bash
BASE_DIR=$HOME/data/anchr/Mabs
cd ${BASE_DIR}

printf "| %s | %s | %s | %s |\n" \
    "Name" "N50" "Sum" "#" \
    > stat3.md
printf "|:--|--:|--:|--:|\n" >> stat3.md

printf "| %s | %s | %s | %s |\n" \
    $(echo "Genome";   faops n50 -H -S -C 1_genome/genome.fa;) >> stat3.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "Paralogs";   faops n50 -H -S -C 1_genome/paralogs.fas;) >> stat3.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "anchor.merge"; faops n50 -H -S -C merge/anchor.merge.fasta;) >> stat3.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "others.merge"; faops n50 -H -S -C merge/others.merge.fasta;) >> stat3.md

cat stat3.md
```

| Name         |     N50 |     Sum |    # |
|:-------------|--------:|--------:|-----:|
| Genome       | 5067172 | 5090491 |    2 |
| Paralogs     |    1580 |   83364 |   53 |
| anchor.merge |    6702 | 5135204 | 1047 |
| others.merge |    1157 |  166634 |  137 |

# *Vibrio cholerae* CP1032(5)

## Vcho: download

* Reference genome

    * *Vibrio cholerae* O1 biovar El Tor str. N16961
        * Taxid: [243277](https://www.ncbi.nlm.nih.gov/Taxonomy/Browser/wwwtax.cgi?id=243277)
        * RefSeq assembly accession: [GCF_000006745.1](ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/006/745/GCF_000006745.1_ASM674v1/GCF_000006745.1_ASM674v1_assembly_report.txt)
    * *Vibrio cholerae* CP1032(5)
        * RefSeq assembly accession: GCF_000279305.1

```bash
mkdir -p ~/data/anchr/Vcho/1_genome
cd ~/data/anchr/Vcho/1_genome

aria2c -x 9 -s 3 -c ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/006/745/GCF_000006745.1_ASM674v1/GCF_000006745.1_ASM674v1_genomic.fna.gz

TAB=$'\t'
cat <<EOF > replace.tsv
NC_002505.1${TAB}I
NC_002506.1${TAB}II
EOF

faops replace GCF_000006745.1_ASM674v1_genomic.fna.gz replace.tsv genome.fa

cp ~/data/anchr/paralogs/model/Results/Vcho/Vcho.multi.fas paralogs.fas

```

* Illumina

    SRX247310, SRR769320

```bash
mkdir -p ~/data/anchr/Vcho/2_illumina
cd ~/data/anchr/Vcho/2_illumina

cat << EOF > sra_ftp.txt
ftp://ftp.sra.ebi.ac.uk/vol1/srr/SRR769/SRR769320
EOF

aria2c -x 9 -s 3 -c -i sra_ftp.txt

cat << EOF > sra_md5.txt
28f49ca6ae9a00c3a7937e00e04e8512        SRR769320
EOF

md5sum --check sra_md5.txt

fastq-dump --split-files ./SRR769320  
find . -name "*.fastq" | parallel -j 2 pigz -p 8

ln -s SRR769320_1.fastq.gz R1.fq.gz
ln -s SRR769320_2.fastq.gz R2.fq.gz
```

* GAGE-B assemblies

```bash
mkdir -p ~/data/anchr/Vcho/8_competitor
cd ~/data/anchr/Vcho/8_competitor

aria2c -x 9 -s 3 -c http://ccb.jhu.edu/gage_b/genomeAssemblies/V_cholerae_MiSeq.tar.gz

tar xvfz V_cholerae_MiSeq.tar.gz abyss_ctg.fasta
tar xvfz V_cholerae_MiSeq.tar.gz soap_ctg.fasta
tar xvfz V_cholerae_MiSeq.tar.gz spades_ctg.fasta
tar xvfz V_cholerae_MiSeq.tar.gz velvet_ctg.fasta

```

## Vcho: combinations of different quality values and read lengths

* qual: 20, 25, and 30
* len: 100, 120, and 140

```bash
BASE_DIR=$HOME/data/anchr/Vcho

cd ${BASE_DIR}
tally \
    --pair-by-offset --with-quality --nozip \
    -i 2_illumina/R1.fq.gz \
    -j 2_illumina/R2.fq.gz \
    -o 2_illumina/R1.uniq.fq \
    -p 2_illumina/R2.uniq.fq

parallel --no-run-if-empty -j 2 "
        pigz -p 4 2_illumina/{}.uniq.fq
    " ::: R1 R2

cd ${BASE_DIR}
parallel --no-run-if-empty -j 2 "
    scythe \
        2_illumina/{}.uniq.fq.gz \
        -q sanger \
        -a /home/wangq/.plenv/versions/5.18.4/lib/perl5/site_perl/5.18.4/auto/share/dist/App-Anchr/illumina_adapters.fa \
        --quiet \
        | pigz -p 4 -c \
        > 2_illumina/{}.scythe.fq.gz
    " ::: R1 R2

cd ${BASE_DIR}
parallel --no-run-if-empty -j 4 "
    mkdir -p 2_illumina/Q{1}L{2}
    cd 2_illumina/Q{1}L{2}
    
    if [ -e R1.fq.gz ]; then
        echo '    R1.fq.gz already presents'
        exit;
    fi

    anchr trim \
        --noscythe \
        -q {1} -l {2} \
        ../R1.scythe.fq.gz ../R2.scythe.fq.gz \
        -o stdout \
        | bash
    " ::: 20 25 30 ::: 100 120 140

```

* Stats

```bash
BASE_DIR=$HOME/data/anchr/Vcho
cd ${BASE_DIR}

printf "| %s | %s | %s | %s |\n" \
    "Name" "N50" "Sum" "#" \
    > stat.md
printf "|:--|--:|--:|--:|\n" >> stat.md

printf "| %s | %s | %s | %s |\n" \
    $(echo "Genome";   faops n50 -H -S -C 1_genome/genome.fa;) >> stat.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "Paralogs";   faops n50 -H -S -C 1_genome/paralogs.fas;) >> stat.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "Illumina"; faops n50 -H -S -C 2_illumina/R1.fq.gz 2_illumina/R2.fq.gz;) >> stat.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "PacBio";   faops n50 -H -S -C 3_pacbio/pacbio.fasta;) >> stat.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "uniq";   faops n50 -H -S -C 2_illumina/R1.uniq.fq.gz 2_illumina/R2.uniq.fq.gz;) >> stat.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "scythe";   faops n50 -H -S -C 2_illumina/R1.scythe.fq.gz 2_illumina/R2.scythe.fq.gz;) >> stat.md

for qual in 20 25 30; do
    for len in 100 120 140; do
        DIR_COUNT="${BASE_DIR}/2_illumina/Q${qual}L${len}"

        printf "| %s | %s | %s | %s |\n" \
            $(echo "Q${qual}L${len}"; faops n50 -H -S -C ${DIR_COUNT}/R1.fq.gz  ${DIR_COUNT}/R2.fq.gz;) \
            >> stat.md
    done
done

cat stat.md
```

| Name     |     N50 |        Sum |       # |
|:---------|--------:|-----------:|--------:|
| Genome   | 2961149 |    4033464 |       2 |
| Paralogs |    3483 |     114707 |      48 |
| Illumina |     251 | 1762158050 | 7020550 |
| PacBio   |         |            |         |
| uniq     |     251 | 1727781592 | 6883592 |
| scythe   |     198 | 1314316931 | 6883592 |
| Q20L100  |     192 | 1161679827 | 6241070 |
| Q20L120  |     193 | 1122792745 | 5957734 |
| Q20L140  |     195 | 1044398184 | 5417172 |
| Q25L100  |     189 | 1056640471 | 5774338 |
| Q25L120  |     190 | 1010327668 | 5437194 |
| Q25L140  |     193 |  922963312 | 4842010 |
| Q30L100  |     184 |  889848287 | 5009608 |
| Q30L120  |     185 |  832476939 | 4590842 |
| Q30L140  |     188 |  732712135 | 3919768 |

## Vcho: down sampling

```bash
BASE_DIR=$HOME/data/anchr/Vcho
cd ${BASE_DIR}

# works on bash 3
ARRAY=(
    "2_illumina/Q20L100:Q20L100:2500000"
    "2_illumina/Q20L120:Q20L120:2500000"
    "2_illumina/Q20L140:Q20L140:2500000"
    "2_illumina/Q25L100:Q25L100:2500000"
    "2_illumina/Q25L120:Q25L120:2500000"
    "2_illumina/Q25L140:Q25L140:2000000"
    "2_illumina/Q30L100:Q30L100:2500000"
    "2_illumina/Q30L120:Q30L120:2000000"
    "2_illumina/Q30L140:Q30L140:2000000"
)

for group in "${ARRAY[@]}" ; do
    
    GROUP_DIR=$(group=${group} perl -e '@p = split q{:}, $ENV{group}; print $p[0];')
    GROUP_ID=$( group=${group} perl -e '@p = split q{:}, $ENV{group}; print $p[1];')
    GROUP_MAX=$(group=${group} perl -e '@p = split q{:}, $ENV{group}; print $p[2];')
    printf "==> %s \t %s \t %s\n" "$GROUP_DIR" "$GROUP_ID" "$GROUP_MAX"

    for count in $(perl -e 'print 500000 * $_, q{ } for 1 .. 5');
    do
        if [[ "$count" -gt "$GROUP_MAX" ]]; then
            continue;
        fi
        
        echo "==> Group ${GROUP_ID}_${count}"
        DIR_COUNT="${BASE_DIR}/${GROUP_ID}_${count}"
        mkdir -p ${DIR_COUNT}
        
        if [ -e ${DIR_COUNT}/R1.fq.gz ]; then
            continue;
        fi
        
        seqtk sample -s${count} \
            ${BASE_DIR}/${GROUP_DIR}/R1.fq.gz ${count} \
            | pigz > ${DIR_COUNT}/R1.fq.gz
        seqtk sample -s${count} \
            ${BASE_DIR}/${GROUP_DIR}/R2.fq.gz ${count} \
            | pigz > ${DIR_COUNT}/R2.fq.gz
    done

done
```

## Vcho: generate super-reads

```bash
BASE_DIR=$HOME/data/anchr/Vcho
cd ${BASE_DIR}

perl -e '
    for my $n (
        qw{
        Q20L100 Q20L120 Q20L140
        Q25L100 Q25L120 Q25L140
        Q30L100 Q30L120 Q30L140
        }
        )
    {
        for my $i ( 1 .. 5 ) {
            printf qq{%s_%d\n}, $n, ( 500000 * $i );
        }
    }
    ' \
    | parallel --no-run-if-empty -j 3 "
        echo '==> Group {}'
        
        if [ ! -d ${BASE_DIR}/{} ]; then
            echo '    directory not exists'
            exit;
        fi        

        if [ -e ${BASE_DIR}/{}/pe.cor.fa ]; then
            echo '    pe.cor.fa already presents'
            exit;
        fi

        cd ${BASE_DIR}/{}
        anchr superreads \
            R1.fq.gz R2.fq.gz \
            --nosr -p 8 \
            -o superreads.sh
        bash superreads.sh
    "

```

Clear intermediate files.

```bash
BASE_DIR=$HOME/data/anchr/Vcho
cd ${BASE_DIR}

find . -type f -name "quorum_mer_db.jf"          | xargs rm
find . -type f -name "k_u_hash_0"                | xargs rm
find . -type f -name "readPositionsInSuperReads" | xargs rm
find . -type f -name "*.tmp"                     | xargs rm
find . -type f -name "pe.renamed.fastq"          | xargs rm
find . -type f -name "pe.cor.sub.fa"             | xargs rm
```

## Vcho: create anchors

```bash
BASE_DIR=$HOME/data/anchr/Vcho
cd ${BASE_DIR}

perl -e '
    for my $n (
        qw{
        Q20L100 Q20L120 Q20L140
        Q25L100 Q25L120 Q25L140
        Q30L100 Q30L120 Q30L140
        }
        )
    {
        for my $i ( 1 .. 5 ) {
            printf qq{%s_%d\n}, $n, ( 500000 * $i );
        }
    }
    ' \
    | parallel --no-run-if-empty -j 3 "
        echo '==> Group {}'

        if [ -e ${BASE_DIR}/{}/anchor/pe.anchor.fa ]; then
            exit;
        fi

        rm -fr ${BASE_DIR}/{}/anchor
        bash ~/Scripts/cpan/App-Anchr/share/anchor.sh ${BASE_DIR}/{} 8 false
    "

```

## Vcho: results

* Stats of super-reads

```bash
BASE_DIR=$HOME/data/anchr/Vcho
cd ${BASE_DIR}

REAL_G=4033464

bash ~/Scripts/cpan/App-Anchr/share/sr_stat.sh 1 header \
    > ${BASE_DIR}/stat1.md

perl -e '
    for my $n (
        qw{
        Q20L100 Q20L120 Q20L140
        Q25L100 Q25L120 Q25L140
        Q30L100 Q30L120 Q30L140
        }
        )
    {
        for my $i ( 1 .. 5 ) {
            printf qq{%s_%d\n}, $n, ( 500000 * $i );
        }
    }
    ' \
    | parallel -k --no-run-if-empty -j 4 "
        if [ ! -d ${BASE_DIR}/{} ]; then
            exit;
        fi

        bash ~/Scripts/cpan/App-Anchr/share/sr_stat.sh 1 ${BASE_DIR}/{} ${REAL_G}
    " >> ${BASE_DIR}/stat1.md

cat stat1.md
```

* Stats of anchors

```bash
BASE_DIR=$HOME/data/anchr/Vcho
cd ${BASE_DIR}

bash ~/Scripts/cpan/App-Anchr/share/sr_stat.sh 2 header \
    > ${BASE_DIR}/stat2.md

perl -e '
    for my $n (
        qw{
        Q20L100 Q20L120 Q20L140
        Q25L100 Q25L120 Q25L140
        Q30L100 Q30L120 Q30L140
        }
        )
    {
        for my $i ( 1 .. 5 ) {
            printf qq{%s_%d\n}, $n, ( 500000 * $i );
        }
    }
    ' \
    | parallel -k --no-run-if-empty -j 8 "
        if [ ! -e ${BASE_DIR}/{}/anchor/pe.anchor.fa ]; then
            exit;
        fi

        bash ~/Scripts/cpan/App-Anchr/share/sr_stat.sh 2 ${BASE_DIR}/{}
    " >> ${BASE_DIR}/stat2.md

cat stat2.md
```

| Name            |   SumFq | CovFq | AvgRead | Kmer |   SumFa | Discard% | RealG |  EstG | Est/Real | SumKU | SumSR |   RunTime |
|:----------------|--------:|------:|--------:|-----:|--------:|---------:|------:|------:|---------:|------:|------:|----------:|
| Q20L100_500000  | 186.17M |  46.2 |     180 |  115 | 150.15M |  19.350% | 4.03M | 3.94M |     0.98 | 4.19M |     0 | 0:01'35'' |
| Q20L100_1000000 | 372.26M |  92.3 |     184 |  121 | 300.12M |  19.379% | 4.03M | 3.96M |     0.98 | 4.25M |     0 | 0:02'35'' |
| Q20L100_1500000 | 558.48M | 138.5 |     187 |  127 |    451M |  19.245% | 4.03M |    4M |     0.99 |  4.5M |     0 | 0:03'48'' |
| Q20L100_2000000 | 744.61M | 184.6 |     191 |  127 | 601.74M |  19.186% | 4.03M | 4.06M |     1.01 | 4.83M |     0 | 0:06'06'' |
| Q20L100_2500000 |  930.6M | 230.7 |     194 |  127 | 755.51M |  18.815% | 4.03M | 4.22M |     1.05 | 6.07M |     0 | 0:08'11'' |
| Q20L120_500000  | 188.42M |  46.7 |     182 |  117 | 152.72M |  18.949% | 4.03M | 3.94M |     0.98 | 4.21M |     0 | 0:02'19'' |
| Q20L120_1000000 | 376.91M |  93.4 |     186 |  121 | 305.19M |  19.028% | 4.03M | 3.96M |     0.98 | 4.27M |     0 | 0:04'04'' |
| Q20L120_1500000 | 565.39M | 140.2 |     189 |  127 |  458.5M |  18.906% | 4.03M |    4M |     0.99 | 4.51M |     0 | 0:06'05'' |
| Q20L120_2000000 | 753.91M | 186.9 |     192 |  127 | 611.92M |  18.834% | 4.03M | 4.06M |     1.01 | 4.85M |     0 | 0:06'15'' |
| Q20L120_2500000 | 942.29M | 233.6 |     195 |  127 | 768.49M |  18.444% | 4.03M | 4.23M |     1.05 | 6.15M |     0 | 0:07'11'' |
| Q20L140_500000  |  192.8M |  47.8 |     186 |  121 | 156.73M |  18.707% | 4.03M | 3.94M |     0.98 | 4.22M |     0 | 0:01'39'' |
| Q20L140_1000000 | 385.57M |  95.6 |     189 |  127 | 313.42M |  18.712% | 4.03M | 3.96M |     0.98 | 4.31M |     0 | 0:02'52'' |
| Q20L140_1500000 | 578.39M | 143.4 |     192 |  127 | 470.65M |  18.628% | 4.03M | 4.01M |     0.99 | 4.55M |     0 | 0:04'18'' |
| Q20L140_2000000 | 771.22M | 191.2 |     195 |  127 |  628.2M |  18.546% | 4.03M | 4.07M |     1.01 | 4.91M |     0 | 0:05'33'' |
| Q20L140_2500000 | 963.99M | 239.0 |     197 |  127 | 786.08M |  18.456% | 4.03M | 4.16M |     1.03 | 5.35M |     0 | 0:06'35'' |
| Q25L100_500000  | 183.03M |  45.4 |     179 |  115 | 154.17M |  15.765% | 4.03M | 3.93M |     0.98 | 4.18M |     0 | 0:01'30'' |
| Q25L100_1000000 | 365.97M |  90.7 |     183 |  119 | 308.21M |  15.784% | 4.03M | 3.96M |     0.98 | 4.22M |     0 | 0:02'27'' |
| Q25L100_1500000 | 548.96M | 136.1 |     187 |  127 | 462.86M |  15.685% | 4.03M | 3.99M |     0.99 | 4.43M |     0 | 0:03'37'' |
| Q25L100_2000000 | 731.93M | 181.5 |     191 |  127 | 617.34M |  15.655% | 4.03M | 4.04M |     1.00 | 4.67M |     0 | 0:05'27'' |
| Q25L100_2500000 | 914.95M | 226.8 |     195 |  127 | 772.42M |  15.577% | 4.03M |  4.1M |     1.02 | 5.02M |     0 | 0:06'19'' |
| Q25L120_500000  | 185.83M |  46.1 |     181 |  117 | 156.99M |  15.517% | 4.03M | 3.94M |     0.98 |  4.2M |     0 | 0:01'47'' |
| Q25L120_1000000 | 371.71M |  92.2 |     186 |  121 | 313.66M |  15.619% | 4.03M | 3.95M |     0.98 | 4.25M |     0 | 0:02'22'' |
| Q25L120_1500000 | 557.49M | 138.2 |     189 |  127 | 470.87M |  15.538% | 4.03M |    4M |     0.99 | 4.46M |     0 | 0:03'31'' |
| Q25L120_2000000 |  743.2M | 184.3 |     193 |  127 | 628.16M |  15.478% | 4.03M | 4.05M |     1.00 | 4.76M |     0 | 0:05'14'' |
| Q25L120_2500000 | 929.13M | 230.4 |     197 |  127 | 785.97M |  15.408% | 4.03M | 4.11M |     1.02 | 5.11M |     0 | 0:06'30'' |
| Q25L140_500000  | 190.62M |  47.3 |     185 |  121 | 160.89M |  15.594% | 4.03M | 3.93M |     0.97 | 4.23M |     0 | 0:01'28'' |
| Q25L140_1000000 | 381.24M |  94.5 |     188 |  127 | 321.81M |  15.589% | 4.03M | 3.96M |     0.98 | 4.31M |     0 | 0:02'44'' |
| Q25L140_1500000 | 571.84M | 141.8 |     192 |  127 | 483.15M |  15.510% | 4.03M |    4M |     0.99 | 4.54M |     0 | 0:03'52'' |
| Q25L140_2000000 | 762.51M | 189.0 |     196 |  127 |  644.8M |  15.437% | 4.03M | 4.06M |     1.01 | 4.83M |     0 | 0:05'11'' |
| Q30L100_500000  | 177.61M |  44.0 |     176 |  111 | 155.06M |  12.697% | 4.03M | 3.93M |     0.98 | 4.17M |     0 | 0:01'48'' |
| Q30L100_1000000 | 355.22M |  88.1 |     182 |  119 | 310.03M |  12.723% | 4.03M | 3.95M |     0.98 | 4.22M |     0 | 0:02'36'' |
| Q30L100_1500000 | 532.82M | 132.1 |     188 |  127 | 465.39M |  12.656% | 4.03M | 3.98M |     0.99 |  4.4M |     0 | 0:03'20'' |
| Q30L100_2000000 | 710.44M | 176.1 |     193 |  127 | 620.95M |  12.597% | 4.03M | 4.01M |     1.00 | 4.62M |     0 | 0:04'05'' |
| Q30L100_2500000 | 888.14M | 220.2 |     198 |  127 | 776.85M |  12.530% | 4.03M | 4.07M |     1.01 | 4.91M |     0 | 0:06'02'' |
| Q30L120_500000  | 181.36M |  45.0 |     180 |  115 | 158.22M |  12.756% | 4.03M | 3.94M |     0.98 |  4.2M |     0 | 0:01'42'' |
| Q30L120_1000000 | 362.64M |  89.9 |     185 |  121 | 316.36M |  12.763% | 4.03M | 3.95M |     0.98 | 4.27M |     0 | 0:02'25'' |
| Q30L120_1500000 | 544.08M | 134.9 |     190 |  127 | 474.96M |  12.704% | 4.03M | 3.98M |     0.99 | 4.46M |     0 | 0:03'38'' |
| Q30L120_2000000 | 725.33M | 179.8 |     195 |  127 | 633.67M |  12.638% | 4.03M | 4.02M |     1.00 | 4.71M |     0 | 0:05'08'' |
| Q30L140_500000  | 186.92M |  46.3 |     184 |  119 |  162.6M |  13.016% | 4.03M | 3.93M |     0.97 | 4.17M |     0 | 0:01'29'' |
| Q30L140_1000000 | 373.84M |  92.7 |     189 |  127 | 325.38M |  12.964% | 4.03M | 3.95M |     0.98 | 4.32M |     0 | 0:03'14'' |
| Q30L140_1500000 | 560.78M | 139.0 |     194 |  127 | 488.39M |  12.908% | 4.03M | 3.99M |     0.99 | 4.55M |     0 | 0:03'46'' |
| Q30L140_2000000 | 732.71M | 181.7 |     198 |  127 |  638.6M |  12.844% | 4.03M | 4.03M |     1.00 |  4.8M |     0 | 0:04'15'' |

| Name            | N50SRclean |   Sum |     # | N50Anchor |   Sum |    # | N50Anchor2 |    Sum |  # | N50Others |     Sum |     # |   RunTime |
|:----------------|-----------:|------:|------:|----------:|------:|-----:|-----------:|-------:|---:|----------:|--------:|------:|----------:|
| Q20L100_500000  |       5129 | 4.19M |  2363 |      5777 | 3.74M |  876 |       1510 |  2.57K |  2 |       476 | 449.41K |  1485 | 0:01'27'' |
| Q20L100_1000000 |       7296 | 4.25M |  2377 |      8208 | 3.87M |  708 |          0 |      0 |  0 |       220 |  381.4K |  1669 | 0:01'49'' |
| Q20L100_1500000 |       4781 |  4.5M |  3919 |      5588 | 3.83M |  927 |          0 |      0 |  0 |       200 | 670.38K |  2992 | 0:02'19'' |
| Q20L100_2000000 |       2915 | 4.83M |  6202 |      3704 | 3.72M | 1203 |          0 |      0 |  0 |       200 |   1.12M |  4999 | 0:02'11'' |
| Q20L100_2500000 |        971 | 6.07M | 14572 |      2000 | 2.96M | 1558 |          0 |      0 |  0 |       220 |   3.11M | 13014 | 0:02'40'' |
| Q20L120_500000  |       4263 | 4.21M |  2629 |      4849 | 3.69M | 1004 |       1554 | 10.22K |  7 |       480 | 510.94K |  1618 | 0:01'12'' |
| Q20L120_1000000 |       6126 | 4.27M |  2554 |      6814 | 3.85M |  801 |       1135 |  1.14K |  1 |       241 | 415.15K |  1752 | 0:01'08'' |
| Q20L120_1500000 |       4068 | 4.51M |  4037 |      4759 | 3.83M | 1024 |          0 |      0 |  0 |       200 | 673.44K |  3013 | 0:01'39'' |
| Q20L120_2000000 |       2772 | 4.85M |  6343 |      3618 | 3.68M | 1243 |          0 |      0 |  0 |       200 |   1.17M |  5100 | 0:02'17'' |
| Q20L120_2500000 |        902 | 6.15M | 15174 |      1956 | 2.88M | 1568 |          0 |      0 |  0 |       227 |   3.28M | 13606 | 0:03'28'' |
| Q20L140_500000  |       3699 | 4.22M |  2833 |      4281 | 3.63M | 1057 |       1213 |   4.7K |  4 |       499 | 590.76K |  1772 | 0:01'16'' |
| Q20L140_1000000 |       4882 | 4.31M |  2873 |      5623 | 3.79M |  949 |       1150 |  4.57K |  4 |       302 | 508.72K |  1920 | 0:01'32'' |
| Q20L140_1500000 |       3595 | 4.55M |  4409 |      4273 | 3.76M | 1114 |       1207 |  1.21K |  1 |       228 | 789.48K |  3294 | 0:01'31'' |
| Q20L140_2000000 |       2319 | 4.91M |  6791 |      3224 | 3.61M | 1349 |          0 |      0 |  0 |       218 |    1.3M |  5442 | 0:02'36'' |
| Q20L140_2500000 |       1563 | 5.35M |  9752 |      2568 | 3.37M | 1479 |          0 |      0 |  0 |       221 |   1.98M |  8273 | 0:03'00'' |
| Q25L100_500000  |       4631 | 4.18M |  2408 |      5143 | 3.73M |  957 |       1071 |  1.07K |  1 |       438 | 448.51K |  1450 | 0:01'48'' |
| Q25L100_1000000 |       7626 | 4.22M |  2146 |      8269 | 3.88M |  681 |       1287 |  2.51K |  2 |       237 | 336.44K |  1463 | 0:01'51'' |
| Q25L100_1500000 |       5062 | 4.43M |  3478 |      5960 | 3.84M |  898 |          0 |      0 |  0 |       201 | 586.72K |  2580 | 0:02'17'' |
| Q25L100_2000000 |       3580 | 4.67M |  5052 |      4497 | 3.78M | 1094 |          0 |      0 |  0 |       197 | 883.68K |  3958 | 0:03'04'' |
| Q25L100_2500000 |       2346 | 5.02M |  7442 |      3294 | 3.65M | 1344 |          0 |      0 |  0 |       198 |   1.37M |  6098 | 0:03'27'' |
| Q25L120_500000  |       3889 |  4.2M |  2710 |      4507 | 3.67M | 1067 |       1185 |  4.76K |  4 |       508 | 535.18K |  1639 | 0:01'19'' |
| Q25L120_1000000 |       5373 | 4.25M |  2528 |      5949 | 3.85M |  878 |          0 |      0 |  0 |       261 | 407.95K |  1650 | 0:01'42'' |
| Q25L120_1500000 |       4062 | 4.46M |  3780 |      4663 | 3.82M | 1040 |       1275 |  1.28K |  1 |       217 | 643.25K |  2739 | 0:02'30'' |
| Q25L120_2000000 |       2810 | 4.76M |  5777 |      3694 |  3.7M | 1236 |          0 |      0 |  0 |       206 |   1.06M |  4541 | 0:02'54'' |
| Q25L120_2500000 |       1990 | 5.11M |  8132 |      2965 | 3.54M | 1412 |          0 |      0 |  0 |       207 |   1.57M |  6720 | 0:03'27'' |
| Q25L140_500000  |       3134 | 4.23M |  3041 |      3723 | 3.58M | 1172 |       1222 | 11.11K |  9 |       532 | 636.36K |  1860 | 0:01'29'' |
| Q25L140_1000000 |       3984 | 4.31M |  2967 |      4570 | 3.77M | 1059 |       1130 |  3.51K |  3 |       395 | 533.49K |  1905 | 0:01'59'' |
| Q25L140_1500000 |       3230 | 4.54M |  4408 |      4080 | 3.73M | 1189 |          0 |      0 |  0 |       254 | 811.78K |  3219 | 0:02'33'' |
| Q25L140_2000000 |       2326 | 4.83M |  6348 |      3190 | 3.62M | 1356 |          0 |      0 |  0 |       231 |   1.21M |  4992 | 0:03'14'' |
| Q30L100_500000  |       4308 | 4.17M |  2418 |      4929 | 3.72M |  979 |       1098 |  3.38K |  3 |       479 | 448.43K |  1436 | 0:01'25'' |
| Q30L100_1000000 |       6451 | 4.22M |  2279 |      7039 | 3.85M |  784 |       1157 |  1.16K |  1 |       272 | 368.64K |  1494 | 0:01'58'' |
| Q30L100_1500000 |       4991 |  4.4M |  3306 |      5678 | 3.86M |  947 |          0 |      0 |  0 |       211 | 541.67K |  2359 | 0:02'40'' |
| Q30L100_2000000 |       3378 | 4.62M |  4811 |      4217 | 3.78M | 1127 |          0 |      0 |  0 |       202 | 844.61K |  3684 | 0:03'08'' |
| Q30L100_2500000 |       2475 | 4.91M |  6782 |      3269 | 3.67M | 1336 |          0 |      0 |  0 |       200 |   1.24M |  5446 | 0:03'30'' |
| Q30L120_500000  |       3387 |  4.2M |  2820 |      3933 |  3.6M | 1134 |       1206 |  3.58K |  3 |       567 | 596.58K |  1683 | 0:01'25'' |
| Q30L120_1000000 |       4569 | 4.27M |  2711 |      5067 | 3.81M |  983 |          0 |      0 |  0 |       345 | 457.06K |  1728 | 0:02'00'' |
| Q30L120_1500000 |       3633 | 4.46M |  3850 |      4163 | 3.79M | 1133 |          0 |      0 |  0 |       253 | 671.86K |  2717 | 0:02'24'' |
| Q30L120_2000000 |       2735 | 4.71M |  5527 |      3430 | 3.72M | 1306 |          0 |      0 |  0 |       217 | 990.37K |  4221 | 0:03'08'' |
| Q30L140_500000  |       3013 | 4.17M |  2832 |      3543 | 3.54M | 1213 |       1275 | 12.46K | 10 |       594 | 619.71K |  1609 | 0:01'27'' |
| Q30L140_1000000 |       3285 | 4.32M |  3271 |      3859 |  3.7M | 1200 |          0 |      0 |  0 |       441 |  617.8K |  2071 | 0:01'47'' |
| Q30L140_1500000 |       2767 | 4.55M |  4606 |      3390 | 3.69M | 1308 |       1089 |  1.09K |  1 |       299 | 854.12K |  3297 | 0:02'20'' |
| Q30L140_2000000 |       2216 |  4.8M |  6273 |      2929 | 3.58M | 1412 |       1089 |  1.09K |  1 |       262 |   1.22M |  4860 | 0:03'04'' |

## Vcho: merge anchors

```bash
BASE_DIR=$HOME/data/anchr/Vcho
cd ${BASE_DIR}

# merge anchors
mkdir -p merge
anchr contained \
    Q20L100_1000000/anchor/pe.anchor.fa \
    Q20L100_1500000/anchor/pe.anchor.fa \
    Q20L120_1000000/anchor/pe.anchor.fa \
    Q20L120_1500000/anchor/pe.anchor.fa \
    Q20L140_1000000/anchor/pe.anchor.fa \
    Q20L140_1500000/anchor/pe.anchor.fa \
    Q25L100_1000000/anchor/pe.anchor.fa \
    Q25L100_1500000/anchor/pe.anchor.fa \
    Q25L120_1000000/anchor/pe.anchor.fa \
    Q25L120_1500000/anchor/pe.anchor.fa \
    Q25L140_1000000/anchor/pe.anchor.fa \
    Q25L140_1500000/anchor/pe.anchor.fa \
    Q30L100_1000000/anchor/pe.anchor.fa \
    Q30L100_1500000/anchor/pe.anchor.fa \
    Q30L120_1000000/anchor/pe.anchor.fa \
    Q30L120_1500000/anchor/pe.anchor.fa \
    Q30L140_1000000/anchor/pe.anchor.fa \
    Q30L140_1500000/anchor/pe.anchor.fa \
    --len 1000 --idt 0.98 --proportion 0.99999 --parallel 16 \
    -o stdout \
    | faops filter -a 1000 -l 0 stdin merge/anchor.contained.fasta
anchr orient merge/anchor.contained.fasta --len 1000 --idt 0.98 -o merge/anchor.orient.fasta
anchr merge merge/anchor.orient.fasta --len 1000 --idt 0.999 -o stdout \
    | faops filter -a 1000 -l 0 stdin merge/anchor.merge.fasta

# merge anchor2 and others
anchr contained \
    Q20L100_1000000/anchor/pe.anchor2.fa \
    Q20L120_1000000/anchor/pe.anchor2.fa \
    Q20L140_1000000/anchor/pe.anchor2.fa \
    Q25L100_1000000/anchor/pe.anchor2.fa \
    Q25L120_1000000/anchor/pe.anchor2.fa \
    Q25L140_1000000/anchor/pe.anchor2.fa \
    Q30L100_1000000/anchor/pe.anchor2.fa \
    Q30L120_1000000/anchor/pe.anchor2.fa \
    Q30L140_1000000/anchor/pe.anchor2.fa \
    Q20L100_1000000/anchor/pe.others.fa \
    Q20L120_1000000/anchor/pe.others.fa \
    Q20L140_1000000/anchor/pe.others.fa \
    Q25L100_1000000/anchor/pe.others.fa \
    Q25L120_1000000/anchor/pe.others.fa \
    Q25L140_1000000/anchor/pe.others.fa \
    Q30L100_1000000/anchor/pe.others.fa \
    Q30L120_1000000/anchor/pe.others.fa \
    Q30L140_1000000/anchor/pe.others.fa \
    --len 1000 --idt 0.98 --proportion 0.99999 --parallel 16 \
    -o stdout \
    | faops filter -a 1000 -l 0 stdin merge/others.contained.fasta
anchr orient merge/others.contained.fasta --len 1000 --idt 0.98 -o merge/others.orient.fasta
anchr merge merge/others.orient.fasta --len 1000 --idt 0.999 -o stdout \
    | faops filter -a 1000 -l 0 stdin merge/others.merge.fasta

# sort on ref
bash ~/Scripts/cpan/App-Anchr/share/sort_on_ref.sh merge/anchor.merge.fasta 1_genome/genome.fa merge/anchor.sort
nucmer -l 200 1_genome/genome.fa merge/anchor.sort.fa
mummerplot -png out.delta -p anchor.sort --large

# mummerplot files
rm *.[fr]plot
rm out.delta
rm *.gp

mv anchor.sort.png merge/

# quast
rm -fr 9_qa
quast --no-check --threads 16 \
    -R 1_genome/genome.fa \
    8_competitor/abyss_ctg.fasta \
    8_competitor/soap_ctg.fasta \
    8_competitor/spades_ctg.fasta \
    8_competitor/velvet_ctg.fasta \
    merge/anchor.merge.fasta \
    merge/others.merge.fasta \
    1_genome/paralogs.fas \
    --label "abyss,soap,spades,velvet,merge,others,paralogs" \
    -o 9_qa

```

* Clear QxxLxxx.

```bash
BASE_DIR=$HOME/data/anchr/Vcho
cd ${BASE_DIR}

rm -fr 2_illumina/Q{20,25,30}L*
rm -fr Q{20,25,30}L*
```

* Stats

```bash
BASE_DIR=$HOME/data/anchr/Vcho
cd ${BASE_DIR}

printf "| %s | %s | %s | %s |\n" \
    "Name" "N50" "Sum" "#" \
    > stat3.md
printf "|:--|--:|--:|--:|\n" >> stat3.md

printf "| %s | %s | %s | %s |\n" \
    $(echo "Genome";   faops n50 -H -S -C 1_genome/genome.fa;) >> stat3.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "Paralogs";   faops n50 -H -S -C 1_genome/paralogs.fas;) >> stat3.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "anchor.merge"; faops n50 -H -S -C merge/anchor.merge.fasta;) >> stat3.md
printf "| %s | %s | %s | %s |\n" \
    $(echo "others.merge"; faops n50 -H -S -C merge/others.merge.fasta;) >> stat3.md

cat stat3.md
```

| Name         |     N50 |     Sum |   # |
|:-------------|--------:|--------:|----:|
| Genome       | 2961149 | 4033464 |   2 |
| Paralogs     |    3483 |  114707 |  48 |
| anchor.merge |   75961 | 3943387 | 135 |
| others.merge |    1021 |   43353 |  41 |