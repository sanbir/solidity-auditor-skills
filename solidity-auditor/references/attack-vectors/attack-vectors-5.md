229 total attack vectors — Extended patterns from DeFi protocol analysis, L2 considerations, behavioral vulnerabilities, EIP-7702, restaking, and modern attack surfaces

**168. L2 Sequencer Grace Period Missing**

- **D:** On L2 chains, when sequencer restarts after downtime, contracts immediately liquidate positions without allowing users a grace period to adjust collateral. Users who were solvent before downtime get unfairly liquidated.
- **FP:** Explicit grace period logic after sequencer restart (e.g., `require(block.timestamp > sequencerUptimeTimestamp + GRACE_PERIOD)`). L1-only deployment. Chainlink L2 Sequencer Uptime Feed checked with grace window.


**169. Liquidation Incentive Insufficient for Trustless Actors**

- **D:** Liquidation reward (bonus %) doesn't cover gas cost for small positions. No one profitably liquidates dust positions, leading to bad debt accumulation. Protocol becomes insolvent as underwater positions compound.
- **FP:** Minimum position size enforced at borrow time. Protocol operates its own liquidation bot. Insurance fund covers unprofitable liquidations. Dynamic liquidation incentive scaled to position size.


**170. Collateral Withdrawal While Position Underwater**

- **D:** User can withdraw partial collateral even when position is underwater, as long as the specific collateral's PNL is positive. Removes liquidation incentive without improving protocol health.
- **FP:** Withdrawal blocked when overall position health factor < 1. Strict overcollateralization lock prevents any withdrawal while underwater.


**171. Dust Loan Griefing (Minimum Loan Size Bypass)**

- **D:** Attacker bypasses or exploits missing `minLoanSize` checks to create many tiny loans that are individually too small to profitably liquidate. Gas cost of liquidation exceeds recovered value, accumulating protocol bad debt.
- **FP:** `require(borrowAmount >= MIN_BORROW)` enforced on all borrow paths. Batch liquidation mechanism handles dust positions. Automatic write-off for sub-threshold bad debt.


**172. Unfair Liquidation via Cherry-Picked Collateral**

- **D:** Liquidator selects which collateral asset to seize, choosing the most liquid/stable asset while leaving volatile collateral. Borrower's position becomes unhealthier post-liquidation despite liquidator profiting.
- **FP:** Collateral seizure follows defined priority ordering. Liquidation enforces health improvement post-seizure (`healthFactorAfter > healthFactorBefore`). Single-collateral system.


**173. Interest Accrual During Emergency Pause**

- **D:** When admin pauses repayments for emergency, interest continues accruing. Users cannot repay but their debt grows, forcing liquidation of positions that were healthy before the pause.
- **FP:** Pause simultaneously halts interest accrual. Symmetric pause behavior (repay and liquidate both paused together). Grace period after unpause before liquidations resume.


**174. Repayment Paused While Liquidation Active**

- **D:** Admin pauses repayments for emergency but liquidations remain active. Borrowers cannot exit positions while protocol can still seize their collateral — asymmetric freeze benefits protocol at user expense.
- **FP:** Synchronized pause (repay and liquidate paused/unpaused together). Documented asymmetric design with economic justification. Emergency pause as intentional protocol safety feature.


**175. Liquidation Leaves Borrower Unhealthier**

- **D:** After partial liquidation, borrower's health factor is lower than before due to incorrect close factor calculation, cherry-picked collateral seizure, or liquidation bonus exceeding available surplus.
- **FP:** Post-liquidation health check (`require(healthAfter >= healthBefore || healthAfter >= 1)`). Full liquidation when partial would worsen position. Documented minimum health improvement requirement.


**176. No LTV Gap Between Borrow and Liquidation Threshold**

- **D:** Liquidation threshold equals max borrow LTV. Positions become immediately liquidatable after borrowing with zero buffer for normal price volatility. Users have no margin to avoid liquidation.
- **FP:** Explicit gap between max borrow LTV and liquidation threshold (e.g., borrow at 75%, liquidate at 80%). Documentation explains chosen parameters. Per-asset configurable thresholds.


**177. First Depositor Reward Stealing (Staking)**

