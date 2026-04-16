# Meta-Cognitive Skill System

A three-layer skill generation system that enables systematic methodology development and skill creation across domains.

## Overview

This project implements a meta-cognitive approach to skill generation, featuring three distinct layers:

| Layer | Name | Description | Location |
|-------|------|-------------|----------|
| **L1** | Meta-Meta Skills | Top-level cognitive principles and methodology generation guides | `meta-skills/` |
| **L2** | Meta Skills | Domain-specific methodologies for generating L3 skills | `examples/web-frontend-skill-creator/` |
| **L3** | Skills | Concrete, scenario-specific execution skills | `examples/web-frontend-skills/` |

## Architecture

```
skill-master/
├── SKILL.md                           # L1 Navigation (meta-cognitive-skill-system)
├── meta-skills/                       # L1: Meta-Meta Skills (6 static files)
│   ├── epistemology.md                # mm-001: Epistemology Framework
│   ├── decomposition.md                 # mm-002: Decomposition & Abstraction
│   ├── causal-thinking.md             # mm-003: Causal & Systems Thinking
│   ├── meta-cognition.md              # mm-004: Meta-Cognition & Self-Reflection
│   ├── value-priority.md             # mm-005: Value & Priority Framework
│   └── interaction.md                 # mm-006: Interaction & Communication Model
├── examples/
│   ├── web-frontend-skill-creator/    # L2: Web Frontend Methodology (generated from L1)
│   │   └── SKILL.md
│   └── web-frontend-skills/           # L3: Concrete frontend skills (generated from L2)
│       ├── form-handling/
│       ├── state-management-implementation/
│       ├── routing-and-navigation/
│       └── ... (20+ more skills)
└── CLAUDE.md                          # Project guidelines
```

## Layer Responsibilities

### L1: Meta-Meta Skills (Static)

L1 skills are **never invoked directly for daily tasks**. They are only activated when generating or revising L2 methodologies.

| Skill | Purpose |
|-------|---------|
| **Epistemology Framework** | Defines "how to judge right from wrong" — establishes cognitive standards |
| **Decomposition & Abstraction** | Defines "how to decompose problems" — establishes structural views |
| **Causal & Systems Thinking** | Defines "how to understand causality" — establishes dynamic models |
| **Meta-Cognition & Self-Reflection** | Defines "how to examine oneself" — establishes quality checkpoints |
| **Value & Priority Framework** | Defines "what matters more" — establishes decision ranking criteria |
| **Interaction & Communication Model** | Defines "how to interact with users" — ensures methodology's user interface quality |

### L2: Meta Skills (Dynamic)

L2 skills are **domain-specific methodologies** that answer "how to think and act in this domain". They can only be used to **generate L3 skills**, not to directly solve problems.

Example: [web-frontend-methodology](examples/web-frontend-skill-creator/SKILL.md)

### L3: Skills (Dynamic)

L3 skills are **concrete execution skills** that answer "how to do this specific thing in this scenario".

Example: [form-handling](examples/web-frontend-skills/form-handling/SKILL.md)

## Generation Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    Generate L2 Methodology                      │
├─────────────────────────────────────────────────────────────────┤
│  1. Load L1 Meta-Meta Skills                                    │
│     │                                                            │
│     ├──→ [Serial] Epistemology Framework → Authority search     │
│     ├──→ [Serial] Decomposition & Abstraction                   │
│     ├──→ [Serial] Causal & Systems Thinking                     │
│     ├──→ [Serial] Meta-Cognition & Self-Reflection              │
│     ├──→ [Serial] Value & Priority Framework                    │
│     └──→ [Parallel] Interaction & Communication Model           │
│                                                                  │
│  2. Integrate and generate L2 methodology (must follow L2      │
│     output format specification)                                 │
│                                                                  │
│  3. Quality review                                               │
│                                                                  │
│  4. Output as SKILL.md to L2 storage directory                  │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Generate L3 Skill                             │
├─────────────────────────────────────────────────────────────────┤
│  Based on validated L2:                                         │
│  1. Determine specific scenario and task                        │
│  2. Extract relevant principles from L2                         │
│  3. Add concrete steps, examples, and precautions               │
│  4. Output to L3 storage directory                               │
└─────────────────────────────────────────────────────────────────┘
```

## Example Skills (L3)

The `web-frontend-skills/` directory contains 20+ concrete skills generated using the web-frontend methodology:

- [form-handling](examples/web-frontend-skills/form-handling/SKILL.md) — Form state management, validation, submission
- [state-management-implementation](examples/web-frontend-skills/state-management-implementation/SKILL.md) — State management patterns
- [routing-and-navigation](examples/web-frontend-skills/routing-and-navigation/SKILL.md) — Client-side routing
- [performance-optimization](examples/web-frontend-skills/performance-optimization/SKILL.md) — Performance best practices
- [error-boundary-and-fallback](examples/web-frontend-skills/error-boundary-and-fallback/SKILL.md) — Error handling
- [api-data-fetching-and-caching](examples/web-frontend-skills/api-data-fetching-and-caching/SKILL.md) — Data fetching patterns
- [lazy-loading-and-code-splitting](examples/web-frontend-skills/lazy-loading-and-code-splitting/SKILL.md) — Code splitting
- [automated-testing-strategy](examples/web-frontend-skills/automated-testing-strategy/SKILL.md) — Testing approaches
- [access-control-implementation](examples/web-frontend-skills/access-control-implementation/SKILL.md) — Authentication & authorization
- [chart-visualization-integration](examples/web-frontend-skills/chart-visualization-integration/SKILL.md) — Data visualization
- ...and more

## Design Principles

1. **Progressive Disclosure** — Skills load context gradually: metadata (~100 tokens) → core instructions (~500 tokens) → detailed references (on-demand)

2. **Immutability of L1** — L1 meta-meta skills, once published, are never modified. New versions are created alongside old ones.

3. **Separation of Concerns** — L2 only generates L3; L3 executes concrete tasks. Direct problem-solving is always done by L3.

4. **Authority-Based Knowledge** — L2 generation requires searching authoritative sources (academic papers, industry standards, expert opinions) before establishing cognitive standards.

## Usage Guide

This section explains how to write effective prompts when using Agent tools like Claude Code or OpenClaw to generate domain methodologies (L2) and concrete skills (L3).

### How the System Works

When you invoke the `meta-cognitive-skill-system` skill (via root `SKILL.md`), the system:
1. Loads the appropriate L1 meta-meta skills
2. Guides you through the generation process based on your prompt
3. Outputs structured SKILL.md files

### Writing Prompts for L2 Generation

When you need to work in a **new domain** without an existing L2 methodology:

**Prompt Template:**
```
I need to work in [domain name]. Please help me generate an L2 methodology
for this domain using the meta-cognitive skill system.

