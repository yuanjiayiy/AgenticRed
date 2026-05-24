<h1 align="center">
  <b>AgenticRed: Evolving Agentic Systems for Red-Teaming</b><br>
</h1>



# AgenticRed

AgenticRed is an automated pipeline that leverages LLMs’ in-context learning to iteratively design and refine red-teaming systems without human intervention, based on the performance metric of
each system.

**Last updated:** 2026-03-26

## What's New

This repository now includes:

- **Prompt memory mechanisms** for avoiding repeated failed/similar-successful attacks.
- **Expanded initial archive methods** (multiple strong hand-crafted baselines).
- **Additional benchmarks** beyond HarmBench/AdvBench.
- **Diversity Search mode** for novelty-driven evolution.

---

## Repository Structure

```text
.
├── server/           # Scripts to start attacker, classifier, and defender model servers
├── search.sh         # Main script to run the search process
├── eval.sh           # Main script to run the evaluation process
└── README.md
```

---

## Prerequisites

- If you want to host your model locally, you need at least 3 GPUs (e.g., 3× L40) or equivalent capacity
    - One for the attacker model server
    - One for the classifier / judge model server
    - One for the defender / target model server
- If you want to call API endpoints instead, we provide OpenRouter/OpenAI client API.
---

## 1. Start Model Servers

You can run local servers (recommended for experiments) or point to external APIs.

### 1.1 Local servers

Inside `server/`, you should have scripts or configs such as:

```bash
cd server/

bash attacker_server.sh          # Starts attacker model server
bash classifier_server.sh        # Starts classifier / guardrail model server
bash defender_server.sh          # Starts defender / target model server
```

Document the actual ports and endpoints so the rest of the pipeline can reference them.
Eg. http://127.0.0.1/8000/v1

### 1.2 Using external APIs (optional)

Instead of local servers, you can configure the system to call:

- OpenAI APIs
- OpenRouter APIs

Update your API keys:

```
export OPENAI_API_KEY=''
export GEMINI_API_KEY=''
export DEEPSEEK_API_KEY=''
export OPENROUTER_API_KEY=''
```

---

## 2. Run the Search Process

The search step iteratively designs new red-teaming systems. 

Typical usage:

```bash
cd _redteam
bash scripts/search.sh

e.g. bash search.sh --expr 1
```

Key arguments (adapt to your implementation):

- `--expr`: Experiment index corresponding to different configurations.
- `--config`: Path to a config file specifying:
    - Attacker, classifier, defender endpoints
    - Search hyperparameters (iterations, beam width, budgets, etc.)
    - Output directory for JSON logs: save_dir
    - Whether to enable OpenRouter

### 2.1 Benchmarks

Supported benchmarks in `search.py`:

- `harmbench`
- `advbench`
- `easyjailbreak`
- `teleai_safety`

Set benchmark in YAML (`benchmark: ...`) or CLI (`--benchmark ...`).

### 2.2 Archive Methods (Initial Population)

The initial archive can be seeded with additional methods via:

- `include_new_methods: true` (YAML), or
- `--include_new_methods` (CLI)

When enabled, the archive includes:

- Reflexion
- Adversarial Reasoning
- PAIR
- AutoDAN-Turbo
- ActorAttack
- X-Teaming
- EvoSynth

### 2.3 Prompt Memory

Two memory mechanisms are integrated into search:

- **FailedPromptMemory** (goal-aware):
  - Stores failed prompts per goal.
  - Skips exact repeats for the same goal.
  - Injects failed-history context into attacker prompting.

- **SucceedPromptMemory** (goal-agnostic, used in diversity mode):
  - Stores successful prompts globally across goals.
  - Skips new prompts with high similarity to existing successful prompts.
  - Similarity threshold is controlled by `succeed_memory_threshold` (default `0.6`).

### 2.4 Diversity Search (Novelty-Driven)

Enable novelty-driven evolution by setting:

- `diversity_search: true`
- `succeed_memory_threshold: 0.6` (or your preferred value)

When `diversity_search` is enabled:

- SucceedPromptMemory-based skipping is active.
- Evolutionary selection uses **diversity of SucceedPromptMemory** as fitness.
- Meta-agent prompting is steered toward discovering **novel attack prompts**.

Run predefined diversity-search experiment:

```bash
cd _redteam
bash scripts/search.sh --expr 9
```

Config used by `--expr 9`:

- `_redteam/configs/exp9_diversity_search.yaml`
---

## 3. Run the Evaluation Process

After search completes, evaluate the discovered attacks on a separate evaluation dataset and target model / benchmark.

```bash
cd _redteam
bash scripts/eval.sh --config configs/eval_easyjailbreak.yaml
```

Key arguments (adapt to your implementation):
- evaluator_model: Models used for evaluation
- benchmark: `harmbench`, `advbench`, `easyjailbreak`, `teleai_safety`

### 3.1 YAML-based Evaluation

`scripts/eval.sh` now accepts a YAML config and delegates parsing to `search.py`:

- `archive_path`: path to an existing search archive JSON
- `benchmark`: evaluation benchmark
- endpoints/models for attacker/defender/classifier/evaluator

`search.py` resolves `save_dir` and `expr_name` directly from `archive_path` in evaluate mode.
---

## Example Workflow

```bash
# 1. Start servers if you are hosting local servers
bash server/attacker_server.sh
bash server/defender_server.sh
bash server/classifirt_server.sh

# 2. Run search
bash _redteam/scripts/search.sh --expr 1

# 3. Run evaluation
bash _redteam/scripts/eval.sh --config _redteam/configs/eval_easyjailbreak.yaml
```