- **D:** In staking/reward contracts, first depositor front-runs initial reward distribution with minimal (1-wei) deposit, capturing 100% of initial rewards intended for later legitimate stakers.
- **FP:** Minimum stake amount enforced. Admin-only initial deposit establishes baseline. Time-weighted reward calculation prevents instant claiming. Initial reward distribution delayed until minimum TVL reached.

---

---


**178. Reward Dilution via Direct Token Transfer**

- **D:** Attacker transfers staking tokens directly to contract (bypassing `stake()` function), inflating `totalSupply` in balance-based calculations without earning tracked stake. Dilutes rewards for legitimate stakers.
- **FP:** Separate reward token tracking independent of raw balance. Internal `totalStaked` variable updated only via `stake()`/`unstake()`. Protocol explicitly handles direct transfer surplus.


**179. Flash Deposit/Withdraw Reward Griefing**

- **D:** Large instantaneous deposit right before reward distribution dilutes per-token reward rate. Attacker withdraws immediately after distribution, capturing disproportionate share intended for steady-state stakers.
- **FP:** Minimum stake duration requirement (cooldown). Time-weighted reward calculation. Reward snapshot at previous block. Anti-flash-loan modifier (`require(block.number > depositBlock)`).


**180. Stale Reward Index After Distribution**

- **D:** Reward distribution updates global reward pool but fails to call `updateReward()` / update `rewardPerTokenStored`. Stale index causes incorrect reward calculations for all users until next state-changing call.
- **FP:** `updateReward()` called in all reward-distributing functions via modifier. Global index updated atomically with distribution. Post-distribution assertions verify consistency.


**181. Balance Caching Issues During Reward Claims**

- **D:** Claiming rewards reads user balance, performs external token transfer, then uses the cached balance for further calculations. Reentrant callback during transfer can manipulate state between read and use.
- **FP:** `nonReentrant` on all claim functions. Balance read after transfer completes. CEI pattern followed — state updates before external calls.


**182. Liquidation Bonus Exceeds Available Collateral**

- **D:** Fixed liquidation bonus (e.g., 110% of debt in collateral) exceeds actual collateral when position is deeply underwater. Liquidation reverts because contract tries to transfer more than available, leaving bad debt permanently stuck.
- **FP:** Dynamic bonus capped at available collateral. Liquidation function handles partial recovery gracefully. Insurance fund covers shortfall. `min(bonus, availableCollateral)` pattern used.


**183. Incorrect Decimal Handling in Multi-Token Liquidations**

- **D:** Liquidation calculations assume uniform 18-decimal tokens but collateral/debt have different decimals (USDC=6, WETH=18). Order-of-magnitude errors in liquidation amounts — either liquidating too much or too little.
- **FP:** Per-token `decimals()` normalization in all cross-token math. Explicit `10 ** (18 - decimals())` scaling factors. Tested with mixed-decimal token pairs.


**184. Interest Accrual During Liquidation Auction**

- **D:** While collateral is being auctioned (Dutch auction, English auction), borrower's debt continues accruing interest. Long auctions make the position progressively worse, potentially causing auction proceeds to be insufficient.
- **FP:** Interest frozen at auction start timestamp. Auction duration bounded. Instant liquidation (no auction). Interest-inclusive reserve price.


**185. No Liquidation Slippage Protection**

- **D:** Liquidator calls `liquidate()` but received collateral amount has no minimum parameter. MEV bot sandwiches the liquidation tx, extracting value via collateral price manipulation.
- **FP:** `minCollateralReceived` parameter in liquidation function. Private mempool for liquidation txs. Protocol-operated liquidation bot with MEV protection.


**186. L2 Sequencer Downtime in Interest Accrual**

- **D:** Interest rate calculations use `block.timestamp` delta without accounting for L2 sequencer downtime periods. If sequencer is down for hours, the first post-restart block has a massive timestamp gap, compounding interest as if the protocol was operating normally.
- **FP:** Interest accrual capped per-update (`maxTimeDelta`). Sequencer uptime feed checked before accruing. Rate-limited compounding.


**187. Precision Loss in Reward-to-Token Conversion**

- **D:** Staking reward calculations use `rewardRate * timeElapsed / totalStaked` where small stakes produce zero due to integer division. Small stakers permanently receive zero rewards despite time passing.
- **FP:** Scaling factor applied (e.g., multiply by 1e18 before division). Minimum stake size enforced above precision loss threshold. Accumulator-based reward tracking.

---

---


**188. Time Unit Confusion in Interest Calculations**

