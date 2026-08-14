# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A ComfyUI custom node pack that integrates LM Studio's local LLM/vision models into ComfyUI workflows via the official `lmstudio` Python SDK. It is installed as a subdirectory of `ComfyUI/custom_nodes/` and loaded by ComfyUI's node-discovery mechanism (there is no standalone entry point or build step — the package only makes sense running inside a ComfyUI process).

## Commands

There is no build, lint, or automated test suite in this repo. Verification is done by:

- **Manual/ad-hoc scripts against a running LM Studio server** — `lmstudio_diag.py`, `lmstudio_list_and_load.py`, `lmstudio_load_model.py`, `lmstudio_test.py`, `lmstudio_test_run.py` are throwaway scripts for probing the `lmstudio` SDK's actual runtime shape (attribute names, method availability differ across SDK versions). Run any of them directly with `python <script>.py` while LM Studio's local server is running. `lmstudio_test.py`/`lmstudio_test_run.py` expect you to replace the `MODEL = "MODELNAME"` placeholder with a real model key first.
- **`upgrade_lmstudio.py`** — helper that upgrades the installed `lmstudio` pip package and prints before/after versions; run with `python upgrade_lmstudio.py` when SDK/LM Studio version mismatches produce errors like missing `bosToken`/`jinjaPromptTemplate`.
- **Real verification requires ComfyUI** — to confirm a node change actually works, install/symlink this repo into a ComfyUI `custom_nodes/` checkout, start ComfyUI with LM Studio's local server running, and exercise the node from the ComfyUI graph UI.

Dependencies: `pip install -r requirements.txt` (`lmstudio`, `requests`). `__init__.py` also auto-installs `lmstudio` via `pip` at import time if it's missing from the environment ComfyUI is running in.

## Architecture

**Entry point (`__init__.py`)**: ensures the `lmstudio` SDK is installed, imports node classes from `expo_lmstudio_imagetotext.py` and `random_list_picker.py`, and populates `NODE_CLASS_MAPPINGS` / `NODE_DISPLAY_NAME_MAPPINGS` — the dict ComfyUI reads to register nodes and their UI labels. **When adding a new node class, register it in both this file's mappings AND the mappings at the bottom of `expo_lmstudio_imagetotext.py`** (that module keeps its own duplicate mapping using different dict keys — `ExpoLmstudioX` vs `"Expo Lmstudio X"` — historical, but both are live since `__init__.py`'s versions are what ComfyUI actually sees).

**Node classes** (all in `expo_lmstudio_imagetotext.py` unless noted) follow the standard ComfyUI custom-node contract: `INPUT_TYPES` classmethod, `RETURN_TYPES`/`RETURN_NAMES`, `FUNCTION` (method name ComfyUI calls), `CATEGORY`, and an `IS_CHANGED` classmethod that hashes all inputs (including image bytes via `np.array(image).tobytes()`) so ComfyUI knows when to re-run vs. use cached output.

- `ExpoLmstudioUnified` — text and/or image input, single combined node.
- `ExpoLmstudioImageToText` — vision-only image captioning; also carries the legacy HTTP fallback path (see below).
- `ExpoLmstudioTextGeneration` — text-only generation; also carries the legacy HTTP fallback path.
- `ExpoLmstudioStructuredOutput` — constrains generation to a JSON Schema via the SDK's `structured` config; extracts up to 6 top-level keys (from a newline-separated `output_keys` list) into individual `value_1`…`value_6` string outputs, tolerating models that wrap JSON in a code fence.
- `RandomListPicker` (`random_list_picker.py`) — standalone utility node, no LM Studio dependency; weighted random selection from a pasted list with template/prefix/suffix, exclude list, shuffle mode, and seeded reproducibility.

**Model interaction pattern** used by every LM Studio node's worker function:
1. `check_lmstudio_connection()` fails fast with a clear error if the SDK isn't installed or the server isn't reachable, so ComfyUI halts the pipeline instead of propagating an opaque error downstream.
2. `get_model_info_with_fallback(model_key, debug)` resolves an explicit `model_key`, or falls back to whatever model LM Studio currently has loaded (inspecting several possible SDK shapes defensively, since `lmstudio` SDK attribute names have changed across versions), or `None` to let the SDK pick its own default.
3. `client.llm.model(...)` acquires the model handle, optionally passing `ttl=unload_delay` for auto-unload-after-idle behavior; `auto_unload=="True"` and `unload_delay==0` triggers an immediate `.unload()` after the call instead.
4. Image inputs (ComfyUI `IMAGE` tensors) are converted `numpy → PIL → temp JPEG file → client.files.prepare_image(path)`, since the SDK takes image handles rather than raw arrays; temp files are cleaned up in a `finally` block.
5. The actual generation call is wrapped in `_collect_response()`, which streams via `respond_stream` when `strip_thinking=True` to filter `<think>...</think>` reasoning blocks — first trying the SDK-native `fragment.reasoning_type` field, and falling back to a regex (`_strip_think_tags`) when that field isn't populated by the installed SDK version. When `strip_thinking=False`, it just calls `model.respond(...)` directly.
6. Every worker runs inside a `ThreadPoolExecutor` with a manual `future.result(timeout=timeout_seconds)` so long/hung LM Studio calls surface as a clean timeout error string rather than blocking the ComfyUI queue indefinitely.

**Legacy HTTP fallback**: `ExpoLmstudioImageToText` and `ExpoLmstudioTextGeneration` accept optional legacy `model`/`ip_address`/`port` inputs for backward compatibility with old saved workflows that predate the SDK integration. If `ip_address` and `port` are both set, the node routes to a `_process_image_legacy_http`/`_generate_text_legacy_http` method that calls the OpenAI-compatible `/v1/chat/completions` endpoint directly via `requests` instead of the SDK. New nodes/features should target the SDK path only — the HTTP path exists solely to not break old workflow JSON files.

**Concurrency/safety details worth preserving when editing generation code**: the outer `ThreadPoolExecutor` timeout wraps the *entire* `do_the_work()` closure (connection + model load + generation), while a second inner `ThreadPoolExecutor` timeout in `_collect_response`'s caller wraps just the model response — both use `timeout_seconds`. Executors are shut down with `cancel_futures=True` and `wait=False` in `finally` blocks so a timed-out call doesn't block node return.
