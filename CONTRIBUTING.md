# Contributing

Thank you for your interest in contributing to the Cloud Bulldozer Skills Marketplace.

## Plugin Versioning Policy

All plugins use [semantic versioning](https://semver.org/):

- **PATCH** (0.0.x): Bug fixes, typo corrections, minor improvements
- **MINOR** (0.x.0): New commands, skills, hooks, or features
- **MAJOR** (x.0.0): Breaking changes to existing skills

### When to bump versions

If your PR modifies plugin code (skills, plugin.json), you **must** bump the version in `<plugin>/.claude-plugin/plugin.json`. Documentation-only changes (README.md files) do not require version bumps.

## Adding a Skill

A skill is a `SKILL.md` file inside a plugin. At minimum:

```
<plugin>/skills/<skill-name>/
├── SKILL.md             # Required — skill definition
├── docs/                # Optional — reference guides, patterns
├── scripts/             # Optional — utility scripts
└── assets/              # Optional — templates, config files
```

### Writing SKILL.md

Every `SKILL.md` has YAML frontmatter followed by markdown instructions:

```yaml
---
name: my-skill-name
description: >-
  One-paragraph description of what this skill does and when to use it.
  This text drives auto-triggering — include keywords users would say.
user-invocable: true
disable-model-invocation: false
allowed-tools: Read Grep Glob Bash Write
argument-hint: "[file-or-directory]"
---
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Slash command name. Kebab-case, max 64 chars. |
| `description` | Yes | When to use this skill. 150-250 chars recommended. |
| `user-invocable` | No | Show in the `/` menu. Default: `true`. |
| `disable-model-invocation` | No | Prevent auto-triggering. Default: `false`. |
| `allowed-tools` | No | Space-separated tool list: `Read Grep Glob Bash Edit Write`. |
| `argument-hint` | No | Autocomplete hint: `"[file]"`, `"<env> [--flag]"`. |

Write clear, specific instructions in the markdown body. Numbered steps work better than unstructured prose.

## Adding a Plugin

A plugin groups related skills under one installable unit:

```
<plugin-name>/
├── .claude-plugin/
│   └── plugin.json        # Required — plugin metadata
└── skills/
    └── <skill-name>/
        └── SKILL.md
```

### plugin.json

```json
{
  "name": "<plugin-name>",
  "version": "1.0.0",
  "description": "What this plugin provides",
  "author": {
    "name": "Cloud Bulldozer Team"
  }
}
```

- `name` must match the directory name (kebab-case)
- `version` follows [semver](https://semver.org)

### Register in marketplace.json

Add an entry to `.claude-plugin/marketplace.json`:

```json
{
  "name": "<plugin-name>",
  "version": "1.0.0",
  "source": "./<plugin-name>",
  "description": "What this plugin provides",
  "license": "Apache-2.0"
}
```

### Add Cursor symlinks

For each skill, create a flat entry in `skills/`:

```bash
mkdir -p skills/<plugin>-<skill>
ln -s ../../<plugin>/skills/<skill>/SKILL.md skills/<plugin>-<skill>/SKILL.md
```

## Development Workflow

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Bump the version in `plugin.json` if modifying plugin code
5. Run `make update` to regenerate documentation
6. Run skill evals to verify nothing broke (see [Evaluating Skills](#evaluating-skills) below)
7. Submit a PR

### Testing locally

Point Claude Code to your local fork:

```bash
/plugin marketplace add <your-fork>
/plugin install <plugin>@<your-marketplace-name>
```

## Evaluating Skills

Every skill has an eval suite that measures how well the skill performs against real-world prompts. Before submitting a PR that modifies a skill, run its evals to confirm your changes don't regress quality.

The eval framework uses the [skill-creator](https://github.com/anthropics/claude-code-plugins/tree/main/skill-creator) plugin. Install it if you haven't:

```bash
/plugin install skill-creator
```

### How it works

Each skill has an `evals/evals.json` file defining test cases. Each test case has:

- **prompt** — a realistic user request
- **expected_output** — what a good response looks like (human-readable)
- **expectations** — a list of specific, verifiable assertions

The eval runner executes each prompt **with the skill** loaded and **without it** (baseline), then grades both against the assertions. This produces a `benchmark.json` showing the pass rate delta — the value the skill adds.

### Workspace layout

Eval results live in an `evals/workspace/` directory inside the skill folder:

```
<plugin>/skills/<skill-name>/
├── SKILL.md
├── evals/
│   ├── evals.json                            # Test case definitions
│   └── workspace/
│       └── iteration-1/
│           ├── benchmark.json                # Aggregate results
│           ├── feedback.json                 # Your qualitative review notes
│           ├── kube-burner-config/           # One directory per eval
│           │   ├── eval_metadata.json        # Prompt + assertions for this eval
│           │   ├── with_skill/
│           │   │   ├── outputs/              # Files the skill produced
│           │   │   ├── grading.json          # Per-assertion pass/fail with evidence
│           │   │   └── timing.json           # Token count and wall clock time
│           │   └── without_skill/
│           │       ├── outputs/
│           │       ├── grading.json
│           │       └── timing.json
│           ├── k8s-netperf-config/
│           │   └── ...
│           └── troubleshooting-debug/
│               └── ...
```

Each iteration directory is self-contained. When you iterate on a skill, create `iteration-2/`, `iteration-3/`, etc. — previous iterations are preserved for comparison.

### Running evals on an existing skill

Use the `skill-creator` skill in Claude Code:

```
/skill-creator Run evals for the orion-regression-analysis skill
```

Or describe what you want more specifically:

```
/skill-creator I updated the kube-burner section in orion-regression-analysis.
               Run the evals to check for regressions.
```

The skill-creator will:

1. Read `evals/evals.json` from the skill directory
2. Spawn subagent runs (with-skill and without-skill) for each test case
3. Grade each run against the assertions
4. Produce `benchmark.json` with aggregate stats
5. Launch a browser-based viewer for qualitative review

#### What to look for

- **Pass rate** — with-skill should be significantly higher than without-skill. A drop from a previous iteration means your change introduced a regression.
- **Assertion failures** — check `grading.json` for the `evidence` field on failed assertions. It explains exactly what went wrong.
- **Timing** — with-skill runs use more tokens (the skill body is in context). That's expected. Watch for large jumps that suggest the skill is causing the model to do unnecessary work.

### Adding evals for a new skill

When you create a new skill, add an `evals/evals.json` in the skill directory:

```json
{
  "skill_name": "my-new-skill",
  "evals": [
    {
      "id": 1,
      "prompt": "A realistic user request that exercises the skill",
      "expected_output": "Human-readable description of what a good response looks like",
      "files": [],
      "expectations": [
        "Specific, verifiable assertion about the output",
        "Another assertion checking a different aspect"
      ]
    }
  ]
}
```

#### Writing good test cases

- **Cover the major use cases.** 2-4 test cases per skill is a good starting point. Each should exercise a different capability.
- **Use realistic prompts.** Write the prompt exactly as a real user would type it — informal language, incomplete information, the way people actually ask for help.
- **Make assertions specific and verifiable.** Bad: "Output is correct." Good: "API server CPU metric uses metricName: containerCPU with labels.namespace.keyword: openshift-kube-apiserver."
- **Test what the skill adds, not general knowledge.** Focus assertions on domain-specific details that Claude wouldn't know without the skill. Assertions that pass without the skill don't prove skill value.

#### Example: orion-regression-analysis evals

The orion skill has 3 eval cases covering its main use cases:

| Eval | Tests | Key assertions |
|------|-------|----------------|
| kube-burner-config | Config generation for kube-burner benchmarks | Correct metricNames, nested `agg:` structure, threshold/direction fields |
| k8s-netperf-config | Config generation for k8s-netperf benchmarks | `metadata.platform` prefix, no aggregation field, quoted boolean strings |
| troubleshooting-debug | ES connectivity debugging guidance | References `discover-es-data.py`, correct `--es-server` CLI flags |

Without the skill, Claude scores ~32% on these assertions (it guesses wrong metricNames, invents fields, uses incorrect CLI flags). With the skill, it scores 100%.

### Interpreting results

#### grading.json

Each run directory contains a `grading.json` with per-assertion results:

```json
{
  "expectations": [
    {
      "text": "Uses metricName: containerCPU",
      "passed": true,
      "evidence": "Found containerCPU in metrics section line 15"
    },
    {
      "text": "Has threshold and direction fields",
      "passed": false,
      "evidence": "No threshold field on any metric. direction: 0 instead of 1."
    }
  ],
  "summary": {"passed": 1, "failed": 1, "total": 2, "pass_rate": 0.5}
}
```

The `evidence` field is the most useful part — it tells you exactly what the model produced vs. what was expected.

#### benchmark.json

The aggregate file at the iteration root shows the overall picture:

```json
{
  "run_summary": {
    "with_skill": {
      "pass_rate": {"mean": 1.0, "stddev": 0.0}
    },
    "without_skill": {
      "pass_rate": {"mean": 0.32, "stddev": 0.05}
    },
    "delta": {
      "pass_rate": "+0.68"
    }
  }
}
```

A healthy skill shows a large positive delta. If the delta shrinks after your changes, investigate which assertions started failing.

### Pre-PR checklist

Before submitting a PR that modifies a skill:

1. Run the skill's evals: `/skill-creator Run evals for <skill-name>`
2. Confirm with-skill pass rate hasn't dropped from the last iteration
3. Review any new assertion failures in `grading.json`
4. If you added new capabilities to the skill, add new eval cases to cover them
5. Bump the version in `plugin.json`
6. Run `make update`

## Auto-generated Files

The following files are auto-generated on merge and should **not** be edited manually:

- `PLUGINS.md`
- `docs/data.json`

These are regenerated by `make update`, which runs:

- `sync/sync_marketplace_versions.py`
- `sync/generate_plugin_docs.py`
- `sync/build-website.py`

## Naming Guidelines

- **Kebab-case**: `kube-burner`, `incident-response`
- **Name by domain**, not team: `benchmarking` not `perf-team-tools`
- **Keep it short**: 1-2 words

## Getting Help

- Check existing plugins for examples
- Review the [Claude Code plugin documentation](https://docs.anthropic.com/en/docs/claude-code/plugins)
- Open an issue for discussion
