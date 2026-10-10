# This fork: Strata on 2x CMP 50HX (sm_75) + 2x Xeon E5-2670 v2 (AVX1)

Upstream is [Niko1221/Strata](https://github.com/Niko1221/Strata). This branch starts from upstream `main`
(0.1.41, `fb58e0d`) and adds a short list of patches for this box. The numbers for each patch are in its commit
message; the older notes in [AVX1_SM75_PLAN.md](AVX1_SM75_PLAN.md) can be out of date.

## Syncing with upstream

```sh
git fetch upstream
git merge upstream/main          # the history is shared, so a plain merge works
```

Rules that keep the next merge small:

- A patch here is small and on its own: new files where possible, a few lines in upstream files.
- A patch that upstream takes (or replaces with its own) is dropped here.
- Each generic patch is also kept as a branch on top of upstream `main` (`pr/*`), ready to send upstream.

## Patches on top of upstream

| commit | what | switch |
|---|---|---|
| `8aa74eb` | Q2_0 expert rows on AVX-only CPUs (`q2_avx1.cpp`) instead of ggml-cpu's scalar dot | always, on an AVX-only CPU |
| `7b07ac7` | tokenizer: a long message's ids remembered whole, on top of upstream's piece cache | always |
| `11a4f15`, `18ec242` | QSA prompt-path block scores: a thread per query head, four lanes per query (Turing) | `STRATA_SCORES_THREAD=0` = old |
| `1eb1010` | prompt attention, int8 K/V: next chunk in registers, K and V in one buffer (Turing) | none (same bits) |
| `94292d6` | gate/up MMQ reads the experts in place (a patched copy of llama.cpp's `mmq.cuh` in the build tree) | `STRATA_MMQ_INPLACE=0` = old |
| `e8f68e0`, `cc3fd60`, `3a5e69e`, `8f93be3` | `dual3090/verify/`: agentbench (gate and compare), flashbench corpus, prefill_probe; `prefill_mmq_bench` | tools |

## Dropped in the sync (from the `avx1-sm75-opt` branch)

| commit | why |
|---|---|
| `87fa125` no-AVX2 support | upstream's older-CPU build (`STRATA_ISA_FLOOR=avx`) does the same |
| `22f2f64` `STRATA_HC_FP16` | upstream's `STRATA_BF16_TC=1` runs these and more of the prompt path's BF16 products on FP16 tensor cores |
| `8de94b5` residency table race | upstream fixed it (`res_put` waits for each copy) |
| `b7bf59d`, `c2e0e79`, `240744b` decode kernels | upstream reworked the same kernels (and stages the IQ2 codebooks itself); about +5 % decode together, not worth the merge conflicts. Bring one back only if the A/B below shows upstream slower |
| `b44bb09` MMQ IQ3_XXS codebook | -0.3 % prompt read in the agent session: too small for a patch to llama.cpp's MMQ |
| dual-3090 peer mode, Docker files | not used here: `--layer-split auto` is much faster on this PCIe Gen2 box |

xrip/llama.cpp-avx1-numa-sm75 was checked too: its SM75 MMQ tuning is for dense Q4_K/IQ4_XS only, its MMVQ and
codebook work is in files Strata does not use (Strata has its own decode kernels), and its AVX1 panel kernels are
repack paths Strata does not call. Strata keeps the pinned llama.cpp (`3cf03257`).

## A/B on the box (to do when it is free)

Build as `build-gcc14` (GCC 14, sm_75, `STRATA_ENABLE_CUDA=ON`) plus `-DSTRATA_ISA_FLOOR=avx`, in a separate folder.
Base: the current `build/strata` (`avx1-sm75-opt`, `b44bb09`). Config: the gate config (`--adapt-swaps 0`,
`--pcie-frac 0.17`), agentbench 34K -> 190K, fresh server for each arm, inside `perf.sh`.

The text will not match the base word for word (upstream changed kernels the base had), so compare speed (time to
first token, incremental read, decode) and check the answers by eye. Arms after the plain sync:

- `STRATA_BF16_TC=1` (instead of the dropped `STRATA_HC_FP16`; other bits)
- `--pipeline-windows` (layer split)
- `STRATA_FS_SLOTS=<n>` (Foresight swap space: per-layer VRAM slots)
- `STRATA_GDN_CHUNKED=2` (other bits)
