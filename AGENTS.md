# AGENTS.md

Unsloth makes LLM fine-tuning and RL faster and lighter by patching Hugging Face
transformers / TRL / PEFT with hand-written Triton kernels and per-model forward
passes. The repo also ships the `unsloth` CLI and Unsloth Studio (a local web UI).
Before changing anything under `studio/`, read `studio/AGENTS.md` for its commands
and gotchas.

## Architecture

- `unsloth/models/loader.py` holds the public entry points. `FastLanguageModel.from_pretrained`
  dispatches on `model_type` with an if/elif chain: llama, mistral, gemma, gemma2, qwen2, qwen3,
  qwen3_moe. Every other type goes to `FastModel` (generic HF classes plus the unsloth_zoo compiler).
  Each hand-optimised family lives in `models/<family>.py` as `Fast<Family>Model(FastLlamaModel)`.
- Import chain: `unsloth/__init__.py` picks one of three paths.
  - On Apple Silicon with mlx installed (the default on a Mac dev box), it takes the MLX path,
    which never imports torch, Triton or `models/*.py`.
  - The GPU path is `_gpu_init.py`, which applies the patches. `import_fixes.py` holds the
    version-skew shims.
  - The CPU path is `_cpu_init.py`.
- `unsloth/kernels/` holds the Triton kernels. They run only on NVIDIA, AMD or Intel GPUs.
- `unsloth_zoo` is a separate package with device detection, the compiler, GGUF export,
  temporary patches and the MLX trainer. A bug whose traceback ends in `site-packages/unsloth_zoo`
  belongs upstream in unslothai/unsloth-zoo. Patch it there, not in site-packages.
- `unsloth_cli/` is the Typer `unsloth` command. Commands are registered in
  `unsloth_cli/__init__.py`, implemented in `unsloth_cli/commands/`, and tested in `unsloth_cli/tests/`.
- `install.sh` and `install.ps1` are the end-user installers.

## Commands (verified on macOS arm64, Python 3.12, no GPU)

```bash
uv venv -p 3.12 .venv && source .venv/bin/activate
uv pip install -e ".[huggingface]"          # torch, unsloth_zoo, transformers (+ mlx on a Mac)
python -m pytest -n 4                        # default = tests/security only (testpaths), ~1 min
python -m pytest -q tests/test_foo.py tests/test_bar.py   # targeted; batch files into ONE call
GLOO_SOCKET_IFNAME=lo0 python -m pytest -n 4 unsloth_cli/tests   # CLI suite, ~15 s
python scripts/run_ruff_format.py path/a.py path/b.py     # the formatter (in place)
ruff check unsloth unsloth_cli studio tests cli.py unsloth-cli.py   # the CI lint gate
python scripts/lint_workflow_triggers.py     # after touching .github/workflows/
```

- **Formatter setup.** `run_ruff_format.py` needs the patched `ruff-unsloth`. On a Mac, install it with:
  `uv pip install "ruff-unsloth @ https://github.com/unslothai/ruff/releases/download/v0.6.9-unsloth.1/ruff_unsloth-0.6.9.post1-py3-none-macosx_11_0_arm64.whl" "ruff==0.15.12"`.
  For other platforms, copy the wheel URL from `.pre-commit-config.yaml`.
- **Test startup cost.** `tests/conftest.py` imports a GPU-spoofing harness that costs about
  25 s per pytest invocation. That is why several files go into one command instead of one command each.
- **gloo on macOS.** Set `GLOO_SOCKET_IFNAME=lo0` for any test that starts torch.distributed gloo ranks.
  Without it, gloo binds an interface it cannot accept on, and the run fails with `uv_accept: invalid argument`
  (`test_doctor_gpus.py::test_allreduce_smoke_gloo_two_ranks`). The timeouts also stretch the CLI suite from
  about 15 s to about 70 s.