- **D:** Interest accrual logic confuses time units — using `block.timestamp` (seconds) in a formula expecting days or blocks, or vice versa. Results in interest rates off by orders of magnitude (e.g., 365x too high or 86400x too low).
- **FP:** Documented time unit constants (`SECONDS_PER_YEAR = 365.25 days`). Unit tests with known interest calculations. Consistent use of `block.timestamp` (seconds) throughout.


**189. Oracle Manipulation via Self-Liquidation**

- **D:** User manipulates oracle price via flash loan (push TWAP or spot price), self-liquidates via second address at the manipulated price, extracts more collateral than debt owed. Profitable when manipulation cost < liquidation bonus.
- **FP:** TWAP with window > manipulation cost threshold. Chainlink/Pyth oracle resistant to single-tx manipulation. Self-liquidation blocked (`require(liquidator != borrower)`).


**190. On-Chain Quoter-Based Slippage Calculation**

- **D:** `minAmountOut` calculated on-chain using current spot price from an AMM quoter. Flash loan manipulates spot price before the tx, setting `minAmountOut` to near-zero, then sandwiches the actual swap.
- **FP:** `minAmountOut` supplied as calldata parameter from off-chain calculation. TWAP-based quoting. Maximum acceptable slippage hardcoded as percentage.


**191. Fixed Fee Tier Assumption in Multi-Pool DEX**

- **D:** Router hardcodes fee tier (e.g., Uniswap V3 0.3% pool) but liquidity migrates to different tier (0.05% or 1%). Swaps execute against low-liquidity pool with excessive slippage or revert.
- **FP:** Fee tier as function parameter. Router queries multiple tiers and uses optimal. Dynamic fee tier detection via factory query.


**192. Multi-Hop Swap Intermediate-Only Protection**

- **D:** Multi-hop swap (A→B→C) protects intermediate amount (B) but not final output (C). MEV extracts value on the final hop where no minimum is enforced.
- **FP:** `minFinalAmountOut` validated against user's actual received balance (balance delta check). Single-hop swap. Per-hop minimums specified by caller.


**193. block.timestamp as Swap Deadline**

- **D:** `deadline = block.timestamp` provides zero deadline protection since every block's timestamp satisfies it. Tx can be held in mempool indefinitely and executed at the worst possible time.
- **FP:** Deadline supplied as calldata from off-chain (e.g., `now + 300 seconds`). Keeper-based execution with enforced timeliness. Private mempool.


**194. Zero minAmountOut on DEX Swap**

- **D:** `swap(tokenIn, tokenOut, amountIn, 0)` — hardcoded zero minimum output. Completely unprotected from MEV/sandwich attacks. Often found in reward harvesting, fee conversion, or liquidation collateral sale paths.
- **FP:** `minAmountOut` parameter exposed to caller or computed from oracle. Internal-only swap with access control. Documented acceptance of slippage risk for small amounts.


**195. Transient Storage Reentrancy Guard in Delegatecall Context**

- **D:** Reentrancy guard uses `TSTORE`/`TLOAD` but in a delegatecall proxy context. Transient storage is per-address, not per-contract — proxy's transient storage is shared across all facets/implementations. Guard in one facet doesn't protect against reentry into different facet.
- **FP:** Shared transient storage slot used by all facets (single guard). Regular storage-based reentrancy guard. No delegatecall architecture.


**196. Uniswap V4 Hook Callback Authorization**

- **D:** Hook contract's callback functions (`beforeSwap`, `afterSwap`, etc.) don't validate `msg.sender == poolManager`. Anyone can call hook functions directly, manipulating cached state or triggering unintended side effects.
- **FP:** `require(msg.sender == address(poolManager))` in every callback. `onlyPoolManager` modifier. Hook uses `BaseHook` from Uniswap V4 periphery.


**197. Uniswap V4 Cached State Desynchronization**

- **D:** Hook caches pool state (`sqrtPriceX96`, `liquidity`, `tick`) in `beforeSwap` but state changes during the swap. `afterSwap` reads stale cached values for fee calculations or rebalancing decisions.
- **FP:** State re-read from pool in `afterSwap`. Cache explicitly invalidated between hooks. No cross-hook state dependency.

---

---


**198. Custom Access Control Without Two-Step Transfer**

