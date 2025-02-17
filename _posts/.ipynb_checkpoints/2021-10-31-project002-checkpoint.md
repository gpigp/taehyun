---
layout: post
title: Project002
subtitle: SNP and INDEL calling
cover-img: /assets/post2-img/indel_blue.jpg
thumbnail-img: https://gpigp.github.io/taehyun/assets/post2-img/marmoset.png
tags: [SNP, INDEL, Marmoset, Stanford, GATK4, BWA, Project]

published: true
---

Call SNPs and INDELs
============================

*❕ docker 환경에서 진행되었습니다.*
*❕ docker environment*
<!-- 1----------------------------------------------------------------------------------------------------------------------------- -->
<h2> 1. Reference & Paired-end whole-genome sequencing data </h2>
<br>
<h6> 2020년 Marmoset Reference </h6>
   
```
wget https://hgdownload.soe.ucsc.edu/goldenPath/calJac4/bigZips/calJac4.fa.gz
gzip -d calJac4.fa.gz
```    
<br>
<h6> SRR7050660 & SRR7050661</h6>
   
```
wget https://sra-downloadb.be-md.ncbi.nlm.nih.gov/sos2/sra-pub-run-13/SRR7050660/SRR7050660.1
wget https://sra-downloadb.be-md.ncbi.nlm.nih.gov/sos2/sra-pub-run-13/SRR7050661/SRR7050661.1
```   
<span style="color: #2D3748; background-color: #fff5b1">2022.04.26.fixed</span>
<br>
<!-- 2----------------------------------------------------------------------------------------------------------------------------- -->
<h2> 2. SRA Toolkit & Fastq-dump </h2>
<br>
SRR7050660.1, SRR7050661.1 각각은 분리시킬수 있는데, SRA Toolkit이 필요합니다.   
[<u>참조</u>](https://github.com/ncbi/sra-tools/wiki/02.-Installing-SRA-Toolkit)[👈](https://github.com/ncbi/sra-tools/wiki/02.-Installing-SRA-Toolkit) 
<br>
<h6> SRA Toolkit 파일 다운 </h6>
<h6> SRA Toolkit downloads </h6>
```
wget --output-document sratoolkit.tar.gz http://ftp-trace.ncbi.nlm.nih.gov/sra/sdk/current/sratoolkit.current-ubuntu64.tar.gz
tar -vxzf sratoolkit.tar.gz
```    
<br>
<h6> 환경변수 설정 </h6>
<h6> Setting Environment Path </h6>
```
export PATH=$PATH:$PWD/sratoolkit.2.11.2-ubuntu64/bin
which fastq-dump
```
*❕ which 이후 아무것도 안뜬다면 설치가 안된 것.*
*❕ If there is nothing after $which, it isn't installed*
<br>
<h6> fastq-dump -I --split-files 파일이름 </h6>
```
fastq-dump -I --split-files SRR7050660.1
fastq-dump -I --split-files SRR7050661.1
```
<br>
<!-- 3----------------------------------------------------------------------------------------------------------------------------- -->
<h2> 3. 2개의 fastq를 reference에 매핑 시키기 </h2>
<h2> 3. mapping fastq_1, fastq_2 to reference </h2>
<br>
매핑 시키기 위해서는 bwa를 설치해야 한다.   
[<u>참조</u>](https://angus.readthedocs.io/en/2013/bwa-tutorial.html)[👈](https://angus.readthedocs.io/en/2013/bwa-tutorial.html)   
<h6> bwa 설치 </h6>   
```
cd /mnt
curl -L -o bwa-0.7.5.tar.bz2 http://sourceforge.net/projects/bio-bwa/files/bwa-0.7.5a.tar.bz2/download
tar xvfj bwa-0.7.5a.tar.bz2
cd bwa-0.7.5a
make
cp bwa /usr/local/bin                // 환경 변수 설정
cd home/Lab_Project_002              // 작업할 파일로 이동
```
<br>
<h6> reference genome 인덱스 만들기 </h6>   
<h6> making reference genome index </h6>   
```
bwa index carJac4.fa
```
<br>
<h6> reference & fastq 파일 매핑 </h6>
<h6> mapping reference & fastq </h6>
```
// Paired-end seqeunce 이므로 bwa mem을 사용한다
// bwa mem -t '# of cores' 'ref file location' 'read1 file' 'read2 file' > 'output'
// 명령어 한 줄당 시간 오래 걸리므로 각각 따로 실행

bwa mem -t 32 Reference/calJac4.fa SRR7050660.1_1.fastq SRR7050660.1_2.fastq > SRR7050660.sam
bwa mem -t 32 Reference/calJac4.fa SRR7050661.1_1.fastq SRR7050661.1_2.fastq > SRR7050661.sam
```
<br>

<!-- 4----------------------------------------------------------------------------------------------------------------------------- -->
<h2> 4 make bam file then sorting and duplicating </h2>
<br>
<h6> GATK 및 Java export </h6>
```
export GATK_LOCAL_JAR=/home/Lab_Project_002/gatk-4.2.2.0/gatk-package-4.2.2.0-local.jar
export JAVA_HOME=~/java
export PATH=$PATH:$JAVA_HOME/bin
```
<br>
<h6> .dict 및 .fai 파일이 있어야 gatk가 동작이 된다. </h6>
<h6>.dict and .fai file is needed to run gatk. </h6>
```
samtools faidx Reference/calJac4.fa
java -jar /downloads/picard.jar CreateSequenceDictionary \ 
      R=Reference/calJac4.fa \ 
      O=Reference/calJac4.dict
```
<br>
<h6> sam to bam </h6>
```
samtools import -@ 32 Reference/calJav4.fa.fai SRR7050660.sam SRR7050660.bam
samtools import -@ 32 Reference/calJav4.fa.fai SRR7050661.sam SRR7050661.bam
```
<br>
<h6> Sort </h6>
```
samtools sort -@ 32 SRR7050660.bam SRR7050660_sorted.bam
samtools sort -@ 32 SRR7050661.bam SRR7050661_sorted.bam
```
<br>
<h6> Duplicate </h6>
```
java -jar /downloads/picard.jar MarkDuplicates \
      I=SRR7050660_sorted.bam.bam \
      O=SRR7050660_dup.bam \
      M=SRR7050660_dup_metrics.txt
      
java -jar /downloads/picard.jar MarkDuplicates \
      I=SRR7050661_sorted.bam \
      O=SRR7050661_dup.bam \
      M=SRR7050661_dup_metrics.txt
```
<br>
<span style="color: #2D3748; background-color: #fff5b1">2022.05.07.fixed</span>
<br>
<!-- 5----------------------------------------------------------------------------------------------------------------------------- -->
<h2> 5. DeepVariant </h2>
<br>
<h6> run deepvariant </h6>
```
run_deepvariant --model_type=WGS \
      --ref=/home/Lab_Project_002/2nd_Trial/Reference/calJac4.fa \
      --reads=/home/Lab_Project_002/2nd_Trial/SRR7050660_dup.bam \
      --output_vcf=/home/Lab_Project_002/2nd_Trial/output/marmoset_VCF_gq0_qual0 \
      --num_shards=$(nproc) \
      --logging_dir=/home/Lab_Project_002/2nd_Trial/log \
      --dry_run=false \
      --postprocess_variants_extra_args="'qual_filter'=0,'cnn_homref_call_min_gq'=0"  
      
run_deepvariant --model_type=WGS \
      --ref=/home/Lab_Project_002/2nd_Trial/Reference/calJac4.fa \
      --reads=/home/Lab_Project_002/2nd_Trial/SRR7050661_dup.bam \
      --output_vcf=/home/Lab_Project_002/2nd_Trial/output/marmoset_VCF_gq0_qual0 \
      --num_shards=$(nproc) \
      --logging_dir=/home/Lab_Project_002/2nd_Trial/log \
      --dry_run=false \
      --postprocess_variants_extra_args="'qual_filter'=0,'cnn_homref_call_min_gq'=0"
```
<br>
<h6> repeat above process changing postprocess variants </h6>
```
"qual_filter=0, cnn_homref_call_min_gq = 0"
"qual_filter=0, cnn_homref_call_min_gq = 0"
"qual_filter=0, cnn_homref_call_min_gq = 0"
"qual_filter=0, cnn_homref_call_min_gq = 0"
```
<br>
<h6> evaluate the outputs </h6>
<br>
<span style="color: #2D3748; background-color: #fff5b1">2022.05.08.fixed</span>
<br>


<h2> 이후 진행할것 </h2>
<h6> 1. GQ filter -> how confident this site is heterozygous, homozygous </h6>
<h6> 2. QUAL filter -> how confident this site is what kind of variation </h6>

<!-- ----------------------------------------------------------------------------------------------------------------------------- -->

<!-- 


<h2> 4. Sam file to Bam file </h2>
<br>
<h6> Samtool Install </h6>
```
cd /root
curl -O -L http://sourceforge.net/projects/samtools/files/samtools/0.1.19/samtools-0.1.19.tar.bz2
tar xvfj samtools-0.1.19.tar.bz2
cd samtools-0.1.19
make
```   

{: .box-error}
```
bam_tview_curses.c:41:10: fatal error: curses.h: No such file or directory
#include "curses.h"
         ^~~~~~~~~~
compilation terminated.
Makefile:129: recipe for target 'bam_tview_curses.o' failed
make: *** [bam_tview_curses.o] Error 1 
```

<h6> ❕ make시 위와 같은 오류가 뜬다면 libncureses5-dev를 설치해야 합니다. </h6>
<h6> ❕ After $make if you have an above error, you have to install libncureses5-dev. </h6>

{: .box-note}
```
apt update && \
apt install -y \
libncurses5-dev \
libncursesw5-dev
```

<br>
<h6> After 'make' </h6>
```
cp samtools /usr/local/bin
cd misc/
cp *.pl maq2sam-long maq2sam-short md5fa md5sum-lite wgsim /usr/local/bin/
cd /home/Lab_Project002
```
<br>
<h6> sam 파일을 bam 파일로 만들기 </h6>
<h6> sam file to bam file </h6>
```
samtools import Reference/calJav4.fa.fai SRR7050660.sam SRR7050660.bam
samtools import Reference/calJav4.fa.fai SRR7050661.sam SRR7050661.bam
```
<br>
<span style="color: #2D3748; background-color: #fff5b1">2022.04.28.fixed</span>
<br>

<h2> 5. Bam file to VCF file </h2>
<br>
<h6>GATK 설치 방법은 추후에 기록하겠습니다.</h6>
<h6>Install sequence for GATK will be recorded later.</h6>

<h6>ref.dict 파일을 만들기 위한 picard.jar 설치 방법도 추후에 기록하겠습니다.</h6>
<h6>picard.jar also will be recorded later.</h6>

<h6> Sort + Mark Duplicates</h6>
```
samtools sort -@ 32 SRR7050660.bam SRR7050660_sorted.bam
samtools sort -@ 32 SRR7050661.bam SRR7050661_sorted.bam
```
<br>

<br>
<h6> Mark Duplicates </h6>
```
java -jar /downloads/picard.jar MarkDuplicates \
      I=SRR7050660_sorted.bam \
      O=SRR7050660_dup.bam \
      M=SRR7050660_dup_metrics.txt
      
java -jar /downloads/picard.jar MarkDuplicates \
      I=SRR7050661_sorted.bam \
      O=SRR7050661_dup.bam \
      M=SRR7050661_dup_metrics.txt
```
<br>
<h6> RG(read group) information adding </h6>
```
java -jar /downloads/picard.jar AddOrReplaceReadGroups \
    I=SRR7050660_dup.bam \
    O=SRR7050660_final.bam \
    SORT_ORDER=coordinate \
    RGID=marmoset \
    RGLB=bar \
    RGPL=PNU \
    RGSM=MLB \
    CREATE_INDEX=True

java -jar /downloads/picard.jar AddOrReplaceReadGroups \
    I=SRR7050661_dup.bam \
    O=SRR7050661_final.bam \
    SORT_ORDER=coordinate \
    RGID=marmoset \
    RGLB=bar \
    RGPL=PNU \
    RGSM=MLB \
    CREATE_INDEX=True
    
samtools index SRR7050660_final.bam
samtools index SRR7050661_final.bam
```
<br>
<h6> Call Variants -> make VCF file </h6>
```
gatk --java-options "-Xmx16g" HaplotypeCaller  \
   -R Reference/calJac4.fa \
   -I SRR7050660_final.bam \
   -O SRR7050660.vcf
   
gatk --java-options "-Xmx16g" HaplotypeCaller  \
   -R Reference/calJac4.fa \
   -I SRR7050661_final.bam \
   -O SRR7050661.vcf   
```
<br>

<br> 
```
run_deepvariant --model_type=WGS \
  --ref=/home/Lab_Project_002/2nd_Trial/Reference/calJac4.fa \
  --reads=/home/Lab_Project_002/2nd_Trial/SRR7050661.bam \
  --output_vcf=/home/Lab_Project_002/2nd_Trial/output/marmoset_VCF_gq0_qual0 \
  --num_shards=$(nproc) \
  --logging_dir=/home/Lab_Project_002/2nd_Trial/log \
  --dry_run=false \
  --postprocess_variants_extra_args="qual_filter=0, cnn_homref_call_min_gq = 0"

```

https://d3g.riken.jp/release/latest/calJac4/

```
bin/hap.py \
  /input/test_nist.b37_chr20_100kbp_at_10mb.vcf.gz \		// 고치기
  /output/output.vcf.gz \					// 고치기
  -f "/home/Lab_Project_002/2nd_Trial/calJac4_ncbiRefSeq.bed12" \
  -r "/home/Lab_Project_002/2nd_Trial/Reference/calJac4.fa" \
  -o "/output/happy.output" \
  --engine=vcfeval \
  --pass-only
```
-->
