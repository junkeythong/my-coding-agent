# My Coding Agent

Runs locally with repository checkouts and persistent memory, using MCP for external integrations.

Five skills: **interview-me**, **coding**, **code review**, **planning**, and **support**.

## Why These Features

- **Local clones:** Avoid repeated MCP source reads, network latency, and token-heavy responses. Reuse checkouts and indexes; remove disposable clones when unused, keeping maps and learnings.
- **Project map:** Quickly locate entry points, modules, and dependencies.
- **Index and memory:** Find code and reuse past decisions. Current source remains authoritative.
- **Security gate:** Require authorization for restricted local and MCP actions.

### Vendors and credits

| Source | Why we use it | Role in this agent |
| --- | --- | --- |
| [andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills) | Simple solutions, explicit assumptions, focused changes. | CODING discipline and REVIEW scope checks. |
| [agent-skills](https://github.com/addyosmani/agent-skills) | Repeatable steps from clarification to verified implementation. | INTERVIEW-ME, PLANNING, CODING, and REVIEW workflows. |
| [superpowers](https://github.com/obra/superpowers) | Root-cause debugging and evidence before claiming success. | SUPPORT investigations and CODING validation. |
| [mattpocock/skills](https://github.com/mattpocock/skills) | Clear instructions, simple module interfaces, shared domain terms. | Knowledge docs, architecture guidance, and retrospectives. |
| [Caveman](https://github.com/JuliusBrussee/caveman/tree/main) | Shorter responses and fewer output tokens; exact code preserved. | Shared voice; full sentences for approvals, clarification, and safety. |

Selected guidance is adapted into local skills. Vendor repositories are references, not runtime dependencies.

## How It Works

### Skill routing

Route by intent; URLs identify the repository or change.

```mermaid
flowchart TD
    REQ[User Request] --> VOICE["Caveman voice - terse output"]
    VOICE --> DET{Skill Detection}
    DET -->|vague ask| INT[INTERVIEW-ME]
    DET -->|review request| REV[REVIEW]
    DET -->|plan / break this down| PLAN[PLANNING]
    DET -->|tool question or error| SUP[SUPPORT]
    DET -->|otherwise| CODE[CODING]

    INT -->|large task| PLAN
    INT -->|concrete| CODE
    PLAN -->|approved| CODE
    REV -->|"fix comments"| CODE
    SUP -->|"our bug"| CODE
```

### Per-skill flows

```mermaid
flowchart LR
    subgraph "SUPPORT Skill"
        direction TB
        S1[Identify repo] --> S2[Refresh target revision]
        S2 --> S3{Fast path?}
        S3 -->|docs question| S4[README first]
        S4 --> S8{Sufficient?}
        S8 -->|Yes| S7[Synthesize answer]
        S8 -->|No| S6
        S3 -->|error/failure| S5[Version check]
        S5 --> S6["Local source + docs + linked issues"]
        S6 --> S7
    end

    subgraph "CODING Skill"
        direction TB
        C1[Read codebase map + recall] --> C2[Read code + tests]
        C2 --> C3[Implement]
        C3 --> C4[Test + lint]
        C4 --> C5[Update map + index memories]
    end

    subgraph "REVIEW Skill"
        direction TB
        R1[Fetch review + diff] --> R2[Check linked requirements + map]
        R2 --> R3["Cover every changed file<br/>(review or skip with reason)"]
        R3 --> R4["Analyze - read beyond the diff"]
        R4 --> R5["Verify positioning (reflection pass)"]
        R5 --> R6[Produce structured review]
    end

    subgraph "PLANNING Skill"
        direction TB
        P1[Read-only investigation] --> P2[Map dependencies]
        P2 --> P3[Slice tasks vertically]
        P3 --> P4[Write plan + checkpoints]
    end

    subgraph "INTERVIEW-ME Skill"
        direction TB
        I1[Hypothesize + confidence] --> I2[Ask one question at a time]
        I2 --> I3[Restate intent]
        I3 --> I4[Confirm - explicit yes]
    end

    subgraph "Caveman voice"
        direction LR
        CV1["Terse output"] --> CV2{Carve-out?}
        CV2 -->|"restricted action · git confirm<br/>confusion · scope flag · safety refusal"| CV3[Full sentences]
        CV2 -->|otherwise| CV4[Grunt]
    end
```

### Components: skills, submodules, MCP servers, memory

```mermaid
flowchart LR
    subgraph AGENT["coding-agent (prompt layer)"]
        ROUTER["AGENT.md — router"]
        subgraph SKILLS["skills/ (workflow)"]
            SK["interview-me · review · planning<br/>support · coding · access-control"]
        end
        CAVE["Caveman voice "]
        ROUTER --> SK
        ROUTER -. applies .-> CAVE
    end

    subgraph VENDOR["Core vendor/ (git submodules — reference only)"]
        V1["superpowers"]
        V2["agent-skills"]
        V3["andrej-karpathy-skills"]
    end
    VENDOR -. adapted into .-> SKILLS

    subgraph LOCAL["Local tooling (this repo)"]
        EXEC["Local execution<br/>shell · files · tests"]
        MEM["memory.py — SQLite + FTS5"]
        DBc[("code_objects<br/>repo code index")]
        DBm[("memory_docs<br/>learnings recall")]
        MEM --> DBc
        MEM --> DBm
    end

    subgraph KIRO["MCP servers (external tools)"]
        M["Git · Jira · Jenkins · Confluence<br/>JFrog · GitLab"]
    end

    GATE["Runtime / tool policy<br/>restricted actions need authorization"]

    SK -->|"index / search / recall"| MEM
    SK -->|authorized local work| EXEC
    SK -->|read-only| KIRO
    SK -->|restricted actions| GATE
    GATE -->|approved| EXEC
    GATE -->|approved| KIRO
    MEM -. indexes .-> KIRONOTE["project-maps · context · plans"]
```

```mermaid
flowchart TD
    subgraph "Repo Indexing — memory.py"
        IX["memory.py index &lt;alias&gt;"] --> CL{Cloned locally?}
        CL -->|No| CLONE[Git clone from remote]
        CLONE --> SYNC["Resolve target revision<br/>fetch if needed"]
        CL -->|Yes| SYNC
        SYNC --> STALE{"Index matches target revision<br/>and working-tree state?"}
        STALE -->|Fresh| SKIP[Skip — already indexed]
        STALE -->|Changed| UPD["Select changed files<br/>remove deleted entries"]
        UPD --> OBJ[Extract and mask code objects]
        OBJ --> DB[(SQLite + FTS5)]
    end

    subgraph "Learnings Recall — memory.py"
        IM["memory.py index-memories"] --> WALK["Walk project-maps + context + plans"]
        WALK --> MT{mtime changed?}
        MT -->|No| MSKIP[Skip]
        MT -->|Yes| MASK[Mask secrets]
        MASK --> MDB[(memory_docs + FTS5)]
        RC["memory.py recall &lt;query&gt;"] --> MDB
        ST["memory.py stale [days]"] --> MDB
    end
```


## Skills

| Skill | Trigger | What it does |
| --- | --- | --- |
| **INTERVIEW-ME** | Vague request, "interview me" | Clarifies intent before plan/code |
| **REVIEW** | "review this commit", review request | Review changes against requirements |
| **PLANNING** | "plan this", large multi-file task | Ordered task list with acceptance criteria |
| **SUPPORT** | Tool question, error report, pasted logs | README-first answers, version checks, Jira/Git searches |
| **CODING** | Everything else | Implement, debug, test, update docs |


## Shared Infrastructure

### `config.yaml` — Repo Registry

Shared registry of repository aliases, clone paths, and search keywords.

### `memory.py` — Repo Indexing

Local SQLite/FTS5 search for source code and saved learnings.

Features:

- **On-demand clones** — reuse local checkouts across tasks; re-clone after cleanup.
- **Cleanup** — remove only disposable, inactive, agent-managed clones with no uncommitted changes or unpushed commits; retain maps and memory.
- **Freshness** — refresh the target revision as needed; track revision and working-tree state; update changed and deleted entries.
- **Code object extraction** — functions, classes, YAML keys, shell functions
- **Credential masking** — regex and entropy checks before storing source or memory.

### `knowledge/` — Reference Material

| File | Purpose |
| --- | --- |
| `troubleshooting.md` | Structured triage: reproduce → localize → narrow → diagnose → resolve |
| `backward-compatibility.md` | Breaking change detection for coding, review, and support |
| `prompt-templates.md` | Reusable patterns for repo identification, version checks, log analysis |
| `complexity-classification.md` | Auto-think: TRIVIAL/STANDARD/COMPLEX task classification |
| `tool-recovery.md` | Tool-specific error recovery with retry hints |
| `writing-for-agents.md` | Guide for authoring skill/steering docs: context pointers, progressive disclosure, no-op pruning |
| `session-state-template.md` | Templates for repo index, hindsight, and session checkpoints |

## Example Requests

```text
# Support: how to use a tool
> how do I use doc tool to generate HTML?

# Support: investigate an error
> project Fun build fail!

# Review a commit:
> review my latest commit

# Plan a large feature:
> plan the implementation for adding this chat feature

# Write code:
> add retry logic to chat client
```

## Access Control

`skills/access-control.md` defines restricted actions. Runtime permissions and tool policies must enforce these boundaries for local commands and MCP calls. The actions below require scoped user authorization; routine edits within the requested scope can proceed.

| Category | Examples |
| --- | --- |
| **Network** | Open ports, change firewall/iptables, modify DNS/proxy/routing |
| **Production** | Any action targeting prod systems, clusters, or databases |
| **Deployment** | helm install, kubectl apply, docker push |
| **Configuration** | System configs, K8s ConfigMaps/Secrets, .env for running services |
| **Credentials** | Tokens, SSH/GPG keys, RBAC, certificates |
| **Data & State** | Delete databases, purge queues, bulk-delete files |

Runtime data (gitignored):

```text
memory.db    # SQLite database for repo indexes + code objects
repos/       # Local clones of repos (auto-cloned)
```
