
# Ebola SNP calling pipeline
# Usage: make vcf or make all-vcf
# To process a specific sample: make vcf SRR=SRR1553425

SHELL = bash

# GenBank accession for reference genome (Zaire ebolavirus)
ACC ?= AF086833

# Default SRR accession (can override: make vcf SRR=SRR1553426)
SRR ?= SRR1553425

# All three samples in this project
SAMPLES = SRR10971381 SRR1553425 SRR1553426

# CPU cores
CPU ?= 4

# Reference genome paths
REF = data/genomes/${ACC}.fa
REF_IDX = ${REF}.fai
GFF = data/genomes/${ACC}.gff
BWA_IDX = ${REF}.amb

# Root name for sample
ROOT = ${SRR}-${ACC}

# File paths for a single sample
BAM = bam/${ROOT}.bam
VCF = vcf/${ROOT}.vcf.gz

# .PHONY targets
.PHONY: help install download-genome index-bam all-vcf vcf clean status vcf-filtered stats compare-snps report igv-prepare

# Check GNU make version >= 4
MAKE_MAJOR := $(firstword $(subst ., ,$(MAKE_VERSION)))
ifeq ($(shell [ $(MAKE_MAJOR) -lt 4 ] && echo yes),yes)
    $(error This Makefile needs make >= 4. Update with: pixi add make)
endif

# Check required programs (optional check)
EXE = samtools bcftools
K := $(foreach exec,$(EXE),$(if $(shell which $(exec)),ok,$(error "Program: $(exec) not found.")))

# Help
help:
	@echo "Ebola SNP calling pipeline"
	@echo ""
	@echo "VCF Generation:"
	@echo "  make download-genome  # Download reference genome (once)"
	@echo "  make index-bam        # Index existing BAM files"
	@echo "  make vcf SRR=SRR1553425  # Generate VCF for specific sample"
	@echo "  make all-vcf          # Generate VCF for all samples"
	@echo ""
	@echo "Analysis & Filtering:"
	@echo "  make vcf-filtered     # Filter VCF by quality (QUAL >= 20)"
	@echo "  make stats            # Show VCF statistics"
	@echo "  make compare-snps     # Compare SNPs between samples"
	@echo "  make report           # Generate comprehensive report"
	@echo ""
	@echo "Visualization:"
	@echo "  make igv-prepare      # Copy files to C:\\ebola_igv\\ (Windows IGV)"
	@echo ""
	@echo "Utilities:"
	@echo "  make status           # Show current BAM and VCF files"
	@echo "  make clean            # Remove generated files"
	@echo ""

# Download reference genome and GFF
download-genome: ${REF} ${GFF}

# Get GFF annotation
${GFF}:
	@mkdir -p data/genomes
	@echo "Downloading GFF for ${ACC}..."
	bio fetch ${ACC} -format gff > ${GFF}

# Get reference FASTA
${REF}: ${GFF}
	@mkdir -p data/genomes
	@echo "Downloading reference FASTA for ${ACC}..."
	bio fetch ${ACC} -format fasta > ${REF}
	samtools faidx ${REF}

# FASTA index
${REF_IDX}: ${REF}
	@echo "Creating FASTA index..."
	samtools faidx ${REF}

# BWA index (optional, not needed for SNP calling from existing BAM)
${BWA_IDX}: ${REF}
	@echo "Creating BWA index..."
	bwa index ${REF}

