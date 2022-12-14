# Add annotation tracks

## CpG islands
```
LTMPDIR=$(mktemp -d)
rsync -avzP rsync://hgdownload.cse.ucsc.edu/goldenPath/hg38/database/cpgIslandExt.txt.gz $LTMPDIR
zcat $LTMPDIR/cpgIslandExt.txt.gz |
    awk '{ printf "%s\t%d\t%d\tOE=%0.1f\n",$2,$3,$4,$NF; }' |
    bgzip -c > annotation/cpgIsland.bed.gz
tabix -f annotation/cpgIsland.bed.gz
rm -rf $LTMPDIR && unset LTMPDIR
```

## Segmental duplications
```
LTMPDIR=$(mktemp -d)
rsync -avzP rsync://hgdownload.cse.ucsc.edu/goldenPath/hg38/database/genomicSuperDups.txt.gz $LTMPDIR
zcat $LTMPDIR/genomicSuperDups.txt.gz |
    awk '{ printf "%s\t%d\t%d\t%s:%'\''d-%'\''d\t%s\t%d\n",$2,$3,$4,$8,$9+1,$10,$7,int(0.5+$27*1000); }' |
    bgzip -c > annotation/segdups.bed.gz
tabix -f annotation/segdups.bed.gz
rm -rf $LTMPDIR && unset LTMPDIR
```

## TODO: Self chains

There are too many self chains in hg38 to properly process them.
I recommend filtering for score > 25,000 to limit the set.
```
LTMPDIR=$(mktemp -d)
rsync -avzP rsync://hgdownload.cse.ucsc.edu/goldenPath/hg38/database/chainSelf.txt.gz $LTMPDIR
rsync -avzP rsync://hgdownload.cse.ucsc.edu/goldenPath/hg38/database/chainSelfLink.txt.gz $LTMPDIR
zcat $LTMPDIR/chainSelfLink.txt.gz | awk '{ print $6 "\t" $3 "\t" $4; }' | bedtools merge -d 100 -i - |
    sort -k1,1 -k2,2g |
    bedtools intersect -sorted -wa -wb -a - \
        -b <(zcat $LTMPDIR/chainSelfLink.txt.gz |
             awk '{ print $6 "\t" $3 "\t" $4 "\t" $5 "\t" $5+$4-$3; }' | sort -k1,1 -k2,2g) |
    sort -k1,1 -k2,2 -k3,3 | datamash -g1,2,3 min 7 max 8 |
    join -t $'\t' - <(zcat $LTMPDIR/chainSelf.txt.gz |
                      awk '{ print $12 "\t" $3 "\t" $7 "\t" $8 "\t" $9 "\t" $13; }' | sort -k1,1) |
    awk '{ qs=($9=="+")?$4:$8-$5; qe=($9=="+")?$5:$8-$4; printf "%s\t%d\t%d\t%s:%'\''d-%'\''d\t%s\t%d\n", $6, $2, $3, $7, qs+1, qe, $9, int(10*$10); }' |
    sort -k1,1 -k2,2g | bgzip -c > annotation/chainSelf.bed.gz
tabix -f annotation/chainSelf.bed.gz
rm -rf $LTMPDIR && unset LTMPDIR
```

## Repeat Masker
```
LTMPDIR=$(mktemp -d)
rsync -avzP rsync://hgdownload.cse.ucsc.edu/goldenPath/hg38/database/rmsk.txt.gz $LTMPDIR
zcat $LTMPDIR/rmsk.txt.gz |
    awk '{ print $6 "\t" $7 "\t" $8 "\t" $12 ":" $11 "\t" $10; }' |
    bgzip -c > annotation/rmsk.bed.gz
tabix -f annotation/rmsk.bed.gz
rm -rf $LTMPDIR && unset LTMPDIR
```

## Window Masker
```
LTMPDIR=$(mktemp -d)
rsync -avzP rsync://hgdownload.cse.ucsc.edu/goldenPath/hg38/database/windowmaskerSdust.txt.gz $LTMPDIR
zcat $LTMPDIR/windowmaskerSdust.txt.gz |
    cut -f2-4 | bgzip -c > annotation/windowmaskerSdust.bed.gz
tabix -f annotation/windowmaskerSdust.bed.gz
rm -rf $LTMPDIR && unset LTMPDIR
```

