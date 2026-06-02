# Changelog — Hasaki 刃先

All notable changes to this project will be documented in this file.
The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]
### Added
- (Next features go here)
### Changed
- (Breaking changes go here)
### Fixed
- (Bug fixes go here)

---

## [3.2.1] - 2026-06-01

### Added
- `--patience <n>` — override early stopping patience from CLI
- `--lr-decay <factor>` — multiply learning rate at end of each epoch; 1.0 disables
- `--batch-size <n>` — configurable mini-batch size; 0 = auto
- `--seed <n>` — reproducible training; fixes weight initialization and data shuffle; -1 = random
- Cross-entropy loss integrated into evaluate() for correct multiclass metrics with softmax

### Changed
- Early stopping patience now scales with dataset size: 200 epochs for small datasets, 1000 for large
- MIN_DELTA threshold added to early stopping — ignores insignificant improvements
- Serial training path optimized — memory reserved upfront, fewer temporary allocations
- OpenMP parallelism removed — overhead exceeded benefit for typical dataset sizes

### Fixed
- Free edition epoch default corrected (was 50000, now 500)
- Batch config `--act` flag corrected in generated commands

---

## [3.2.0] - 2026-05-23

### Added
- Dropout regularisation (Pro)
- L2 regularisation (Pro)
- He/Xavier/Softmax weight initialization by activation type
- Loss abstraction — CrossEntropy and MSE fully integrated in backprop
- `network.save()` returns bool with error handling
- ETA in real time during training (10-epoch sliding window)
- `HASAKI_BUILD_VERSION` injected from release workflow

### Changed
- Complete rebrand from Xyron to Hasaki 刃先
- Binary renamed: `hasaki_free_*` and `hasaki_pro_*`

### Fixed
- Small dataset split threshold — datasets < 10 samples use full data for train and validation
- `--act` flag corrected in CLI help examples
- README fabricated benchmarks removed

---

## [3.0.2] - 2026-05-20

### Added
- Initial public release of Hasaki Free
- Cross-platform binaries: Linux, macOS, Windows
- SGD optimizer with configurable learning rate
- Early stopping based on validation loss
- Mini-batch gradient descent
- Training resumption from saved models
- Example datasets: XOR, 1D regression, multiclass classification
- Hasaki Pro edition: unlimited architecture, Adam, INT8/INT4, batch processing

### Features
- **Architecture limits (Free)**: Max 2 hidden layers, 64 neurons per layer, 500 epochs
- **Quantization (Free)**: FP32 float only
- **Output**: Self-contained C header with `predict()` function, no dependencies, no heap allocation

---

*Built for microcontrollers at the bottom of the drawer — still useful, still available, still cheap.*
