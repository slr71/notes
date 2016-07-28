# NCBI WGS Submission

## Submission XML

### Genome Element

Sequence
- Description (complex type)

### Description Element

Sequence
- GenomeAssemblyMetadataChoice (complex type)
- GapInfo (typeGapInfo)
- GenomeRepresentation (complex type)
- SequenceAuthors (com:typeAuthorSet)
- Publication (com:typePublication)
- ExpectedFinalVersion (typeYesNo)
- SingleCellAmplification (xs:string)
- AnnotateWithPGAP (typeYesNo)
- ExistingWGSAccession (xs:string)
- Corresponding16SrRNA (xs:string)
- BacteriaAndSourceDNAIsAvailableFrom (xs:string)
- Notes (xs:string)

### GenomeAssemblyMetadataChoice Element

Choice
- GenomeAssemblyMetadata (complex type)
- StructuredComment (simple element)

### GenomeAssemblyMetadata Element

Sequence
- SequencingTechnologies (complex type)
- Assembly (complex type)
- AssemblyName (typeNonEmptyString)

### SequencingTechnologies Element

Sequence
- Technology (complex type)
- Coverage (attribute, typeCoverage)

### Technology Element

This element is an extension of typeSequencingTechnology. Only the extensions will be documented here.

Sequence
- other_description (attribute, typeNonEmptyString)
- coverage (attribute, typeCoverage)

### Assembly Element

Sequence
- Method (complex type)
- reference_assembly (attribute, xs:string)
- assembly_date (attribute, xs:date)

### Method Element

This element is an extension of typeAssemblyMethod. Only the extensions will be documented here.

Simple Content
- other_description (attribute, typeNonEmptyString)
- version (attribute, typeNonEmptyString)

### GenomeRepresentation Element

This element is an extension of typeGenomeRepresentation. Only the extensions will be documented here.

Simple Content
- genome_representation_description (attribute, typeNonEmptyString)

### typeNonEmptyString Type

This type is self-explanatory.

### typeCoverage Type

This is a restriction of xs:decimal with a range from 0 to 10000, exclusive.

### typeYesNo Type

Enumeration
- Yes
- No

### typeGapInfo Type

Simple Content
- min_gap_size (attribute, com:typeNumber)
- unknown_gap_size (attribute, com:typeNumber)
- linkage_evidence (attribute, typeLinkageEvidence)

### typeLinkageEvidence Type

Enumeration
- paired-ends
- align-genus
- align-xgenus
- strobe
- map
- within-clone
- clone-contig
- align-trnscpt

### typeGenomeRepresentation Type

Enumeration
- Full
- Partial

### typeSequencingTechnology Type

Enumeration
- ABI3730
- Sanger
- 454
- Illumina
- Illumina GAII
- Illumina GAIIx
- Illumina HiSeq
- Illumina MiSeq
- Illumina NextSeq
- Illumina NextSeq 500
- PacBio
- IonTorrent
- SOLiD
- Helicos
- Complete Genomics
- Other

### typeAssemblyMethod Type

Enumeration
- Newbler
- Celera Assembler
- SOAPdenovo
- SPAdes
- Velvet
- AllPaths
- GS De Novo Assembler
- MIRA
- phredPhrap
- ABySS
- CLC NGS Cell
- Arachne
- JAZZ
- MaSuRCA
- Other

### typeChromosomeType Type

Enumeration
- Chromosome
- Plasmid
- Mitochondrion
- Chloroplast
- Apicoplast
- Leucoplast
- Plastid
- Proplastid
- Hydrogenosome
- Kinetoplast

### typeRepliconMolType Type

Enumeration
- eChromosome
- ePlasmid
- eLinkageGroup
- eSegment
- eExtrachrom
- eOther

### typeRepliconLocation Type

Enumeration
- eNuclearProkaryote
- eMacronuclear
- eNucleomorph
- eMitochondrion
- eKinetoplast
- eChloroplast
- eChromoplast
- ePlastid
- eVirionPhage
- eProviralProphage
- eViroid
- eCyanelle
- eApicoplast
- eLeucoplast
- eProplastid
- eHydrogenosome
- eChromatophore
- eOther
