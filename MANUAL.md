# Hasaki User Manual
**v3.2.1 — Neural Inference Engine for Embedded Systems**

---

## Table of Contents

1. [Overview](#1-overview)
2. [Editions: Free vs Pro](#2-editions-free-vs-pro)
3. [Installation](#3-installation)
4. [Command structure](#4-command-structure)
5. [Flags reference](#5-flags-reference)
6. [Actions](#6-actions)
   - [train](#61-train)
   - [predict](#62-predict)
   - [validate](#63-validate)
   - [export](#64-export)
   - [quantize_test](#65-quantize_test)
7. [Activation functions](#7-activation-functions)
8. [CSV format](#8-csv-format)
9. [Training behaviour](#9-training-behaviour)
10. [Resuming training](#10-resuming-training)
11. [Recovering the best model after a manual stop](#11-recovering-the-best-model-after-a-manual-stop)
12. [Exporting to C header](#12-exporting-to-c-header)
13. [Batch processing (Pro)](#13-batch-processing-pro)
14. [Complete examples](#14-complete-examples)
15. [Error reference](#15-error-reference)

---

## 1. Overview

Hasaki trains a neural network on your desktop and exports a single, self-contained C header file ready to drop into any embedded project. No runtime, no framework, no TensorFlow Lite. The exported header contains the trained weights and a `predict()` function that runs on any MCU that can compile C.

```c
#include "model.h"

float input[2]  = { 1.0f, 0.0f };
float output[1];

predict(input, output);
// output[0] → 0.987
```

No heap allocation. No dependencies. No runtime overhead.

---

## 2. Editions: Free vs Pro

Hasaki ships in two editions compiled at build time via a `#define`. The binary itself enforces limits — there is no runtime unlock.

| Feature | Free | Pro |
|---|---|---|
| Hidden layers | Max 2 | Unlimited |
| Neurons per layer | Max 64 | Unlimited |
| Epochs | Max 500 | Unlimited |
| Optimizer | SGD | SGD + Adam (`--adam`) |
| Quantization | Float (FP32) | Float · INT8 · INT4 |
| Batch processing | — | ✅ (`-batch`) |
| Dropout regularisation | — | ✅ |
| L2 regularisation | — | ✅ |
| Commercial use | Personal / Educational | ✅ |

> The Free edition is fully functional for learning, prototyping, and non-commercial projects.

---

## 3. Installation

Download the binary for your platform and place it somewhere in your `PATH`:

| Platform | Binary |
|---|---|
| Linux | `hasaki_free_linux` |
| Windows | `hasaki_free_windows.exe` |
| macOS | `hasaki_free_macos` |

Verify the installation:

```bash
hasaki -h
```

---

## 4. Command structure

```
hasaki -d <dims> -act <activations> -a <action> [options]
```

`-d`, `-act`, and `-a` are required for all actions except `-batch`. When using `-m` with a model that already contains architecture metadata, `-d` and `-act` become optional — the model file is the source of truth. The order of flags does not matter.

---

## 5. Flags reference

### Core

| Flag | Value | Description |
|---|---|---|
| `-d` | `int,int,...` | Network dimensions — input, hidden layers, output. Example: `2,4,1` or `784,128,64,10` |
| `-act` | `func,func,...` | Activation function per layer (one per layer transition). Example: `sigmoid,sigmoid` |
| `-a` | `action` | Action to execute. See [Actions](#6-actions) |

### Files

| Flag | Value | Description |
|---|---|---|
| `-f` | `path` | CSV data file (required for `train`, `validate`, `quantize_test`) |
| `-m` | `path` | Model file to load (required for `predict`, `validate`, `export`, `quantize_test`; optional for `train` to resume) |
| `-o` | `path` | Output file (required for `train` and `export`) |
| `-test-file` | `path` | Separate validation CSV (optional, overrides auto-split in `train`) |

### Training

| Flag | Value | Description |
|---|---|---|
| `-e` | `int` | Number of epochs. Default: `500` (Free) / `50000` (Pro) |
| `-l` | `float` | Learning rate. Default: `0.05` |
| `--batch-size` | `int` | Mini-batch size for training. `0` keeps auto sizing based on dataset size |
| `--seed` | `int` | Random seed for reproducible weight initialization and data shuffling. `-1` keeps random behavior |
| `--patience` | `int` | Early-stopping patience in epochs. `0` keeps automatic sizing |
| `--lr-decay` | `float` | Multiply the learning rate by this factor after each epoch. `1.0` disables decay |
| `--adam` | — | Use Adam optimizer instead of SGD **(Pro only)** |
| `--dropout` | `float` | Dropout rate for hidden layers, e.g. `0.5`. Applied during training only **(Pro only)** |
| `--l2` | `float` | L2 regularisation coefficient, e.g. `0.0001` **(Pro only)** |

### Prediction

| Flag | Value | Description |
|---|---|---|
| `-v` | `float,float,...` | Input values for a single inference. Count must match the input dimension in `-d` |

### Export

| Flag | Value | Description |
|---|---|---|
| `-q` | `float` \| `int8` \| `int4` | Quantization type for export. Default: `float`. INT8/INT4 are **Pro only** |

### Batch (Pro only)

| Flag | Value | Description |
|---|---|---|
| `-batch` | `path` | Path to a `.ini` batch config file |
| `--model` | `name` | Execute only one named model from the batch config |

---

## 6. Actions

### 6.1 `train`

Trains a new model from a CSV file and saves it to disk. If `-m` is provided, resumes training from an existing model instead of initialising from scratch.

**Required flags:** `-d`, `-act`, `-f`, `-o`  
**Optional flags:** `-e`, `-l`, `-m` (resume), `-test-file`, `--adam` (Pro)

```bash
hasaki -d 2,4,1 -act sigmoid,sigmoid -a train \
      -f examples/xor.csv -e 500 -l 0.1 -o xor.txt
```

**Training output columns:**

```
Epoch: 250 | Train Loss: 0.11985 | Train Acc: 1.0000 | Val Loss: 0.11985 | Val Acc: 1.0000
```

- **Train Loss** — mean loss over the training set
- **Train Acc** — fraction of training samples classified correctly (threshold 0.5 for binary outputs)
- **Val Loss / Val Acc** — same metrics over the validation set

Hasaki prints progress every 50 epochs and at the final epoch. The log also includes `Elapsed` and `ETA` so long runs can be tracked directly from the CLI.

**Training control notes**

- `--batch-size` is optional. If omitted or set to `0`, Hasaki picks a batch size automatically from the dataset size.
- `--seed` makes both initialization and shuffling reproducible. Use `-1` to keep non-deterministic behavior.
- `--patience` overrides early stopping directly. Use small values for fast iterations and larger values for long runs.
- `--lr-decay` reduces the learning rate after each epoch. Use values like `0.99` or `0.95` for long runs.
- Larger batch sizes usually improve throughput on medium and large datasets, but may change convergence behavior slightly.

---

### 6.2 `predict`

Runs a single forward pass on one input vector and prints the output.

**Required flags:** `-m`, `-v`  
`-d` and `-act` are optional if the model file already contains the architecture metadata.

```bash
hasaki -m xor.txt -a predict -v "1.0,0.0"
```

Output:

```
Input:  1, 0
Output: 0.987431
```

The number of values in `-v` must exactly match the input dimension in `-d`.

---

### 6.3 `validate`

Runs inference on every row of a CSV file and reports per-sample error plus aggregate statistics.

**Required flags:** `-m`, `-f`  
`-d` and `-act` are optional if the model file already contains the architecture metadata.

```bash
hasaki -m xor.txt -a validate -f examples/xor.csv
```

Output:

```
In: 0.000000 1.000000  | Expected: 1.000000  | Got: 0.987431  | Error: 0.012569
...
--------------------------------------------------------
Max error:  0.012569
Mean error: 0.008341
```

Error is the Euclidean distance between the expected and predicted output vectors.

---

### 6.4 `export`

Exports the trained model as a self-contained C header file with a `predict()` function.

**Required flags:** `-m`, `-o`  
`-d` and `-act` are optional if the model file already contains the architecture metadata.  
**Optional flags:** `-q` (default `float`)

```bash
hasaki -m xor.txt -a export -o xor.h -q float
```

The resulting header can be included directly in any C or C++ project. No other files are needed.

**Quantization options:**

| Value | Size | Notes |
|---|---|---|
| `float` | 4 bytes/weight | Default. Full precision. Works on all targets |
| `int8` | 1 byte/weight | 4× smaller. **Pro only** |
| `int4` | 0.5 bytes/weight | 8× smaller. **Pro only** |

---

### 6.5 `quantize_test`

Evaluates the model in float and quantized mode against a reference CSV, then reports the quantization error and a pass/fail result.

**Required flags:** `-m`, `-f`  
`-d` and `-act` are optional if the model file already contains the architecture metadata.  
**Optional flags:** `-q` (`int8` or `int4`, default `int8`)

```bash
hasaki -m mnist.txt -a quantize_test -f mnist.csv -q int8
```

Output includes float metrics, quantized metrics, a threshold, and `Result: PASSED` or `FAILED`.

**(Pro only)**

---

## 7. Activation functions

One activation is required per layer transition. For a network `-d 2,4,1`, two activations are needed: one for the hidden layer, one for the output layer.

| Name | Flag value | Typical use |
|---|---|---|
| Sigmoid | `sigmoid` | Binary classification output, hidden layers |
| Tanh | `tanh` | Hidden layers, especially with small datasets |
| ReLU | `relu` | Deep networks, image classification |
| Leaky ReLU | `leaky_relu` | Hidden layers where ReLU may die |
| Linear | `linear` | Regression output |
| Softmax | `softmax` | Multiclass classification output |

**Examples:**

```bash
# Binary classification
-act tanh,sigmoid

# Regression
-act sigmoid,linear

# Multiclass
-act relu,softmax
```

---

## 8. CSV format

Hasaki expects a headerless CSV with all inputs followed by all outputs on each row.

```
input_1,input_2,...,output_1,output_2,...
```

The number of inputs and outputs must match the first and last values in `-d`.

**XOR example** (`examples/xor.csv`):

```
0.0,0.0,0.0
0.0,1.0,1.0
1.0,0.0,1.0
1.0,1.0,0.0
```

**Regression example** (`examples/regression_1d.csv`):

```
0.0,0.0
0.1,0.01
0.2,0.04
...
1.0,1.0
```

**Multiclass example** (`examples/softmax_toy.csv`) — one-hot output:

```
0.8,0.1,0.0,1.0,0.0
0.1,0.8,0.0,0.0,1.0
0.1,0.1,1.0,0.0,0.0
```

---

## 9. Training behaviour

### Auto-split vs manual split

By default Hasaki splits the dataset 80/20 into training and validation sets. If the dataset has fewer than 10 samples, the full dataset is used for both training and validation to avoid producing a validation set of a single sample.

```
[Small dataset (4 samples < 10): using full data for training and validation]
  Train samples: 4
  Val samples:   4 (same as train)
```

To use a separate validation file instead of the auto-split:

```bash
hasaki -d 2,4,1 -act sigmoid,sigmoid -a train \
      -f train.csv -test-file val.csv -e 500 -l 0.1 -o model.txt
```

### Early stopping

Training stops early if the validation loss does not improve for a number of consecutive epochs. The patience scales with dataset size:

- **Small datasets** (< 10,000 samples): patience = 200 epochs
- **Large datasets** (≥ 10,000 samples): patience = 1,000 epochs

If you expose `--patience` in a workflow or wrapper, use it as an operational knob:

- `--patience 5`: fast iterations, Workbench runs, and CI checks where you want to stop as soon as the curve stalls.
- `--patience 50` or `--patience 100`: longer training runs before export, where short plateaus are acceptable.

The model saved to disk is the one at the epoch with the best validation loss, not the last epoch. Early stopping is evaluated every epoch internally, regardless of the display interval.

```
Early stopping at epoch 1850 (no improvement for 1000 epochs, min_delta=5e-05)
=== Training Complete === (Final Val Loss: 0.043210)
```

### Mini-batch training

Training uses mini-batch gradient descent. The batch size is controlled via `--batch-size`. If omitted or set to `0`, Hasaki selects a batch size automatically based on dataset size.

### Regularisation (Pro)

Two regularisation techniques are available in Pro to combat overfitting on large datasets:

**Dropout** randomly deactivates neurons during training, forcing the network to learn redundant representations. Applied to hidden layers only — automatically disabled during inference (`validate`, `predict`, `export`).

```bash
--dropout 0.3    # typical range: 0.2–0.5
```

**L2 regularisation** penalises large weights during gradient updates, discouraging memorisation.

```bash
--l2 0.0001    # typical range: 0.0001–0.001
```

Both can be combined:

```bash
hasaki -d 784,64,10 -act relu,softmax -a train \
      -f mnist_train.csv -e 50000 -l 0.001 --adam \
      --dropout 0.3 --l2 0.0001 -o mnist.txt
```

---

## 10. Resuming training

Pass `-m` together with `-a train` to continue training from a saved model instead of reinitialising weights randomly.

```bash
# Initial training
hasaki -d 2,4,1 -act sigmoid,sigmoid -a train \
      -f examples/xor.csv -e 500 -l 0.1 -o xor.txt

# Resume for 2000 more epochs with a lower learning rate
hasaki -d 2,4,1 -act sigmoid,sigmoid -a train \
      -f examples/xor.csv -e 2000 -l 0.01 \
      -m xor.txt -o xor.txt
```

The `-d` and `-act` flags must match the architecture of the saved model.

---

## 11. Recovering the best model after a manual stop

During training, Hasaki continuously saves the best model found so far to a temporary file named `<output>.best_tmp`. For example, if your output is `mnist.txt`, the temporary file is `mnist.txt.best_tmp`.

**When training completes normally** — whether by reaching the epoch limit or by early stopping — Hasaki saves the best model to the output file and automatically deletes the `.best_tmp` file. Nothing extra is needed.

**When training is interrupted manually** — by pressing Ctrl+C, clicking Stop in Workbench, or killing the process — the `.best_tmp` file is left on disk. The output file may be incomplete or absent. To recover the best model reached before the interruption:

**Linux / macOS:**
```bash
cp mnist.txt.best_tmp mnist.txt
```

**Windows (PowerShell):**
```powershell
Copy-Item mnist.txt.best_tmp mnist.txt
```

**Windows (Command Prompt):**
```cmd
copy mnist.txt.best_tmp mnist.txt
```

After recovering, you can validate, export, or resume training normally using `-m mnist.txt`.

> **Note:** If you stop training very early — before the first validation cycle completes — the `.best_tmp` file may not exist yet. In that case, no checkpoint is available and training must be restarted from scratch.

---

## 12. Exporting to C header

After training, export the model to a C header:

```bash
hasaki -m xor.txt -a export -o xor.h -q float
```

Drop `xor.h` into your embedded project and call `predict()`:

```c
#include "xor.h"

float input[2]  = { 1.0f, 0.0f };
float output[1];

predict(input, output);
// output[0] ≈ 0.987  →  XOR result for (1, 0)
```

The header is fully self-contained — no other Hasaki files are needed on the target. It compiles with any C99-compatible toolchain and makes no dynamic memory allocations.

---

## 13. Batch processing (Pro)

Batch processing allows running multiple train/export/validate operations from a single `.ini` configuration file. Each model is a named section.

```bash
# Run all models in the config
hasaki -batch models.ini

# Run only one model from the config
hasaki -batch models.ini --model xor_model
```

### Config file format

```ini
# Optional: path to the hasaki binary (default: ./hasaki)
hasaki_binary = ./hasaki

[xor_model]
dims         = 2,4,1
activations  = sigmoid,sigmoid
csv          = examples/xor.csv
output       = xor.h
action       = train_and_export
epochs       = 500
learning_rate = 0.1
quantization = float

[regression_model]
dims         = 1,4,1
activations  = sigmoid,linear
csv          = examples/regression_1d.csv
output       = regression.h
action       = train_and_export
epochs       = 500
learning_rate = 0.05
quantization = float
```

### Supported actions in batch

| Action | Description |
|---|---|
| `train` | Train and save model file |
| `export` | Export existing model to C header |
| `train_and_export` | Train, then export in one step |
| `validate` | Validate existing model against a CSV |

### Config parameters per model

| Key | Description |
|---|---|
| `dims` | Network dimensions (same format as `-d`) |
| `activations` | Activation functions (same format as `-act`) |
| `csv` | Path to data CSV |
| `model` | Path to existing model file (for `export` / `validate`) |
| `output` | Output file path |
| `action` | One of the supported batch actions |
| `epochs` | Number of training epochs |
| `learning_rate` | Learning rate |
| `quantization` | `float`, `int8`, or `int4` |

---

## 14. Complete examples

### XOR — binary classification

```bash
# Train
hasaki -d 2,4,1 -act sigmoid,sigmoid -a train \
      -f examples/xor.csv -e 500 -l 0.1 -o xor.txt

# Predict one sample
hasaki -m xor.txt -a predict -v "1.0,0.0"

# Validate against the full dataset
hasaki -m xor.txt -a validate -f examples/xor.csv

# Export to C header
hasaki -m xor.txt -a export -o xor.h -q float
```

### 1D Regression

```bash
hasaki -d 1,4,1 -act sigmoid,linear -a train \
      -f examples/regression_1d.csv -e 500 -l 0.05 -o regression.txt

hasaki -m regression.txt -a validate -f examples/regression_1d.csv
```

### Multiclass classification

```bash
hasaki -d 3,4,2 -act sigmoid,linear -a train \
      -f examples/softmax_toy.csv -e 500 -l 0.1 -o softmax_toy.txt

hasaki -m softmax_toy.txt -a validate -f examples/softmax_toy.csv
```

### MNIST on ESP32-C3 (Pro)

```bash
# Train with Adam, dropout and L2 regularisation
hasaki -d 784,128,10 -act relu,softmax -a train \
      -f mnist_train.csv -e 50000 -l 0.001 --adam \
      --dropout 0.3 --l2 0.0001 -o mnist.txt

# Test INT8 quantization before export
hasaki -m mnist.txt -a quantize_test -f mnist_test.csv -q int8

# Export as INT8 for tighter memory
hasaki -m mnist.txt -a export -o mnist_int8.h -q int8
```

Full project and firmware: [https://github.com/AlexRosito67/hasaki-mnist-esp32](https://github.com/AlexRosito67/hasaki-mnist-esp32)

---

## 15. Error reference

| Error message | Cause | Fix |
|---|---|---|
| `Error: No arguments provided` | Hasaki called with no flags | Run `hasaki -h` |
| `Error: Unknown option '-x'` | Unrecognised flag | Check flag spelling. Run `hasaki -h` |
| `Error: No dimensions provided` | `-d` missing | Add `-d input,hidden,output` |
| `Error: No training file provided` | `-f` missing for `train` | Add `-f path/to/data.csv` |
| `Error: No output file provided` | `-o` missing | Add `-o output_file` |
| `Error: No model file provided` | `-m` missing | Add `-m path/to/model.txt` |
| `Error: No input values provided` | `-v` missing for `predict` | Add `-v "val1,val2,..."` |
| `Error: Prediction input count mismatch` | `-v` count ≠ input dim in `-d` | Match value count to first number in `-d` |
| `Error: Number of activations must match number of layers` | `-act` count wrong | One activation per layer transition: `dims.size() - 1` |
| `Error: Unknown activation 'x'` | Typo in activation name | Use: `sigmoid`, `tanh`, `relu`, `leaky_relu`, `linear`, `softmax` |
| `Error: Unknown action 'x'` | Typo in `-a` value | Use: `train`, `predict`, `validate`, `export`, `quantize_test` |
| `Error: Free edition allows max 2 hidden layers` | Architecture too deep | Reduce hidden layers or upgrade to Pro |
| `Error: Free edition allows max 64 neurons per hidden layer` | Layer too wide | Reduce neuron count or upgrade to Pro |
| `Error: Free edition allows max 500 epochs` | `-e` too high | Reduce epochs or upgrade to Pro |
| `Error: Adam optimizer is a Pro feature` | `--adam` used in Free | Remove `--adam` or upgrade to Pro |
| `Error: INT8/INT4 quantization is a Pro feature` | `-q int8` or `-q int4` in Free | Use `-q float` or upgrade to Pro |
| `Error: Batch processing is a Pro feature` | `-batch` used in Free | Upgrade to Pro |
| `Error: Dropout is a Pro feature` | `--dropout` used in Free | Upgrade to Pro |
| `Error: L2 regularisation is a Pro feature` | `--l2` used in Free | Upgrade to Pro |
