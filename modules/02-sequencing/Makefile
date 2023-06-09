threads := 8 # adjust this to your liking

all: data/wgs_illumina_R1.fastq.gz data/wgs_illumina_R2.fastq.gz data/wes_illumina_R1.fastq.gz data/wes_illumina_R2.fastq.gz data/wgs_ont.fastq.gz data/wgs_pacbio.fastq.gz \
	data/wes_illumina_mutect.funcotated.vcf \
	data/wgs_illumina_manta.vcf.gz data/wgs_pacbio_sniffles.vcf.gz data/wgs_ont_sniffles.vcf.gz \
	data/wes_illumina_R1.fastqc.html data/wes_illumina_R2.fastqc.html data/wgs_illumina_R1.fastqc.html data/wgs_illumina_R2.fastqc.html data/wgs_ont.fastqc.html data/wgs_pacbio.fastqc.html 

data ref tmp: 
	mkdir -p $@
	sleep 2

# download reference genome
ref/hs37d5.fa ref/hs37d5.fa.fai ref/hs37d5.dict: ref
	-wget -O- -q --show-progress http://ftp.ncbi.nlm.nih.gov/1000genomes/ftp/technical/reference/phase2_reference_assembly_sequence/hs37d5.fa.gz | gzip -dc > ref/hs37d5.fa 2> /dev/null
	samtools faidx ref/hs37d5.fa
	picard CreateSequenceDictionary -R ref/hs37d5.fa -O ref/hs37d5.dict
	sleep 2

# download SK-BR-3 RNA-seq bam files from AWS bucket (https://aws.amazon.com/marketplace/pp/prodview-4wmr6pg5beutw)
data/rna_seq.bam data/rna_seq.bam.bai: data tmp
	aws s3 cp --no-sign-request s3://gdc-ccle-2-open/4eb9d31a-ea92-4846-8791-59f3ad7837e3/G27493.SK-BR-3.2.bam data/rna_seq.bam
	aws s3 cp --no-sign-request s3://gdc-ccle-2-open/4eb9d31a-ea92-4846-8791-59f3ad7837e3/G27493.SK-BR-3.2.bam.bai data/rna_seq.bam.bai

# download RNA-seq expression values from the Cancer Depedency Map (https://depmap.org/portal/download/all/), subset to just our sample
data/rna_seq_genes_tpm.csv: data tmp
	wget -O- -q --show-progress https://ndownloader.figshare.com/files/34989919 > tmp/CCLE_expression.csv
	head -n 1 tmp/CCLE_expression.csv > $@
	grep "ACH-000017" tmp/CCLE_expression.csv >> $@

# download WGS data from Schatz lab website
data/wgs_illumina.bam data/wgs_illumina.bam.bai: data tmp
	wget -O- -q --show-progress http://labshare.cshl.edu/shares/schatzlab/www-data/skbr3/SKBR3_550bp_pcrFREE_S1_L001_AND_L002_R1_001.101bp.bwamem.ill.mapped.sort.bam > tmp/wgs_illumina.bam
	samtools sort -@ $(threads) -n tmp/wgs_illumina.bam | samtools addreplacerg -r '@RG\tID:SK-BR-3\tSM:SK-BR-3' - | samtools fixmate -m - tmp/wgs_illumina.fixmate.bam
	samtools sort -@ $(threads) tmp/wgs_illumina.fixmate.bam | samtools markdup - data/wgs_illumina.bam
	samtools index data/wgs_illumina.bam

data/wgs_ont.bam data/wgs_ont.bam.bai: data
	wget -O- -q --show-progress http://labshare.cshl.edu/shares/schatzlab/www-data/skbr3/skbr3.ont.sort.bam > data/wgs_ont.bam
	samtools index data/wgs_ont.bam

data/wgs_pacbio.bam data/wgs_pacbio.bam.bai: data
	wget -O- -q --show-progress http://labshare.cshl.edu/shares/schatzlab/www-data/skbr3/reads_lr_skbr3.fa_ngmlr-0.2.3_mapped.bam > tmp/wgs_pacbio.bam
	samtools sort -@ $(threads) tmp/wgs_pacbio.bam > data/wgs_pacbio.bam
	samtools index data/wgs_pacbio.bam

