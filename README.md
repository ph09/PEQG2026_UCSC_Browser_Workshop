# PEQG2026_UCSC_Browser_Workshop

Build your own UCSC Genome Browser for a genome that **isn't in UCSC/GenArk yet** — turn a FASTA into a `.2bit`, attach your own tracks (for example, **Cactus** and **minimap2** alignments to a reference species), host it, and load it. If you just assembled a genome but cannot submit to GenBank just yet, you can make an **assembly hub**!

## 0. What is an assembly hub?

A **track hub** adds tracks on top of a reference UCSC already hosts (e.g. hg38, or a publicly available assembly in GenArk). An **assembly hub** also supplies the reference sequence itself, as a `.2bit` file, so you can browse a genome UCSC has never seen.

A hub is just a folder of files reachable over `https://`, organized into three plain-text config files plus your data, with **each genome in its own subdirectory**:

```
myHub/
├── hub.txt                      # names the hub, points to genomes.txt
├── genomes.txt                  # one stanza per assembly: its sequence + trackDb
└── toyGenome/                   # one directory per genome
    ├── toyGenome.2bit           # the reference sequence
    ├── trackDb.txt              # the tracks for this genome
    ├── toyRegions.bb            # bigBed annotations
    ├── refReads.sorted.bam      # minimap2 alignment ...
    ├── refReads.sorted.bam.bai  # ... and its index
    └── alignment.hal            # Cactus alignment
```

Three files, three jobs:

- **hub.txt** — top level: the hub's name and a pointer to `genomes.txt`.
- **genomes.txt** — one stanza per assembly: where its `.2bit` and `trackDb.txt` live, plus display defaults.
- **trackDb.txt** — inside each genome's directory: one stanza per track.

(For a single genome you *can* collapse all three into one file with `useOneFile on`, but this split layout extends cleanly to multiple assemblies — just add another stanza and another subdirectory.)

## 1. Install the kent tools

The UCSC "kent" utilities do the conversions.

```
conda create -c conda-forge -c bioconda -n ucsc_hub \
  ucsc-fatotwobit ucsc-twobitinfo ucsc-bedtobigbed ucsc-hubcheck samtools
conda activate ucsc_hub
```

(The kent tools are also precompiled at `http://hgdownload.soe.ucsc.edu/admin/exe/`.)

## 2. FASTA → 2bit

This is the step that makes the genome browsable. Work inside the genome's directory (`myHub/toyGenome/`):

```
faToTwoBit toyGenome.fa toyGenome.2bit
twoBitInfo toyGenome.2bit stdout | sort -k2,2nr > toyGenome.chrom.sizes
cat toyGenome.chrom.sizes
```

Expected:

```
scaffold_1    5000
scaffold_2    3000
```

> The names in `chrom.sizes` are now the *only* names the browser knows. Every coordinate you give it later — track features, `defaultPos`, and the genome names inside your alignments — must use these exact names.

> If your BED/VCF, collaborators, or a reference use different sequence names (GenBank `CM…`, RefSeq `NC_…`, Ensembl `1`, UCSC `chr1`), add a `chromAlias` file so the browser accepts all of them for searches and custom tracks. Make a tab-separated `toyGenome/toyGenome.chromAlias.txt` — a header naming each scheme, then one row per sequence with your assembly's names in the first column:
>
> ```
> # ucsc	assembly	genbank	refseq
> scaffold_1	1	CM000001.1	NC_000001.1
> scaffold_2	2	CM000002.1	NC_000002.1
> ```
>
> Then add `chromAlias toyGenome/toyGenome.chromAlias.txt` to the genome stanza in `genomes.txt` (the path is relative to `hub.txt`). For an assembly with thousands of sequences, convert it to bigBed and use `chromAliasBb` instead so name lookups stay fast.

## 3. Build the tracks

All track data files go in the genome's directory (`myHub/toyGenome/`), next to `trackDb.txt`.

### 3a. Your annotations (BED → bigBed) — *runs live*

`toyRegions.bed` holds five regions in BED4 (`chrom start end name`). bigBed input must be sorted, then indexed against `chrom.sizes`:

```
sort -k1,1 -k2,2n toyRegions.bed > toyRegions.sorted.bed
bedToBigBed toyRegions.sorted.bed toyGenome.chrom.sizes toyRegions.bb
```

Same recipe covers most data: signal → `bigWig`, variants → `bgzip`+`tabix` VCF, gene models → `bigGenePred`.

### 3b. minimap2 alignment → BAM track

To compare a reference species, align it **to your assembly** so the result is in *your* coordinates, then sort and index.

```
minimap2 -a toyGenome.fa refSpecies.fa | samtools sort -o refReads.sorted.bam
samtools index refReads.sorted.bam        # produces refReads.sorted.bam.bai
```

Hosting it is then just one stanza in `trackDb.txt`, `type bam`:

```
track refMinimap2
shortLabel Ref (minimap2)
longLabel Reference species aligned to my assembly with minimap2
type bam
visibility pack
bigDataUrl refReads.sorted.bam
```

> A `bam` track needs `refReads.sorted.bam.bai` hosted in the same folder, same root name. No `.bai`, no track. (Same rule for `vcfTabix` and its `.tbi`.)
> Your assembly must be the minimap2 *target* (`-a target.fa query.fa`). If you aligned the other way round, the BAM is in the reference's coordinates and won't display on your hub.

### 3c. Cactus alignment → HAL snake track — *pre-made*

Conceptually:

```
# seqFile lists the tree + each genome's FASTA, e.g.:
#   (refSpecies,toyGenome);
#   refSpecies   refSpecies.fa
#   toyGenome    toyGenome.fa
cactus ./jobstore seqFile.txt alignment.hal
halStats --genomes alignment.hal          # confirm the genome names in the HAL
```

