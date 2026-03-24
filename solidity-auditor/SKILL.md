---
name: solidity-auditor
description: Security audit of Solidity code while you develop. Trigger on "audit", "check this contract", "review for security". Modes - default (full repo), DEEP (+ DeFi protocol agent), or a specific filename.
---

# Smart Contract Security Audit

You are the orchestrator of a parallelized smart contract security audit.

## Mode Selection

**Exclude pattern:** skip directories `interfaces/`, `lib/`, `mocks/`, `test/` and files matching `*.t.sol`, `*Test*.sol` or `*Mock*.sol`.

- **Default** (no arguments): scan all `.sol` files using the exclude pattern. Use Bash `find` (not Glob).
- **deep**: same scope as default, but also spawns the DeFi protocol agent (Agent 9, opus). Use for thorough reviews. Slower and more costly.
- **`$filename ...`**: scan the specified file(s) only.

**Flags:**

- `--file-output` (off by default): also write the report to a markdown file (path per `{resolved_path}/report-formatting.md`). Never write a report file unless explicitly passed.

## Constraints (optional)

If a `.pashov-skills-constraints.yaml` file exists in the repository root, the orchestrator reads it during discovery and includes it in every agent bundle. Agents use declared constraints to fast-track **Skip** classification during triage. Constraints describe factual codebase properties, not security assumptions. Code overrides constraints.

Supported fields (all optional, unknown fields ignored):

```yaml
tokens: [USDC, WETH]              # accepted token list (skip exotic-token vectors if set)
standards: [ERC20]                 # implemented standards (skip unlisted standard vectors)
cross_chain: false                 # no bridge/LayerZero/cross-chain logic
proxy_pattern: none                # none | transparent | uups | diamond | beacon
oracle: chainlink                  # chainlink | twap | pyth | custom | none
account_abstraction: false         # no ERC-4337 / smart account logic
```

## Orchestration

**Turn 1 — Discover.** Print the banner, then make these parallel tool calls in one message:

a. Bash `find` for in-scope `.sol` files per mode selection
b. Glob for `**/references/attack-vectors/attack-vectors-1.md` — extract the `references/` directory (two levels up) as `{resolved_path}`
c. ToolSearch `select:Agent`
d. Read the local `VERSION` file from the same directory as this skill
e. Bash `curl -sf https://raw.githubusercontent.com/sanbir/solidity-auditor-skills/main/solidity-auditor/VERSION`
f. Bash `mktemp -d /tmp/audit-XXXXXX` — store as `{bundle_dir}`
g. Read `.pashov-skills-constraints.yaml` from the repository root (if not found, continue without constraints)

If the remote VERSION fetch succeeds and differs from local, print `⚠️ You are not using the latest version. Please upgrade for best security coverage. See https://github.com/sanbir/solidity-auditor-skills`. If it fails, skip silently.

**Turn 2 — Prepare.** In one message, make parallel tool calls: (a) Read `{resolved_path}/report-formatting.md`, (b) Read `{resolved_path}/judging.md`.

Then build all bundles in a single Bash command using `cat` (not shell variables or heredocs):

1. `{bundle_dir}/source.md` — ALL in-scope `.sol` files, each with a `### path` header and fenced code block.
2. Agent bundles = `source.md` + per-agent bundles with cat:

| Bundle               | Appended files (relative to `{resolved_path}`)                                                                                                  |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `agent-1-bundle.md`  | `attack-vectors/attack-vectors-1.md` + `attack-vectors/attack-vectors-2.md` + `attack-vectors/attack-vectors-3.md` + `attack-vectors/attack-vectors-4.md` + `attack-vectors/attack-vectors-5.md` + `hacking-agents/vector-scan-agent.md` + `hacking-agents/shared-rules.md` |
| `agent-2-bundle.md`  | `hacking-agents/math-precision-agent.md` + `hacking-agents/shared-rules.md`                                     |
| `agent-3-bundle.md`  | `hacking-agents/access-control-agent.md` + `hacking-agents/shared-rules.md`                                     |
| `agent-4-bundle.md`  | `hacking-agents/economic-security-agent.md` + `hacking-agents/shared-rules.md`                                  |
| `agent-5-bundle.md`  | `hacking-agents/execution-trace-agent.md` + `hacking-agents/shared-rules.md`                                    |
| `agent-6-bundle.md`  | `hacking-agents/invariant-agent.md` + `hacking-agents/shared-rules.md`                                          |
| `agent-7-bundle.md`  | `hacking-agents/periphery-agent.md` + `hacking-agents/shared-rules.md`                                          |
| `agent-8-bundle.md`  | `hacking-agents/first-principles-agent.md` + `hacking-agents/shared-rules.md`                                   |

