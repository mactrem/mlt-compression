# MLT v2: Compression Strategy

A proposal for discussion. 


## Summary

- Tiles are typically delivered compressed, since for bandwidth-bound map delivery, network transfer is usually a larger
  bottleneck than general-purpose decompression, especially on mobile networks. MLT v2 should therefore optimize tile
  size *after* gzip, brotli or zstd.
- The model for MLT v2 should be inspired by OpenZL (lightweight, format-aware transforms followed by a
  general-purpose backend), not analytics formats such as BtrBlocks, FastLanes or Vortex, which are designed for a
  different performance regime.
- Concretely: use a small set of transforms that make data more compressible; apply whole-tile gzip, brotli or zstd
  as the only general-purpose compression stage (terminal/leaf encoding); and omit encodings that provide no gain, or a negative gain, after compression (FastPFOR, bit-packing, FSST, rANS, most forms of RLE, ...).

## Motivation

- **The network is usually the bottleneck, not decompression.** Across the cited German and US operator results, typical mobile download speed is roughly 100 Mbit/s (12 MB/s) ([Opensignal Germany](https://insights.opensignal.com/reports/2025/11/germany/mobile-network-experience), [Opensignal USA](https://insights.opensignal.com/reports/2026/07/usa/mobile-network-experience)). A single CPU core decompresses gzip or brotli at roughly 400 MB/s and zstd at over 1,000 MB/s ([zstd benchmark](https://github.com/facebook/zstd#benchmarks), [lzbench](https://github.com/inikep/lzbench)). In browsers, this runs natively while tile encodings run in JavaScript. Analytics formats such as [BtrBlocks](https://www.cs.cit.tum.de/fileadmin/w00cfj/dis/papers/btrblocks.pdf) target the opposite case: fast cloud networks (over 100 Gbit/s networks), where decompression is the bottleneck (CPU-bound).
- **Tiles are delivered compressed.** [Planetiler](https://github.com/onthegomap/planetiler/blob/main/planetiler-core/src/main/java/com/onthegomap/planetiler/config/PlanetilerConfig.java) and [tippecanoe](https://github.com/felt/tippecanoe#readme) gzip tiles by default, because users want the smallest tiles.
- **gzip is the baseline.** Every browser decodes gzip and brotli as Content-Encoding, and zstd since Chrome and
  Edge 123, Firefox 126 and Safari 26.3 ([caniuse](https://caniuse.com/zstd)). Clients that decompress tiles
  themselves, such as PMTiles readers, use `DecompressionStream`, which supports only gzip and deflate in every browser
  ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/DecompressionStream/DecompressionStream#browser_compatibility)).



## Goals and non-goals

**Goals**

1. Minimize tile size after gzip, brotli and zstd over the raw encoded tile size.
2. Web first: cheap to decode in JavaScript, with the transport compressor as the only entropy coder.
3. Keep the set of encodings small, as MLT v1 did: an encoding is added only if it shows a measurable gain after
   compression. This
   - keeps the implementations simple and maintainable;
   - keeps the decoder small, which matters because every web client downloads it;
   - keeps conversion fast. On-the-fly MVT-to-MLT conversion is not the primary use case, but several adopters need
     it; for example Jia Yu (co-founder and Chief Architect of Wherobots) named it an adoption criterion for MLT over MVT;
   - leaves room for what comes next (SIMD- and GPU-friendly encodings, encodings for 3D, possibly routing) instead
     of spending the budget on encodings that only serve 2D rendering on a CPU.
4. Fast encoding, so that choosing encodings by trial against the compressor stays affordable.

**Non-goals**

- The smallest raw size. That is the standalone profile.
- Entropy coding inside the tile (Huffman, FSE, rANS).
- Deep, OpenZL-style transform graphs per tile.
- General-purpose compression per stream, as in Nimble.
- Maximum native decode speed through SIMD or GPU layouts. Those can become separate profiles later.


## Encoding strategy

### Profiles

| Profile | Optimized for | Intended for | Encodings |
|---|---|---|---|
| Standalone | Raw size | No transport compression: fast networks, offline storage, CPU-bound clients | MLT v1's set plus front coding |
| Web (default) | Size after gzip, brotli or zstd | Web and mobile delivery | Section 3.2 |

### Web profile

**Terminal codec.**  Always use general purpose compression as leaf/terminal codec. Every tile ends in gzip, brotli or zstd, applied once to the whole tile, with no per-stream entropy or LZ stage compared to OpenZL. The encoder chooses encodings by trial against the deployed compressor.

**Lightweight opaque transforms.** Like OpenZL's transforms, they need not shrink the data on their own; they make it more compressible for the backend.
- Delta
- Dictionary, shared dictionary, Morton-ordered vertex dictionary
- Zigzag + Varint: compared to bit aligned encodings like FastPfor byte aligned encodings like Varint showed real gains in my experience when combined with compression
- ALP: decimal floats to integers with exceptions, byte-aligned output
- Front coding 
- To evaluate, from OpenZL: Divide-by (GCD), float deconstruct, partition, sentinel, sparse, ...

**Lightweight transparent encodings.** Constant and sequence encodings provide O(1) access without materializing a
decoded buffer, reducing decode overhead and improving cache locality, for example during filtering. In v1, they
were special cases of RLE exposed as `ConstantVector` and `SequenceVector` in the in-memory format.
- **Constant:** one value and a count.
- **Sequence:** a start value, step and count.

**Spatial predictors for geometry** (work in progress by Benedict). Use previously decoded neighbouring vertices as
context to predict each vertex, and store only the residual. Inspired by PNG filters (5 predictors), WebP
(14 predictors) and Draco, use a small set of cheap predictors selected by trial compression per layer or, where
worthwhile, at finer feature granularity. Learned predictors such as LSTMs are a longer-term alternative to hand-designed predictors; our initial trials reduce on that topics size but decode too slowly for practical use.

**Correlation-aware encodings.** Exploit correlations between columns.
- Use shared dictionaries between related columns such as `name` and `name:*`, as already supported by MLT.
- **To evaluate:** [C3](https://github.com/cwida/C3)-style equality, one-to-one and one-to-many dictionaries,
  integer relations and Dictionary-FOR, plus [FastLanes external dictionaries](https://www.vldb.org/pvldb/vol18/p4629-afroozeh.pdf).

**Data restructuring and conversion.** Inspired by OpenZL's restructuring and conversion phase; for example, split
geometry into jump, free and residual streams, and evaluate semantic casts.

**Shared data outside the tile.** Move stable schema, metadata and common values into per-tileset dictionaries.

**Index buffers.** For pre-tessellated polygons and 3D, compare Edgebreaker-style connectivity coding with a
byte-oriented index codec designed for a following gzip or zstd stage, as used by meshoptimizer.


The following encodings should remain outside the web profile unless they demonstrate a net gain after compression: FSST, FastPFOR, rANS, most forms of RLE, bit-packing and more than one (or max two) boolean encodings. RLE may still help on exceptional streams with very long runs, but it should be retained only when measurement shows a gain. Delta-of-delta should not be an isolated encoding; where useful, it should be incorporated into the geometry predictor profiles.



