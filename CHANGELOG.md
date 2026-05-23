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

## [3.0.2] - 2026-05-20

### Added
- Initial public release of Hasaki Free
- Command-line tool for training neural networks and exporting to C headers
- Cross-platform binaries: Linux, macOS, Windows
- Support for multiple activation functions: Sigmoid, Tanh, ReLU, Leaky ReLU, Linear, Softmax
- SGD optimizer with configurable learning rate
- Early stopping based on validation loss (20 epochs without improvement)
- Mini-batch gradient descent (batch size: 32)
- Training resumption from saved models
- Model inference via CLI
- Batch validation against datasets
- Example datasets: XOR (binary classification), 1D regression, multiclass classification
- Hasaki Pro edition with unlimited architecture, Adam optimizer, INT8/INT4 quantization, and batch processing

### Features
- **Architecture limits (Free)**: Max 2 hidden layers, 64 neurons per layer, 500 epochs
- **Quantization (Free)**: FP32 float only
- **Commercial use**: Free edition restricted to personal/educational use
- **Output**: Self-contained C header with `predict()` function, no dependencies, no heap allocation

### Known Limitations
- Free edition: max 2 hidden layers
- Free edition: max 64 neurons per layer
- Free edition: max 500 epochs
- Free edition: SGD optimizer only (Adam available in Pro)
- Free edition: FP32 quantization only (INT8/INT4 in Pro)
- Free edition: no batch processing (available in Pro)

---

*Built for microcontrollers at the bottom of the drawer — still useful, still available, still cheap.*
