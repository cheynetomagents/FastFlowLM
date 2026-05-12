# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

All build commands run from `src/`:

```bash
# Linux — configure + build
cmake --preset linux-default      # configures into src/build/
cd build && cmake --build . -j$(nproc)

# Install (system-wide, requires sudo on Linux)
sudo cmake --install .

# Windows (Developer Command Prompt)
cmake --preset windows-default
cmake --build build --preset windows-default
```

**Required CMake variables** (set in `CMakePresets.json`; pass manually if not using presets):
- `-DFLM_VERSION=<ver>` (e.g. `0.9.40`)
- `-DNPU_VERSION=<ver>` (e.g. `32.0.203.304`)

**Linux build dependencies:**
```bash
sudo apt install ninja libavformat-dev libavutil-dev libavcodec-dev \
  libswresample-dev libswscale-dev libxrt-dev uuid-dev libdrm-dev
```

XRT headers/libs default to `/opt/xilinx/xrt/{include,lib}`; override with `-DXRT_INCLUDE_DIR` / `-DXRT_LIB_DIR`.

## Running

```bash
flm run llama3.2:1b          # interactive CLI
flm serve llama3.2:1b        # OpenAI-compatible REST server (port 52525)
flm list                     # show available models
flm validate                 # verify NPU hardware/driver setup
flm pull <tag> --force       # re-download corrupted model
```

Model storage: `~/.config/flm/` on Linux, `%USERPROFILE%\Documents\flm\models\` on Windows.  
Override on Linux: `FLM_MODEL_PATH=<path>`.

## Testing

Tests live under `src/test/` with their own `CMakeLists.txt`. Each test is a standalone executable built via `add_npu_test()`. There is no top-level `ctest` target in the main build; tests are built and run independently per model under `src/test/<model>/`.

```bash
# Build a single test (example: llama3 model test)
cd src/test/llama3_npu
cmake -B build && cmake --build build
./build/test_llama3
```

## Architecture

```
src/src/main.cpp          → parses args → dispatches to Runner or Server
src/runner/runner.cpp     → interactive CLI loop, model switching, in-session commands
src/server/server.cpp     → HTTP server (OpenAI-compatible REST API)
src/common/AutoModel/     → AutoModel class: model dispatch, tokenizer setup, shared inference logic
src/common/AutoEmbeddingModel/ → same pattern for embedding models
src/common/modules/       → shared inference primitives (sampler, etc.)
src/common/tokenizer/     → tokenizer wrapper (tokenizers-cpp submodule)
src/common/<arch>_npu/    → per-architecture NPU kernel wrappers (llama, qwen3, gemma, phi4, etc.)
src/include/models/       → per-model header definitions (llama/, qwen3/, gemma/, etc.)
src/common/image*/        → image preprocessing (AVX512-accelerated)
src/common/audio*/        → audio preprocessing (AVX512-accelerated)
src/pull/                 → HuggingFace model download logic
```

**Data flow:** `main` → `Runner`/`Server` → `AutoModel::setup()` → `modeling_<arch>.cpp` → NPU kernel via XRT → token sampler → streamed output.

**Adding a new model family:**
1. Add NPU kernel wrappers under `src/common/<arch>_npu/`
2. Add model headers under `src/include/models/<arch>/`
3. Add a `modeling_<arch>.cpp` source in `src/common/`
4. Register in `AutoModel` dispatch and `src/model_list.json`
5. Add a test under `src/test/<arch>_npu/`

## Key Constraints

- C++20 required throughout. AVX512 flags are set per-file on image/audio sources — do not apply globally.
- NPU kernels are **proprietary precompiled binaries** in `src/lib/` and `src/xclbins/`. Do not attempt to modify or recompile them.
- `FLM_VERSION` and `NPU_VERSION` must always be passed to CMake; the build will hard-fail without them.
- Linux memlock must be `unlimited` for XRT NPU buffers; see `flm validate` output.
- `FASTFLOWLM_LINUX_LIMITED_MODELS` define gates whisper and embedding support on Linux.