## Simple Repeats from the Tandem Repeats Finder (TRF)
```
LTMPDIR=$(mktemp -d)
rsync -avzP rsync://hgdownload.cse.ucsc.edu/goldenPath/hg38/database/simpleRepeat.txt.gz $LTMPDIR
zcat $LTMPDIR/simpleRepeat.txt.gz |
    cut -f2-4,7,17 | awk '{  print $1"\t"$2"\t"$3"\t"$4"x"length($5)"bp:" $5; }' |
    bgzip -c > annotation/simpleRepeat.bed.gz
tabix -f annotation/simpleRepeat.bed.gz
rm -rf $LTMPDIR && unset LTMPDIR
```

## Gap+centromere track
```
LTMPDIR=$(mktemp -d)
rsync -avzP rsync://hgdownload.cse.ucsc.edu/goldenPath/hg38/database/gap.txt.gz $LTMPDIR
rsync -avzP rsync://hgdownload.cse.ucsc.edu/goldenPath/hg38/database/centromeres.txt.gz $LTMPDIR
zcat $LTMPDIR/gap.txt.gz | cut -f2-4,8 |
    awk '{n=$4; if(n=="clone" || n=="contig" || n=="scaffold" || n=="short_arm") { n="gap"; } print $1"\t"$2"\t"$3"\t"n; }' |
    cat - <(zcat $LTMPDIR/centromeres.txt.gz | awk '{ print $2"\t"$3"\t"$4"\tcentromere"; }') |
    sort -k1,1 -k2,2g | bgzip -c > annotation/gap.bed.gz
tabix -f annotation/gap.bed.gz
rm -rf $LTMPDIR && unset LTMPDIR
```

## Centromere track
```
LTMPDIR=$(mktemp -d)
rsync -avzP rsync://hgdownload.cse.ucsc.edu/goldenPath/hg38/database/gap.txt.gz $LTMPDIR
zcat $LTMPDIR/gap.txt.gz | cut -f2-4,8 |
    awk '{n=$4; if(n=="clone" || n=="contig" || n=="scaffold" || n=="short_arm") { n="gap"; } print $1"\t"$2"\t"$3"\t"n; }' |
    sort -k1,1 -k2,2g | bgzip -c > annotation/gap.bed.gz
tabix -f annotation/gap.bed.gz
rm -rf $LTMPDIR && unset LTMPDIR
```

## RefSeq exons
```
LTMPDIR=$(mktemp -d)
rsync -avzP rsync://hgdownload.cse.ucsc.edu/goldenPath/hg38/database/refGene.txt.gz $LTMPDIR
zcat $LTMPDIR/refGene.txt.gz |
    awk '{ split($10,ES,","); split($11,EE,","); for(i=1;i<=$9;i++) { s=ES[i]; e=EE[i]; if($8>$7) { print $3"\t"s"\t"e"\t"$2":"$13;  } } }' |
    sort -k1,1 -k2,2g | bgzip -c > annotation/refGene.exon.bed.gz
tabix -f annotation/refGene.exon.bed.gz
zcat $LTMPDIR/refGene.txt.gz |
    awk '{ split($10,ES,","); split($11,EE,","); for(i=1;i<=$9;i++) { s=ES[i]>$7?ES[i]:$7; e=EE[i]<$8?EE[i]:$8; if(e>s) { print $3"\t"s"\t"e"\t"$2":"$13;  } } }' |
    sort -k1,1 -k2,2g | bgzip -c > annotation/refGene.cds.bed.gz
tabix -f annotation/refGene.cds.bed.gz
zcat $LTMPDIR/refGene.txt.gz |
    awk '{ split($10,ES,","); split($11,EE,","); for(i=1;i<=$9;i++) { if(ES[i]<$7) { s=ES[i]; e=EE[i]<$7?EE[i]:$7; { print $3"\t"s"\t"e"\t"$2":"$13; } } if(EE[i]>$8) { s=ES[i]>$8?ES[i]:$8; e=EE[i]; print $3"\t"s"\t"e"\t"$2":"$13; } } }' |
    sort -k1,1 -k2,2g | bgzip -c > annotation/refGene.utr.bed.gz
tabix -f annotation/refGene.utr.bed.gz
rm -rf $LTMPDIR && unset LTMPDIR
```

## Pseudoautosomal regions
```
mysql --user=hgtestuser --host=genome-testdb.cse.ucsc.edu -A \
    -Ne "SELECT chrom, chromStart, chromEnd, name FROM hg38.par" |
    sort -k1,1 -k2,2g | bgzip -c > annotation/par.bed.gz
tabix -f annotation/par.bed.gz
```

