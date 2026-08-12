# frus-semantic-vectors

Per-volume semantic vectors for [FRUS Explorer](https://github.com/joshbotts/FRUS-Explorer) — the
Tier-2 "exact scoring" layer, fetched by the app alongside each volume it downloads.

**This repository holds derived data only.** No document text, no user data. Each `.vec` file is a
block of quantized embedding vectors, one per document, for a single *Foreign Relations of the
United States* volume.

## What is here

```
shards/<volumeId>.vec    552 files, ~82 MB total, ~148 KB each
shards-manifest.json     per-shard SHA-256 + byte length + the provenance pin
```

`shards-manifest.json` is authoritative for integrity: the app verifies a downloaded shard against
its digest, and every shard's own 64-byte header carries the same provenance digest the app has
bundled. A shard whose pin does not match the app's bundled tier is refused rather than blended —
vectors from two different generations must never be scored against each other.

## Shard format

All little-endian.

| offset | size | field |
|---|---|---|
| 0 | 4 | magic `FRSS` |
| 4 | 4 | layout version (`1`) |
| 8 | 4 | dimensions (`256`) |
| 12 | 4 | document count |
| 16 | 32 | provenance digest (SHA-256) |
| 48 | 16 | zero padding |
| 64 | `count × 256` | int8 codes, one 256-byte row per document |
| … | `count × 4` | `Float32` per-vector quantization scale |

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
| dimensions | 768 native, truncated to 256 (Matryoshka), then int8 |
| chunking | 3200 characters, 480 overlap |
| document prompt | `title: none | text: ` (trailing space significant) |
| pooling | character-length-weighted mean of unit-norm chunk vectors, L2-renormalised |
| corpus | 552 volumes, 314,483 documents, 605,643 chunks |
| harvested | 2026-08-10 |
| provenance digest | `57fb4c4b3aea961595df87dcbd28c0af7ec8a671bc4c11f886827a20ca4bfb14` |

## Quality, honestly

Measured against exact float-768 neighbours over 3,600 era-stratified queries: the shipping
configuration (256-bit Hamming candidate generation → int8-256 rerank at a pool of 800) recalls
**0.745** of the exact top ten. Flat int8-256 without the binary funnel is 0.749; int8-512 would be
0.864 at double the size.

The semantic feature these support ships **experimental**. Quality on pre-1900 volumes is an
*unmeasured unknown* rather than a measured pass — the automatic gate available (editorial
cross-references) reaches only 572 pre-1900 queries corpus-wide, because the citation idiom it
relies on postdates 1945.

## Licence

The source documents are U.S. Government works in the public domain. The vectors are model outputs;
under the Gemma terms, output is not claimed by the model licence. Published for use by FRUS Explorer
and anyone else who finds them useful.

Regenerate with `SemanticVectorsGenerator` in the app repository.