- **D:** Hand-rolled `setOwner(newOwner)` with single-step transfer. Typo in `newOwner` address permanently locks out admin access with no recovery mechanism.
- **FP:** OZ `Ownable2Step` used. Multisig as owner (typo requires multiple signers). Timelock with cancel capability. Recovery mechanism via governance.


**199. Inconsistent Pausable Coverage**

- **D:** Contract imports `Pausable` but applies `whenNotPaused` modifier inconsistently. Some fund-moving operations (withdraw, transfer, liquidate) lack pause protection, allowing drain during emergency pause intended to freeze all operations.
- **FP:** All state-changing functions have `whenNotPaused`. Intentionally unpaused functions documented (e.g., emergency withdraw). Pause coverage verified in tests.


**200. OpenZeppelin Version Confusion (v4 vs v5)**

- **D:** Contract overrides `_beforeTokenTransfer` (OZ v4 hook) while importing OZ v5, where the hook was replaced with `_update`. Override silently never executes — access control, transfer restrictions, or enumerable tracking bypassed.
- **FP:** Confirmed OZ version consistency. Contract uses `_update` override for v5. No OZ token base inherited.


**201. Approval to Arbitrary User-Supplied Address (Aggregator/Router Pattern)**

- **D:** Router/aggregator calls `token.approve(userSuppliedPool, MAX_UINT)` where pool address comes from user calldata without allowlist validation. Attacker supplies malicious "pool" that calls `transferFrom` to drain all approved tokens.
- **FP:** Pool addresses validated against factory or hardcoded allowlist. Approval limited to exact amount per operation (`approve(pool, amountIn)` followed by `approve(pool, 0)`). No persistent approvals.


**202. External Call Failure DoS in Batch Operations**

- **D:** Batch operation (reward distribution, multi-user withdrawal, liquidation queue) reverts entirely if any single external call fails. One blacklisted address or reverting contract blocks all users in the batch.
- **FP:** Per-item `try/catch` with skip-on-failure. Pull-over-push pattern. Failed items queued for retry. Event emitted for failed transfers.


**203. Storage Bloat Attack (Unbounded Mapping/Array Growth)**

- **D:** Attacker fills user-controlled mappings/arrays without economic limits (e.g., `userTokens[user].push(attacker_token)` for each of thousands of fake tokens). Functions iterating over this array hit block gas limit.
- **FP:** Array size bounded (`require(arr.length < MAX)`). Economic deterrent (cost per entry). Pagination for iteration. EnumerableSet with bounded operations.


**204. Timestamp Griefing (Lock Period Reset by Third Party)**

- **D:** Any address can trigger a timestamp reset on another user's lock. Pattern: `deposit(address user, uint256 amount)` where `amount` can be 1 wei and resets `lockExpiry[user] = block.timestamp + LOCK_DURATION`. Attacker calls `deposit(victim, 1)` perpetually extending victim's lock.
- **FP:** Lock timestamp only updatable by the locked user themselves. Minimum deposit enforced. Lock extension only possible, never reset (uses `max(existing, new)`).


**205. Self-Destruct Force-Feed Breaking Strict Balance Equality**

- **D:** Contract uses `require(address(this).balance == expectedBalance)` with strict equality. Attacker force-feeds ETH via `selfdestruct(target)`, permanently breaking the equality check and DoS-ing the function.
- **FP:** Internal balance tracking (never reads `address(this).balance` for logic). Uses `>=` or `<=` comparisons. Contract designed to accept arbitrary ETH.


**206. Inconsistent Guard Coverage Across Semantically Equivalent Functions**

- **D:** Contract has multiple functions that perform the same logical operation (e.g., `transfer` and `transferFrom`, or `deposit` and `depositFor`) but security guards (pause, reentrancy, access control) are only applied to one. Attacker uses the unguarded variant.
- **FP:** Shared internal function with guards called by all public variants. Modifier applied uniformly. Single entry point for each operation.


**207. Missing Initialization of Inherited State in Upgradeable Contracts**

- **D:** Upgradeable contract calls `__Ownable_init()` but forgets to call `__ReentrancyGuard_init()`, `__Pausable_init()`, or other inherited initializers. Reentrancy guard status is 0 (uninitialized), which may not provide protection. Pausable state is unpredictable.
- **FP:** All inherited `__*_init()` functions called in `initialize()`. OZ Upgradeable contracts used with complete initialization chain. Tests verify all state correctly initialized post-deployment.

