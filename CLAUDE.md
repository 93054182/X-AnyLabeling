# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**X-AnyLabeling** is a PyQt6-based auto-labeling tool for computer vision annotation tasks. It integrates AI models for automatic annotation of images and videos, supporting multiple annotation formats (COCO, VOC, YOLO, DOTA, MOT, etc.) and various computer vision tasks (detection, segmentation, pose estimation, OCR, etc.).

## Build and Development Commands

### Installation
```bash
# CPU development environment
pip install -e .[cpu,dev]

# GPU CUDA 12.x development environment
pip install -e .[gpu,dev]

# GPU CUDA 11.x development environment
pip install -e .[gpu-cu11,dev]
```

### Running the Application
```bash
# Launch GUI
xanylabeling

# Open specific file/folder
xanylabeling --filename /path/to/image.jpg

# Debug mode
xanylabeling --logger-level debug
```

### Code Quality
```bash
# Format code (Black, line length 79)
bash scripts/format_code.sh

# Run linting (Flake8)
flake8 .

# Run tests
pytest tests/
```

### Building Executables
```bash
# Windows CPU
bash scripts/build_executable.sh win-cpu

# Windows GPU
bash scripts/build_executable.sh win-gpu

# Linux/macOS variants available
```

## Architecture

### Directory Structure
- **`anylabeling/`** - Main application package
  - **`views/`** - PyQt6 UI components
    - **`labeling/`** - Core annotation widget (`label_widget.py`, ~2500 lines)
    - **`common/`** - Shared utilities (converter, device_manager, toaster)
  - **`services/`** - Business logic
    - **`auto_labeling/`** - AI model implementations
      - **`__base__/`** - Base classes for model types (YOLO, SAM, CLIP, etc.)
      - **`model.py`** - Abstract Model base class
      - **`model_manager.py`** - Model loading/unloading management
  - **`configs/auto_labeling/`** - YAML model configuration files

### Key Components

**Model System:**
- All models inherit from `Model` class in `model.py`
- Each model has a corresponding YAML config in `configs/auto_labeling/`
- Model types are registered in `services/auto_labeling/__init__.py`
- `ModelManager` handles model lifecycle, downloading, and execution threads

**Label Format Conversion:**
- `views/common/converter.py` - Bidirectional format conversion (YOLO, VOC, COCO, etc.)
- Native format is XLABEL (JSON)

**Configuration:**
- `config.py` - Application config management with nested key support
- User config stored in `~/.xanylabelingrc`
- Model configs use YAML with `type`, `name`, `display_name`, and model-specific fields

### Model Integration

**Adding a new model:**
1. Create model class inheriting from appropriate base class (e.g., `YOLO`, `SegmentAnythingModel`)
2. Add config YAML to `configs/auto_labeling/`
3. Register in `services/auto_labeling/__init__.py` in appropriate model lists
4. Add to `models.yaml` for built-in model listing

**Model metadata lists** (in `__init__.py`):
- `_CUSTOM_MODELS` - All available models
- `_AUTO_LABELING_CONF_MODELS` - Models with confidence threshold
- `_AUTO_LABELING_IOU_MODELS` - Models with IoU threshold
- `_AUTO_LABELING_MASK_FINENESS_MODELS` - Segmentation mask smoothness control
- `_AUTO_LABELING_PROMPT_MODELS` - Text/visual prompt support

### Code Style

- **Line length:** 79 characters
- **Docstrings:** Google-style with type hints mandatory
- **Formatter:** Black with project-specific exclusions
- **Linter:** Flake8 with configured ignores
- **Language:** All comments, docs, and messages in English (project is bilingual EN/CN)

### Testing

- Tests in `tests/` directory
- Unit tests use `unittest` framework
- Some tests mock PyQt6 modules to avoid GUI dependencies

### PyQt6 Notes

- The app uses PyQt6 (not PySide6 except for development)
- Main window: `MainWindow` → `LabelingWrapper` → `LabelingWidget`
- Models run in separate QThreads to avoid blocking UI
- Resources loaded via Qt's resource system (`:/` prefix)

### Model Storage

- Downloaded models stored in `~/xanylabeling_data/models/`
- Model URLs point to GitHub Releases
- ONNX format preferred for cross-platform compatibility
