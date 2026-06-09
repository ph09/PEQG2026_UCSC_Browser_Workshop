# PEQG2026_UCSC_Browser_Workshop
Build your own UCSC Genome Browser for a genome that **isn't in UCSC/GenArk yet** — turn a FASTA into a `.2bit`, attach your own tracks, host it, and load it. If you just assembled a genome but cannot submit to GenBank just yet, you can make an **assembly hub**!

This walkthrough uses a short chromosome as the stand-in: the *Drosophila melanogaster* **Y chromosome** from dm6.

## 0. What is an assembly hub?
A **track hub** adds tracks on top of a reference UCSC already hosts (e.g. hg38, or a publicly available assembly in GenArk). An **assembly hub** also supplies the reference sequence itself, as a `.2bit` file, so you can browse a new genome.
A hub is just a folder of files reachable over `https://`, organized into three plain-text config files plus your data, with **each genome in its own subdirectory**:
```
myHub/
├── hub.txt                      # names the hub, points to genomes.txt
├── genomes.txt                  # one stanza per assembly: its sequence + trackDb
└── dmelY/                       # one directory per genome
    ├── dmelY.2bit               # the reference sequence (dm6 chrY)
    ├── trackDb.txt              # the tracks for this genome
    ├── exampleRegions.bb        # bigBed annotations
    ├── refReads.sorted.bam      # minimap2 alignment ...
    ├── refReads.sorted.bam.bai  # ... and its index
    └── alignment.hal            # Cactus alignment
```
- **hub.txt** — top level: the hub's name and a pointer to `genomes.txt`.
- **genomes.txt** — one stanza per assembly: where its `.2bit` and `trackDb.txt` live, plus display defaults.
- **trackDb.txt** — inside each genome's directory: one stanza per track.
(For a single genome you *can* collapse all three into one file with `useOneFile on`, but this split layout extends cleanly to multiple assemblies — just add another stanza and another subdirectory.)
## 1. Install the kent tools
The UCSC "kent" utilities do the conversions.
```
conda create -c conda-forge -c bioconda -n ucsc_hub \
  ucsc-fatotwobit ucsc-twobitinfo ucsc-twobittofa ucsc-bedtobigbed ucsc-hubcheck samtools
conda activate ucsc_hub
```
(The kent tools are also precompiled at `http://hgdownload.soe.ucsc.edu/admin/exe/`.)
## 2. Get the sequence and convert: FASTA → 2bit
We'll grab the real Y chromosome as FASTA or you can just use the one in this repo. Just pull the `chrY` out of the complete `2bit` file. Work inside the genome's directory (`myHub/dmelY/`):
```
mkdir myHub
cd myHub
mkdir dmelY
cd dmelY
wget https://hgdownload.soe.ucsc.edu/goldenPath/dm6/bigZips/dm6.2bit
twoBitToFa dm6.2bit:chrY dm6_chrY.fa        # extract just the Y chromosome
```
Now the step that makes the genome browsable — FASTA → 2bit — plus a `chrom.sizes`:
```
faToTwoBit dm6_chrY.fa dmelY.2bit
twoBitInfo dmelY.2bit stdout | sort -k2,2nr > dmelY.chrom.sizes
cat dmelY.chrom.sizes
```
Expected:
```
chrY    3667352
```
(Your own project would start right here at `faToTwoBit yourAssembly.fa …`)
> The names in `chrom.sizes` (here just `chrY`) are now the *only* names the browser knows. Every coordinate you give it later — track features, `defaultPos`, and the genome names inside your alignments — must use these exact names.
> If your BED/VCF, collaborators, or a reference use different sequence names (GenBank `CM…`, RefSeq `NC_…`, Ensembl `Y`, UCSC `chrY`), add a `chromAlias` file so the browser accepts all of them for searches and custom tracks. It's a tab-separated table — a header naming each scheme, then one row per sequence with your assembly's names in the first column:
>
> ```
> # ucsc	assembly	genbank	refseq
> chrY	Y	<GenBank-id>	<RefSeq-id>
> ```
>
> Then add `chromAlias dmelY/dmelY.chromAlias.txt` to the genome stanza in `genomes.txt` (the path is relative to `hub.txt`). For an assembly with thousands of sequences, convert it to bigBed and use `chromAliasBb` so name lookups stay fast.
## 3. Build the tracks
All track data files go in the genome's directory (`myHub/dmelY/`), next to `trackDb.txt`.
### 3a. Your annotations (BED → bigBed)
`exampleRegions.bed` holds five demo windows along chrY in BED4 (`chrom start end name`) — stand-ins for anything you'd annotate (genes, repeats, candidate regions). bigBed input must be sorted, then indexed against `chrom.sizes`:
```
sort -k1,1 -k2,2n ../../dm6_chrY_exampleRegion.bed > exampleRegions.sorted.bed
bedToBigBed exampleRegions.sorted.bed dmelY.chrom.sizes exampleRegions.bb
```
Same recipe covers most data: signal → `bigWig`, variants → `bgzip`+`tabix` VCF, gene models → `bigGenePred`. (You could also pull dm6's NCBI RefSeq gene models for chrY and convert those in case you want to try this out!)
Add this to the `trackDb.txt` file in the same directory.
```
track exampleRegions
shortLabel Example regions
longLabel Demo regions of interest along chrY
type bigBed 4
visibility pack
color 20,90,160
bigDataUrl exampleRegions.bb
```
### 3b. Alignment BAM track
You can align a different genome sequence or reads using minimap2 (or your aligner of choice!) **to your assembly** so the result is in *your* coordinates, then sort and index.
```
minimap2 -a chrY.fa refSpecies.fa | samtools sort -o refReads.sorted.bam
samtools index refReads.sorted.bam        # produces refReads.sorted.bam.bai
```
Hosting it is then just one stanza in `trackDb.txt`, `type bam`:
```
track AlignmentOfX
shortLabel Ref (alignment)
longLabel Some sort of alignment
type bam
visibility pack
bigDataUrl refReads.sorted.bam
```
> A `bam` track needs `refReads.sorted.bam.bai` hosted in the same folder, same root name. 
> Your assembly must be the minimap2 *target* (`-a target.fa query.fa`). If you aligned the other way round, the BAM is in the reference's coordinates and won't display on your hub.
## 4. Wire it together: hub.txt, genomes.txt, trackDb.txt
**`myHub/hub.txt`** — names the hub and points at `genomes.txt`:
```
hub PEQG2026AssemblyHub
shortLabel Dmel Y (dm6)
longLabel Assembly hub demo: D. melanogaster Y chromosome (dm6)
genomesFile genomes.txt
email you@institution.edu
```
**`myHub/genomes.txt`** — one stanza for the assembly. Paths are relative to `hub.txt`, so each is prefixed with the genome's directory:
```
genome dmelY
trackDb dmelY/trackDb.txt
twoBitPath dmelY/dmelY.2bit
organism D. melanogaster Y
scientificName Drosophila melanogaster
description D. melanogaster Y chromosome (dm6, Aug 2014)
defaultPos chrY:800000-900000
orderKey 10
```
> The `genome` value (`dmelY`) is a label *you* invent — that, plus `twoBitPath` supplying the sequence, is what makes this an *assembly* hub rather than a track hub on an existing UCSC/GenArk assembly. Note it differs from the *sequence* name inside the `.2bit` (`chrY`), which is what `defaultPos` uses. To add a second genome, leave a blank line and add another stanza pointing at its own subdirectory.
**`myHub/dmelY/trackDb.txt`** — one stanza per track:
```
track exampleRegions
shortLabel Example regions
longLabel Demo regions of interest along chrY
type bigBed 4
visibility pack
color 20,90,160
bigDataUrl exampleRegions.bb

track refMinimap2
shortLabel Ref (minimap2)
longLabel Reference species aligned to chrY with minimap2
type bam
visibility pack
bigDataUrl refReads.sorted.bam

track refCactusSnake
shortLabel Ref (Cactus)
longLabel Cactus/HAL alignment of reference species onto chrY
type halSnake
otherSpecies refSpecies
visibility full
bigDataUrl alignment.hal
```
> `bigDataUrl` here is relative to `trackDb.txt`. Since the data files sit in the same `dmelY/` directory, the paths are just bare filenames.
## 5. Host it
Upload the whole `myHub/` directory to a server that supports **HTTP byte-range requests**. Because every path inside the hub is relative, you upload the folder as-is; your hub URL is the path to `hub.txt`:
```
https://wherever/your/folder/is/myHub/hub.txt
```
Make sure the BAM index (`refReads.sorted.bam.bai`) is uploaded alongside its `.bam`. 
## 6. Load it in the browser
- **Menu:** My Data → Track Hubs → **My Hubs** → paste your `hub.txt` URL → **Add Hub**. Your assembly appears in the genome drop-down.
- **Direct link**:
```
https://genome.ucsc.edu/cgi-bin/hgTracks?genome=dmelY&hubUrl=<URL-to-myHub/hub.txt>
```
You should land on `chrY` at the `defaultPos` window with one track.
## 7. Validate & iterate
```
hubCheck <URL-to-your-hub.txt>
```
The browser caches hub files for ~5 minutes; append `udcTimeout=1` to your browser URL to see edits immediately.
## 8. Add a BLAT / In-Silico PCR server
A `.2bit` is also all you need to give your assembly its own **BLAT** ("align my sequence to this genome") and **In-Silico PCR** ("where do these primers land"). You run one or two `gfServer` processes on a machine reachable from the internet, then point the hub at them from `genomes.txt`. 

From the directory holding `dmelY.2bit`, start the servers. Convention is a **translated** (amino-acid) server on `17777` and a **DNA** server on `17779` — the DNA one also handles In-Silico PCR:
```
rsync -avP hgdownload.soe.ucsc.edu::genome/admin/exe/linux.x86_64/blat/ ~/bin/
chmod +x ~/bin/gfServer

# translated / protein queries  -> transBlat
~/bin/gfServer start yourServer.yourInstitution.edu 17777 -trans -mask -log=dmelY.trans.log dmelY.2bit &

# DNA queries + In-Silico PCR    -> blat + isPcr
~/bin/gfServer start yourServer.yourInstitution.edu 17779 -stepSize=5 -log=dmelY.untrans.log dmelY.2bit &
```
Then add three lines to this assembly's stanza in `genomes.txt`, with **matching ports**:
```
transBlat yourServer.yourInstitution.edu 17777
blat yourServer.yourInstitution.edu 17779
isPcr yourServer.yourInstitution.edu 17779
```
BLAT and PCR now work for your assembly. This URL opens the BLAT page with your genome in the drop-down:
```
https://genome.ucsc.edu/cgi-bin/hgBlat?hubUrl=<URL-to-myHub/hub.txt>
```
- **Use a real public hostname/IP, not `localhost`** (unless you're running your own local browser/GBiB) — `genome.ucsc.edu` has to be able to reach the server — and make sure the chosen ports are open through your firewall.
- **The server is persistent.** A dedicated `gfServer` loads the index into memory and must keep running; add the commands to a startup script (e.g. `rc.local`) so they survive reboots. For many assemblies or infrequent use, set up a **dynamic** `gfServer` instead (pre-built indexes served on demand via an `xinetd` super-server) — see UCSC's "Running your own gfServer".
- **`-mask`** makes the translated server skip soft-masked (lowercase) repeats; drop it if you'd rather match repeats too.
- **Debug outside the browser first.** Check the server with `gfServer status yourServer.yourInstitution.edu 17779`, and run a test query with `gfClient` from the `.2bit` directory.
## 9. Going further
- **Description page / track groups / cytoband ideogram** — add a `description.html` (referenced with `htmlPath` in the genome stanza), a `groups.txt`, and a `cytoBandIdeo` track inside the genome directory to polish a real hub.
- **Make it public / GenArk** — if your assembly is in NCBI with a `GCA_/GCF_` accession, request it at `https://genome.ucsc.edu/assemblyRequest` and UCSC will host the browser for you.
## References
- Assembly Hub User Guide — https://genome.ucsc.edu/goldenPath/help/assemblyHubHelp.html
- Quick Start Guide to Assembly Hubs — https://genome.ucsc.edu/goldenPath/help/hubQuickStartAssembly.html
- Running your own gfServer — https://genomewiki.ucsc.edu/index.php/Running_your_own_gfServer
- Track Database (trackDb) settings — https://genome.ucsc.edu/goldenPath/help/trackDb/trackDbHub.html
- dm6 downloads (bigZips) — https://hgdownload.soe.ucsc.edu/goldenPath/dm6/bigZips/
