# EdgeAI - Keyword Spotting

**How efficiently can edge devices recognise a small spoken vocabulary?**

> [!IMPORTANT]
> This repository preserves the Keyword Spotting experiments from the completed EdgeAI Honours project at commit `7d31a9c`. It includes the historical DS-CNN implementations and recorded results.

The benchmark compares keyword recognition across an x86 CPU, a dedicated GPU, a Raspberry Pi 5, an Android smartphone and an ESP32-S3. Classification measurements are considered alongside workload inference time, throughput and estimated operational carbon emissions.

These are **adapted MLPerf Tiny experimental evaluations**, not official MLPerf Tiny submissions.

## Model and dataset

The reference model is a **depthwise separable convolutional neural network (DS-CNN)**. Depthwise and pointwise convolutions reduce the computation required to classify a short audio feature tensor.

The workload uses **Speech Commands v0.02**, evaluated on **1,000 samples** selected from the TensorFlow Datasets test split. The model predicts **12 classes**: down, go, left, no, off, on, right, stop, up, yes, silence and unknown. The MLPerf Tiny reference quality target used here is **90% top-1 accuracy**.

The reference preprocessing accepts one-second, **16 kHz** audio clips. A 30 ms analysis window, 20 ms stride and ten MFCC coefficients produce a **49 x 10 x 1** feature tensor. The retained `get_dataset.py` implements feature extraction; the model is supplied, rather than trained as part of benchmark execution. See the [Speech Commands dataset](https://www.tensorflow.org/datasets/catalog/speech_commands) and [MLCommons reference](https://github.com/mlcommons/tiny).

## Evaluated configurations

<table align="center">
  <thead>
    <tr><th>Device</th><th>Deployment configuration</th></tr>
  </thead>
  <tbody>
    <tr><td>AMD Ryzen 7 6800H</td><td>Full-precision TensorFlow; x86 CPU reference</td></tr>
    <tr><td>NVIDIA GeForce RTX 4090</td><td>Full-precision TensorFlow; RunPod GPU reference</td></tr>
    <tr><td>Raspberry Pi 5</td><td>Full-precision TensorFlow and INT8 LiteRT</td></tr>
    <tr><td>Samsung Galaxy S26 Ultra</td><td>INT8 LiteRT through tflite_runtime</td></tr>
    <tr><td>Waveshare ESP32-S3</td><td>INT8 TensorFlow Lite Micro; 240 MHz, 16 MB flash, 8 MB Octal PSRAM</td></tr>
  </tbody>
</table>

Full-precision TensorFlow loads the `trained_models/kws_ref_model/` SavedModel. Raspberry Pi INT8 and Android load `kws_ref_model.tflite` through LiteRT. The ESP32-S3 executes the matching embedded INT8 DS-CNN through TensorFlow Lite Micro.

CPU, GPU and ESP host paths prepare test features through TensorFlow Datasets. Android and **both Raspberry Pi configurations** load separately exported `benchmark_data/x_test.npy` and `y_test.npy`. The exporter and its helper dependencies are included; the dataset arrays are excluded.

## Implementation and procedure

Each configuration uses one warm-up followed by three measured runs. Each measured run repeats the complete 1,000-sample workload until at least 30 seconds of elapsed prediction-loop time is reached, completing its current repetition. Time and emissions are normalized to one workload. The ESP32-S3 instead accumulates device-side invocation time for the interval.

The ESP host sends **490-byte INT8 tensors at 921600 baud**. Firmware uses a **128 KiB PSRAM arena** and returns 12 quantised outputs with the time surrounding `Invoke()`. Host feature extraction, serial transfers and the firmware's 500 ms command delay are excluded from reported ESP inference time and throughput.

See the [DS-CNN firmware documentation](MLPerf_Tiny/MLPerf_Tiny_ESP32-S3/training/keyword_spotting/firmware/README.md) for ESP-IDF **6.0.3**, TensorFlow Lite Micro component **1.4.0**, ESP-NN **1.3.2** and the build/flash procedure.

> [!NOTE]
> The Android and Raspberry Pi INT8 scripts retain historical input quantisation and output probability handling. Later clipping/probability corrections have not been benchmarked again. These CSVs describe the historical implementations, not the corrected branch.

## Recorded results

Time covers **all 1,000 samples**, throughput is samples/s, and emissions are estimated kgCO2e per workload. Accuracy, workload time and estimated emissions average the three measured runs, excluding warm-up. Throughput is the workload sample count divided by the mean recorded workload time. All six configurations met the 90% target.

<table align="center">
  <thead>
    <tr><th>Configuration</th><th>Accuracy (%)</th><th>Time / workload (s)</th><th>Throughput (samples/s)</th><th>Estimated emissions (kgCO2e)</th></tr>
  </thead>
  <tbody>
    <tr><td>AMD Ryzen 7 6800H</td><td>91.9</td><td>1.587</td><td>630.12</td><td>2.673e-06</td></tr>
    <tr><td>NVIDIA GeForce RTX 4090</td><td>91.9</td><td>0.59</td><td>1695.87</td><td>8.266e-06</td></tr>
    <tr><td>Raspberry Pi 5 - full precision</td><td>91.9</td><td>1.248</td><td>801.28</td><td>6.570e-07</td></tr>
    <tr><td>Raspberry Pi 5 - INT8</td><td>91.2</td><td>0.339</td><td>2946.95</td><td>1.812e-07</td></tr>
    <tr><td>Samsung Galaxy S26 Ultra</td><td>91.1</td><td>0.046</td><td>21897.81</td><td>1.395e-09</td></tr>
    <tr><td>ESP32-S3</td><td>91.2</td><td>19.823</td><td>50.45</td><td>1.130e-06</td></tr>
  </tbody>
</table>

The [six-record summary](historical-results/MLPerf_Tiny/mlperf_tiny_all_devices_summary.csv) retains the other classification metrics, repetitions and status. [Historical result folders](historical-results/MLPerf_Tiny/) preserve raw per-run CSVs, prediction labels, confusion matrices, ROC curves and available CodeCarbon logs.

The full-precision configurations recorded 91.9% accuracy; the INT8 configurations recorded 91.1–91.2%. The Android and Pi INT8 paths have higher recorded throughput, with runtime and output handling differences that limit an isolated hardware comparison. Saved prediction files contain labels rather than complete probability vectors, so they cannot reconstruct metrics for the later probability corrections.

## Installation and execution

### Python environment

Install common analysis packages from the repository root:

```text
python -m pip install -r requirements/base.txt
```

For CPU/GPU/Pi full precision, separately install a compatible **TensorFlow/Keras 2** build and `requirements/tensorflow.txt`. CPU/GPU preprocessing also requires TensorFlow Datasets. Android/Pi INT8 need `requirements/litert.txt` and a platform-compatible `tflite-runtime` wheel.

The ESP host needs `requirements/esp32-host.txt` plus compatible TensorFlow/Keras and TensorFlow Datasets for MFCC preparation. KWS helpers import the legacy TensorFlow microfrontend module even when MFCC features are selected. These import-derived package lists are not a historical version lock or a verified fresh environment; a generic latest TensorFlow installation is not an established substitute. GPU drivers and TensorFlow GPU support must be prepared separately.

### Prepare the evaluation features

Obtain Speech Commands through [TensorFlow Datasets](https://www.tensorflow.org/datasets/catalog/speech_commands). CPU/GPU/ESP scripts load `speech_commands` through `get_dataset.py`; `--data_dir` selects the TFDS cache (the default is `~/data`). Data may be prepared or downloaded if absent. The TFDS package/builder version and sample ordering are not pinned.

On a compatible desktop TensorFlow environment, run the included exporter from the Android workload directory:

```text
cd MLPerf_Tiny/MLPerf_Tiny_Android/training/keyword_spotting
python export_test_set.py --data_dir PATH_TO_TFDS_CACHE
```

It selects `ds_test.unbatch().take(1000).batch(1)` and creates:

```text
benchmark_data/x_test.npy    # 1000 x 49 x 10 x 1 feature values
benchmark_data/y_test.npy    # 1000 class labels
```

Provide the generated `benchmark_data/` directory under both the Android and Raspberry Pi `training/keyword_spotting/` directories. Keep preprocessing and dataset ordering consistent when comparing a new experiment. A fresh export is not proof of identical historical tensors. No datasets or retained arrays from the original laptop were copied during extraction.

### Run a configuration

Change into the selected platform's `training/keyword_spotting/` directory, then execute:

<table align="center">
  <thead>
    <tr><th>Platform directory under MLPerf_Tiny/</th><th>Script in training/keyword_spotting/</th></tr>
  </thead>
  <tbody>
    <tr><td>MLPerf_Tiny_PC_CPU</td><td>mlperf_keyword_spotting_CPU_bench.py</td></tr>
    <tr><td>MLPerf_Tiny_PC_GPU</td><td>mlperf_keyword_spotting_RunPod_GPU.py</td></tr>
    <tr><td>MLPerf_Tiny_RaspberryPi5</td><td>mlperf_keyword_spotting_Pi5_full.py or mlperf_keyword_spotting_Pi5_quant.py</td></tr>
    <tr><td>MLPerf_Tiny_Android</td><td>mlperf_keyword_spotting_Android_bench.py</td></tr>
    <tr><td>MLPerf_Tiny_ESP32-S3</td><td>mlperf_keyword_spotting_ESP32S3_bench.py</td></tr>
  </tbody>
</table>

For example, from the CPU workload directory:

```text
python mlperf_keyword_spotting_CPU_bench.py --data_dir PATH_TO_TFDS_CACHE
```

For ESP32-S3, follow the firmware README to build/flash the matching DS-CNN project first. The historical host uses `COM8`; choose the actual port for a future run and close any serial monitor. `PING/PONG` confirms responsiveness without identifying the model. No firmware build or flash was performed during extraction.

Paths for models, arrays and outputs are relative to the workload working directory. Retained `training/results_helper.py` and `training/esp32_energy_helper.py` provide shared result and energy helpers. New outputs go to ignored `results/`, or Pi `results_full/` and `results_quant/`, separately from historical artifacts.

To compile **new** outputs, from `MLPerf_Tiny/` use `python compile_results.py MLPerf_Tiny_PC_CPU` for each platform folder and then `python compile_all_devices.py` after all five summaries exist. The standalone compiler discovers only Keyword Spotting. It does not regenerate or overwrite `historical-results/`.

## Measurement limitations and reproducibility

The CPU, GPU, Raspberry Pi and Android record elapsed Python prediction-loop time per complete workload, including runtime and output handling. The ESP32-S3 sums device-side `Invoke()` durations. These timing scopes differ; the tables do not measure end-to-end application latency.

CodeCarbon provides software-based energy and carbon estimates on the four larger platforms. Logs use different regional carbon intensities and power estimates, including a 42.5 W CPU fallback on Android. The ESP32-S3 helper applies **0.29007 W** and **0.707686 kgCO2e/kWh** to device invocation time. Neither method directly measures whole-device or host electricity consumption.

Python package versions, delegates and operating-system settings are not fully locked. CPU/GPU labels describe the reported experimental configurations; `gpu_ids=[]` configures CodeCarbon and does not itself force TensorFlow placement. The LiteRT scripts import `tflite_runtime.interpreter.Interpreter`; no explicit Android NPU delegate is configured. LiteRT denotes the TensorFlow Lite runtime used by these INT8 scripts, while TensorFlow Lite Micro executes the embedded model on the microcontroller.

See [result provenance](MLPerf_Tiny/RESULTS_PROVENANCE.md) and [static verification](STATIC_VERIFICATION.md). Extraction validation checks syntax, artifacts, links and recorded values. It does not establish a fresh environment, clean firmware build or successful new inference.

## Attribution

This workload derives from [MLCommons MLPerf Tiny](https://github.com/mlcommons/tiny). The [Apache 2.0 licence](MLPerf_Tiny/LICENSE.md), original source notices and [focused attribution](MLPerf_Tiny/ATTRIBUTION.md) are retained. Dataset and downloaded dependency terms apply separately.

This is a standalone extraction of **EdgeAI: Performance Analysis of Edge Nodes for AI Tasks**, a 2026 BSc Honours Computer Science project. See the [original Honours project](https://github.com/STU-4202626/EdgeAI-Performance-Analysis/tree/7d31a9c273d1e4b088faed55508907e7ef04c3d3) for the complete comparison and research context.
