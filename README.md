# rkvdec-vdpu383-h264-hevc — mainline VDPU383 HEVC/H.264 throughput (and H.264 deblock) fixes

Fixes for the **mainline Linux `rkvdec-vdpu383` HEVC and H.264 backends** on the
Rockchip **RK3576** (VDPU383) — the decoders that already decode *correctly* on
mainline, but leave a large amount of performance (and one H.264 correctness case)
on the table. Unlike the sibling AV1/VP9 repos, this is **not** new-codec work; it
is a small set of fixes to **Collabora's already-merged** HEVC/H.264 support.

> **Headline: the mainline HEVC/H.264 backends never configure the VDPU383 read
> caches, so they decode ~7× (HEVC) / ~2.4× (H.264) slower than the hardware can.**
> Configuring the caches — exactly as the vendor MPP driver and the AV1/VP9 backends
> do — recovers it, with **byte-identical output**.

## 1. Read-cache throughput fix (HEVC 7×, H.264 2.4×)

### Root cause

The VDPU383 has three reference **read caches** (`CACHE0/1/2`). The vendor MPP
driver configures them per decode (`CACHEABLE | READ_ALLOC | LINE_64`, a per-frame
clear, and a max-outstanding-reads value). The AV1 and VP9 backends do too. **The
mainline HEVC and H.264 backends issue zero cache config** — so every
reference/data read misses to DRAM, and decode runs several times slower than the
silicon is capable of. This is an upstream performance bug inherited by anything
built on mainline `rkvdec-vdpu383`.

It was found by timing MMIO→completion on the *same* clip through both the vendor
(BSP/MPP) and mainline stacks: same silicon, ~7× difference — and our DDR is
*faster* than the BSP's, so it was never bandwidth, it was a missing cache config.

### Measured (1080p, pure-HW decode time, RK3576 / 7.0.1-edge-rockchip64)

| codec | cache **off** (mainline today) | cache **on** (this fix) | speedup |
|-------|--------------------------------|-------------------------|---------|
| HEVC  | ~12 ms | **1.70 ms** | **~7×** (= vendor MPP) |
| H.264 | 8.7 ms | **3.6 ms** | **~2.4×** |

**Correctness unaffected** — the read cache is a pure throughput optimisation. HEVC
output is **byte-identical** cache-off vs cache-on (md5 `18f9bbc46cf0528a` both); it
does not change decoded pixels.

### The fix

Map the dedicated read-cache MMIO window (DT reg-name `"cache"`) at probe, and write
the same cache config the AV1/VP9/MPP paths use, immediately before the decode kick
(`DEC_E`), on both the HEVC and H.264 kick paths. Gated by a `hevc_cache` module
parameter (**default on** — it is a safe, large win; set `0` for A/B). Full hunks in
[`patches/0001-vdpu383-hevc-h264-read-cache.patch`](patches/0001-vdpu383-hevc-h264-read-cache.patch);
the essence:

```c
/* before the DEC_E kick, in the HEVC and H.264 backends */
if (hevc_cache && rkvdec->cache) {
    writel(0x1u | 0x2u | 0x10u, rkvdec->cache + 0x1c); /* CACHE0 cfg: CACHEABLE|READ_ALLOC|LINE_64 */
    writel(0x1u | 0x2u | 0x10u, rkvdec->cache + 0x5c); /* CACHE1 cfg */
    writel(0x1u | 0x2u | 0x10u, rkvdec->cache + 0x9c); /* CACHE2 cfg */
    writel(0x1u, rkvdec->cache + 0x10);                /* CLR_CACHE0 */
    writel(0x1u, rkvdec->cache + 0x50);                /* CLR_CACHE1 */
    writel(0x1u, rkvdec->cache + 0x90);                /* CLR_CACHE2 */
    writel(0x1cu, rkvdec->cache + 0x18);               /* MAX_READS */
    wmb();
}
```
(`rkvdec->cache` is `devm_platform_ioremap_resource_byname(pdev, "cache")`, mapped
optionally at probe; offsets are relative to that window.)

### Honest residual

After the cache fix there is a **consistent ~1.9× residual** vs the vendor stack
across *all* codecs (HEVC and VP9 alike). It is two further, identified factors —
not a mystery and not addressed here: (1) **FBC** (AFBC reference compression — the
vendor halves reference-read bandwidth; mainline reads references uncompressed), and
(2) **link/CCU pipelining** (~20%, HW armed across frames). The read-cache fix is the
dominant, cleanest, byte-identical-output piece — the others are storage-format and
scheduling changes for a later day.

## 2. H.264 deblock-at-resume correctness fix (RK3576 warmup)

A second, H.264-specific contribution (correctness, not throughput): on RK3576 the
VDPU383 needs a hardware **warmup sequence at runtime-resume** that the BSP performs
and the mainline path missed; without it, H.264 produces an intermittent deblocking
error. Replicating the warmup makes H.264 **64/64 bit-exact**, default-on. This was
reported to the linux-media list (`66768711@symple.nz`). It ships in the full driver
in the sibling repos; it is noted here because it is the other mainline-H.264 fix
from this effort. (The warmup is shared infrastructure — see the sibling drivers'
`rkvdec-link.c` / the RK3576 init path.)

## Applying

The hunks in `patches/` are written against the mainline `rkvdec-vdpu383` backends
(`rkvdec.c`, `rkvdec-vdpu383-hevc.c`, `rkvdec-vdpu383-h264.c`). They require the DT
node to expose the `"cache"` reg window (it is a distinct sub-region of the decoder
MMIO; the RK3576 decoder node already has the address — add the `reg`/`reg-names`
entry if absent). The full, buildable driver carrying these fixes is in the sibling
repos' `src/` (they ship the complete VDPU38x driver).

## Status & intent

Published **downstream-first**: a measured, byte-identical-output performance fix to
mainline `rkvdec-vdpu383`, offered as a clean candidate for the mainline maintainers
(Detlev Casanova / Collabora). It takes no feature requests and is not a fork — it is
the throughput (and H.264 deblock) fixes from the broader RK3576 VDPU383 effort,
surfaced where they are useful to anyone running mainline rkvdec on this SoC.

## Licence & credits

GPL-2.0 (see [LICENSE](LICENSE)) — these are fixes to the GPL-2.0 mainline `rkvdec`
driver. Built on **Detlev Casanova / Collabora's** mainline VDPU383 HEVC/H.264
support; the vendor `rockchip-linux/mpp` driver was the ground-truth reference that
showed the cache config the mainline path was missing.

## Related

- [`rkvdec-vdpu383-av1`](https://github.com/SympleNZ/rkvdec-vdpu383-av1) / [`rkvdec-vdpu383-vp9`](https://github.com/SympleNZ/rkvdec-vdpu383-vp9) — the mainline V4L2 AV1 / VP9 drivers (which ship these fixes in their full `src/`).
- [`rkvdec-vdpu383-mpp-mainline`](https://github.com/SympleNZ/rkvdec-vdpu383-mpp-mainline) — the vendor MPP decoder on a mainline kernel (the same-silicon reference that exposed the cache gap).

