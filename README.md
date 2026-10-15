# frus-semantic-vectors

Per-volume semantic vectors for [FRUS Explorer](https://github.com/joshbotts/FRUS-Explorer) — the
Tier-2 "exact scoring" layer, fetched by the app alongside each volume it downloads — and, under
Releases, the query encoders the app can download.

**This repository holds derived data only.** No document text, no user data. Each `.vec` file is a
block of quantized embedding vectors, one per document, for a single *Foreign Relations of the
United States* volume.

## What is here

```
generations/89481b18/shards/<volumeId>.vec    553 files, 155 MB   EmbeddingGemma 2; builds 51 and later
generations/89481b18/shards-manifest.json     their lengths, SHA-256s and provenance pin
shards/<volumeId>.vec                         553 files, 155 MB   EmbeddingGemma 300m; builds 42 to 50
shards-manifest.json                          their lengths, SHA-256s and provenance pin
LICENSES/Apache-2.0.txt                       the licence of EmbeddingGemma 2's weights (release encoder-2)
```

Shard sizes track volume length, not an average: the smallest is 1 KB and the largest 965 KB, with a
median of 242 KB, in either generation (a file's size depends on its document count alone).

Each `shards-manifest.json` lists every file of its generation with its byte length and SHA-256. It
is a copy of the manifest the app bundles (`FRUSExplorer/Resources/semantic-shards-manifest.json` in
the app repository), and the bundled copy is the one that decides: a build verifies each shard it
downloads against the length and digest it shipped with, never against this file. Every shard's own
64-byte header carries the same provenance digest the app has bundled. A shard whose pin does not
match the app's bundled tier is refused rather than blended — vectors from two different
generations must never be scored against each other.

## Generations

**Each generation has a folder of its own, named by the first eight hex digits of its provenance
digest, and keeps it.** A build reads one generation at one address, and verifies what it fetches
against the manifest it shipped with, so a new generation published over the files an older build
reads would leave that build refusing every fetch. EmbeddingGemma 2's shards, published for build
51, are therefore under `generations/89481b18/`, and the 300m's stay in `shards/`, the folder
builds 42 to 50 were compiled against.

| generation | model | dims | folder | read by | provenance digest |
|---|---|---|---|---|---|
| current | EmbeddingGemma 2 | 512 | `generations/89481b18/` | build 51 and later | `89481b18…ad2809b7e6101` |
| previous | EmbeddingGemma 300m | 512 | `shards/` | builds 42 to 50 | `a726ca60…6aa6e3a64` |
| retired | EmbeddingGemma 300m | 256 | none: the 512 set replaced it in place on 2026-08-16, when every build was a TestFlight build | builds before 42 | `57fb4c4b…0ca4bfb14` |

The digest's inputs are the model, its weights' SHA-256, the width, the chunking, the document
prompt, the pooling and the quantization, so changing any of them makes a new generation. Each
generation is reproducible from its raw harvest with `SemanticVectorsGenerator` in the app
repository, verified byte for byte across independent runs for both 512 generations, over all three
bundled artifacts and all 553 shards. Nothing here needs to be archived to be recoverable.

## Changes since publication

**EmbeddingGemma 2 (`generations/89481b18/`)**, published 2026-10-15 for build 51: none.

**EmbeddingGemma 300m (`shards/`)**, published 2026-08-16. Three files have changed, each because
the Office of the Historian changed the volume it describes.

| date | file | what changed |
|---|---|---|
| 2026-09-09 | `frus1981-88v16.vec` | added: the volume was published, in part (88 documents) |
| 2026-10-10 | `frus1952-54v09p1.vec` | replaced: Documents 900–946 were restored to the volume, 899 → 946 rows |
| 2026-10-10 | `frus1902app1.vec` | replaced: two divisions stopped being documents, 198 → 196 rows |

A re-pack leaves every other file byte-identical: on 2026-10-10, 551 of the 553.

A volume whose documents change is embedded again as a whole, and embedding is not bit-reproducible
from one machine to another. So a replaced file's rows for the documents that did not change can
differ from their predecessors' in a few low bits. For the two files of 2026-10-10 the new vectors
agree with the ones they replace at a cosine of 0.999999 or better, measured before quantization
over the 2,264 text chunks both runs embedded.

**A replacement within a generation is made in place, and an older build of the app will not take
it.** Within a generation the address carries no version, and a build verifies a download against
the manifest it shipped with. A FRUS Explorer build whose bundled manifest predates a replacement
therefore refuses a fresh download of that file and scores that volume from its bundled tier alone;
a copy it already holds stays. For the two files of 2026-10-10 that is build 49 and earlier. The
files those builds expect are in this repository's history at commit `dd8295f`. An added file has
no such cost, since no earlier build names it.

## Shard format

All little-endian, the same in every generation.

| offset | size | field |
|---|---|---|
| 0 | 4 | magic `FRSS` |
| 4 | 4 | layout version (`1`) |
| 8 | 4 | dimensions (`512`) |
| 12 | 4 | document count |
| 16 | 32 | provenance digest (SHA-256) |
| 48 | 16 | zero padding |
| 64 | `count × 512` | int8 codes, one 512-byte row per document |
| … | `count × 4` | `Float32` per-vector quantization scale |

The width is read from the header rather than assumed, so a consumer needs no compile-time constant
to parse any generation — only the digest decides whether it may *use* what it parsed.

Vectors are L2-normalised before quantization, so `dot(a, b) × scaleA × scaleB` reconstructs cosine
similarity.

**Rows are positional, and identity does not live here.** Row *k* is the *k*th document of that
volume *in the order of the app's bundled index*, not in the order of whatever XML is on disk. The
mapping from row to document id lives in the app's bundled `semantic-vectors-index.json`. A consumer
that infers identity from a volume's XML will be wrong for any volume republished upstream since
these vectors were built.

## Provenance

| | EmbeddingGemma 2 (current) | EmbeddingGemma 300m (previous) |
|---|---|---|
| model | `embeddinggemma-2-Q8_0` (EmbeddingGemma 2, the ggml-org Q8_0 GGUF) | `text-embedding-embeddinggemma-300m-qat` (EmbeddingGemma-300M, QAT release) |
| weights SHA-256 | `2188ac1deca4b77dffefd603c2776a9d76d9d74ec01841392982ebb840b09135` | `5a9e0645541b06367dec3fd14d0019015b872cfe36d35dfb2a9d752dade09020` |
| embedded through | llama.cpp `a0e6dc73` (`llama-server`) | LM Studio |
| dimensions | 768 native, truncated to 512 (Matryoshka), then int8 | the same |
| chunking | 3200 characters, 480 overlap | the same |
| document prompt | `title: none \| text: ` (trailing space significant) | the same |
| pooling | character-length-weighted mean of unit-norm chunk vectors, L2-renormalised | the same |
| corpus | 553 volumes, 314,616 documents, 605,978 chunks | the same |
| harvested | 2026-10-14 to 15, in one run | 2026-08-10; `frus1981-88v16` on 2026-09-09; `frus1952-54v09p1` and `frus1902app1` again on 2026-10-10 |
| packed | 2026-10-15 | 2026-10-10 |
| provenance digest | `89481b1869da1880c599a6923df9a90bef19fc402c3db5a370ead2809b7e6101` | `a726ca606bdf4d1984ba7cfda4d5605c2e9dc1a8320654a1b5742e06aa6e3a64` |

Fifty documents of the 300m's generation, in five volumes, were embedded from text the corpus
corrected afterwards (punctuation and repaired characters); EmbeddingGemma 2's were embedded from
the current text.

## Quality, honestly

Recall@10 against each model's own exact float-768 neighbours, over 3,600 era-stratified queries.
The shipping configuration is binary Hamming candidate generation over the 512 sign bits at a pool
of 800, then an int8 rerank:

| configuration | EmbeddingGemma 2 | EmbeddingGemma 300m |
|---|---|---|
| **shipping** — Hamming-800 over 512 sign bits → int8-512 | **0.9021** | 0.8610 |
| flat int8-512, no binary funnel | 0.9054 | 0.8615 |
| Hamming-800 over 256 sign bits → int8-256 | 0.7845 | 0.7413 |

(The 300m's index states 0.851, measured in August with 256 sign bits ahead of the int8-512 rerank;
0.8610 is the funnel builds 42 to 50 run, measured 2026-10-15.)

Against the editors' own cross-references (83,119 citation queries, MRR@10 at the shipping shape),
the 300m scores 0.2574 and EmbeddingGemma 2 0.2503, a gap read as comparable quality; all of it is
in citations to a document of the same volume, and cross-volume citations are level. How either
ranks typed questions is measured by 25 judged queries, for the 300m in August; EmbeddingGemma 2's
lists share 3.6 documents a query with those and are not yet judged. The app repository's
`Planning/semantic-vectors/EmbeddingGemma-2-Stage-2-2026-10-15.md` has the figures.

The semantic feature these support ships **experimental**. Quality on pre-1900 volumes is an
*unmeasured unknown* rather than a measured pass — the automatic gate available (editorial
cross-references) reaches few pre-1900 queries, because the citation idiom it relies on postdates
1945. Agreement with the float-768 model is not the same as agreement with a historian.

## Model weights

This repository's **Releases** host the query encoders the FRUS Explorer app can optionally
download, one release a model. The app downloads the file its bundled artifacts pin and verifies it
against the pinned length and SHA-256 before keeping it.

- **`encoder-2`, from build 51:** Google's EmbeddingGemma 2, as the unmodified ggml-org Q8_0 GGUF
  build of `google/embeddinggemma-2` (`embeddinggemma-2-Q8_0.gguf`, 309,855,456 bytes, SHA-256
  `2188ac1deca4b77dffefd603c2776a9d76d9d74ec01841392982ebb840b09135`). The model is licensed under
  the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0); a copy is
  [`LICENSES/Apache-2.0.txt`](LICENSES/Apache-2.0.txt), and the release carries it too. Neither the
  model's repository nor the GGUF's carries a NOTICE file.
- **`encoder-1`, builds 42 to 50:** Google's EmbeddingGemma 300m, as the unmodified
  lmstudio-community Q4_0 QAT GGUF build (`embeddinggemma-300m-qat-Q4_0.gguf`, 229,093,184 bytes,
  SHA-256 `5a9e0645541b06367dec3fd14d0019015b872cfe36d35dfb2a9d752dade09020`). These weights are
  **not** covered by this repository's licence. Gemma is provided under and subject to the
  [Gemma Terms of Use](https://ai.google.dev/gemma/terms), including the
  [Gemma Prohibited Use Policy](https://ai.google.dev/gemma/prohibited_use_policy). By downloading
  or using that model file you agree to those terms, including their use restrictions.

The vector artifacts in this repository are model *outputs* and carry no model-licence conditions.

`encoder-1` also carries the debug symbols of the llama.cpp builds (MIT) the app embeds to run the
models, archived for crash symbolication: `llama-xcframework-dSYMs-8663224.zip` (through build 50)
and `llama-xcframework-dSYMs-a0e6dc7.zip` (from build 51).

## Licence

The source documents are U.S. Government works in the public domain. The vectors are model outputs:
the Gemma terms do not claim the 300m's, and EmbeddingGemma 2, under Apache 2.0, places no condition
on its own. Published for use by FRUS Explorer and anyone else who finds them useful.

Regenerate with `SemanticVectorsGenerator` in the app repository.