If constraints were found in Turn 1, prepend a `## Constraints` section containing the raw YAML content to every bundle.

Print line counts for every bundle and `source.md`. Do NOT inline file content into agent prompts.

**Turn 3 — Spawn.** In one message, spawn all agents as parallel foreground Agent calls. Always spawn Agents 1–8 with `model: "sonnet"`. Only spawn Agent 9 when the mode is **DEEP**.

- **Agents 1–8** — spawn with `model: "sonnet"`. Prompt template (substitute real values):

```
Your bundle file is {bundle_dir}/agent-N-bundle.md (XXXX lines).
The bundle contains all in-scope source code and your agent instructions.
Read the bundle fully before producing findings.
```

- **Agent 9** (DeFi protocol analysis, DEEP only) — spawn with `model: "opus"`. Receives the in-scope `.sol` file paths and the instruction: your reference directory is `{resolved_path}`. If constraints were found in Turn 1, also include: the constraints file is at `.pashov-skills-constraints.yaml` in the repository root. Read `{resolved_path}/hacking-agents/defi-protocol-agent.md` for your full instructions.

**Turn 4 — Deduplicate, validate & output.** Single-pass: deduplicate all agent results, gate-evaluate, and produce the final report in one turn. Do NOT print an intermediate dedup list — go straight to the report.

1. **Deduplicate.** Parse every FINDING and LEAD from all agents. Group by `group_key` field (format: `Contract | function | bug-class`). Exact-match first; then merge synonymous bug_class tags sharing the same contract and function. Keep the best version per group, number sequentially, annotate `[agents: N]`.

   Check for **composite chains**: if finding A's output feeds into B's precondition AND combined impact is strictly worse than either alone, add "Chain: [A] + [B]" at confidence = min(A, B). Most audits have 0–2.

2. **Gate evaluation.** Run each deduplicated finding through the four gates in `judging.md` (do not skip or reorder). Evaluate each finding exactly once — do not revisit after verdict.

   **Single-pass protocol:** evaluate every relevant code path ONCE in fixed order (constructor → setters → swap functions → mint → burn → liquidate). One-line verdict per path: `BLOCKS`, `ALLOWS`, `IRRELEVANT`, or `UNCERTAIN`. Commit after all paths — do not re-examine. `UNCERTAIN` = `ALLOWS`.

3. **Lead promotion & rejection guardrails.**
   - Promote LEAD → FINDING (confidence 75) if: complete exploit chain traced in source, OR `[agents: 2+]` demoted (not rejected) the same issue.
   - `[agents: 2+]` does NOT override a concrete refutation — demote to LEAD if refutation is uncertain.
   - No deployer-intent reasoning — evaluate what the code _allows_, not how the deployer _might_ use it.

4. **Fix verification** (confidence ≥ 80 only): trace the attack with fix applied; verify no new DoS, reentrancy, or broken invariants (use `safeTransfer` not `require(token.transfer(...))`); list all locations if the pattern repeats. If no safe fix exists, omit it with a note.

5. **Format and print** per `report-formatting.md`. Exclude rejected items. If `--file-output`: also write to file.

## Banner

Before doing anything else, print this exactly:

```

███████╗ ██████╗ ██╗     ██╗██████╗ ██╗████████╗██╗   ██╗
██╔════╝██╔═══██╗██║     ██║██╔══██╗██║╚══██╔══╝╚██╗ ██╔╝
███████╗██║   ██║██║     ██║██║  ██║██║   ██║    ╚████╔╝
╚════██║██║   ██║██║     ██║██║  ██║██║   ██║     ╚██╔╝
███████║╚██████╔╝███████╗██║██████╔╝██║   ██║      ██║
╚══════╝ ╚═════╝ ╚══════╝╚═╝╚═════╝ ╚═╝   ╚═╝      ╚═╝
       █████╗ ██╗   ██╗██████╗ ██╗████████╗ ██████╗ ██████╗
      ██╔══██╗██║   ██║██╔══██╗██║╚══██╔══╝██╔═══██╗██╔══██╗
      ███████║██║   ██║██║  ██║██║   ██║   ██║   ██║██████╔╝
      ██╔══██║██║   ██║██║  ██║██║   ██║   ██║   ██║██╔══██╗
      ██║  ██║╚██████╔╝██████╔╝██║   ██║   ╚██████╔╝██║  ██║
      ╚═╝  ╚═╝ ╚═════╝ ╚═════╝ ╚═╝   ╚═╝    ╚═════╝ ╚═╝  ╚═╝

  328 attack vectors · 8 specialized agents · DeFi protocol analysis

```