---

---


**208. EIP-7702 Cross-Chain Delegation Replay**

- **D:** EIP-7702 authorization signatures for EOA-to-contract delegation miss `chainId` in the signed tuple. Attacker replays the same delegation signature on another chain, hijacking the EOA's execution context on chains the user never intended to delegate on.
- **FP:** `chainId` included in EIP-7702 authorization tuple. Wallet UI displays target chain before signing. Per-chain delegation with separate signatures.


**209. EIP-7702 Delegate Hijacking**

- **D:** Malicious contract becomes an EOA's delegate via social engineering or phishing. Once delegated, all transactions to the EOA execute the malicious contract's code in the EOA's context — draining funds, approving tokens, or modifying state. Persists until the user explicitly revokes delegation.
- **FP:** Delegation target is a well-known, audited contract (e.g., Safe module). Wallet prompts clearly distinguish delegation from normal signing. Revocation mechanism is accessible and documented. Time-limited delegation with automatic expiry.


**210. EIP-7702 Delegation Phishing**

- **D:** User is socially engineered into signing what appears to be a normal typed-data message, but is actually an EIP-7702 authorization tuple. Once submitted, any subsequent transaction to the user's EOA executes attacker-controlled code. Unlike approval phishing, this controls ALL future interactions with the EOA.
- **FP:** Wallet UI clearly distinguishes EIP-7702 delegation requests from standard EIP-712 messages. Warning displayed for delegation to unverified contracts. Transaction simulation shows delegation effect before signing.


**211. EIP-7702 EOA Nested Reentrancy**

- **D:** With EIP-7702, EOAs can have code. A delegated EOA receiving ETH or tokens triggers fallback/receive in the delegate contract, creating new reentrancy surfaces that didn't exist when the counterparty was a plain EOA. Protocols assuming EOAs can't have callbacks are vulnerable.
- **FP:** `nonReentrant` on all external-call-bearing functions regardless of counterparty type. No assumption that `tx.origin == msg.sender` means "safe EOA." CEI pattern followed universally.


**212. ERC-4337 Paymaster Drain via Crafted UserOperations**

- **D:** Attacker crafts UserOperations that pass paymaster validation but consume maximum gas during execution. Paymaster pays for gas but the operation accomplishes nothing useful for the paymaster's business model. Repeated submissions drain the paymaster's deposit in the EntryPoint.
- **FP:** Paymaster validates operation purpose (not just signature). Gas limits per UserOperation and per-user rate limits enforced. Paymaster deposit monitored with automatic pause at low threshold. Off-chain simulation before on-chain validation.


**213. ERC-4337 Validation-Execution Phase Confusion**

- **D:** Logic that should only run during validation phase (signature checks, nonce verification) executes during the execution phase or vice versa. Banned opcodes in validation (ERC-7562) cause bundler rejection, while missing validation in execution allows unauthorized operations.
- **FP:** Clear separation between `validateUserOp` and execution functions. No storage access in validation beyond sender's associated storage. Compliance with ERC-7562 opcode restrictions verified. Bundler simulation tests pass.


**214. Uniswap V4 Hook Data Manipulation**

- **D:** Hook reads parameters from `hookData` bytes passed through the swap path. Attacker crafts `hookData` to manipulate hook behavior — bypassing fee calculations, altering routing decisions, or triggering unintended state changes in the hook contract.
- **FP:** `hookData` validated against expected schema (length, types). Critical parameters derived from pool state, not `hookData`. Hook ignores or doesn't use `hookData`. Authenticated `hookData` (signed by trusted relayer).


**215. Known Solidity Compiler Bug Exploitation**

- **D:** Contract compiled with a Solidity version containing a known CVE or documented compiler bug (e.g., ABI encoding bugs, optimizer errors, memory corruption). The specific code patterns in the contract trigger the bug, producing incorrect bytecode that behaves differently from source-level semantics.
- **FP:** Compiler version checked against https://docs.soliditylang.org/en/latest/bugs.html. Version is latest stable or a version where known bugs don't affect patterns used. `forge inspect` or `slither --detect solc-version` run. No vulnerable code patterns present for the specific compiler bugs.


**216. Unsafe ABI Decoding of Untrusted Calldata**

