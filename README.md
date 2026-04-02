<p align="center">
  <img src="https://img.shields.io/badge/Version-1.1.0-brightgreen.svg" alt="Version">
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-Apache_2.0-blue.svg" alt="License"></a>
  <img src="https://img.shields.io/badge/Claude_Code-Plugin-purple.svg" alt="Claude Code Plugin">
  <img src="https://img.shields.io/badge/Agents-18-orange.svg" alt="18 Agents">
  <img src="https://img.shields.io/badge/Skills-10-green.svg" alt="10 Skills">
  <a href="https://github.com/MJbae/awesome-novel-studio/stargazers"><img src="https://img.shields.io/github/stars/MJbae/awesome-novel-studio?style=social" alt="GitHub Stars"></a>
</p>

# Awesome Novel Studio

**AI Web Novel Creation Harness** — A Claude Code Plugin

**English** | [한국어](README_KO.md)

An end-to-end web novel production system built on Claude Code. Combines 18 specialist agents and 10 skills to automate the full pipeline: **propose → design → create → polish → rewrite**.

> **Production-Proven** — A web novel written with this workflow secured a publishing deal.
> Daily views 2,500+ / Likes 1,000+ / Subscribers 300+

---

## Installation

### Via Marketplace

#### Add the marketplace
```shell
/plugin marketplace add MJbae/awesome-novel-studio
```

#### Install the plugin
```shell
/plugin install novel-studio@awesome-ai-studio
```

#### Restart session to activate
```shell
/exit
```

---

## Quick Start

```bash
# 1. Generate 3 novel proposals, pick one
/propose

# 2. Big design (bootstrap + character sheet + plot-hook guide)
/design-big

# 3. Small design (25-episode detailed design)
/design-small

# 4. Write episodes
/create

# 5. Polish
/polish
```

## Pipeline

```
Propose ────── Design ─────────────── Create ──── Polish ──── Publish
/propose        /design-big                /create   /polish
                /design-small
                     ↕ On design changes
                  /rewrite → /polish
```

---

## The Three Walls of Long-Form AI Fiction

Writing a single 5,000-character episode with AI is easy. Scaling that to 300 episodes and 1.5 million characters is where things break.

| The Wall | Symptom | Awesome Novel Studio's Solution |
|----------|---------|------------------------|
| **Character depth collapse** | Distinct personalities, speech patterns, and motivations blur into sameness as episodes accumulate. | **Character sheet + voice table** — Define each character's tone, sentence endings, and non-verbal palette at design time. Automatically verified every episode via the `VOICE`, `TITLE`, and `ALIVE` polish axes. |
| **Story coherence breakdown** | Cause-and-effect chains and foreshadowing unravel across a sprawling world. | **Plot-hook guide + continuity-bridge** — Decompose the full narrative arc into 25-episode beat structures. Before each episode, `continuity-bridge` collects timeline, foreshadowing, and character state from the previous 2 episodes and feeds it to the creation agent. |
| **Number and setting inconsistency** | Currency values, historical dates, character ages — the hard facts that anchor the story start contradicting each other. | **guard_rails + LOGIC axis** — Define absolute rules in `novel-config.md`. The `LOGIC` polish axis cross-checks numbers and timeline against the previous 2 episodes. |

---

## Commands

| Command | Description | Notes |
|---------|-------------|-------|
| `/propose` | Generate 3 proposals from genre + Korean platform + concept | Starting point for a new novel |
| `/design-big` | Full novel design (bootstrap, characters, plot) | Includes auto-research |
| `/design-small` | 25-episode detailed design | Requires big design first |
| `/design` | Design router (big + small combined) | Use when scope is unclear |
| `/bootstrap` | Generate bootstrap document only | World-building and concept only |
| `/character` | Generate character sheet only | Character design only |
| `/plot-hook` | Generate plot-hook guide only | Narrative structure only |
| `/create` | Write episodes sequentially | Based on design documents |
| `/polish` (= `/lint`) | 16-axis polish (6 agents in parallel) | Auto-sequential progression |
| `/rewrite` (= `/revise`) | Rewrite episodes after design changes | Auto-calculates impact scope |

### Scope Targeting

All create/polish/rewrite commands accept episode range arguments:

```bash
/create EP051          # Start from EP051
/create EP001-EP010    # EP001 through EP010
/polish start          # From EP001
/rewrite project-name EP001-EP010
```

## Workflow