# subset WGS bam over exome intervals to generate a WES bam
data/wes_illumina.bam data/wes_illumina.bam.bai: ref/hs37d5.fa ref/hs37d5.dict data/wgs_illumina.bam
	# download panel and convert contigs from hg19 to hs37d5
	wget -O- -q --show-progress https://sfvideo.blob.core.windows.net/sitefinity/docs/default-source/supplementary-product-info/xgen-exome-hyb-panel-v2-targets-hg19.bed | \
		sed s,chr,, | \
		perl -pe 's/[^\s_]+_([^\s_]+)_random/$1.1/' | \
		tr "gl" "GL" > ref/exome_panel.bed

	# extract reads in the panel intervals
	picard BedToIntervalList -I ref/exome_panel.bed -O ref/exome_panel.interval_list -SD ref/hs37d5.dict
	picard FilterSamReads -I data/wgs_illumina.bam -O data/wes_illumina.bam --INTERVAL_LIST ref/exome_panel.interval_list --FILTER includePairedIntervals
	samtools index data/wes_illumina.bam

# make fastq files from subsampling the WGS bam
data/%_illumina_R1.fastq.gz data/%_illumina_R2.fastq.gz: data/%_illumina.bam
	samtools sort -@ $(threads) -n $< | samtools fastq -@ $(threads) -1 data/$*_illumina_R1.fastq.gz -2 data/$*_illumina_R2.fastq.gz -0 /dev/null -s /dev/null -n -

data/wgs_ont.fastq.gz: data/wgs_ont.bam
	samtools sort -@ $(threads) -n $< | samtools fastq -@ $(threads) - | gzip > $@

data/wgs_pacbio.fastq.gz: data/wgs_pacbio.bam
	samtools sort -@ $(threads) -n $< | samtools fastq -@ $(threads) - | gzip > $@

data/%.fastqc.html: data/%.fastq.gz
	fastqc -t $(threads) -o data $<

# Call variants from illumina data using Mutect2
data/wes_illumina_mutect.vcf.gz: data/wes_illumina.bam ref/hs37d5.fa
	gatk Mutect2 -R ref/hs37d5.fa -I data/wes_illumina.bam -O $@ --native-pair-hmm-threads $(threads)

# download funcotator data sources
ref/funcotator_dataSources: ref
	wget -O- -q --show-progress https://storage.googleapis.com/broad-public-datasets/funcotator/funcotator_dataSources.v1.7.20200521s.tar.gz | tar zxf -
	mv funcotator_dataSources.v1.7.20200521s $@

# annotate variants with Funcotator
data/wes_illumina_mutect.funcotated.vcf: data/wes_illumina_mutect.vcf.gz ref/funcotator_dataSources ref/hs37d5.fa
	gatk Funcotator \
		--variant data/wes_illumina_mutect.vcf.gz \
		--reference ref/hs37d5.fa \
		--ref-version hg19 \
		--data-sources-path ref/funcotator_dataSources \
		--output $@ \
		--output-file-format VCF

# Download structural variant calls
data/wgs_illumina_manta.vcf.gz:
	wget -O- -q --show-progress https://labshare.cshl.edu/shares/schatzlab/www-data/skbr3/SKBR3_550bp_pcrFREE_S1_L001_AND_L002_R1_001.101bp.bwamem.ill.mapped.sort.bam.manta_noalt.vcf.gz | gzip -dc | bgzip -c > $@

data/wgs_pacbio_sniffles.vcf.gz:
	wget -O- -q --show-progress https://labshare.cshl.edu/shares/schatzlab/www-data/skbr3/bcorganoid/SKBR3_PACBIO_sniffles.vcf | gzip -c > $@

data/wgs_ont_sniffles.vcf.gz:
	wget -O- -q --show-progress https://labshare.cshl.edu/shares/schatzlab/www-data/skbr3/bcorganoid/SKBR3_ONT_sniffles.vcf | gzip -c > $@

clean: 
	rm -rf tmp