- **D:** Raw `abi.decode(data, (...))` called on user-supplied or cross-contract calldata without length or schema validation. Truncated, malformed, or oversized payloads cause unexpected reverts, type confusion, or silent misinterpretation of parameters. In assembly-based decoders, out-of-bounds reads return zero.
- **FP:** `data.length >= expectedMinLength` checked before decoding. `try/catch` wrapping untrusted decode. Assembly decoders validate bounds. Only trusted internal calldata decoded without checks.


**217. Donation Attack on Balance-Based Vault Accounting**

- **D:** Vault uses `token.balanceOf(address(this))` as `totalAssets` instead of internal accounting. Attacker donates tokens directly (bypassing `deposit()`), inflating the share price. Subsequent depositors receive fewer shares than expected. Combined with flash loans, enables share price manipulation for profit extraction.
- **FP:** Internal `totalAssets` tracking updated only via `deposit()`/`withdraw()`. `balanceOf` never used for share price calculation. Virtual assets/shares offset applied. Donation amount ignored or swept separately.


**218. LP Token Price Manipulation via Reserve Ratio**

- **D:** Protocol prices LP tokens using `reserve0 * price0 + reserve1 * price1 / totalSupply` — directly reading pool reserves that are manipulable via flash loans. Attacker skews reserves in a single transaction, inflates LP token value, uses overpriced LP as collateral, borrows against it.
- **FP:** LP token pricing uses invariant-based formula (Alpha Homora fair LP pricing). Reserves not read directly — price derived from oracle prices and the constant-product invariant. TWAP-based LP valuation. Flash loan resistant pricing.


**219. Circular Flash Loan Amplification Across Protocols**

- **D:** Attacker uses flash-loaned assets to deposit in Protocol A, borrows from A, deposits borrowed assets in Protocol B, borrows from B, and repeats. Creates leveraged positions across multiple protocols in a single transaction with zero initial capital, amplifying any exploit (oracle manipulation, governance attack) by orders of magnitude.
- **FP:** Flash loan detection via `require(block.number > depositBlock)` or same-block withdrawal restriction. Cross-protocol exposure limits. Deposit cooldown periods. Conservative LTV ratios that make circular amplification unprofitable after fees.


**220. Off-Chain Infrastructure Compromise (Frontend/Signer)**

- **D:** Frontend UI, signer service, npm dependency, or multisig wallet interface is compromised. Users interact with a legitimate-looking but attacker-controlled interface that crafts malicious transactions (different calldata than displayed). The on-chain contracts are correct but users are tricked into signing harmful transactions. Classic pattern: Bybit $1.4B (2025) via compromised Safe{Wallet} UI.
- **FP:** Transaction simulation shown in wallet before signing. Calldata decoded and displayed in human-readable form. Subresource integrity (SRI) headers on frontend. Reproducible builds. Multi-party verification of transaction calldata before multisig signing. Hardware wallet with on-device transaction parsing.


**221. Embedded Library Code Missing Security Patches**

- **D:** Contract contains copy-pasted library source code (e.g., OpenZeppelin, Solmate) instead of importing from a versioned dependency. When the library publishes security fixes, the embedded copy remains vulnerable. The vulnerability is invisible in dependency audits since no import exists to version-check.
- **FP:** All library code imported from versioned npm/git dependencies. No copy-pasted external code in project source. Dependency versions pinned and regularly updated. Custom modifications to libraries documented and tracked.


**222. Restaking Cascading Slashing Risk (EigenLayer-style)**

- **D:** Same stake secures multiple AVSs (Actively Validated Services) via restaking. A slashing event in one AVS can cascade — the same ETH is slashed by AVS-A, then AVS-B detects reduced stake and triggers its own slashing condition. Total slashing across all registered AVSs can exceed 100% of the original stake, creating insolvency.
- **FP:** Maximum aggregate slash exposure capped across all AVSs. Slashing amounts deducted from future AVS registrations. Insurance or reserve fund for cascading scenarios. Per-AVS stake isolation. Slashing events rate-limited across AVSs.


**223. Queue/List Poisoning via Dust Entries**

- **D:** Attacker fills a withdrawal queue, reward distribution list, or processing queue with thousands of dust-amount entries (1 wei each). Processing the queue requires iterating all entries, and the gas cost grows linearly. Eventually the queue becomes too expensive to process within block gas limits, permanently blocking legitimate withdrawals.
- **FP:** Minimum entry size enforced (`require(amount >= MIN_AMOUNT)`). Queue implements pagination/batch processing. Economic deterrent per queue entry (fee or deposit). Admin can prune dust entries. Max queue length enforced.


