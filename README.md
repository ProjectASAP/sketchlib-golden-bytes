# sketchlib-golden-bytes

Golden byte-vectors for the ASAPv1 sketch wire format.

Each `.hex` file pins the exact bytes ASAPv1 emits for one fixed, known sketch
state. Every implementation of ASAPv1 must encode that state to these bytes and
decode these bytes back to that state. The spec is
[`docs/asapv1_wire_format.md`](https://github.com/ProjectASAP/asap_sketchlib/blob/main/docs/asapv1_wire_format.md)
in `asap_sketchlib`.

Each file is one line of lowercase hex (no `0x`, no whitespace) = the complete
ASAPv1 envelope `[ magic | version | kind_id | metadata_len | payload_len |
metadata | payload ]`.

## Using this repo

Consumers mount it as a git submodule at `asapv1_golden/`:

```bash
git submodule add https://github.com/ProjectASAP/sketchlib-golden-bytes.git asapv1_golden
```

| Consumer | Test |
| -------- | ---- |
| `asap_sketchlib` | `tests/asapv1_golden.rs` — serializes each known state and asserts `== golden`; decodes each golden and asserts the state and a byte-identical re-encode |

## Design principle: state is fixed, not hashed

Every fixture is built from a **known raw sketch state** (specific register
bytes / matrix values set directly), never by hashing input values. So the
golden tests the **wire encoding**, isolated from the hash functions, except
that decoding Coco checks each key's placement against its hash.

## Fixtures

| File | Sketch | kind_id | State |
| ---- | ------ | ------- | ----- |
| `hll_classic_p12` | HLL Classic, P12 | `01 01` | 4096 registers, set: `[0]=1, [1]=7, [100]=42, [4095]=3` |
| `hll_ertl_mle_p12` | HLL Ertl-MLE, P12 | `01 02` | same register pattern |
| `hll_hip_p12` | HLL HIP, P12 | `01 03` | same registers + `hip_kxq0=1.5, hip_kxq1=2.5, hip_est=3.0` |
| `hll_classic_p12_admission` | HLL Classic, P12, sampled | `01 01` | the `hll_classic_p12` registers; `admission_rate=0.5` |
| `hll_classic_p14` | HLL Classic, P14 | `01 01` | 16384 registers, set: `[0]=1, [1]=7, [8192]=42, [16383]=51` |
| `hll_ertl_mle_p14` | HLL Ertl-MLE, P14 | `01 02` | same register pattern |
| `hll_hip_p14` | HLL HIP, P14 | `01 03` | same registers + `hip_kxq0=16380.5, hip_kxq1=0.25, hip_est=4.125` |
| `hll_classic_p14_delta` | HLL Classic delta, P14 | `01 81` | indices `[0,2,200,8192,16383]`, registers `[51,1,9,7,1]`; merged into `hll_classic_p14` gives `[0]=51, [1]=7, [2]=1, [200]=9, [8192]=42, [16383]=51` |
| `hll_ertl_mle_p14_delta` | HLL Ertl-MLE delta, P14 | `01 82` | same delta; merged into `hll_ertl_mle_p14` gives the same registers |
| `cms_i64_regular_2x3` | Count-Min i64, RegularPath | `02 00` | 2×3 row-major `[[0,1,127],[128,300,65536]]` |
| `cms_i64_regular_2x3_admission` | Count-Min i64, RegularPath, sampled | `02 00` | the `cms_i64_regular_2x3` matrix; `admission_rate=0.1` |
| `cms_f64_fast_2x3` | Count-Min f64, FastPath | `02 00` | 2×3 row-major `[[0.0,1.5,2.25],[3.75,4.125,5.0625]]` |
| `cms_i64_regular_2x3_delta` | Count-Min i64 delta, RegularPath, 2×3 | `02 80` | indices `[0,1,2,4,5]`, deltas `[127,-1,-33,-32769,4294967296]`; merged into `cms_i64_regular_2x3` gives `[[127,0,94],[128,-32469,4295032832]]` |
| `cms_f64_fast_2x3_delta` | Count-Min f64 delta, FastPath, 2×3 | `02 80` | indices `[0,3,5]`, deltas `[2.5,-0.75,-5.0625]`; merged into `cms_f64_fast_2x3` gives `[[2.5,1.5,2.25],[3.0,4.125,0.0]]` |
| `cs_i64_regular_2x4` | Count Sketch i64, RegularPath | `04 00` | 2×4 row-major `[[0,127,128,65536],[-1,-33,-32768,-2147483648]]` |
| `cs_i64_regular_2x4_admission` | Count Sketch i64, RegularPath, sampled | `04 00` | the `cs_i64_regular_2x4` matrix; `admission_rate=0.25` |
| `cs_i64_fast_2x4` | Count Sketch i64, FastPath | `04 00` | same matrix — differs from the above only by `mode` |
| `cs_i32_regular_2x4` | Count Sketch i32, RegularPath | `04 00` | same matrix — differs from the first only by `counter_type` |
| `cs_f64_fast_2x4` | Count Sketch f64, FastPath | `04 00` | 2×4 row-major `[[0.0,-0.0,1.5,-2.25],[127.0,-32768.5,0.125,-1e300]]` |
| `cs_i64_regular_2x4_delta` | Count Sketch i64 delta, RegularPath, 2×4 | `04 80` | indices `[0,1,2,4,6,7]`, deltas `[-32,-128,255,-129,65535,-2147483649]`; merged into `cs_i64_regular_2x4` gives `[[-32,-1,383,65536],[-130,-33,32767,-4294967297]]` |
| `cs_f64_fast_2x4_delta` | Count Sketch f64 delta, FastPath, 2×4 | `04 80` | indices `[1,3,5,7]`, deltas `[-0.75,2.25,0.5,1e300]`; merged into `cs_f64_fast_2x4` gives `[[0.0,-0.75,1.5,0.0],[127.0,-32768.0,0.125,0.0]]` |
| `kll_f64_k200` | KLL f64, k=200 | `06 00` | integers `1..=50`, compaction seed 42 (recorded in metadata as `compaction_seed`) |
| `kll_i64_k200` | KLL i64, k=200 | `06 00` | integers `1..=50`, compaction seed 42 (recorded in metadata as `compaction_seed`) |
| `kll_dynamic_f64_k200` | KLLDynamic f64, k=200 | `06 01` | `[2.5, -1.0, 0.0, 1e300, -0.125, 42.0, 3.0e-5]` in that order, compaction seed 42 (not in metadata) |
| `kll_dynamic_i64_k200` | KLLDynamic i64, k=200 | `06 01` | `[0, 1, -1, 127, -32, 128, -33, 255, -128, 256, -129, 65535, -32768, 65536, -32769, 4294967295, -2147483648, 4294967296, -2147483649, i64::MAX, i64::MIN]` in that order, compaction seed 42 (not in metadata) |
| `ddsketch_positive_a001` | DDSketch, α=0.01, positive only | `05 00` | `metadata_version` 1; positive store `[1,0,127,128,300,65536,4294967296]` at offset `-40`; empty negative store at offset `0`; `zero_count=0`; `sum=2181071000.0, min=0.453125, max=0.5078125` |
| `ddsketch_signed_a001` | DDSketch, α=0.01, signed | `05 00` | `metadata_version` 1; positive store `[3,0,2]` at offset `310`; negative store `[5,1]` at offset `-208`; `zero_count=7`; `sum=2523.90625, min=-0.016, max=515.0` |
| `ddsketch_signed_a001_admission` | DDSketch, α=0.01, signed, sampled | `05 00` | the `ddsketch_signed_a001` state; `admission_rate=0.125` |
| `ddsketch_empty_a001` | DDSketch, α=0.01, empty | `05 00` | `metadata_version` 1; empty stores at offset `0`; `zero_count=0`; `sum=0.0, min=+inf, max=-inf` |
| `ddsketch_signed_a001_delta` | DDSketch delta, α=0.01 | `05 80` | indices `[-300,-100,-5,7,200,400]`, deltas `[1,127,128,256,65536,4294967296]`; negative indices `[-208,-33,-1,0,130,1000]`, negative deltas `[2,3,3,1,3,1]`; `zero_count=9`; merged into `ddsketch_signed_a001` gives positive buckets `{-300:1, -100:127, -5:128, 7:256, 200:65536, 310:3, 312:2, 400:4294967296}`, negative buckets `{-208:7, -207:1, -33:3, -1:3, 0:1, 130:3, 1000:1}`, `zero_count=16`, `min=-r(1000)`, `max=r(400)`, where `r(k) = 1.01·γ^k` |
| `ddsketch_buckets_positive_a001` | DDSketch (buckets only), α=0.01, positive only | `05 01` | `metadata_version` 1; positive store `[1,0,127,128,300,65536,4294967296]` at offset `-40`; empty negative store at offset `0`; `zero_count=0` |
| `ddsketch_buckets_signed_a001` | DDSketch (buckets only), α=0.01, signed | `05 01` | `metadata_version` 1; positive store `[3,0,2]` at offset `310`; negative store `[5,1]` at offset `-208`; `zero_count=7` |
| `ddsketch_buckets_signed_a001_admission` | DDSketch (buckets only), α=0.01, signed, sampled | `05 01` | the `ddsketch_buckets_signed_a001` state; `admission_rate=0.125` |
| `ddsketch_buckets_empty_a001` | DDSketch (buckets only), α=0.01, empty | `05 01` | `metadata_version` 1; empty stores at offset `0`; `zero_count=0` |
| `ddsketch_buckets_signed_a001_delta` | DDSketch delta, α=0.01 | `05 81` | the `ddsketch_signed_a001_delta` payload under the current kind; merged into `ddsketch_buckets_signed_a001` gives the same buckets and `zero_count=16` as that fixture |
| `cmsheap_i64_regular_2x3_strkeys` | CMSHeap i64, RegularPath, `string` keys | `03 00` | 2×3 row-major `[[0,1,127],[128,300,65536]]`; `heap_size=5`; heap `{"hot":65536, "warm":300, "mild":300, "cold":1}` |
| `cmsheap_i32_fast_2x3_i64keys` | CMSHeap i32, FastPath, `i64` keys | `03 00` | same matrix; `heap_size=3`; heap `{-1:7, -129:7, 4294967296:3}` |
| `cmsheap_i64_regular_2x3_i64tie` | CMSHeap i64, RegularPath, `i64` keys | `03 00` | same matrix; `heap_size=5`; heap `{2:9, -1:5, 1:5, 0:5, i64::MIN:5}` |
| `cmsheap_i64_regular_2x3_strtie` | CMSHeap i64, RegularPath, `string` keys | `03 00` | same matrix; `heap_size=5`; heap `{"hot":9, "b":5, "aa":5, "Z":5, "a":5}` |
| `cmsheap_i64_regular_2x3_tuplekeys` | CMSHeap i64, RegularPath, `bytes` tuple keys | `03 00` | same matrix; `heap_size=5`; heap `{["us-east",443,true]:300, ["us-east",80,null]:300, [null,-1,2.5]:65536, ["",i64::MIN,-0.0]:1, ["é",0,NaN,false]:127}` |
| `cmsheap_i64_regular_2x3_empty` | CMSHeap i64, RegularPath, empty heap | `03 00` | same matrix; `heap_size=4`; no entries |
| `cmsheap_f64_fast_2x3_f64keys` | CMSHeap f64, FastPath, `f64` keys | `03 00` | the Count-Min f64 matrix `[[0.0,1.5,2.25],[3.75,4.125,5.0625]]`; `heap_size=6`; heap `{2.0:+inf, 1.5:300.25, -0.5:300.25, 8.0:0.0, 7.0:-0.0, 9.0:-inf}` |
| `cmsheap_i64_regular_2x3_delta` | CMSHeap i64 delta, RegularPath, 2×3, `i64` keys | `03 80` | indices `[0,1,2,4,5]`, deltas `[127,-1,-33,-32769,4294967296]`; `heap_size=5`; heap `{-1:4295032832, 7:128, -129:128}`; merged into `cmsheap_i64_regular_2x3_strkeys` gives `[[127,0,94],[128,-32469,4295032832]]` and the delta's heap |
| `csheap_i64_regular_2x4_strkeys` | CSHeap i64, RegularPath, `string` keys | `0a 00` | the Count Sketch 2×4 matrix `[[0,127,128,65536],[-1,-33,-32768,-2147483648]]`; `heap_size=5`; heap `{"alpha":4294967296, "beta":127, "delta":127, "gamma":-33}` |
| `csheap_i64_regular_2x4_i64tie` | CSHeap i64, RegularPath, `i64` keys | `0a 00` | the Count Sketch 2×4 matrix; `heap_size=6`; heap `{2:9, -1:-3, 1:-3, 0:-3, i64::MIN:-3, -129:-40}` |
| `csheap_i64_regular_2x4_tuplekeys` | CSHeap i64, RegularPath, `bytes` tuple keys | `0a 00` | the Count Sketch 2×4 matrix; `heap_size=6`; heap `{["us-east",443,true]:127, [null,-1,2.5]:-33, ["",i64::MIN,-0.0]:127, ["é",0,NaN,false]:65536}` |
| `csheap_f64_fast_2x4_strkeys` | CSHeap f64, FastPath, `string` keys | `0a 00` | the Count Sketch f64 matrix `[[0.0,-0.0,1.5,-2.25],[127.0,-32768.5,0.125,-1e300]]`; `heap_size=4`; heap `{"alpha":63.5625, "aa":-0.75, "beta":-0.75, "gamma":-16384.25}` |
| `csheap_i64_regular_2x4_delta` | CSHeap i64 delta, RegularPath, 2×4, `i64` keys | `0a 80` | indices `[0,1,2,4,6,7]`, deltas `[-32,-128,255,-129,65535,-2147483649]`; `heap_size=5`; heap `{-1:65536, 7:-33, -129:-4294967297}`; merged into `csheap_i64_regular_2x4_strkeys` gives `[[-32,-1,383,65536],[-130,-33,32767,-4294967297]]` and the delta's heap |
| `cmsspacesaving_i64_regular_2x3_strkeys` | CMSSpaceSaving i64, RegularPath, `string` keys | `1e 00` | the Count-Min i64 matrix `[[0,1,127],[128,300,65536]]`; `capacity=4`; summary `(key, count, error)` `{("hot",300,0), ("warm",128,0), ("aa",7,1), ("b",7,0)}`, `total=442`, `discarded_max=1` |
| `cmsspacesaving_nototal_i64_regular_2x3_strkeys` | CMSSpaceSaving i64, RegularPath, `string` keys, no `total` | `1e 01` | the `cmsspacesaving_i64_regular_2x3_strkeys` matrix and summary without `total`; `discarded_max=1` |
| `csspacesaving_f64_fast_2x4_i64keys` | CSSpaceSaving f64, FastPath, `i64` keys | `1f 00` | the Count Sketch f64 matrix `[[0.0,-0.0,1.5,-2.25],[127.0,-32768.5,0.125,-1e300]]`; `capacity=3`; summary `{(-1,65536,0), (7,6,5), (i64::MIN,6,0)}`, `total=65548`, `discarded_max=5` |
| `csspacesaving_nototal_f64_fast_2x4_i64keys` | CSSpaceSaving f64, FastPath, `i64` keys, no `total` | `1f 01` | the `csspacesaving_f64_fast_2x4_i64keys` matrix and summary without `total`; `discarded_max=5` |
| `hydra_kll_2x2_k200` | Hydra, KLL counter (k=200, m=8), schema `["region","service"]` | `07 00` | 2×2 grid, row-major cells: `[1.0..=5.0]` seed 1, empty seed 2, `[2.5, -1.0, 0.0, 1e300, -0.125]` seed 3, `[3.0e-5]` seed 4; each coin is `[seed, 0, 0]` |
| `hydra_cm_2x2_counter_2x2` | Hydra, Count-Min counter (2×2 i32, FastPath), same schema | `07 01` | 2×2 grid, row-major cells: `[[0,1],[127,128]]`, `[[255,256],[300,65535]]`, `[[65536,1000000],[2147483647,0]]`, all zero |
| `hydra_cs_2x2_counter_2x2` | Hydra, Count Sketch counter (2×2 i32, FastPath), same schema | `07 02` | 2×2 grid, row-major cells: `[[0,-1],[127,-32]]`, `[[-33,128],[-128,-129]]`, `[[-32768,65536],[-32769,2147483647]]`, `[[-2147483648,1],[0,0]]` |
| `hydra_hll_1x2_p14` | Hydra, HLL Ertl-MLE counter (P14), same schema | `07 03` | 1×2 grid; cell 0 registers `[0]=1, [1]=7, [100]=42, [16383]=3`; cell 1 `[0]=2, [8192]=51`; all others 0 |
| `hydra_univmon_1x2` | Hydra, UnivMon counter (2 layers of 1×2, heap 2, `u64` keys), same schema | `07 04` | 1×2 grid; cell 0: layer 0 counts `[5,-3]`, l2 `34`, heap `{7:5, 300:2}`, incomplete; layer 1 counts `[0,2]`, l2 `4`, heap `{4294967296:2}`, complete; total weight 7, standard mode; cell 1 empty |
| `coco_3x7` | Coco, 3×7 table | `0c 00` | 15 occupied buckets `(row, col): key=value`: `(0,0) "uint32-min"=65536`, `(0,1) "uint16-max"=65535`, `(0,2) "fixint-max"=127`, `(0,3) ""=1`, `(0,4) "uint8-max"=255`, `(0,5) "emoji-😀"=5`, `(1,0) "uint16-min"=256`, `(1,1) "str8-min-32-bytes-0123456789abcd"=3`, `(1,2) "uint8-min"=128`, `(1,3) "clé-ünïcode-流量"=4`, `(1,4) "fixstr-max-31-bytes-0123456789a"=2`, `(1,5) "uint64-min"=4294967296`, `(1,6) "zero"=0`, `(2,1) "uint64-max"=u64::MAX`, `(2,5) "uint32-max"=4294967295`; the other 6 unoccupied |
| `elastic_4b_2x4` | Elastic, 4 heavy buckets, light 2×4 i32 RegularPath | `0b 00` | heavy `(flow_id, vote+, vote-, eviction)`: free with the flag set, `("10.0.0.1:443>192.168.10.20:5123",127,128,false)`, `("",1,65535,true)`, `("10.0.0.1:443>192.168.10.20:51234",2147483647,256,true)`; light row-major `[[0,255,65536,2147483647],[-1,-33,-32768,-2147483648]]`; `stale_copies=false` |
| `elastic_4b_2x4_stale` | Elastic, same geometry | `0b 00` | same state — differs from the above only by `stale_copies=true` |
| `univmon_str_l3_2x4_h5` | UnivMon, 3 layers of 2×4, heap 5, `string` keys | `10 00` | layer 0 counts `[[0,127,128,65536],[-1,-33,-32768,-2147483648]]`, l2 `[4294999809, 4611686019501130818]`, heap `{"alpha":65536, "beta":300, "delta":128}`, complete; layer 1 counts `[[3,-2,0,1],[0,0,5,-4]]`, l2 `[14, 41]`, heap `{"gamma":5}`, incomplete; layer 2 counts `[[0,7,0,0],[-6,0,0,0]]`, l2 `[49, 36]`, heap `{"epsilon":9, "zeta":2}`, complete; total weight 70000, standard mode |
| `univmon_i64_l3_2x4_h5` | UnivMon, same shape, `i64` keys | `10 00` | same layers — differs from the above only by `key_type` and keys: layer 0 heap `{i64::MIN:65536, -1:300, -129:128}`, layer 1 heap `{128:5}`, layer 2 heap `{4294967296:9, 7:2}` |
| `univmon_empty_l3_2x4_h5` | UnivMon, same shape | `10 00` | freshly constructed: all counts and l2 zero, heaps empty, every layer complete, total weight 0, unset mode; `key_type` `u64` |
| `count_l2hh_2x4_seed7` | CountL2HH, 2×4, seed index 7 | `19 00` | counts row-major `[[127,128,65535,-32768],[-32,-33,-2147483648,i64::MIN]]`; l2 `[65536, i64::MAX]` |
| `set_aggregator_strings` | SetAggregator | `08 00` | `{"", "ab", "abc", "abcdefghijklmnopqrstuvwxyz012345", "api", "fixstr-max-31-bytes-0123456789a", "web", "é", "中", "～", "😀"}` |
| `set_aggregator_empty` | SetAggregator, empty | `08 00` | `{}` |
| `delta_result_strings` | DeltaResult | `08 80` | added `{"queue", "été", "😀"}`; removed `{"", "cache", "db", "中"}` |
| `delta_result_empty` | DeltaResult, empty | `08 80` | added `{}`; removed `{}` |
| `kmv_k8` | KMV, k=8 | `0e 00` | digests `[1, 127, 128, 65535, 4294967296, 0xfedcba9876543210]` |
| `bloom_regular_3x128` | Bloom, RegularPath, 3 slices of 128 bits | `17 00` | words row-major `[1, 2^63, 0, 0xff, u64::MAX, 0x10000]`; `inserted=300` |
| `bloom_fast_5x32` | Bloom, FastPath, 5 slices of 32 bits | `17 00` | words `[1, 0x80000000, 0, 0xffffffff, 0x10000]`; `inserted=9` |
| `bloom_bits_regular_3x128` | Bloom, RegularPath, 3 slices of 128 bits | `17 01` | the `bloom_regular_3x128` words, no `inserted` |
| `bloom_bits_fast_5x32` | Bloom, FastPath, 5 slices of 32 bits | `17 01` | the `bloom_fast_5x32` words, no `inserted` |
| `space_saving_i64_c8` | Space-Saving, `i64` keys, capacity 8 | `18 00` | `(key, count, error)`: `(-1,70000,0)`, `(300,9,2)`, `(0,5,0)`, `(1,5,1)`, `(i64::MIN,5,4)`, `(-129,5,0)`; `total=70030`, `discarded_max=4` |
| `space_saving_bytes_c3` | Space-Saving, `bytes` keys, capacity 3 | `18 00` | `(ff 00, 9, 0)`, `("", 4, 1)`, `("ab", 4, 0)`; `total=20`, `discarded_max=3` |
| `space_saving_nototal_i64_c8` | Space-Saving, `i64` keys, capacity 8, no `total` | `18 01` | the `space_saving_i64_c8` triples; `discarded_max=4` |
| `space_saving_nototal_bytes_c3` | Space-Saving, `bytes` keys, capacity 3, no `total` | `18 01` | the `space_saving_bytes_c3` triples; `discarded_max=3` |
| `uniform_sampling_r0p25` | UniformSampling, rate 0.25 | `0d 00` | `(priority, value)`: `(3,-0.0)`, `(3,0.0)`, `(2^63,1e300)`, `(u64::MAX,NaN)`; `total_seen=20`, `rng_state=0x0123456789abcdef` |
| `univmon_optimized_l5_e2_3x8_1x4_h6` | UnivMon Optimized, 5 layers, 2 elephant layers of 3×8, mouse 1×4, heap 6, `string` keys | `11 00` | nonzero cells `(row, col)=value`: layer 0 `(0,1)=127, (0,2)=-1, (0,5)=128, (0,7)=-33, (1,0)=65536, (1,7)=-32768, (2,6)=300`; layer 1 `(0,0)=5, (1,2)=-3, (2,7)=2`; layer 2 `(0,1)=4, (0,3)=-2`; layer 3 `(0,0)=1`; each row's `l2` its sum of squares; heaps `{"alpha":40, "beta":7, "gamma":7}`, `{"delta":3}`, `{}`, `{"eps":1}`, `{}`; total weight 70, unset mode, every layer complete |
| `univmon_q_i32_l3_c8_r3` | UnivMon-Q, 3 layers, `top_layer_cols=8` halving every layer, 3 rows, `i32`, heap 2, 3 ordered samples, seed index 5 | `1a 00` | `counters[i]` over layer runs of 24, 12 and 6 cells: `P[(5·layer + 3·row + col) mod 14]`, `P = [0,1,-1,127,-128,32767,-32769,i32::MAX,i32::MIN,65536,-2,300,0,9]`; candidates `{o(-2.5):4, o(3.0):2}`, `{o(0.5):1, o(5.0):3}`, `{o(0.0):2}`; `ever_evicted=[false,true,false]`; `count=12`, `min=o(-2.5)`, `max=o(5.0)`; `source_id=7`, `next_sequence=12`; occurrences `(1, u64::MAX, o(3.0))`, `(0x0123456789abcdef, 0, o(0.0))`, `(0xfedcba9876543210, 0x42, o(-2.5))`; `o(x)` is the order-preserving `u64` of `x` |
| `ehsketchlist_cm_i32_fast_2x4` | EHSketchList, Count-Min i32 FastPath 2×4 | `14 00` | `[[127,-1,0,65536],[-32768,128,i32::MAX,i32::MIN]]` |
| `ehsketchlist_ddsketch_a001` | EHSketchList, DDSketch | `14 00` | the `ddsketch_signed_a001` state |
| `ehsketchlist_ddsketch_buckets_a001` | EHSketchList, DDSketch (buckets only) | `14 00` | the `ddsketch_buckets_signed_a001` state, nested under kind_id `05 01` |
| `eh_w100_k2_cm_2x4` | ExponentialHistogram, window 100, k 2, Count-Min i32 FastPath 2×4 | `13 00` | buckets `[[1,0,2,0],[0,3,0,0]]` size 3 over `[10,30]` and `[[0,0,0,1],[1,0,0,0]]` size 1 over `[40,40]`; prototype all zero |

The HLL fixtures set the register bytes directly. Each estimator has a P12 and
a P14 fixture; the P14 set holds register value 51, the largest a P14 register
takes, and touches the first, a middle and the last index.

The HLL delta fixtures carry the metadata block of the P14 state fixture they
merge into, byte for byte, under kind_id `01 81` or `01 82`. Their registers
hold 1 and 51, the smallest and largest a delta carries, and their indices the
first, a middle and the last; the delta raises `[0]` and leaves `[8192]` and
`[16383]`, which hold more, unchanged.

The CMS i64 fixture deliberately spans the msgpack integer width boundaries
(positive fixint / uint8 / uint16 / uint32) to lock the "non-negative integer →
uint family, minimal width" rule (spec Section 4).

The Count-Min delta fixtures carry the metadata block of the state fixture they
merge into, byte for byte, under kind_id `02 80`. The i64 delta leaves cell 3
out, and its deltas span positive fixint `127`, negative fixint `-1`, int8
`-33`, int32 `-32769` and uint64 `4294967296`; each is the first or last value
of its width.

The Count Sketch fixtures cover the **negative** side, because Count Sketch
cells are signed — it adds `±weight`: negative fixint / int8 / int16 / int32,
alongside positive fixint / uint8 / uint32.

The three integer Count Sketch files hold the same matrix, so each pair isolates
one metadata key: the two i64 files differ only by `mode`, and `cs_i32_regular_2x4`
differs from `cs_i64_regular_2x4` only in the `counter_type` value, `"i64"`
against `"i32"`. The payloads are byte-identical, because msgpack encodes an
integer at its minimal width whatever the source type is. So the i32 fixture
pins that the counter type reaches the bytes, and that nothing else does.

The Count Sketch delta fixtures carry the metadata block of the state fixture
they merge into, byte for byte, under kind_id `04 80`. The i64 delta leaves
cells 3 and 5 out, and its deltas span negative fixint `-32`, int8 `-128`,
uint8 `255`, int16 `-129`, uint16 `65535` and int64 `-2147483649`; each is the
first or last value of its width.

The KLL fixtures are a special case of "state is fixed, not hashed": KLL never
hashes — it orders raw numeric values — so inserting known values places exactly
those retained samples. `k=200` keeps every input below the level-0 capacity, so
no compaction fires (`num_levels = 1`, `levels = [0, n]`, items in input order)
and the state is fully deterministic. The fixed compaction seed (42) pins the
carried coin state, `[42, 0, 0]`. Compact KLL records the seed in metadata;
KLLDynamic never emits the `compaction_seed` key, so its metadata differs only by omitting
it. The two variants share the payload shape and differ by `kind_id`. The
dynamic f64 items are negative, zero and fractional; the dynamic i64 items span
positive fixint / uint8 / uint16 / uint32 / uint64 and negative fixint / int8 /
int16 / int32 / int64.

DDSketch never hashes, so its fixtures set the bucket stores, offsets, zero
count and the `sum` / `min` / `max` scalars directly. The positive fixture's
counts span positive fixint / uint8 / uint16 / uint32 / uint64 and its offset is
an int8. Every payload carries both stores and the zero count, so the positive
and empty fixtures hold an empty negative store at offset `0` and a zero
`zero_count`. The signed fixture's offsets are a uint16 and an int16. The empty
fixture is a fresh sketch: no buckets and the `0.0` / `+inf` / `-inf` scalars.
α is a single metadata `f64`, so all three use 0.01.

The DDSketch delta fixture carries the metadata block of `ddsketch_signed_a001`,
byte for byte, under kind_id `05 80`. Its indices in each store span int16,
int8, negative fixint, positive fixint, uint8 and uint16, and its positive
deltas span positive fixint / uint8 / uint16 / uint32 / uint64. Merged, each
bucket's representative `r(k) = (1 + α)·γ^k`, negated in the negative store, is
folded into `sum` (as `r·count`), `min` and `max`, positive store first, in
payload order.

The `ddsketch_*` fixtures above carry the retired kinds `05 00` / `05 80`, whose
state also holds `sum` / `min` / `max`; they stay as decode-only fixtures for
bytes written before the change. The current kinds `05 01` / `05 81` carry the
two stores and the zero count alone, so the `ddsketch_buckets_*` fixtures are
the same states and delta with those scalars dropped: a decoder reads a retired
fixture into the same stores and re-encodes it as its `ddsketch_buckets_*`
twin. Sum, minimum and maximum are not DDSketch state; a separate exact
aggregate keeps them.

The four `_admission` fixtures each hold the state of the fixture they are
named after, and their metadata is that fixture's map with one more key,
`admission_rate`, last, as float64. The payloads are byte-identical to that
fixture's. `0.1` is the nearest float64, `3fb999999999999a`.

The CMSHeap fixtures set the matrix and the heap entries directly. The integer
ones reuse the Count-Min i64 matrix; between them they cover two counter types,
both modes and two key types. Entries are emitted by descending count, ties by
key: a signed key compares as its two's-complement bit pattern read unsigned, so
`i64tie` emits `0, 1, i64::MIN, -1`; a string compares byte-wise, a proper
prefix first, so `strtie` emits `"Z", "a", "aa", "b"`. The `i64keys` keys span
negative fixint / int16 / uint64. The empty heap emits `key_type` `"u64"` with
two empty arrays.

The f64 CMSHeap fixture carries float64 heap counts, ordered by IEEE-754
`totalOrder` (Rust `f64::total_cmp`): `+inf` first, `0.0` before `-0.0`, `-inf`
last. Its `300.25` tie falls to the keys' bits, so `1.5` precedes `-0.5`.

The CMSHeap delta fixture carries the metadata block of
`cmsheap_i64_regular_2x3_strkeys` except `key_type`, `"i64"` for its own heap,
under kind_id `03 80`. Its cells are the Count-Min i64 delta's.

The `tuplekeys` fixtures hold keys made of typed columns, each a `bytes` key
in the spec's tuple key encoding (Section 2): per column a tag byte, then an
8-byte big-endian `i64` or `f64`, or a big-endian `u32` length and UTF-8.
`-0.0` is written as `0.0` and `NaN` as `0x7ff8000000000000`. Ties fall to the
bytes: the two `["us-east", …]` keys at 300 differ first in the `i64` column.

The CSHeap state fixtures set the Count Sketch matrix and the heap entries
directly, so their `counts` arrays are byte-identical to the Count Sketch
fixtures'. The i64 one's heap is one entry short of `heap_size` and holds a
count tie (`beta` before `delta`); its heap counts span uint64 / positive
fixint / negative int8: a CSHeap heap count is a signed median. The `i64tie`
one ties four keys at a negative count, emitted `0, 1, i64::MIN, -1` as in
`cmsheap_i64_regular_2x3_i64tie`.

The f64 CSHeap fixture carries float64 heap counts, the unrounded signed median:
`-16384.25` is a negative one and `63.5625` a fractional one. Its `-0.75` tie
falls to the keys, so `"aa"` precedes `"beta"`.

The CSHeap delta fixture carries the metadata block of
`csheap_i64_regular_2x4_strkeys` except `key_type`, `"i64"` for its own heap,
under kind_id `0a 80`. Its cells are the Count Sketch i64 delta's; its heap
counts span uint32 / negative int8 / int64.

The `space_saving_*`, `cmsspacesaving_*` and `csspacesaving_*` fixtures without
`nototal` carry the retired kinds `18 00` / `1e 00` / `1f 00`, whose summary
also holds a `total`; they stay as decode-only fixtures for bytes written before
the change. The current kinds `18 01` / `1e 01` / `1f 01` drop `total`, so each
`*_nototal_*` fixture is the same state without it: a decoder reads a retired
fixture into the same triples and ceiling and re-encodes it as its
`*_nototal_*` twin. The stream's total weight is not Space-Saving state; a
separate exact count keeps it.

The CMSSpaceSaving and CSSpaceSaving fixtures set the matrix directly and
build the summary by weighted inserts, whose triples do not depend on the hash.
Each summary is full and has displaced one key, so it carries a non-zero error
and `discarded_max`; `rows`, `cols` and `capacity` are pairwise distinct. The
count ties fall to the keys: `"aa"` precedes `"b"`, and `7` precedes
`i64::MIN`, whose two's-complement bits read higher. The summary counts span
positive fixint / uint8 / uint16 / uint32 and the `i64` keys negative fixint /
positive fixint / int64.

The Hydra fixtures set every cell's state directly, so neither the subkeys nor
the values are hashed: the matrix cells from storage, the HLL registers by
pre-hashed values crafted to land on each index and rank, the KLL cells by
inserting raw values (k=200, so no compaction fires) under a distinct
compaction seed per cell, and the UnivMon layers by counter deltas at named
cells plus explicit heap entries. Each grid keeps the empty cell's shape in the
bytes. The 2×2 grids pin grid row-major order; the matrix counters' 2×2 runs pin
row-major order inside a cell. The Count-Min counters span positive fixint /
uint8 / uint16 / uint32 up to `i32::MAX`; the Count Sketch counters add negative
fixint / int8 / int16 / int32 down to `i32::MIN`. The HLL fixture holds register
value 51, the largest a P14 register takes.

The Coco fixture sets every bucket's key and value directly. Each key sits in
the column its row hashes it to, folded `% cols`, because a decoder rejects any
other placement; the bytes themselves carry no hash. The width is 7, not a power
of two, so a port that folds with a mask (`& (cols-1)`) instead of `% cols`
misplaces the keys and fails the fixture. The values span positive fixint /
uint8 / uint16 / uint32 / uint64 at both ends of each width. The keys cover the
empty string (an occupied bucket, distinct from an unoccupied `nil` one), a
31-byte fixstr, a 32-byte str8, and UTF-8 code points of two, three and four
bytes; `"zero"` is an occupied bucket holding 0.

The Elastic fixtures set the heavy buckets and the light Count-Min cells
directly; no flow id is hashed. The heavy table holds a free bucket (`nil`)
whose eviction flag is set, as a vacated bucket keeps it, an empty flow id
(`""`) and ids of 31 and 32 bytes, the fixstr / str8 boundary. The
votes span positive fixint / uint8 / uint16 / uint32 up to `i32::MAX`; the light
row 0 spans uint8 / uint32 up to `i32::MAX` and row 1 negative fixint / int8 /
int16 / int32 down to `i32::MIN`. The two files differ in one byte, the
`stale_copies` bool.

The UnivMon fixtures set every layer directly: counters by deltas at named
cells, which carry each row's `l2` accumulator, and heap entries by explicit
`(key, count)` pairs; no hash reaches the bytes. The shape's four parameters
(3 layers, 2 rows, 4 columns, heap 5) are pairwise distinct, so no dimension can
stand in for another; the heaps hold 3, 1 and 2 entries. Layer 0 holds the
Count Sketch matrix, so the counters span positive fixint / uint8 / uint32 and
negative fixint / int8 / int16 / int32, and both of its `l2` values are uint64.
The heap counts span uint32 / uint16 / uint8 / positive fixint and the `i64`
keys int64 / negative fixint / int16 / uint8 / uint64 / positive fixint. The two
populated files differ only in `key_type` and `keys`. The empty file pins the
encoding of a pyramid with no keys, whose `key_type` is `u64`. The fixtures
cover `update_mode` 0 (unset) and 1 (standard), not 2 (terminal-only): only a
hashed insert selects that mode.

The CountL2HH fixture sets the counts, the per-row `l2` accumulators and the
seed index directly, through the sketch's serde form; no key is hashed. Each
`l2` is set apart from the counts, since it is carried state rather than the
row's sum of squares. The cells span positive fixint / uint8 / uint16 and
negative fixint / int8 / int16 / int32 / int64 down to `i64::MIN`; the `l2`
values are a uint32 and `i64::MAX`, the accumulator's saturation value, a
uint64. Between them every msgpack integer width appears.

The SetAggregator and DeltaResult fixtures set the string sets directly; neither
type hashes. Each array is in ascending UTF-8 byte order. The strings cover the
empty string, a proper-prefix chain (`"ab"` before `"abc"` before the 32-byte
key), a 31-byte fixstr and a 32-byte str8, and UTF-8 code points of two, three
and four bytes;
`"～"` (U+FF5E) precedes `"😀"` (U+1F600), the reverse of their UTF-16 order.

The KMV, Bloom, Space-Saving and UniformSampling fixtures set their state
directly: KMV's digests are inserted as digests, and the other three go through
their serde form. KMV's last digest is above 2^63 and still a uint64. Bloom
column `c` of row `r` is bit `c mod 64` of word `r·ceil(cols/64) + c/64`; the
3×128 grid has two words per row, and the 5×32 grid one, whose top 32 bits are
padding and zero. The Space-Saving `i64` fixture holds fewer counters than its
capacity, and its count-5 tie emits by two's-complement bits: `0, 1, i64::MIN,
-129`. The `bytes` fixture is full and its keys are `bin`, one of them not
UTF-8. The UniformSampling priority-3 tie orders `-0.0` before `0.0`, and the
`NaN` sample is `0x7ff8000000000000`.

The `bloom_regular_3x128` and `bloom_fast_5x32` fixtures carry the retired kind
`17 00`, whose payload `[words, inserted]` also holds an insert counter; they
stay as decode-only fixtures for bytes written before the change. The current
kind `17 01` carries `[words]` alone, so each `bloom_bits_*` fixture is its twin
with `inserted` dropped: a decoder reads a retired fixture into the same grid
and re-encodes it as its `bloom_bits_*` twin. An insert count is not Bloom
state; a separate exact count keeps it.

The UnivMon Optimized fixture sets counters by deltas at named cells and heap
entries by explicit pairs, as the UnivMon fixtures do. Its seven layout values
(5, 2, 3, 8, 1, 4, 6) are pairwise distinct. Its mode is unset and every layer
complete; the UnivMon fixtures cover mode 1 and an incomplete layer through the
same payload.

The UnivMon-Q state is not reachable through an update stream: its counters are
fixed values, not the sum of its observations. Each candidate key hashes into
the layer it sits in, because the decoder checks it. Layer widths are 8, 4 and
2. Candidates ascend by key within a layer, and occurrences by `(priority_high,
priority_low, key)`.

The EHSketchList and ExponentialHistogram fixtures carry each nested sketch's
metadata and payload blocks byte for byte: the DDSketch triple holds
`ddsketch_signed_a001`'s blocks, and each Count-Min block is the `0x02 0x00`
encoding of a 2×4 `i32` FastPath matrix.

## Coverage

The fixtures cover thirty-eight `kind_id`s, every one the spec's registry marks
*implemented*: HLL's three estimators (at P12 and P14) and the Classic and
Ertl-MLE deltas, Count-Min and its delta, CMSHeap and its delta, Count Sketch
and its delta, CSHeap and its delta, CMSSpaceSaving, CSSpaceSaving, DDSketch and its delta, both KLL variants
(compact and dynamic), Hydra's five counter variants, Coco, Elastic,
UniformSampling, KMV, UnivMon, UnivMon Optimized, ExponentialHistogram,
EHSketchList, Bloom, Space-Saving, CountL2HH, UnivMon-Q, SetAggregator and
DeltaResult. The `admission_rate` key is covered on the HLL Classic,
Count-Min, Count Sketch and DDSketch states, not on a delta. EHSketchList is
covered for its Count-Min and DDSketch variants, standalone, and for Count-Min
inside an ExponentialHistogram.

## Changing a fixture

The bytes are authored by `asap_sketchlib` (rmp_serde is the reference
encoder); other implementations conform to them, never the reverse.

1. Commit the new or changed `.hex` here, with its row in the table above.
2. In each consumer, bump the `asapv1_golden` submodule to that commit and
   update its golden test in the same PR.