```
/propose                Generate 3 proposals, select 1
       ↓
/design-big             Bootstrap + character sheet + plot-hook guide
       ↓                → novel-config.md auto-generated
/design-small           25-episode detailed design (optional, recommended)
       ↓
/create                 Episode creation (auto-sequential)
       ↓
/polish                 16-axis polish (auto-sequential)
       ↓
[On design changes] /rewrite → /polish   Rewrite then re-polish
```

## Directory Structure

Running the pipeline generates this project structure:

```
{project-name}/
├── novel-config.md              # Project config (central file)
├── design/                      # Design documents
│   ├── {name}_bootstrap.md      # World-building, concept, platform strategy
│   ├── {name}_character.md      # Character profiles, relationships, dialogue DNA
│   └── {name}_plot-hook.md      # 3-act structure, 25-ep beats, hook strategy
├── episode/                     # Episode manuscripts
│   ├── ep001.md
│   └── ...
├── revision/                    # Working files
│   ├── fix_plan.md              # Polish progress tracker
│   └── learnings.md             # Patterns discovered during polish
└── _workspace/                  # Temporary workspace
    └── 00_research/             # Auto-research results
```

## novel-config.md

The central configuration file for the entire pipeline. Auto-generated by `/design-big`, then reviewed and edited by the user.

```yaml
project:
  name: "My Novel"
  target_platform: "문피아"
  target_genre: "regression+specialist"
  episode_dir: "episode/"
  work_dir: "revision/"
  design_dir: "design/"

design_documents:
  bootstrap: "design/my-novel_bootstrap.md"
  character_core: "design/my-novel_character.md"
  plot_guide: "design/my-novel_plot-hook.md"

ep_range_table:
  - range: "EP001-EP025"
    label: "Act 1: Origin"
    plot_guide: "design/my-novel_plot-hook.md"

guard_rails:
  - "The protagonist's regression ability only allows recalling past information"

custom_axes:
  EXPERTISE: "Domain knowledge should be hinted at through dialogue, never explained"
```

- **ep_range_table**: Episode ranges auto-extracted from the plot guide
- **guard_rails**: Absolute rules enforced at every creation and polish step
- **custom_axes**: Project-specific additional polish criteria

## 16-Axis Polish System

| Axis | Name | Description |
|------|------|-------------|
| 1 | BANNED | Banned expressions (time-skip cliches, meta-commentary) |
| 2 | VOICE | Character dialogue consistency (checked against voice table) |
| 3 | TITLE | Forms of address (speaker, listener, context) |
| 4 | SILENCE | Silence pattern overuse (max 4 per episode) |
| 5 | TRANS | Translationese, AI tone, semantic literal translation detection |
| 6 | SCENE | Scene structure (beats, conflict, resolution) |
| 7 | LOGIC | Narrative logic, timeline, numerical consistency |
| 8 | SUMMARY | Scene-level justification of existence |
| 9 | UNIFORM | Cross-episode consistency |
| 10 | HOOK | Hook intensity (opening, midpoint, cliffhanger) |
| 11 | OPENING | Opening hook within 200 characters |
| 12 | MOBILE | Mobile readability (paragraph length, dialogue ratio) |
| A1 | ALIVE | Echo dialogue resolution |
| A2 | ALIVE | Silence-to-nonverbal replacement |
| A3 | ALIVE | Character vitality at tension points |
| A4 | ALIVE | Emotional distance management |

## Agent Architecture

### Design Agents (5)
`concept-builder` · `character-architect` · `plot-hook-engineer` · `proposal-generator` · `domain-researcher`

### Creation Agents (4)
`episode-architect` · `episode-creator` · `continuity-bridge` · `quality-verifier`

### Polish Agents (6)
`rule-checker` · `story-analyst` · `platform-optimizer` · `alive-enhancer` · `revision-executor` · `revision-reviewer`

### Rewrite Agents (4)
`revision-analyst` · `character-sculptor` · `episode-rewriter` · `quality-verifier`

> `quality-verifier` is shared between Creation and Rewrite phases (18 unique agents total).

## Acknowledgments

Special thanks to [Minho Hwang (revfactory)](https://github.com/revfactory) — his [Harness](https://github.com/revfactory/harness) plugin made it easy to bootstrap the initial harness architecture. The insights he shares on LinkedIn and other channels have been a constant source of inspiration.

## License

[Apache 2.0](LICENSE)