The domain is: [brief description of the domain]

Please:
1. Load the meta-cognitive-skill-system skill
2. Execute the L2 generation workflow
3. Search for authoritative sources in this domain
4. Generate a complete L2 methodology SKILL.md
```

**Example Prompts:**
- `"I need to develop backend services in Go. Generate an L2 methodology for backend development."`
- `"Help me create a methodology for data engineering. I want systematic approaches for building data pipelines."`
- `"We're entering the mobile development space. Generate an L2 methodology for iOS development."`

### Writing Prompts for L3 Generation

When you have an L2 methodology and need a **concrete skill** for a specific scenario:

**Prompt Template:**
```
I need to [specific task] in [domain]. Please use the [domain] methodology
(L2) to generate an L3 skill for this scenario.

The specific task is: [detailed description]

Please:
1. Load the [domain]-methodology skill
2. Extract relevant principles for this scenario
3. Generate a concrete L3 skill SKILL.md
```

**Example Prompts:**
- `"I need to implement JWT authentication in our React app. Use the web-frontend methodology to generate an L3 skill for JWT authentication."`
- `"We're building REST APIs with error handling. Use the backend methodology to generate an L3 skill for API error handling patterns."`
- `"I need to handle file uploads with progress tracking. Generate an L3 skill using the web-frontend methodology."`

### Writing Prompts for L2 Revision

When an existing L2 methodology has problems:

**Prompt Template:**
```
I found an issue with the [domain] methodology. [Describe the problem].

The specific issue is: [what's wrong]
The affected dimension is likely: [epistemology / decomposition / causal-thinking / meta-cognition / value-priority / interaction]

Please:
1. Load the meta-cognitive-skill-system skill
2. Identify the relevant L1 skill(s)
3. Revise only the affected section
4. Update the methodology version
```

### Activation Triggers Quick Check

| Your Situation | Correct Prompt Style | Wrong Prompt Style |
|---------------|---------------------|-------------------|
| "Need to work in backend development" | `"Generate an L2 methodology for backend development"` | `"Help me design this database schema"` |
| "Need a form skill" | `"Use web-frontend methodology to generate an L3 skill for form handling"` | `"How do I validate forms in React?"` |
| "Design a login form" | `"Use the form-handling L3 skill to guide the implementation"` | `"Generate L2 for frontend"` |

### Prompt Keywords Reference

| Intent | Recommended Keywords |
|--------|---------------------|
| Generate L2 | `"generate L2 methodology"`, `"create domain methodology"`, `"build methodology for [domain]"` |
| Generate L3 | `"generate L3 skill"`, `"create [scenario] skill"`, `"generate skill using [L2 name]"` |
| Revise L2 | `"revise methodology"`, `"update L2"`, `"methodology has issue with"` |
| Enter new domain | `"new domain"`, `"first time working in"`, `"no existing methodology"` |

## Quick Reference

| Action | Trigger |
|--------|---------|
| Enter a new domain | Run full L2 generation flow with all L1 skills |
| Existing L2 partially failing | Identify issue, invoke relevant L1 only |
| Generate L3 skill | Use L2 methodology to generate specific skill |

## License

MIT
