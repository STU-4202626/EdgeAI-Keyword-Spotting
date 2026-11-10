# Historical Results and Extraction Provenance

All benchmark implementations and recorded artifacts were extracted from published EdgeAI commit `7d31a9c273d1e4b088faed55508907e7ef04c3d3`. [SOURCE_MANIFEST.json](../SOURCE_MANIFEST.json) records the original Git paths and SHA-256 hashes. No experiment, model training, firmware build or result regeneration was performed.

Raw benchmark-result CSVs, predictions, confusion matrices and available ROC curves are copied unchanged under [`historical-results/`](../historical-results/). Their original relative paths are retained below that directory. The platform and combined summary files select only Keyword Spotting records from the existing published CSVs: six combined configurations, with two Raspberry Pi configurations and one on each other platform. Selection preserves complete original record lines and all recorded cell strings; it does not average runs or recompute measurements.

Active benchmark output directories start without historical results and are ignored. Keep recorded results in `historical-results/` when running future experiments. The compilation utilities operate only on the active output directories. Predictions and figures describe the historical implementations.

The Android and Raspberry Pi INT8 scripts are the historical pre-correction implementations. They convert quantised KWS inputs to INT8 without clipping and apply an additional Softmax after dequantisation, although the deployed model already contains Softmax. Later fixes change input/probability handling and have not been benchmarked again. Existing results are measurements of the retained historical implementations, not measurements of the corrected branch. New experiments are necessary before reporting corrected-code results. The original provenance discussion is retained in the [published project](https://github.com/STU-4202626/EdgeAI-Performance-Analysis/blob/7d31a9c273d1e4b088faed55508907e7ef04c3d3/MLPerf_Tiny/RESULTS_PROVENANCE.md).

The local Honours report and IST-Africa draft informed the explanatory documentation; they are not redistributed. Numerical tables use the pinned published CSV values when report rounding differs. Original MLCommons reference material supplies model, dataset and quality-target background, rather than evidence of an official submission.

Times on CPU/GPU/Pi/Android cover the Python prediction loop. ESP32-S3 times sum device-side `Invoke()` only, excluding serial communication, preprocessing and the firmware command delay. CodeCarbon and the ESP32 datasheet-based power helper estimate electricity/carbon; they do not directly measure whole-device power.

Exact historical Python environments and complete dataset ordering cannot be reconstructed from these artifacts alone. See the root [README](../README.md) for workload-specific limitations. New measurements must be labelled as new experiments.

## CodeCarbon Public-Copy Privacy Reduction

The 5 historical `emissions.csv` files contain **114 original rows**, in their original order. On 10 November 2026 the standalone public copies removed only these six geographical columns:

- `latitude`
- `longitude`
- `country_name`
- `country_iso_code`
- `region`
- `cloud_region`

Every retained header and cell value was compared as an exact string with the matching source at commit `7d31a9c273d1e4b088faed55508907e7ef04c3d3`. No numeric parsing, recalculation, rounding or measurement replacement was used. Duration, energy, emissions, emissions rate, estimated power and all other retained values are unchanged. Timestamps, `run_id` and existing `experiment_id` values are retained; the identifiers are valid UUIDs rather than personal text. Empty `cloud_provider` and the `on_cloud` infrastructure flag are retained.

`SOURCE_MANIFEST.json` distinguishes the original Git blob (`source_sha256`, `source_bytes`) from its public copy (`sha256`, `bytes`). Each transformed entry records `kind = privacy-reduced-columns`, `removed_columns`, ordered `retained_columns`, `row_count` and `retained_values_sha256`. The last digest is computed from the original retained-column projection as canonical UTF-8 JSON `[header, *rows]` with `ensure_ascii=False` and `separators=(',', ':')`; every cell remains a string.

`tools/verify_static.py` verifies public hashes, column exclusion, row counts and the retained-value digest without requiring the original repository. The preparation also compared every retained cell and row directly with the original Git blob. No removed geographical values are included in the manifest or this note.

These files remain historical CodeCarbon estimates, influenced by the original regional factors. Removing metadata does not recalculate emissions or establish a common geographic basis for comparisons. The original source files and both branches were not modified. The original public EdgeAI repository still tracks the geographical fields at the pinned baseline; this privacy reduction applies only to these standalone copies.