Snake tracks are UCSC's native HAL visualization. Hosting is one stanza in `trackDb.txt`, `type halSnake`:

```
track refCactusSnake
shortLabel Ref (Cactus)
longLabel Cactus/HAL alignment of reference species onto my assembly
type halSnake
otherSpecies refSpecies
visibility full
bigDataUrl alignment.hal
```

> The hub's `genome` value (e.g. `toyGenome`) must be one of the genome names *inside* the `.hal`, and `otherSpecies` must be another. Use the names `halStats --genomes` reports — these are the names you set in the Cactus `seqFile`, so make them match your assembly's name from the start.
>
> *Shortcut:* `hal2assemblyHub.py alignment.hal outDir` (from the HAL toolkit) auto-builds an entire comparative assembly hub — 2bit, snake tracks for every pair, the works — instead of hand-writing the stanza. Handy once you have several genomes.

## 4. Wire it together: hub.txt, genomes.txt, trackDb.txt

**`myHub/hub.txt`** — names the hub and points at `genomes.txt`:

```
hub PEQG2026AssemblyHub
shortLabel My VGP genome
longLabel Assembly hub for a genome not yet in GenArk (PEQG 2026 demo)
genomesFile genomes.txt
email you@institution.edu
```

**`myHub/genomes.txt`** — one stanza for the assembly. Paths are relative to `hub.txt`, so each is prefixed with the genome's directory:

```
genome toyGenome
trackDb toyGenome/trackDb.txt
twoBitPath toyGenome/toyGenome.2bit
organism My species
scientificName Genus species
description PEQG 2026 demo assembly
defaultPos scaffold_1:1-5000
orderKey 10
```

> The `genome` value (`toyGenome`) is a label *you* invent — that, plus `twoBitPath` supplying the sequence, is what makes this an *assembly* hub rather than a track hub on an existing UCSC/GenArk assembly. To add a second genome, leave a blank line and add another stanza pointing at its own subdirectory.

**`myHub/toyGenome/trackDb.txt`** — one stanza per track:

```
track candidateRegions
shortLabel Candidate regions
longLabel Regions of interest I generated
type bigBed 4
visibility pack
color 20,90,160
bigDataUrl toyRegions.bb

track refMinimap2
shortLabel Ref (minimap2)
longLabel Reference species aligned to my assembly with minimap2
type bam
visibility pack
bigDataUrl refReads.sorted.bam

track refCactusSnake
shortLabel Ref (Cactus)
longLabel Cactus/HAL alignment of reference species onto my assembly
type halSnake
otherSpecies refSpecies
visibility full
bigDataUrl alignment.hal
```

> `bigDataUrl` here is relative to `trackDb.txt`. Since the data files sit in the same `toyGenome/` directory, the paths are just bare filenames.

## 5. Host it

Upload the whole `myHub/` directory to a server that supports **HTTP byte-range requests** — an institutional web server, an S3/cloud bucket, or GitHub Pages (Settings → Pages → deploy from `main`) for a small demo. Because every path inside the hub is relative, you upload the folder as-is; your hub URL is the path to `hub.txt`:

```
https://<your-user>.github.io/<your-repo>/myHub/hub.txt
```

Make sure the BAM index (`refReads.sorted.bam.bai`) is uploaded alongside its `.bam`. A real vertebrate `.2bit`, BAM, and HAL run from hundreds of MB to several GB and will exceed GitHub's 100 MB/file limit — host those on institutional or cloud storage and keep GitHub for small demos.

## 6. Load it in the browser

- **Menu:** My Data → Track Hubs → **My Hubs** → paste your `hub.txt` URL → **Add Hub**. Your assembly appears in the genome drop-down.
- **Direct link** (good for slides):

```
https://genome.ucsc.edu/cgi-bin/hgTracks?genome=toyGenome&hubUrl=<URL-to-myHub/hub.txt>
```

You should land on `scaffold_1` with three tracks: your candidate regions, the minimap2 reads, and the Cactus snake showing where the reference aligns.

## 7. Validate & iterate

```
hubCheck <URL-to-your-hub.txt>
```

The browser caches hub files for ~5 minutes; append `udcTimeout=1` to your browser URL to see edits immediately.

## 8. Going further (real genomes)

- **Description page / track groups / cytoband ideogram** — add a `description.html` (referenced with `htmlPath` in the genome stanza), a `groups.txt`, and a `cytoBandIdeo` track inside the genome directory to polish a real hub.
- **BLAT / In-Silico PCR** — run a local `gfServer` and add `blat`/`transBlat`/`isPcr` lines to the genome stanza so users can search your sequence.
- **Automate** — `hal2assemblyHub.py` (HAL toolkit) for comparative hubs; [MakeHub](https://github.com/Gaius-Augustus/MakeHub) and [G-OnRamp](https://g-onramp.org/) for annotation hubs.
- **Make it public / GenArk** — if your assembly is in NCBI with a `GCA_/GCF_` accession, request it at `https://genome.ucsc.edu/assemblyRequest` and UCSC will host the browser for you.

## References

- Assembly Hub User Guide — https://genome.ucsc.edu/goldenPath/help/assemblyHubHelp.html
- Track Database (trackDb) settings — https://genome.ucsc.edu/goldenPath/help/trackDb/trackDbHub.html
- HAL toolkit / hal2assemblyHub — https://github.com/ComparativeGenomicsToolkit/hal
- GenArk (VGP & HPRC assemblies) — https://hgdownload.soe.ucsc.edu/hubs/
