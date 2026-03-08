# Solidity Auditor Skills

> The ultimate Claude Code skill for Solidity smart contract security auditing — 210 attack vectors, 7 parallel agents, DeFi protocol checklists, and adversarial reasoning.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## Install & Run

Works with **Claude Code CLI**, the **VS Code Claude extension**, and **Cursor**.

**Claude Code CLI:**

```bash
git clone https://github.com/anthropics/solidity-auditor-skills.git && mkdir -p ~/.claude/commands && cp -r solidity-auditor-skills/solidity-auditor ~/.claude/commands/solidity-auditor
```

**Cursor:**

```bash
git clone https://github.com/anthropics/solidity-auditor-skills.git && mkdir -p ~/.cursor/skills && cp -r solidity-auditor-skills/solidity-auditor ~/.cursor/skills/solidity-auditor
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
