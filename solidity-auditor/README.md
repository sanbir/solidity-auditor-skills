# Solidity Auditor

A security agent for **Solidity and EVM systems**.

Attribution: this fork keeps the v2 packaging and audit workflow lineage from [pashov/skills](https://github.com/pashov/skills), then extends the Solidity corpus further.

Built for:

- **Solidity developers** who want fast security feedback before every commit
- **Security researchers** who need a first pass over a codebase before manual review
- **Auditors** who want broad attack-vector coverage before deeper protocol reasoning

It is not a substitute for a formal audit. It is the fast pass you should never skip.

## Demo

_Portrayed below: running the skill in a terminal workflow_

![Running solidity-auditor in terminal](../static/skill_pag.gif)

## Usage

```bash
/solidity-auditor
/solidity-auditor deep          # adds DeFi protocol agent (opus)
/solidity-auditor src/Vault.sol src/Router.sol
/solidity-auditor --file-output
```

## Coverage

- **321 attack vectors** tuned for EVM smart-contract security
- **8 specialized parallel agents** (vector-scan, math-precision, access-control, economic-security, execution-trace, invariant, periphery, first-principles)
- **Deep mode** adds DeFi protocol analysis (opus)

## What It Looks For

- authorization, ownership, and privileged-path mistakes
- proxy, initializer, upgrade, and storage-collision bugs
- reentrancy across functions, callbacks, hooks, and external integrations
- oracle manipulation, liquidation, rounding, and accounting drift
- permit, EIP-712, replay, and signature-validation mistakes
- bridge, messaging, and cross-domain trust failures
- exotic ERC-20 behaviors and unsafe integrations
- low-level call, returndata, gas-griefing, and edge-case execution issues

## Tips

- **Run it on the hot contracts you are actively changing.** Smaller scoped runs usually produce denser context and better findings.
- **Re-run after every fix set.** Model output is non-deterministic, and second passes often surface different bugs.
- **Use `deep` for DeFi, proxy-heavy, bridge, oracle, and smart-account systems.** Those architectures benefit most from the extra adversarial and protocol passes.
