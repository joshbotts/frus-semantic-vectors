# frus-semantic-vectors

Per-volume semantic vectors for [FRUS Explorer](https://github.com/joshbotts/FRUS-Explorer) — the
Tier-2 "exact scoring" layer, fetched by the app alongside each volume it downloads.

**This repository holds derived data only.** No document text, no user data. Each `.vec` file is a
block of quantized embedding vectors, one per document, for a single *Foreign Relations of the
United States* volume.

## What is here

```
shards/<volumeId>.vec    553 files, 155 MB total, 287 KB each on average
shards-manifest.json     per-shard SHA-256 + byte length + the provenance pin
```

Shard sizes track volume length, not an average: the smallest is 1 KB and the largest 965 KB, with a
median of 242 KB.

`shards-manifest.json` lists every file here with its byte length and SHA-256. It is a copy of the
manifest the app bundles (`FRUSExplorer/Resources/semantic-shards-manifest.json` in the app
repository), and the bundled copy is the one that decides: a build verifies each shard it downloads
against the length and digest it shipped with, never against this file. Every shard's own 64-byte
header carries the same provenance digest the app has bundled. A shard whose pin does not match the
app's bundled tier is refused rather than blended — vectors from two different generations must
never be scored against each other.

## Generations

**These shards are the 512-dimension generation, published 2026-08-16.** They replaced a
256-dimension set built from the same harvest.

The two are not interchangeable, and the mechanism that stops them mixing is the provenance digest:
`shippingDims` is one of its inputs, so widening the truncation moves the digest, and an app bundled
against one generation refuses every shard of the other. That refusal is the design working. In
practice it means **the shards and the app's bundled tier must be updated together** — an app on the
256 pin fetching these files rejects each one on its byte length before the digest is even checked.

| generation | dims | total | per shard | provenance digest |
|---|---|---|---|---|
| current | 512 | 155 MB | 287 KB mean | `a726ca60…6aa6e3a64` |
| previous | 256 | 78 MB | 145 KB mean | `57fb4c4b…0ca4bfb14` |

Both are reproducible from the same raw harvest with `SemanticVectorsGenerator` at the matching
`DIMS` — verified byte-for-byte across independent runs, over all three bundled artifacts and all
552 shards of the first publication. Nothing here needs to be archived to be recoverable.

## Changes since publication

The generation has not changed since 2026-08-16: the same model file, prompt, chunking and width,
and so the same provenance digest. Three files have changed, each because the Office of the
Historian changed the volume it describes.

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

**A replacement is made in place, and an older build of the app will not take it.** The address
carries no version, and a build verifies a download against the manifest it shipped with. A FRUS
Explorer build whose bundled manifest predates a replacement therefore refuses a fresh download of
that file and scores that volume from its bundled tier alone; a copy it already holds stays. For the
two files of 2026-10-10 that is build 49 and earlier. The files those builds expect are in this
repository's history at commit `dd8295f`. An added file has no such cost, since no earlier build
names it.

## Shard format

All little-endian.

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
to parse either generation — only the digest decides whether it may *use* what it parsed.

Vectors are L2-normalised before quantization, so `dot(a, b) × scaleA × scaleB` reconstructs cosine
similarity.

**Rows are positional, and identity does not live here.** Row *k* is the *k*th document of that
volume *in the order of the app's bundled index*, not in the order of whatever XML is on disk. The
mapping from row to document id lives in the app's bundled `semantic-vectors-index.json`. A consumer
that infers identity from a volume's XML will be wrong for any volume republished upstream since
these vectors were built.

## Provenance

| | |
|---|---|
| model | `text-embedding-embeddinggemma-300m-qat` (EmbeddingGemma-300M, QAT release) |
| weights SHA-256 | `5a9e0645541b06367dec3fd14d0019015b872cfe36d35dfb2a9d752dade09020` |
| dimensions | 768 native, truncated to 512 (Matryoshka), then int8 |
| chunking | 3200 characters, 480 overlap |
| document prompt | `title: none \| text: ` (trailing space significant) |
| pooling | character-length-weighted mean of unit-norm chunk vectors, L2-renormalised |
| corpus | 553 volumes, 314,616 documents, 605,978 chunks |
| harvested | 2026-08-10; `frus1981-88v16` on 2026-09-09; `frus1952-54v09p1` and `frus1902app1` again on 2026-10-10 |
| packed | 2026-10-10 |
| provenance digest | `a726ca606bdf4d1984ba7cfda4d5605c2e9dc1a8320654a1b5742e06aa6e3a64` |

The harvest was unchanged from the 256 generation when the shards were widened — only the
Matryoshka truncation and the quantization width differ, which is why re-embedding was not needed.
The three volumes embedded since (above) were embedded under the same model file, prompt and
chunking.

## Quality, honestly

Measured against exact float-768 neighbours over 3,600 era-stratified queries. The shipping
configuration is binary Hamming candidate generation at a pool of 800, then an int8 rerank:

| configuration | recall@10 |
|---|---|
| **512 (current)** — Hamming-800 → int8-512 | **0.851** |
| 256 (previous) — Hamming-800 → int8-256 | 0.745 |
| flat int8-512, no binary funnel | 0.864 |
| raw binary alone, no rerank | 0.53 |

So the funnel costs 0.013 against a flat int8-512 scan, and the width buys 0.106 over the generation
it replaced — for twice the bytes and about 1.6× the query time. That time figure is an **M1 Max**
measurement (≈1.0 ms → ≈1.6 ms for a full-corpus scan of 314,483 documents); the ratio should
transfer to a phone but the absolute numbers will not, and no on-device measurement has been taken.

The semantic feature these support ships **experimental**. Quality on pre-1900 volumes is an
*unmeasured unknown* rather than a measured pass — the automatic gate available (editorial
cross-references) reaches only 572 pre-1900 queries corpus-wide, because the citation idiom it
relies on postdates 1945. Widening to 512 does not change that: it improves agreement with the
float-768 model, which is not the same as improving agreement with a historian.

Every figure in this section was measured on the first publication (552 volumes, 314,483 documents)
and has not been measured again. The three files changed since hold 1,230 of the 314,616 documents.

## Model weights

Beside the per-volume vector shards, this repository's **Releases** host the query encoder the
FRUS Explorer app can optionally download: Google's EmbeddingGemma model, as the unmodified
lmstudio-community Q4_0 QAT GGUF build (`embeddinggemma-300m-qat-Q4_0.gguf`, 229,093,184 bytes,
SHA-256 `5a9e0645541b06367dec3fd14d0019015b872cfe36d35dfb2a9d752dade09020`).

The model weights are **not** covered by this repository's licence. Gemma is provided under and
subject to the [Gemma Terms of Use](https://ai.google.dev/gemma/terms), including the
[Gemma Prohibited Use Policy](https://ai.google.dev/gemma/prohibited_use_policy). By downloading
or using the model file you agree to those terms, including their use restrictions. The vector
artifacts in this repository are model *outputs* and carry no model-licence conditions.

The release also carries `llama-xcframework-dSYMs-8663224.zip` — debug symbols for the llama.cpp
build (MIT) the app embeds to run the model, archived for crash symbolication.

## Licence

The source documents are U.S. Government works in the public domain. The vectors are model outputs;
under the Gemma terms, output is not claimed by the model licence. Published for use by FRUS Explorer
and anyone else who finds them useful.

Regenerate with `SemanticVectorsGenerator` in the app repository.