- **Expected failures on macOS.** These come from the environment, not from your change, so leave them alone:
  - 5 tests in `tests/security/test_release_desktop_appimage.py` exit 127 because they need Linux-only tools.
  - `unsloth_cli/tests/test_studio_update_uv_cache.py::test_a_cache_path_that_is_not_utf_8_is_recorded_and_read_back`
    fails because APFS rejects non-UTF-8 file names (Errno 92).
  - `test_start.py::test_studio_token_is_issued_for_the_owner_not_a_managed_account` sometimes fails under
    `-n 4` but passes when run alone.
  - For any other failure, rerun it with your change stashed before you treat it as yours.
- **Markers.** `addopts` deselects `gpu`, `slow`, `server` and `native_mxc`. If you name a
  GPU test file, pytest reports "no tests ran". Pass `-m gpu` only on a machine with a real GPU.

## Conventions

- **Kwarg spacing.** Keyword arguments and defaults are written with spaces: `f(x, dtype = None)`
  and `def g(a, b = 1)`. Stock `ruff format` rewrites these to `a=b` (a 120-line diff on
  `models/qwen3.py` alone). Format with `run_ruff_format.py` only.
- **Test idiom: contract tests.** Most tests use `read_text()` or `ast.parse()` on a source file
  and assert an invariant, without importing unsloth or needing CUDA. A regression test is named
  `tests/test_<topic>_<issue>.py`, and its one-line docstring starts with `unslothai/unsloth#NNNN:`.
  Model the next one on `tests/test_harmony_tool_call_eos_5162.py`. On a Mac this is the only way
  to cover `models/*.py`, because `import unsloth` takes the MLX path there.
- **User-facing output.** Messages are `print("Unsloth: ...")` or `warnings.warn`. Patches that may
  not apply are wrapped in `try/except Exception`. Version gates compare
  `transformers_version >= Version("x.y")`, with `Version` from `unsloth_zoo.utils`.
- **Comments.** Comments explain *why* and cite the issue or PR (`#NNNN`) that forced the code.
- **Python floor.** The floor is 3.9. A file that uses `X | Y` in annotations needs
  `from __future__ import annotations`, which `tests/test_python39_compatibility.py` enforces.
  A stdlib API newer than 3.9 needs a `# novermin` comment with the reason
  (`scripts/lint_backend_python_floor.py`).
- **License header.** New files copy the license header of their neighbours. That is
  `# SPDX-License-Identifier: AGPL-3.0-only` in `studio/`, `unsloth_cli/`, `scripts/`, `tests/`
  and newer core files. Older `unsloth/` files keep the Apache header.

## Guardrails

- **Vendored code:** `unsloth/_vendor/` and `studio/backend/vendor/`. These are byte-identical
  upstream copies pinned by sha256 manifests, and any edit fails the digest tests.
- **Data tables:** `models/mapper.py`, `chat_templates.py` and `ollama_template_mappers.py`.
  They are excluded from ruff. Add or change entries in the existing layout, and leave the rest
  of the file unformatted.
- **Generated or pinned files.** Regenerate them with their tool instead of editing by hand:
  - `studio/frontend/dist/`, built by `build.sh`.
  - `studio/frontend/src/features/chat/model-catalog-snapshot.ts`, from its refresh script.
  - `studio/*_prebuilt_pins.json`, which are sha256 trust anchors.
  - `scripts/*_baseline.json`.
  - `package-lock.json`.
- **Installer parity.** `install.sh` and `install.ps1` change together, or
  `tests/python/test_cross_platform_parity.py` fails. PowerShell helpers shared with
  `studio/setup.ps1` are synced with `python scripts/sync_shared_ps1_helpers.py`.
- **Workflows.** Files under `.github/workflows/` are themselves under test
  (`tests/studio/test_*`). Pin actions to a commit SHA, and leave out `pull_request_target`.
- **Never run `pytest tests/`.** It walks the GPU and checkpoint-download suites and takes
  16 min at `-n 16`. Name the files or directories you need instead.
