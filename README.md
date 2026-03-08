# Solidity Auditor Skills

> The ultimate Claude Code skill for Solidity smart contract security auditing — 210 attack vectors, 7 parallel agents, DeFi protocol checklists, and adversarial reasoning.

Forked from [pashov/skills](https://github.com/pashov/skills) and extended with knowledge aggregated from 20+ open-source audit skill repositories.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## What Changed from pashov/skills

This repo builds on Pashov Audit Group's excellent [solidity-auditor](https://github.com/pashov/skills) skill — the same 4-turn orchestration, the same FP gate, the same report format. Everything below is additive; nothing from the original was removed.

### +40 Attack Vectors (170 → 210)

Pashov's 170 vectors are the best publicly available collection of Solidity attack patterns. They cover syntactic/structural bugs comprehensively — reentrancy variants, proxy pitfalls, ERC standard edge cases, assembly traps, cross-chain messaging, and more.

The 40 new vectors (V171–V210) target gaps in three areas:

**DeFi protocol economics (V171–V179, V185–V189, V192).** Liquidation mechanics are the #1 source of high-severity findings in Sherlock and Code4rena lending protocol contests. Pashov touches this with V41 (dust positions) and V43 (self-liquidation), but the broader class — insufficient liquidation incentives, cherry-picked collateral seizure, interest accrual during pause, liquidation bonus exceeding available collateral, multi-decimal confusion in liquidation math — was not covered. These bugs are behavioral and economic, not syntactic, which is why pattern-matching agents miss them.

**Staking and reward manipulation (V180–V184, V190).** First-depositor reward stealing, flash deposit/withdraw griefing, reward dilution via direct transfer, stale reward indices, and precision loss zeroing small stakers. Pashov has V144 (checkpoint ordering) but not the broader class of reward gaming patterns that show up in nearly every staking protocol audit.

**Modern EVM and library evolution (V193–V210).** EIP-1153 transient storage reentrancy in delegatecall contexts (V198), Uniswap V4 hook callback authorization and state desync (V199–V200), OpenZeppelin v4→v5 migration confusion where `_beforeTokenTransfer` overrides silently stop executing (V203), on-chain quoter-based slippage that looks correct but is flash-loan manipulable (V193), and behavioral consistency patterns like inconsistent `whenNotPaused` coverage across semantically equivalent functions (V202, V209). These are post-2024 attack surfaces.

### +1 Agent Type: DeFi Protocol Analysis

Pashov's architecture runs 4 vector-scan agents (Sonnet) in default mode, adding a 5th adversarial-reasoning agent (Opus) in DEEP mode. We keep all of those and add:

**Agent 7 — DeFi Protocol Agent** (Opus, DEEP only). Instead of scanning for known patterns, this agent classifies the protocol type and runs domain-specific checklists:

- **Lending/Borrowing** (14 items): health factor includes accrued interest, liquidation incentive covers gas for min positions, self-liquidation not profitable, collateral withdrawal blocked when underwater, interest/liquidation pause symmetry
- **AMM/DEX** (9 items): slippage from calldata not from on-chain quoter, deadline not `block.timestamp`, multi-hop protection on final output, fee tier not hardcoded, LP value not from spot reserves
- **Vault/ERC-4626** (9 items): first-depositor inflation mitigated, rounding direction correct in all 4 operations, round-trip profit impossible, share price not manipulable via direct transfer
- **Staking/Rewards** (10 items): rewardPerToken updated before balance changes, no flash deposit/withdraw capture, precision loss doesn't zero small stakers, cooldown not griefable by dust deposits
- **Bridge/Cross-Chain** (11 items): message replay protection, peer validation, rate limits, DVN diversity, supply invariant, decimal conversion, sequencer uptime + grace period
- **Governance** (6 items): vote weight from past block, timelock, quorum, no double-voting via transfer
- **Proxy/Upgradeable** (11 items): `_disableInitializers`, atomic deploy+init, storage append-only, EIP-1967 slots, UUPS authorization, diamond namespaced storage
- **Account Abstraction/ERC-4337** (7 items): entryPoint restriction, signature binding, no banned opcodes in validation, paymaster prefund penalty, factory salt includes owner

This catches an entirely different class of bugs than vector scanning. "Does this lending protocol's liquidation math actually work end-to-end?" is a different question from "is there a reentrancy here?"

### +1 Scan Agent (4 → 5 vector-scan agents)

210 vectors distributed across 5 agents (~42 each) instead of 170 across 4. Same per-agent cognitive load, better parallelism.

### Enhanced Adversarial Reasoning Agent

The adversarial agent (Agent 6) adds three structured reasoning strategies from [nemesis-auditor](https://github.com/sainikethan/nemesis-auditor):

- **Feynman questioning:** For each function, "What would happen if I called this with the most adversarial possible inputs?"
- **State inconsistency analysis:** For every pair of functions sharing state, check whether Function A can leave state in a condition Function B doesn't expect.
- **Invariant hunting:** Identify implicit invariants (totalSupply == sum(balances), conservation laws) and find functions that violate them.

### +2 Confidence Score Deductions

Pashov's scoring: privileged caller (-25), partial path (-20), self-contained impact (-15).

We add:
- **Requires specific token behavior** (fee-on-transfer, rebasing, ERC-777 hooks) → **-10**
- **Requires external precondition** (oracle failure, L2 sequencer downtime, bridge delay) → **-10**

These reduce noise from the most common source of low-value contest submissions — "technically exploitable but only if you're using a weird token on an L2 where the sequencer went down."

### What's Unchanged

The core architecture is identical: 4-turn orchestration (discover → prepare → spawn → report), chunked bundle reading, triage → deep pass → composability check → hard stop, 3-check FP gate, confidence scoring from 100, and the same report format. If you know pashov/skills, this works the same way — just with more coverage.

---

## Install & Run

Works with **Claude Code CLI**, the **VS Code Claude extension**, and **Cursor**.

**Claude Code CLI:**

```bash
git clone https://github.com/sanbir/solidity-auditor-skills.git && mkdir -p ~/.claude/commands && cp -r solidity-auditor-skills/solidity-auditor ~/.claude/commands/solidity-auditor
```

**Cursor:**

```bash
git clone https://github.com/sanbir/solidity-auditor-skills.git && mkdir -p ~/.cursor/skills && cp -r solidity-auditor-skills/solidity-auditor ~/.cursor/skills/solidity-auditor
```

The skill is then invocable as `/solidity-auditor`. See the [skill README](solidity-auditor/README.md) for usage.

**Update to latest:** `cd` into the cloned repo and run:

```bash
git pull
# Claude Code CLI:
cp -r solidity-auditor/  ~/.claude/commands/solidity-auditor
# Cursor:
cp -r solidity-auditor/ ~/.cursor/skills/solidity-auditor
```

---

## Skills

| Skill | Description |
| --- | --- |
| [solidity-auditor](solidity-auditor/) | 210-vector security audit with 5–7 parallel agents, DeFi protocol checklists, and adversarial reasoning |

---

## What's Included

### 210 Attack Vectors (5 reference files)

| File | Vectors | Focus Areas |
| --- | --- | --- |
| [attack-vectors-1](solidity-auditor/references/attack-vectors/attack-vectors-1.md) | 1–42 | Signatures, ERC721/1155, snapshots, timestamps, proxies, LayerZero, msg.value, access control, CREATE2, reentrancy callbacks |
| [attack-vectors-2](solidity-auditor/references/attack-vectors/attack-vectors-2.md) | 43–84 | Liquidation, overflow/underflow, diamond proxies, force-fed ETH, oracle pricing, ERC4626 rounding, Chainlink, fee-on-transfer, Merkle trees, rebasing tokens |
| [attack-vectors-3](solidity-auditor/references/attack-vectors/attack-vectors-3.md) | 85–126 | Assembly pitfalls, flash loans, governance hijacks, transient storage (EIP-1153), cross-contract reentrancy, proxy initialization, minimal proxies, slippage |
| [attack-vectors-4](solidity-auditor/references/attack-vectors/attack-vectors-4.md) | 127–170 | Cross-chain replay, TWAP manipulation, Merkle reuse, bridge security, DVN diversity, L2 sequencer, 63/64 gas rule, storage layout, metamorphic contracts, calldata malleability |
| [attack-vectors-5](solidity-auditor/references/attack-vectors/attack-vectors-5.md) | 171–210 | DeFi liquidation economics, staking reward griefing, L2 sequencer grace periods, interest accrual edge cases, Uniswap V4 hooks, OZ v4/v5 confusion, behavioral vulnerabilities |

### 3 Specialized Agents

| Agent | Mode | Model | Approach |
| --- | --- | --- | --- |
| Vector Scan (x5) | Default + Deep | Sonnet | Systematic triage of ~42 vectors each against full codebase |
| Adversarial Reasoning | Deep only | Opus | Free-form exploit hunting with Feynman questioning and invariant analysis |
| DeFi Protocol | Deep only | Opus | Domain-specific checklists for lending, AMM, vaults, staking, bridges, governance, proxies, ERC-4337 |

### Quality Controls

- **FP Gate:** 3-check filter (concrete path, reachable entry, no existing guard)
- **Confidence Scoring:** Base 100 with deductions for privileged callers (-25), partial paths (-20), self-contained impact (-15), token assumptions (-10), external preconditions (-10)
- **Threshold:** Findings below 75 confidence reported without fix suggestions

---

## Attributions

This skill aggregates knowledge from the following open-source repositories. We are grateful to all contributors.

### Core Architecture

| Repository | Author | Contribution |
| --- | --- | --- |
| [pashov/skills](https://github.com/pashov/skills) | Pashov Audit Group | Parallelized agent orchestration, 170 attack vectors (V1–V170), FP gate, confidence scoring, vector-scan and adversarial-reasoning agent patterns, report formatting |

### Attack Vectors & Patterns

| Repository | Author | Contribution |
| --- | --- | --- |
| [quillai-network/qs_skills](https://github.com/quillai-network/qs_skills) | QuillAI Network | Behavioral state analysis, semantic guard analysis, state invariant detection, reentrancy pattern classification, oracle/flash loan patterns, proxy upgrade safety, input/arithmetic safety, external call safety, signature replay taxonomy, DoS/griefing patterns |
| [auditmos/skills](https://github.com/auditmos/skills) | Auditmos | Lending protocol vulnerabilities, liquidation mechanics (fairness, DoS, calculation precision), staking reward edge cases, concentrated liquidity management, auction security, slippage/MEV patterns, oracle manipulation |
| [carni-ships/SolidSecs](https://github.com/carni-ships/SolidSecs) | SolidSecs | 100+ vulnerability classes, protocol-specific checklists (AMM/DEX, lending, vault, bridge, governance, proxy, staking, account abstraction, Uniswap V4), modern EVM patterns (EIP-7702, ERC-4337) |
| [max-taylor/Claude-Solidity-Skills](https://github.com/max-taylor/Claude-Solidity-Skills) | Max Taylor | 115+ audit checklist items, 37 SWC vulnerability types, 20+ weird ERC20 behaviors, gas optimization categories |
| [alt-research/SolidityGuard](https://github.com/alt-research/SolidityGuard) | AltResearch | 104 vulnerability patterns, Slither/Aderyn/Mythril integration patterns, entry point analysis |
| [GuardianAudits/SolidityLab](https://github.com/GuardianAudits/SolidityLab) | Guardian Audits | Documented bug encyclopedia (17 patterns), attack vector encyclopedia (15+ patterns), auditor's handbook methodology |
| [devdacian/ai-auditor-primers](https://github.com/devdacian/ai-auditor-primers) | Devdacian | ERC-4626 vault specialist patterns, general auditor heuristics |

### Methodology & Agents

| Repository | Author | Contribution |
| --- | --- | --- |
| [trailofbits/skills](https://github.com/trailofbits/skills) | Trail of Bits | Building secure contracts methodology, secure workflow guide, token integration analysis, code maturity assessment framework, multi-chain vulnerability scanners |
| [sainikethan/nemesis-auditor](https://github.com/sainikethan/nemesis-auditor) | Nemesis | Dual-pass auditing (Feynman questioning + state inconsistency analysis), iterative convergence methodology |
| [Archethect/sc-auditor](https://github.com/Archethect/sc-auditor) | Archethect | MAP-HUNT-ATTACK methodology, Cyfrin audit checklist integration, Solodit findings search |
| [kadenzipfel/scv-scan](https://github.com/kadenzipfel/scv-scan) | Kaden Zipfel | 4-phase systematic audit methodology (load → sweep → validate → report), severity guidelines |
| [slvDev/weasel](https://github.com/slvDev/weasel) | slvDev | Rust-based parallel analysis, PoC writing patterns, finding triage/filter methodology |
| [auditor-addon/auditor-addon](https://github.com/auditor-addon/auditor-addon) | Auditor Addon | Map & Probe methodology, threat modeling (STRIDE), call chain tracing, code metrics |

### DeFi Protocol Knowledge

| Repository | Author | Contribution |
| --- | --- | --- |
| [ethskills.com](https://ethskills.com) | EthSkills | 500+ audit checklist items across 19 domains, building blocks (Uniswap, Aave, flash loans), protocol composability |
| [BowTiedSwan/solodit-api-skill](https://github.com/BowTiedSwan/solodit-api-skill) | BowTied Swan | 50,000+ real-world findings database integration, audit firm taxonomy |
| [OpenZeppelin/openzeppelin-skills](https://github.com/OpenZeppelin/openzeppelin-skills) | OpenZeppelin | Secure contract development patterns, upgrade safety |
| [Cyfrin/solskill](https://github.com/Cyfrin/solskill) | Cyfrin / Patrick Collins | Solidity development standards, testing emphasis |
| [LLM-Driven-Development/aether](https://github.com/LLM-Driven-Development/aether) | Aether | 180+ static detectors, 14 protocol archetypes, 75+ exploit knowledge base, taint analysis |
| [Sir-Shaedy/Diablo](https://github.com/Sir-Shaedy/Diablo) | Diablo | 50k+ Solodit findings search, academy study modules from historical reports |
| [hackenproof-public/skills](https://github.com/hackenproof-public/skills) | HackenProof | Bug bounty triage workflow, severity assignment methodology |
| [Lemon-Weasel/wake-ai](https://github.com/Lemon-Weasel/wake-ai) | Wake AI | Multi-step workflow orchestration, progressive validation |

---

## License

[MIT](LICENSE) — see individual attribution repos for their respective licenses.
