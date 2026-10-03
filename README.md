# HarmonyOS Code Agent Benchmark Tool

A Code Agent benchmarking tool for HarmonyOS projects, covering three complex task types: **bug fixing / feature development / HarmonyOS migration**.

Distributed as **compiled executables (exe)**. **No Python, Node.js, or other runtime dependencies need to be installed**; extract and run on Windows.

---

## 1. Deliverables

After extraction, the directory structure is:

```
harmony-bench/
├── bin/                    # All executables (required)
│   └── harmony_bench.exe   # Unified entry point; it invokes the other executables automatically
├── configs/                # Built-in configuration files
│   ├── harmonization.yaml  # Migration pipeline configuration
│   ├── pipeline.yaml       # Bug-fixing pipeline configuration
│   └── dataset.yaml
└── README.md
```

> **Note:** All executables in `bin/` work together. `harmony_bench.exe` invokes other executables in the same directory (such as `harmonization_main.exe`, `stage*.exe`, and `run_eval.exe`). **Do not move, rename, or delete any executable individually**; keep the entire `bin/` directory intact.

---

## 2. Requirements

- Windows 10 / 11 (64-bit)
- Running an agent (`run` / `batch`) requires an LLM API key; see [Section 6](#6-api-key-configuration).
- Generating tasks (`generate`) requires project data that you provide; see [Section 5](#5-data).

---

## 3. Quick Check

Open the extracted directory and verify that the tool runs:

```powershell
bin\harmony_bench.exe --help

# dry-run: print the commands that would run without executing them (checks the environment and arguments)
bin\harmony_bench.exe generate migration `
  --android D:\data\android --harmonyos D:\data\harmony `
  --config configs\harmonization.yaml --dry-run
```

All subcommands support `--dry-run`.

---

## 4. Workflow

The unified entry point is `bin\harmony_bench.exe`. Run all commands below from the root of the extracted directory so relative paths such as `configs\...` resolve correctly.

### 1. Generate tasks

Use one of the three `generate` subcommands for each task type.

**Migration tasks** (Android → HarmonyOS):

```powershell
bin\harmony_bench.exe generate migration `
  --android D:\data\pair1_android `
  --harmonyos D:\data\pair1_harmony `
  --config configs\harmonization.yaml `
  --api-mapping D:\data\api_mapping.json `
  --output-dir outputs\migration
```

| Argument | Description |
|------|------|
| `--android` / `--harmonyos` | Paths to the Android / HarmonyOS projects that form a pair |
| `--config` | Migration pipeline configuration (built into `configs\`; use this path) |
| `--api-mapping` | HomeTran API mapping library JSON file (provide your own) |
| `--output-dir` | Output directory for task packages |

**Feature development tasks:**

```powershell
bin\harmony_bench.exe generate feature `
  --project D:\data\harmony_project `
  --feature-types route_navigation,network_capability `
  --output D:\data\feature_tasks
```

| Argument | Description |
|------|------|
| `--feature-types` | Comma-separated feature types (all types by default) |
| `--max-tasks` | Maximum number of tasks |
| `--difficulty` | Filter by difficulty |

**Bug-fixing tasks:**

```powershell
bin\harmony_bench.exe generate defect `
  --config configs\pipeline.yaml `
  --project D:\data\harmony_project `
  --project-name sample_clock `
  --output D:\data\outputs
```

| Argument | Description |
|------|------|
| `--config` | Bug-fixing pipeline configuration (built into `configs\`) |
| `--project-name` | Project name to use in reports |
| `--output` | Root output directory |

### 2. Evaluate (prepare → run → eval)

After generating tasks, evaluate them in three steps. Migration tasks are used as an example:

**Step 1: prepare** — Set up the workspace:

```powershell
bin\harmony_bench.exe prepare `
  --task-dir outputs\migration\task_packages\<task_id> `
  --output-dir task_runs
```

**Step 2: run** — Run the agent (requires an API key):

```powershell
bin\harmony_bench.exe run `
  --task-dir task_runs\<task_id> `
  --model deepseek-v4 --seed 42 --prompt migration
```

| Argument | Description |
|------|------|
| `--model` | `gpt5-codex` / `qwen3-coder` / `deepseek-v4` / `claude-sonnet5` |
| `--prompt` | Task type: `defect` / `feature` / `migration` |
| `--seed` | Random seed (use 42/43/44 for multiple runs) |
| `--step-limit` / `--cost-limit` | Maximum steps / cost |

**Step 3: eval** — Evaluate the run:

```powershell
bin\harmony_bench.exe eval `
  --task-run-dir task_runs\<task_id> --model deepseek-v4 --seed 42
```

This produces `eval_result.json` (resolved status, resolution rate, individual checks, and trajectory information).

### 3. Run the full workflow in batch

One command performs prepare + run + eval:

```powershell
# Migration tasks (seeds are fixed at 42/43/44)
bin\harmony_bench.exe batch --task-type migration --models deepseek-v4 gpt5-codex

# Bug-fixing / feature tasks
bin\harmony_bench.exe batch `
  --task-type feature --models deepseek-v4 --seeds 42 43 44 `
  --pilot-tasks D:\data\pilot_tasks.jsonl --workers 4
```

> Batch mode requires a task manifest: migration tasks use `bin\curated_tasks.jsonl`; for bug-fixing and feature tasks, specify one with `--pilot-tasks`. Generate tasks with `generate`, then prepare the manifest yourself (one JSON object per line, with a `task_path` field pointing to the task package directory).

---

## 5. Data

- **Project data is not included in the tool:** Android/HarmonyOS project pairs, output directories, and `api_mapping.json` are your own data; pass their paths as arguments.
- **Task manifest:** Required for batch mode; see the previous section for its format.
- The tool only includes `configs\` (pipeline configuration files) and compiled logic; it does not include project data.

---

## 6. API Key Configuration

Running an agent (`run` / `batch`) requires an LLM key, supplied through an environment variable (PowerShell):

```powershell
$env:CHATANYWHERE_API_KEY = "sk-xxx"
```

In cmd, use:

```cmd
set CHATANYWHERE_API_KEY=sk-xxx
```

Then run `run` or `batch`.

---

## 7. FAQ

- **Can I keep only `harmony_bench.exe` from `bin\`?** No. It is the unified entry point and invokes other executables in the same directory as needed. Removing them will disable the corresponding functionality.
- **Seeing `Project path not found`?** The path passed to `--android`, `--harmonyos`, or `--project` does not exist. Check for typos in forward slashes or relative paths.
- **Does `run` keep failing?** First confirm that `CHATANYWHERE_API_KEY` is set and valid.
- **Does a migration task produce no results?** Confirm that `--api-mapping` points to the correct mapping JSON file.

---

## 8. Distribution

This tool is distributed as compiled binaries and **does not include Python or Node.js source code**. Recipients can use it directly without setting up a runtime environment.