## Immunoglobulin / VDJ recombination regions track

The three immunoglobulin loci in the human genome are prone to false positive structural variant
calls that are really detecting [VDJ recombination] (https://en.wikipedia.org/wiki/V%28D%29J_recombination):
* IGH@ is on chr14 (telomeric end of q arm)
* IGK@ is on chr2 (centromeric end of p arm)
* IGL@ is on chr22 (q arm)
```
# Download the old v8 UCSC genes tracks, which has gene models named "abParts" that cover
# the IG regions of interest.
LTMPDIR=$(mktemp -d)
rsync -avzP rsync://hgdownload.cse.ucsc.edu/goldenPath/hg38/database/kgXrefOld8.txt.gz $LTMPDIR
zcat $LTMPDIR/kgXrefOld8.txt.gz | awk -F'\t' '($5=="abParts") { print $1; }' | sort > $LTMPDIR/abPartsGenes.txt

rsync -avzP rsync://hgdownload.cse.ucsc.edu/goldenPath/hg38/database/knownGeneOld8.txt.gz $LTMPDIR
zcat $LTMPDIR/knownGeneOld8.txt.gz | sort -k1,1 | join -t $'\t' - $LTMPDIR/abPartsGenes.txt |
    awk '{ print $2 "\t" $4 "\t" $5; }' | sort -k1,1 -k2,2g | bedtools merge -d 1000000 -i - |
    awk '($1 == "chr14" || $1=="chr2" || $1=="chr22") { print $0 "\tVDJ"; }' | bgzip -c > annotation/vdj.bed.gz
tabix -f annotation/vdj.bed.gz

rm -rf $LTMPDIR
```

## Combined tracks for display
### Segdups + self chains
```
cat <(zcat annotation/segdups.bed.gz | awk 'BEGIN{OFS="\t"}{$4="segdup:"$4; print $0;}') \
    <(zcat annotation/chainSelf.bed.gz | awk 'BEGIN{OFS="\t"}{$4="selfchain:"$4; print $0;}') |
    sort -k1,1 -k2,2g | bgzip -c > annotation/segdups+selfchain.bed.gz
tabix -f annotation/segdups+selfchain.bed.gz
```

### Repeats combine rmsk and simple repeats
```
cat <(zcat annotation/rmsk.bed.gz | awk 'BEGIN{OFS="\t"}{$4="rmsk:"$4; print $0;}') \
    <(zcat annotation/simpleRepeat.bed.gz | awk 'BEGIN{OFS="\t"}{$4="trf:"$4; print $0;}') |
    sort -k1,1 -k2,2g | bgzip -c > annotation/repeats.bed.gz
tabix -f annotation/repeats.bed.gz
```

### Low-complexity repeats that are most difficult for SV calling (Justin Zook suggestion)
```
zcat annotation/repeats.bed.gz | grep -E '(rmsk:Low_complexity|rmsk:Simple_repeat|trf)' |
    cut -f1-3 | bedtools merge -i - | bgzip -c > annotation/mergedLowComplexityRepeats.bed.gz
tabix -f annotation/mergedLowComplexityRepeats.bed.gz
```

### "Odd" regions combine gaps, PARs, and VDJ regions
```
zcat annotation/{gap,par,vdj}.bed.gz | sort -k1,1 -k2,2g | bgzip -c > annotation/oddRegions.bed.gz
tabix -f annotation/oddRegions.bed.gz
```

### ClinVar
```
LTMPDIR=$(mktemp -d)
curl -s ftp://ftp.ncbi.nlm.nih.gov/pub/clinvar/vcf_GRCh38/clinvar.vcf.gz > $LTMPDIR/clinvar.vcf.gz
# Extract variants with at least one classification as:
#     4 - Likely pathogenic,
#     5 - Pathogenic,
#     6 - drug response,
#     7 - histocompatibility
zcat $LTMPDIR/clinvar.vcf.gz | sed -re 's/^.*CLNSIG(=[^;]*).*$/\1\t\0/' | awk '($1 ~ /[=,|][456]/) {print $0; }' | cut -f2- |
    sed -r -e 's/[\t][A-Z0-9]+=.*CLNDBN=/\tCLNDBN=/' -e 's/[;].*//' |
    map_columns - ncbi2ucsc.txt - | cat <(zgrep '^#' $LTMPDIR/clinvar.vcf.gz) - | bgzip -c > annotation/clinvar.vcf.gz
tabix -f annotation/clinvar.vcf.gz
```
