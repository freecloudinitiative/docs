# Getting Started

This page explains how to preview and contribute to the documentation site locally.

## Prerequisites

- Python 3.8+
- `pip` package manager

## Local Setup & Preview

1. **Install required packages**:
   ```bash
   pip install -r requirements.txt
   ```

2. **Run live preview server**:
   ```bash
   mkdocs serve
   ```

3. **View in Browser**:
   Open [http://127.0.0.1:8000](http://127.0.0.1:8000) to see live-reloaded changes as you edit documentation files under `docs/`.

## Building Static Output

To compile the Markdown docs into static HTML/CSS/JS (`site/` folder):

```bash
mkdocs build
```