**224. Governance Spam Proposals via Low Deposit Threshold**

- **D:** Governance proposal creation requires a deposit or token threshold too low relative to token supply. Attacker floods governance with spam proposals, overwhelming voters and hiding malicious proposals among noise. Legitimate governance activity becomes impractical. Combined with voter apathy, a malicious proposal may pass unnoticed.
- **FP:** Proposal deposit proportional to token supply (e.g., 1% of total supply). Proposal rate limiting per address. Quorum requirements filter out low-engagement proposals. Proposal screening by guardians/council before on-chain vote. Deposit forfeited if proposal doesn't reach quorum.


**225. EIP-2612 Permit Frontrun DoS**

- **D:** A workflow like `permitAndDeposit`, `permitAndSwap`, or `permitAndRemoveLiquidity` assumes the inline `permit()` call must succeed. Attacker copies the signature from the mempool, front-runs `permit()`, consumes the nonce, and causes the victim's full transaction to revert on invalid nonce. The attacker does not steal approval-controlled funds directly; they brick the primary action and can repeatedly grief users.
- **FP:** `permit()` wrapped in `try/catch` or equivalent fallback logic. Main action proceeds if allowance is already sufficient. Failure of `permit()` does not revert the rest of the workflow. UX explicitly tolerates already-consumed nonces.


**226. Non-Standard Permit Integration Mismatch**

- **D:** Integration assumes all tokens implement EIP-2612 `permit(owner, spender, value, deadline, v, r, s)`. Token instead uses a non-standard permit shape (e.g., DAI-style `allowed` boolean, different nonce semantics, different domain separator, custom expiry rules). Contract signs or decodes the wrong payload, leading to failed approvals, replay assumptions, or broken authz fallbacks.
- **FP:** Token-specific permit paths implemented and tested per asset. DAI/RAI/GLM-style variants explicitly supported where relevant. If permit support is absent or non-standard, integration falls back to approve flow instead of assuming EIP-2612 compatibility.


**227. Unprotected SELFDESTRUCT**

- **D:** Contract exposes `selfdestruct`/`suicide` through a public or weakly restricted path. Attacker can permanently destroy the contract on pre-Dencun chains, force-send ETH to an arbitrary address, or brick dependent integrations. Even post-Dencun, the presence of `selfdestruct` in implementations, factories, or same-tx create/destroy flows can still be exploitable.
- **FP:** No reachable `selfdestruct` path from untrusted callers. Opcode absent entirely from business logic. Upgradeable implementations, clones, and delegatecall targets explicitly forbid `selfdestruct`. Chain-specific assumptions about EIP-6780 are documented and not relied on for safety.


**228. Sender Confusion Under Multicall / Forwarder Context**

- **D:** Code that should reason about the original signer uses raw `msg.sender` inside multicall, relayer, or trusted-forwarder flows. Hooks, accounting, authz, or recipient attribution then execute against the batching contract / forwarder instead of the real user. Common pattern: helper libraries like `_msgSender()` / `LibMulticaller.senderOrSigner()` exist, but one or more internal paths bypass them.
- **FP:** Every authorization- or attribution-sensitive path consistently uses the canonical sender abstraction for the architecture (`_msgSender()`, trusted forwarder context, multicaller helper). Tests cover direct call, multicall, and forwarded execution paths and assert identical authorization semantics.


**229. Missing Native-ETH Receive Path Bricks Protocol Flow**

- **D:** Contract participates in a native ETH flow (direct deposits, unwrap-to-ETH withdrawals, auction refunds, keeper payouts) but lacks a compatible `receive()` / payable fallback path, or assumes recipients can always accept ETH. Funds become stuck, user actions revert, or protocol state advances while value transfer fails. This shows up frequently in vaults, routers, auction houses, and refund paths that mix ERC-20 and ETH handling.
- **FP:** Contract has an explicit payable receive path when native ETH is part of the design. Outbound ETH sends use checked `call{value: ...}` with fallback accounting or pull-payment recovery. Native and ERC-20 code paths are separated clearly and tested against smart-contract recipients (Safe, proxies, multisigs) as well as EOAs.