# Index existing BAM files (for samples that already have BAM)
index-bam:
	@echo "Indexing BAM files..."
	@for file in bam/*.bam; do \
		if [ -f "$$file" ] && [ ! -f "$$file.bai" ]; then \
			echo "Indexing $$file"; \
			samtools index "$$file"; \
		fi; \
	done
	@echo "Done. BAM files indexed."

# VCF generation from existing BAM file
${VCF}: ${REF_IDX} bam/${ROOT}.bam
	@mkdir -p vcf
	@echo "Calling variants for ${ROOT}..."
	@if [ ! -f "bam/${ROOT}.bam" ]; then \
		echo "ERROR: BAM file bam/${ROOT}.bam not found"; \
		exit 1; \
	fi
	@if [ ! -f "bam/${ROOT}.bam.bai" ]; then \
		echo "Indexing BAM file..."; \
		samtools index bam/${ROOT}.bam; \
	fi
	bcftools mpileup -O v -f ${REF} bam/${ROOT}.bam | \
	bcftools call --ploidy 1 -mv -O z -o ${VCF}
	bcftools index ${VCF}
	@echo "VCF generated: ${VCF}"
	@ls -lh ${VCF}

# Generate VCF for default sample
vcf: ${VCF}

# Generate VCF for all samples
all-vcf: download-genome index-bam $(foreach sample,${SAMPLES},vcf/${sample}-${ACC}.vcf.gz)

# Pattern rule for all samples
vcf/%-${ACC}.vcf.gz: ${REF_IDX}
	@mkdir -p vcf
	@echo "Calling variants for $*..."
	@if [ ! -f "bam/$*-${ACC}.bam" ]; then \
		echo "ERROR: BAM file bam/$*-${ACC}.bam not found"; \
		exit 1; \
	fi
	@if [ ! -f "bam/$*-${ACC}.bam.bai" ]; then \
		echo "Indexing bam/$*-${ACC}.bam..."; \
		samtools index bam/$*-${ACC}.bam; \
	fi
	bcftools mpileup -O v -f ${REF} bam/$*-${ACC}.bam | \
	bcftools call --ploidy 1 -mv -O z -o $@
	bcftools index $@
	@echo "Generated: $@"
	@ls -lh $@

# Summary of BAM and VCF files
status:
	@echo "=== BAM FILES ==="
	@ls -lh bam/*.bam 2>/dev/null || echo "No BAM files found"
	@echo ""
	@echo "=== BAM INDEXES ==="
	@ls -lh bam/*.bam.bai 2>/dev/null || echo "No BAM indexes found"
	@echo ""
	@echo "=== VCF FILES ==="
	@ls -lh vcf/*.vcf.gz 2>/dev/null || echo "No VCF files found"
	@echo ""
	@echo "=== REFERENCE GENOME ==="
	@ls -lh ${REF} ${REF_IDX} 2>/dev/null || echo "Reference genome not yet downloaded"

# VCF filtering by quality (QUAL >= 20)
vcf-filtered: 
	@echo "Filtering VCF files by quality (QUAL >= 20)..."
	@mkdir -p vcf/filtered
	@for vcf in vcf/*-${ACC}.vcf.gz; do \
		if [ -f "$$vcf" ]; then \
			base=$$(basename $$vcf); \
			echo "Filtering $$base..."; \
			bcftools filter -O z -o vcf/filtered/$$base -i 'QUAL >= 20' $$vcf; \
			bcftools index vcf/filtered/$$base; \
		fi; \
	done
	@echo "Filtered VCF files are in vcf/filtered/"
	@ls -lh vcf/filtered/*.vcf.gz

# VCF statistics summary
stats:
	@echo "=== VCF STATISTICS ==="
	@for vcf in vcf/*-${ACC}.vcf.gz; do \
		if [ -f "$$vcf" ]; then \
			echo ""; \
			echo "File: $$(basename $$vcf)"; \
			echo "---"; \
			bcftools stats $$vcf | grep "^SN" | cut -f 3-; \
		fi; \
	done

# Compare SNPs between samples (shared and unique)
compare-snps:
	@echo "=== SNP COMPARISON ==="
	@mkdir -p reports
	@echo "Extracting positions from each VCF..."
	@bcftools query -f '%CHROM\t%POS\n' vcf/SRR1553425-${ACC}.vcf.gz | sort > /tmp/SRR1553425.pos
	@bcftools query -f '%CHROM\t%POS\n' vcf/SRR1553426-${ACC}.vcf.gz | sort > /tmp/SRR1553426.pos
	@echo ""
	@echo "=== RESULTS ==="
	@echo "SRR1553425 total SNPs: $$(wc -l < /tmp/SRR1553425.pos)"
	@echo "SRR1553426 total SNPs: $$(wc -l < /tmp/SRR1553426.pos)"
	@echo "Shared SNPs: $$(comm -12 /tmp/SRR1553425.pos /tmp/SRR1553426.pos | wc -l)"
	@echo "Unique to SRR1553425: $$(comm -23 /tmp/SRR1553425.pos /tmp/SRR1553426.pos | wc -l)"
	@echo "Unique to SRR1553426: $$(comm -13 /tmp/SRR1553425.pos /tmp/SRR1553426.pos | wc -l)"
	@echo ""
	@echo "Shared SNP positions saved to reports/shared_snps.txt"
	@comm -12 /tmp/SRR1553425.pos /tmp/SRR1553426.pos > reports/shared_snps.txt

# Generate comprehensive report
report: stats compare-snps
	@echo "=== PIPELINE SUMMARY ==="
	@echo "Reference: ${ACC}"
	@echo "Samples analyzed: SRR1553425, SRR1553426"
	@echo "VCF files: vcf/*-${ACC}.vcf.gz"
	@echo "Reports: reports/"
	@ls -lh reports/

# Prepare files for IGV (copy to Windows-accessible location)
igv-prepare:
	@echo "Preparing IGV files..."
	@mkdir -p /mnt/c/ebola_igv
	@echo "Copying reference genome..."
	@cp data/genomes/${ACC}.fa /mnt/c/ebola_igv/
	@cp data/genomes/${ACC}.fa.fai /mnt/c/ebola_igv/ 2>/dev/null || true
	@echo "Copying BAM files..."
	@cp bam/*.bam* /mnt/c/ebola_igv/
	@echo "Copying VCF files..."
	@cp vcf/*-${ACC}.vcf.gz* /mnt/c/ebola_igv/
	@echo ""
	@echo "✓ Files ready for IGV:"
	@echo "  Location: C:\\ebola_igv\\"
	@echo ""
	@echo "IGV usage:"
	@echo "  1. Open IGV Desktop (Windows)"
	@echo "  2. Genomes → Load Genome from File → C:\\ebola_igv\\${ACC}.fa"
	@echo "  3. File → Load from File → C:\\ebola_igv\\SRR1553425-${ACC}.bam"
	@echo "  4. File → Load from File → C:\\ebola_igv\\SRR1553425-${ACC}.vcf.gz"
	@echo "  5. Type a position in the search bar (e.g., '1-1000')"
	@echo ""
	@ls -lh /mnt/c/ebola_igv/

# Clean up generated files
clean:
	@echo "Cleaning up..."
	rm -f ${VCF} vcf/*.vcf.gz* vcf/filtered/*.vcf.gz* bam/*.bam.bai ${BWA_IDX} ${REF}* ${GFF}
	rm -f /tmp/*.pos
	@echo "Done."

# Installation instructions
install:
	@echo "Install required tools with:"
	@echo "  pixi add bio samtools bcftools"
