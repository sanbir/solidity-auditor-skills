# Solidity Auditor

The ultimate AI-powered security audit skill for Solidity — 229 attack vectors, 5 parallel scan agents, adversarial reasoning, and DeFi protocol analysis.

Built for:

- **Solidity devs** who want a security check before every commit
- **Security researchers** looking for fast wins before a manual review
- **Auditors** who want systematic vector coverage as a first pass

Not a substitute for a formal audit — but the most comprehensive AI check you can run.

## What's Inside

- **229 attack vectors** across 5 reference files — covering reentrancy, access control, arithmetic, ERC token standards, proxies, cross-chain (LayerZero), DeFi protocols, L2 considerations, flash loans, oracle manipulation, assembly pitfalls, and more
- **5 parallel vector-scan agents** — each assigned ~45 vectors, scanning the full codebase simultaneously
- **Adversarial reasoning agent** (DEEP mode) — free-form exploit hunting using Feynman questioning, state inconsistency analysis, and invariant hunting
- **DeFi protocol agent** (DEEP mode) — domain-specific checklists for lending, AMM/DEX, ERC-4626 vaults, staking, bridges, governance, proxies, and account abstraction (ERC-4337)
- **False-positive gate** — every finding must pass 3 checks (concrete path, reachable entry point, no existing guard)
- **Confidence scoring** — base 100 with deductions for privileged callers, partial paths, self-contained impact, token assumptions, and external preconditions

## Usage

```bash
# Scan the full repo (default — 5 agents)
/solidity-auditor

# Full repo + adversarial reasoning + DeFi protocol analysis (7 agents)
/solidity-auditor deep

# Review specific file(s)
/solidity-auditor src/Vault.sol
/solidity-auditor src/Vault.sol src/Router.sol

# Write report to a markdown file (terminal-only by default)
/solidity-auditor --file-output
```

## Constraints (optional)

Drop a `.pashov-skills-constraints.yaml` in your repo root to tell the skill what your codebase does and doesn't use. Agents will skip irrelevant attack vectors during triage, reducing noise and scan time.

```yaml
tokens: [USDC, WETH]        # skip exotic-token vectors
standards: [ERC20]           # skip ERC721/ERC1155/ERC4626 vectors
cross_chain: false           # skip LayerZero/bridge vectors
proxy_pattern: none          # none | transparent | uups | diamond | beacon
oracle: chainlink            # chainlink | twap | pyth | custom | none
account_abstraction: false   # skip ERC-4337 vectors
```

All fields optional. Code overrides constraints.

## Known Limitations

**Codebase size.** Works best up to ~2,500 lines of Solidity. Past ~5,000 lines, triage accuracy and mid-bundle recall drop noticeably. For large codebases, run per module rather than everything at once.

**What AI misses.** AI is strong at pattern matching — missing access controls, unchecked return values, known reentrancy shapes. It struggles with relational reasoning: multi-transaction state setups, specification/invariant bugs, cross-protocol composability, game-theory attacks, and off-chain assumptions. AI catches what humans forget to check. Humans catch what AI cannot reason about. You need both.
