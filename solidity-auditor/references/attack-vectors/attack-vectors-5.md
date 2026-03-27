341 total attack vectors

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

**230. Algorithmic Complexity Gas DoS**

- **D:** Nested loops, combinatorial matching, or recursive computation with superlinear gas cost (O(n²), O(2ⁿ)). At production scale, execution exceeds block gas limit, bricking the function.
- **FP:** O(n) or O(n log n) algorithm. Input capped (`require(n <= MAX)` gas-tested). Computation paginated/batched. Off-chain compute with on-chain verification.

**231. Blacklist and Whitelist Not Mutually Exclusive**

- **D:** Address holds both `BLACKLISTED` and `WHITELISTED` roles. Whitelist-gated paths don't check blacklist — blacklisted address bypasses restrictions via whitelist.
- **FP:** Adding to one auto-removes from other. Single enum role per address. Both checks applied on every restricted path.

**232. Cross-Chain Sandwich via Bridge Parameter Exposure**

- **D:** Bridge tx on source chain exposes destination swap params (`amountOutMin`, token, amount) in plaintext. Attacker frontruns on destination L2 to manipulate pool, backruns after bridge tx executes.
- **FP:** Encrypted/committed bridge payloads. Destination swap recalculates slippage via oracle. Intent-based bridge (solver fills off-chain).

**233. Dead Code After Return Statement**

- **D:** Critical state update or validation placed after `return` — unreachable. Failures undetected, events never emitted, state never updated.
- **FP:** All critical logic precedes `return`. Compiler unreachable-code warnings addressed.

**234. Deprecated Gauge Blocks Claiming Accrued Rewards**

- **D:** Killing/deprecating gauge blocks `claimReward()` for already-accrued, unclaimed rewards — users who earned before deprecation cannot retrieve.
- **FP:** Kill stops future accrual only — claim remains active for pre-kill balances. Emergency claim bypasses active check.

**235. Dutch Auction Price Decay Underflow**

- **D:** `currentPrice = startPrice - (decayRate * elapsed)`. Past zero-point: underflow reverts (0.8+) or wraps to `type(uint256).max` (<0.8). Auction unfinishable.
- **FP:** Floor price via `min()` or ternary at duration boundary. Reserve price enforced.

**236. EIP-7702 Code Inspection Opcode Invalidation**

- **D:** `extcodesize`, `extcodehash`, `extcodecopy` on delegated EOA operate on the 23-byte `0xef0100` delegation stub, not the delegate's code. `isContract()` checks misroute delegated EOAs. `extcodehash` comparisons against known implementation hashes fail. Proxy detection and ERC-1167 clone verification return unexpected results.
- **FP:** No security-critical branching on `extcodesize`/`extcodehash`. Uses `CODESIZE`/`CODECOPY` within execution context (which follow delegation) rather than `EXT*` variants.

**237. EIP-7702 Dual Signature Validation Confusion**

- **D:** Delegated EOA supports both ECDSA (private key) and ERC-1271 (`isValidSignature` from delegate). Protocol checking only one path lets attacker exploit the other. Signature replay across redelegation — message signed under Delegate A interpreted differently by Delegate B.
- **FP:** OZ `SignatureChecker.isValidSignatureNow` used. ERC-1271 checked first for accounts with code, ECDSA fallback for codeless. Signatures include delegate address in EIP-712 domain.

**238. EIP-7702 ERC-721/ERC-1155 Callback Revert on Delegated EOA**

- **D:** `safeTransferFrom` to delegated EOA triggers `onERC721Received`/`onERC1155Received` (recipient has code). If delegate doesn't implement callback, transfer reverts — breaks distribution loops and airdrops.
- **FP:** Uses `transferFrom` (no callback). Fallback path on callback failure. Skip-and-accrue pattern.

**239. EIP-7702 tx.origin == msg.sender Bypass**

- **D:** `require(tx.origin == msg.sender)` as EOA gate or reentrancy guard. Delegated EOA passes check while executing arbitrary contract logic — enables flash loans, atomic governance manipulation, and reentrancy through "EOA-only" functions.
- **FP:** Additional `require(msg.sender.code.length == 0)` check (delegated EOAs have 23-byte `0xef0100` stub). Function protected by time-lock, multi-sig, or past-block snapshot.

**240. ERC4626 convertToAssets Used Instead of previewWithdraw**

- **D:** Integration calls `convertToAssets(shares)` to estimate withdrawal proceeds — excludes fees/slippage per spec. Downstream logic (health checks, rebalancing) operates on inflated values.
- **FP:** `previewWithdraw()`/`previewRedeem()` used for estimates. No withdrawal fees. Fee delta accounted separately.

**241. ERC4626 maxDeposit Returns Non-Zero When Paused**

- **D:** `maxDeposit()` returns `type(uint256).max` when paused. Integrators read "deposits accepted," attempt deposit, revert. Per ERC4626, must return 0 when deposits would revert.
- **FP:** `maxDeposit()` returns 0 when paused. Integrators use `try deposit()`.

**242. Emergency Mode State Machine Incompleteness**

- **D:** Emergency mode pauses operations but accrual continues (rewards, interest, fees). On exit, accumulated state not reconciled — stuck funds. Also: emergency withdrawal omits cleanup of associated records (staking, reward debt, queue), leaving orphaned state.
- **FP:** Emergency freezes all accrual. Exit reconciles before resuming. Emergency withdrawal atomically cleans associated records. Invariants re-validated on exit.

**243. Expired Oracle Version Silently Assigned Previous Price**

- **D:** In request-commit oracle patterns (Pyth, keepers), expired/unfulfilled request assigned last valid price instead of reverting. Pending orders execute at stale prices.
- **FP:** Expired versions return `price = 0` or `valid = false`, forcing cancellation. Staleness threshold per-request. Fallback oracle.

**244. False Existence Detection via Balance Check at Computed Address**

- **D:** Contract checks pool/pair existence via `balanceOf()` at computed CREATE2 address. Pre-sent tokens make `balanceOf > 0` before deployment — logic assumes pool exists, attempts swap, reverts.
- **FP:** Existence via factory: `factory.getPair(A, B) != address(0)`. `code.length > 0` checked. Pool verified by calling pool-specific function (`getReserves()`, `token0()`).

**245. Fixed-End Auction Last-Block Sniping**

- **D:** On-chain auction with fixed `endBlock`/`endTime` and no extension. Validator places winning bid in final block, censors competitors. Pattern: `require(block.timestamp <= endTime)` without anti-snipe extension.
- **FP:** Auction extends on late bids. Candle auction (random end). Sealed-bid second-price. Batch/uniform-price clearing.

**246. Funding Rate Derived from Single Trade Price**

- **D:** Perp funding rate uses last trade price as mark. Single self-trade at extreme price skews funding — attacker profits on opposing position.
- **FP:** Mark from TWAP or external oracle. Funding rate capped per period. VWAP used.

**247. JIT Liquidity on Deterministic TWAMM Virtual Order Execution**

- **D:** TWAMM virtual orders execute at deterministic intervals with publicly readable sale rates. Attacker adds concentrated liquidity before execution, captures fees from predictable flow, removes immediately. No mempool needed — timing fully on-chain.
- **FP:** Time-weighted fee discount for liquidity added within execution window. Execution boundary randomized. Same-block positions excluded from TWAMM fee capture.

**248. LST Redemption-Rate vs Market-Price Divergence**

- **D:** LST collateral valued at protocol redemption rate (`stETH.getPooledEthByShares()`) while market trades at discount. Borrower posts overvalued collateral, borrows against inflated value. During stress, redemption rate stays high while market drops — bad debt.
- **FP:** Market price feed (Chainlink stETH/ETH) used. `min(redemptionRate, marketPrice)` for valuation. LTV haircut for historical deviation. Circuit breaker on divergence.

**249. Loss-Versus-Rebalancing (LVR) in Constant-Function AMMs**

- **D:** AMM with constant-function pricing and static fees. Searchers continuously arbitrage stale pool price against external markets, extracting from LPs on every price movement. Concentrated liquidity amplifies extraction.
- **FP:** Dynamic fee adjusting for volatility (Uniswap V4 hooks). MEV-aware design (batch auctions, CoW AMM). Fee tier covers expected LVR for pair volatility.

**250. Memory Struct Copy Not Written Back to Storage**

- **D:** `MyStruct memory s = myMapping[key]` creates a copy — mutations don't persist. Also: internal function with `memory` parameter silently copies storage on call. Pattern: `_updatePosition(Position memory pos)` called with `positions[user]`.
- **FP:** Uses `storage` keyword. Explicitly writes back: `myMapping[key] = s`. Internal function parameter declared as `storage`.

**251. Permissionless accrueInterest Griefing**

- **D:** Permissionless `accrueInterest()` called at short intervals — each computes zero interest (rounding) but advances timestamp, systematically suppressing accumulation.
- **FP:** Minimum accrual interval enforced. Precision ensures per-block interest > 0. Access-restricted accrual.

**252. Profit Tracking Underflow Blocks Withdrawals**

- **D:** Vault tracks cumulative profit. Strategy loss exceeding recorded profit causes `totalProfit -= loss` to underflow (revert on 0.8+), bricking all withdrawals.
- **FP:** Loss capped: `totalProfit -= min(loss, totalProfit)`. Signed integer for profit/loss. Per-strategy tracking.

**253. Quorum Computed from Live Supply, Not Snapshot**

- **D:** `quorum = totalSupply() * quorumBps / 10000` reads current supply. Attacker inflates supply after proposal creation, lowering effective quorum percentage.
- **FP:** Quorum snapshotted at proposal creation. Fixed absolute quorum. Supply changes don't affect active proposals.

**254. Self-Matched Orders Enable Wash Trading**

- **D:** Order matching doesn't verify `maker != taker`. User submits both sides to farm rewards, inflate volume, bypass royalties, or extract fee rebates.
- **FP:** `require(maker != taker)`. Volume rewards use time-weighted averages. Royalty enforced regardless of counterparty.

**255. Timelock Anchored to Deployment, Not Action**

- **D:** Timelock measured from deployment, not action queue time. Once initial delay elapses, all future actions execute instantly — permanent bypass.
- **FP:** `executeAfter = block.timestamp + delay` set at queue time. OZ TimelockController.

**256. TWAP Accumulator Not Updated During Sync or Skim**

- **D:** `sync()`/`skim()` updates reserves but doesn't call `_update()` to advance TWAP accumulator. Stale TWAP enables manipulation via sync-then-trade.
- **FP:** `sync()` calls `_update()` before overwriting reserves. TWAP from external oracle. Uniswap V3 `observe()` used.

**257. Vault Harvest Front-Running**

- **D:** `harvest()` claims yield and increases `totalAssets()` atomically, raising share price. Attacker deposits before harvest, withdraws after — captures yield without duration exposure.
- **FP:** Yield distributed over time (drip/streaming). Deposit lock spans harvest cycle. Harvest via keeper in private mempool. Performance fee at harvest.

**258. Withdrawal Queue Bricked by Zero-Amount Entry**

- **D:** FIFO withdrawal queue hits cancelled/zeroed entry that causes `break` or revert instead of skip, permanently blocking all subsequent withdrawals.
- **FP:** Queue skips zero-amount entries. Cancellation removes or marks entry processed. Linked list allows removal.

**259. Withdrawal Queue Rate Lock-In Front-Run**

- **D:** `requestWithdraw()` locks exchange rate at request time, not claim time. Attacker front-runs pending loss event (slashing, depeg), locks pre-loss rate. Remaining depositors absorb full loss.
- **FP:** Conversion at claim time using worst of request/claim rate. Same-block deposit+request prevented. Loss realization atomic with share price update.

**260. notifyRewardAmount Overwrites Active Reward Period**

- **D:** `notifyRewardAmount(newAmount)` replaces current period — undistributed rewards silently lost, not carried forward.
- **FP:** New notification adds remaining: `rewardRate = (newAmount + remaining) / duration`. Only callable by designated distributor with timelock.

**261. Reward Rate Changed Without Settling Accumulator**

- **D:** Admin updates emission rate without calling `updateReward()` first. New rate retroactively applied to entire elapsed period, overpaying or underpaying.
- **FP:** Rate-change calls `updateReward()` before applying new rate. Modifier auto-settles on every state change.

**262. Reward Accrual During Zero-Depositor Period**

- **D:** Time-based reward distribution starts at vault deployment but no depositors exist yet. First depositor claims all rewards accumulated during the empty period regardless of deposit size or timing.
- **FP:** Rewards only accrue when `totalSupply > 0`. Reward start time set on first deposit. Unclaimed pre-deposit rewards sent to treasury or burned.

**263. Unclaimed Reward Tokens from Underlying Protocol**

- **D:** Vault deposits into yield protocol (Morpho, Aave, Convex) that emits reward tokens but never calls `claim()`. Rewards accumulate inaccessibly in vault or underlying.
- **FP:** Explicit `claimRewards()` harvests and distributes. Reward tokens tracked dynamically. Vault sweeps unexpected balances to treasury.

**264. Vault Insolvency via Accumulated Rounding Dust**

- **D:** Vault tracks `totalAssets` as a storage variable separate from `token.balanceOf(vault)`. Solidity's floor rounding on each deposit/withdrawal creates tiny overages — user receives 1 wei more than burned shares represent. Over many operations `totalAssets` exceeds actual balance, causing last withdrawers to revert.
- **FP:** Rounding consistently favors the vault (round shares up on deposit, round assets down on withdrawal). OZ Math with `Rounding.Ceil`/`Rounding.Floor` applied correctly.

**265. Adverse Selection — Passive LP Value Extraction via Selective JIT**

- **D:** No time-weighting or lock on fee distribution. JIT providers enter only during high-fee moments and exit during adverse moves. Passive LPs bear 100% IL but share fees with JIT providers bearing zero IL.
- **FP:** Fee share time-weighted by duration. Dynamic fee increases during volatility. Withdrawal cooldown makes selective entry costly.

**266. Tick Crossing Fee Accounting Manipulation via JIT**

- **D:** On tick crossing, `feesPerLiquidityOutside` flips (`global - outside`). JIT provider adds at tick boundary after fees accumulate but before crossing — flip credits position with pre-existing fees it didn't earn.
- **FP:** `feesPerLiquidityInsideLast` set at creation; crossing correctly partitions pre/post fees. Same-block creation and crossing yield zero claimable.

**267. Fee Accumulation Rounding Extraction via Large JIT Position**

- **D:** `feesPerLiquidity += (feeAmount << 128) / totalLiquidity`. Large JIT position inflates `totalLiquidity` — per-unit increment rounds to zero for existing LPs while JIT provider captures truncated amount.
- **FP:** Sufficient precision (Q128+) ensures rounding loss < 1 wei at realistic ratios. Protocol minimum fee increment.

**268. Atomic JIT Liquidity via Flash Accounting**

- **D:** Flash accounting / lock-callback allows add-liquidity + swap + remove-liquidity atomically with zero capital. No minimum hold duration or fee decay. Attacker adds concentrated liquidity at current tick, swap executes through it, liquidity removed — all in one callback.
- **FP:** Minimum hold duration enforced. Fee share weighted by time-in-pool. Withdrawal fee on short-lived positions. Flash callback restricted from `updatePosition`.

**269. Bridge Global Rate Limit Griefing**

- **D:** Bridge enforces global throughput cap not segmented by user. Attacker fills limit bridging cheap tokens back and forth, blocking all legitimate users during cooldown.
- **FP:** Per-user rate limits. Segmented by token/route. Whitelist for high-value transfers.

**270. Governance Proposal Executable Before Voting Period Ends**

- **D:** `execute()` checks quorum/majority but not `block.timestamp >= proposal.endTime`. Once quorum met, proposal executable immediately — cuts voting window short.
- **FP:** `require(block.timestamp >= proposal.endTime)`. OZ Governor enforces `ProposalState.Succeeded`.

**271. Idle Asset Dilution from Sub-Vault Deposit Caps**

- **D:** Aggregator vault accepts deposits without checking sub-vault capacity. Excess assets sit idle earning zero yield but dilute share price for all depositors.
- **FP:** `maxDeposit()` reflects combined sub-vault remaining capacity. Deposits revert when no capacity remains. Idle assets auto-routed to fallback yield.

**272. Lazy Epoch Advancement Skips Reward Periods**

- **D:** Epoch advances only on user interaction. No interaction = never advanced — rewards miscalculated or lost when next interaction retroactively applies to wrong epoch.
- **FP:** Keeper advances epochs independently. Catch-up loop processes skipped epochs. Continuous (non-epoch) reward accrual.

**273. Liquidated Position Continues Accruing Rewards**

- **D:** Position liquidated (balance zeroed) but not removed from reward distribution. `rewardDebt` not reset — phantom rewards accrue or are locked permanently.
- **FP:** Liquidation calls `_withdrawRewards()` before zeroing. Reward system checks `balance > 0` before accruing.

**274. Liquidation Arithmetic Reverts at Extreme Price Drops**

- **D:** Collateral drops 95%+ — `collateralNeeded = debt / price` exceeds available collateral, liquidation math overflows/reverts. Position unliquidatable, bad debt locked.
- **FP:** `collateralSeized` capped at position's total. Full seizure with remaining bad debt socialized.

**275. Liquidation Blocked by External Pool Illiquidity**

- **D:** Liquidation swaps collateral for debt token via external DEX. Drained pool reverts swap, making liquidation impossible. Bad debt accumulates.
- **FP:** Liquidation accepts collateral directly. Fallback path uses different DEX. Liquidator provides debt token.

**276. Liquidation Discount Applied Inconsistently Across Code Paths**

- **D:** One path calculates debt at face value, another applies discount. Mismatch causes underflow or leaves residual bad debt unaccounted.
- **FP:** Discount applied consistently across all liquidation paths. Single source of truth for discounted value.

**277. Loan State Transition Before Interest Settlement**

- **D:** Repay sets loan to `Repaid` before interest settled. Accrual skips `Repaid` loans — accumulated interest permanently uncollectable.
- **FP:** `settleInterest()` before state transition. `require(msg.value >= principal + accruedInterest)`.

**278. No-Bid Auction Fails to Clear State**

- **D:** Auction expires with no bids but finalization doesn't clear lien/escrow data — collateral locked with no return path or re-auction mechanism.
- **FP:** No-bid finalization returns collateral and clears state. Auto re-auction. Timeout-based release.

**279. Partial Redemption Fails to Reduce Tracked Total**

- **D:** Partial redemption fill doesn't reduce `totalQueuedShares`/`totalPendingAssets` proportionally. Inflated total skews share price.
- **FP:** Partial fill reduces tracked totals proportionally. Per-request tracking. Atomic full-or-nothing redemptions.

**280. Repeated Liquidation of Same Position**

- **D:** Liquidation doesn't flag position as processed. After partial liquidation, position still appears undercollateralized — second liquidator seizes collateral beyond intent.
- **FP:** Position marked `liquidated` or deleted. `require(status != Liquidated)`. Post-liquidation health check.

**281. Share Redemption at Optimistic Rate**

- **D:** Shares redeemed at projected end-of-term rate rather than current realized rate. Early redeemers take more than proportional share — late redeemers find vault depleted.
- **FP:** Redemption uses current realized rate (`totalAssets() / totalSupply()`). Withdrawal queue enforces proportional access. Early redemption penalty applied.

**282. State Record Overwrite Without Existence Check**

- **D:** Mapping entry (refund, withdrawal, order) written without checking if key occupied. Overwrites legitimate user's record — blocks claim, redirects funds, or poisons state. Pattern: `records[key] = newData` without `require(records[key].amount == 0)`.
- **FP:** Existence check before write. Nonce/hash-based keys prevent collision. Append-only structure. Old entry processed before overwrite.

**283. Withdrawal Rate Limit Bypassed via Share Transfer**

- **D:** Per-address withdrawal limit bypassed by transferring shares to fresh addresses — each gets a fresh limit.
- **FP:** Limit tracks underlying position, not address. Shares non-transferable or transfer resets allowance.

---

---


**284. EIP-7702 Whitelist / Allowlist Privilege Borrowing**

- **D:** Whitelisted address signs EIP-7702 delegation. Attacker includes that authorization in their tx, calls the delegated address — target contract sees `msg.sender == whitelisted_address`. One phished signature becomes a permanent gateway for unlimited actors.
- **FP:** Access control rejects delegation designator prefix (`0xef0100`). Whitelist requires per-call signature, not just address check.


**285. Missing Slippage Protection on Vault Withdraw/Redeem**

- **D:** ERC4626 `withdraw`/`redeem` accept no slippage parameter. Exchange rate changes between submission and execution (yield, donations, losses). Users receive fewer assets or burn more shares than expected.
- **FP:** Fixed 1:1 exchange rate. Custom `withdrawWithSlippage` wrapper. Frontend simulation with revert. Loss-proof yield source.


**286. Nonce Not Incremented on Reverted Execution**

- **D:** Meta-tx nonce checked before execution but incremented only on success. Reverted inner call leaves nonce unchanged — same signed message replayable until it succeeds.
- **FP:** Nonce incremented before execution (CEI). Incremented in both success/failure paths. Deadline-based expiry.


**287. Minimum Lock Period Bypass via Position Modification**

- **D:** Lock enforced on creation/removal but not `increaseLiquidity`/`decreaseLiquidity`. Attacker maintains minimal position, increases massively before profitable swap, decreases after — bypassing lock.
- **FP:** Lock applies to any increase — `lastModifiedBlock` updated on every change. Fee accrual begins after lock for newly added liquidity.


**288. Governance Precondition Manipulation**

- **D:** Parameter updates have preconditions based on manipulable state (TVL, liquidity). Adversary inflates/deflates state to block updates — DoS on governance prevents critical changes (fee adjustments, security patches, oracle swaps).
- **FP:** Preconditions use time-weighted/snapshot values. No state-dependent preconditions. Admin emergency override. Absolute thresholds, not relative to manipulable state.


**289. Hook Callback Reentrancy for Fee Bypass**

- **D:** User-controlled hook (beforeClaim, onReceive) fires mid-operation before fee accounting finalized. User reenters different contract via alternate path that skips fee deduction.
- **FP:** Global reentrancy lock (not per-function). Hook fires after all state changes and fees finalized. Cross-contract mutex.


**290. Cross-Message Token Identity Mismatch**

- **D:** Multi-hop/cross-chain flow uses user-controlled token fields per leg without cross-validation. Attacker deposits token A but encodes token B — destination withdraws contract's balance of B. Pattern: `depositedToken`, `swapFromToken`, `swapToToken`, `withdrawalToken` specified independently.
- **FP:** `require(depositedToken == message.fromToken)` at deposit. Swap output validated against withdrawal token. Stateless relay holds no funds. Fields derived from on-chain state.


**291. First-Swap Extraction on Newly Created Pools**

- **D:** New pool with minimal liquidity — first significant swap is extremely fee-rich. Attacker front-runs by adding concentrated liquidity, captures outsized fees, removes. Extreme: initializes at skewed price, profits from arb correction.
- **FP:** Minimum locked seed liquidity (Uniswap V2 `MINIMUM_LIQUIDITY`). Fee ramp-up for new pools. Anti-sniping delay on creation.


**292. Self-Delegation Doubles Voting Power**

- **D:** Self-delegation adds votes to delegate (self) without subtracting undelegated balance — power counted twice: held tokens + delegated votes.
- **FP:** Delegation subtracts from holder's direct balance. Self-delegation is no-op or explicitly handled. OZ Votes used.


**293. MEV Withdrawal Before Bad Debt Socialization**

- **D:** External event (liquidation, exploit, depeg) causes vault loss. MEV actor observes pending loss-causing tx in mempool and front-runs a withdrawal at pre-loss share price, leaving remaining depositors to absorb the full loss.
- **FP:** Withdrawals require time-delayed request queue (epoch-based or cooldown). Loss realization and share price update are atomic. Private mempool used for liquidation txs.


**294. Empty Swap Path Bypasses Token Validation**

- **D:** Empty swap data/zero-length path returns input amount without swapping — and without validating input == output token. Attacker skips swap, receives output token from contract's balance. Pattern: `if (swapData.length == 0) return amount;` without `require(fromToken == toToken)`.
- **FP:** Empty path enforces `require(fromToken == toToken)`. Swap mandatory. Reverts on empty data. Post-swap balance delta check.


**295. Sentinel / Placeholder Address Operations**

- **D:** Code branches on sentinel (`address(0)`, `0xEeEe...`, `type(uint256).max`) for ETH/special cases. Special branch omits validations the normal branch performs. Also: ERC20 calls on sentinel — high-level reverts (no code), low-level succeeds silently.
- **FP:** Sentinel branch has equivalent validation. No ERC20 calls on sentinels. WETH wrapping instead of dual-path. Early detection routes to independent handler.


**296. EIP-7702 Delegation Persists on Transaction Revert**

- **D:** Delegation designator is set BEFORE transaction execution. If tx body reverts, delegation is NOT rolled back — EOA permanently has new code despite reverted state changes.
- **FP:** Delegation requires EOA holder's explicit signature. Wallet UI shows active delegation status.


**297. Open Interest Tracked with Pre-Fee Position Size**

- **D:** OI incremented by full position size before fee deduction. Actual exposure < recorded OI. Permanently inflated OI hits caps, blocking new positions.
- **FP:** OI incremented by post-fee size. OI decremented on close by same amount used at open.


**298. Admin Parameter Change During Active Multi-Step Operation**

- **D:** Multi-step operation (auction, epoch, bridge transfer, pending callback) spans multiple blocks. Admin setter takes effect mid-operation — later steps use new value, or dependency swap makes pending callback unfulfillable. Pattern: `setOracle(new)` while old oracle's callback pending.
- **FP:** Setter blocked while operation active. Operation snapshots config at start. Pending callbacks resolved before dependency swap. Changes queued for next boundary.


**299. Interest Accrual Rounds to Zero but Timestamp Advances**

- **D:** `interest = rate * timeDelta / SECONDS_PER_YEAR` rounds to zero for small `timeDelta`, but `lastAccrualTime` still advances — fractional interest permanently lost.
- **FP:** Accumulator uses sufficient precision (RAY = 1e27). `lastAccrualTime` only advances when interest > 0.


**300. Oracle Extractable Value (OEV) Liquidation Leakage**

- **D:** Liquidation callable by anyone with full `liquidationBonus` going to `msg.sender`. Oracle price update → MEV race to liquidate → value leaks from protocol to searchers with no recapture.
- **FP:** OEV-aware oracle (API3 OEV Network, Chainlink SVR). Liquidation bonus auctioned with proceeds to protocol. Dutch auction liquidation. Keeper priority window.


**301. DoS via Reverting External Call in Loop**

- **D:** Distribution/claim loops over dynamic list with external call per iteration. Any single revert (paused token, blacklisted recipient, illiquid pool) blocks entire function for all users.
- **FP:** `try/catch` per iteration (skip on failure). Separate per-item withdrawal. Admin can remove problematic entries.


**302. Intent Solver Collusion / Suboptimal Execution**

- **D:** Intent-based protocol with no on-chain execution quality enforcement. Colluding solvers provide suboptimal fills and split extracted value. Pattern: `fillOrder(order, solverParams)` with no price benchmark.
- **FP:** On-chain oracle comparison with max deviation. User `minOutput` enforced on-chain. Solver bond slashed for suboptimal execution.


**303. Pause Modifier Blocks Liquidations**

- **D:** `whenNotPaused` applied to all external functions including `liquidate()`. During pause, interest accrues and prices move but positions can't be liquidated — bad debt accumulates.
- **FP:** Liquidation exempt from pause. Separate `pauseLiquidations` flag. Interest accrual also paused.


**304. Capacity Competition Between Accounting Variables**

- **D:** Multiple accounting variables (user shares, fee shares, rewards) share a common cap. One can fill the cap, making another permanently unfulfillable. Pattern: `maxDeposit = supplyCap - totalSupply` ignores pending fee shares — deposits fill cap, fees can never mint.
- **FP:** `maxDeposit` accounts for all future obligations. Fee shares minted eagerly. Separate caps for user/protocol shares.


**305. Position Reduction Triggers Liquidation**

- **D:** Partial repay/withdrawal creates intermediate state below liquidation threshold — bot liquidates before atomic completion. Health check applied to intermediate, not final state.
- **FP:** Repay and collateral changes atomic. Health check on final state only. Grace period after modification.


**306. Checkpoint Overwrite on Same-Block Operations**

- **D:** Multiple delegate/transfer operations in same block overwrite `_writeCheckpoint()` at same key — binary search returns incomplete checkpoint, losing intermediate state.
- **FP:** Same-block operations accumulate into existing checkpoint. Off-chain indexer used.


**307. msg.value vs Computed Amount Mismatch**

- **D:** Payable function computes `netAmount` after fees but forwards full `msg.value` downstream. Or trusts user-supplied `amount` without `require(msg.value == amount)`.
- **FP:** `require(msg.value == expectedAmount)` at entry. Fee-adjusted amount used consistently. Excess refunded.


**308. ERC4626 maxDeposit vs Actual Deposit Method Mismatch**

- **D:** Vault queries `maxDeposit()` but deposits via `mint()` (or vice versa). Per ERC4626, `maxDeposit` governs `deposit()` and `maxMint` governs `mint()` — limits may differ. Same for withdrawal: `convertToAssets(maxRedeem(...))` instead of `maxWithdraw(...)` overstates amount (excludes fees/slippage).
- **FP:** Method-matched queries (`maxMint` for `mint`, `maxDeposit` for `deposit`). `previewWithdraw`/`previewRedeem` for estimates. Underlying `max*` verified consistent.


**309. Delegation to address(0) Blocks Token Transfers**

- **D:** Delegating to `address(0)` causes `_update` hooks to revert modifying zero-address checkpoint. All transfers/burns for that holder permanently revert.
- **FP:** Delegation to `address(0)` treated as undelegation. Hook skips checkpoint when delegate is zero. OZ Votes handles this.


**310. FIFO Withdrawal Ordering Degrades Yield**

- **D:** Aggregator vault withdraws from sub-vaults in fixed FIFO order, depleting highest-APY vaults first. Remaining capital concentrates in lowest-yield positions, reducing overall returns for all depositors.
- **FP:** Withdrawal ordering sorted by APY ascending (lowest-yield first). Dynamic rebalancing after withdrawals. Single underlying vault (no ordering issue).


**311. Emission Distribution Before Period Update**

- **D:** `distribute()` reads token balance before `updatePeriod()` mints new emissions. Rewards arrive after distribution — idle until next cycle, underpaying current period.
- **FP:** `updatePeriod()` called before `distribute()`. Emissions pre-funded before distribution window.


**312. Same-Block Vote-Transfer-Vote**

- **D:** Governance reads voting power at current block. User votes, transfers tokens to second wallet, votes again — doubling effective weight in same block.
- **FP:** `getPastVotes(block.number - 1)` or proposal-creation snapshot. Voting power locked on first vote. ERC20Votes with checkpoints.


**313. Pending Async Callback with Dependency Swap**

- **D:** Contract requests async operation (randomness, oracle, cross-chain message) fulfilled via callback. Dependency swapped before callback arrives — new provider can't fulfill old request, old rejected as unregistered. Request stuck permanently. Pattern: `setProvider(new)` while `pendingRequestId != 0`.
- **FP:** Swap blocked while requests pending. Callback validates request ID, not sender. Transition fulfills/cancels pending before registering new provider. Timeout for stuck requests.


**314. Generalized Frontrunner on Permissionless Value Functions**

- **D:** Value-extracting function (liquidation, reward claim) pays `msg.sender` with no access control, auction, or commit-reveal. Searcher always captures by frontrunning. Pattern: `token.transfer(msg.sender, bonus)` in permissionless external function.
- **FP:** Keeper priority window. Dutch auction. Commit-reveal with caller binding. Value returned to protocol/affected user.


**315. Reward Snapshot JIT (Same-Block Deposit-Withdraw for LP Rewards)**

- **D:** LP rewards accrue based on instantaneous liquidity share, not time-weighted contribution. Attacker adds massive liquidity before accrual, captures disproportionate share, removes after.
- **FP:** Time-weighted liquidity (`rewardPerLiquiditySecond` accumulator). Minimum staking duration. Previous-block snapshots.


**316. EIP-7702 Cross-Chain Authorization Replay**

- **D:** Authorization tuple with `chain_id = 0` valid on ALL EVM chains supporting EIP-7702. Each chain has independent nonce — same signature replayed across L1, rollups, testnets to delegate victim's EOA everywhere.
- **FP:** Authorization signed with explicit `chain_id != 0`. Delegate contract checks `block.chainid`.


**317. EIP-7702 Storage Collision on Redelegation**

- **D:** EOA redelegates from Contract A to Contract B. Storage persists and is reinterpreted under B's layout — corruption, privilege escalation, or fund loss.
- **FP:** ERC-7201 namespaced storage used. ERC-7779 redelegation process followed. Delegate has no persistent storage dependency.


**318. Cached Reward Debt Not Reset After Claim**

- **D:** After `claimRewards()`, `pendingReward`/`rewardDebt` not zeroed. Next claim pays full cached amount again — double payout.
- **FP:** `pendingReward[user] = 0` after transfer. `rewardDebt` recalculated from current balance and accumulator.


**319. Borrower Front-Runs Liquidation**

- **D:** Borrower front-runs `liquidate()` with minimal repayment/top-up to push health above threshold, then re-borrows after liquidation fails. Repeatable indefinitely.
- **FP:** Flash-loan-resistant health check (same-block deposit excluded). Minimum repayment cooldown. Dutch auction liquidation.


**320. Protocol Fee Inflates Reward Accumulator**

- **D:** Treasury fee processed through same `rewardPerToken` accumulator as staker rewards. Accumulator increments as if all goes to stakers — `earned()` exceeds contract balance.
- **FP:** Fee deducted before accumulator: `rewardPerToken += (reward - fee) / totalStaked`. Separate fee accounting.


**321. EIP-7702 Delegation Initialization Front-Run**

- **D:** EOA delegates to smart wallet requiring separate `initialize(owner)` call. Attacker front-runs with victim's authorization, calls `initialize()` first — takes ownership of EOA's wallet and assets.
- **FP:** Delegation and initialization bundled atomically. Owner derived from authorization signature via `ecrecover`. No permissionless `initialize()` step.


**322. Namespace / ID Reuse Across Subsystems**

- **D:** Multiple subsystems populate the same identifier space (`positionId`, `vaultId`, `requestId`, `orderId`), but authorization and state transitions only validate the ID, not the originating subsystem. An ID created in subsystem A is accepted by subsystem B, bypassing assumptions about ownership or lifecycle.
- **FP:** IDs are namespaced per subsystem, or every call validates both `id` and subsystem/type discriminator. Cross-subsystem direction table reviewed and impossible states rejected.


**323. Compliance Bypass via Privileged Auth Transfer**

- **D:** A privileged transfer primitive (`authTransfer`, `forceTransfer`, compliance-admin move) skips blacklist / sanction / allowlist checks by design, and a user-reachable path can indirectly invoke it. The calling wrapper assumes the privileged callee already enforced compliance, but it did not.
- **FP:** Privileged transfer is reachable only from isolated admin flows. User-facing paths re-run the same compliance checks before invoking the privileged primitive. Trace from all external entry points to `authTransfer` / `forceTransfer` shows no user-controlled route.


**324. Sentinel Collision on Exhausted Quota**

- **D:** `0` or another sentinel means "unset" / "unlimited", but the same value is also reachable through normal exhaustion (`remaining = 0`). Once a finite quota decrements to the sentinel value, the contract interprets the exhausted state as unlimited and re-enables access.
- **FP:** Exhausted state is represented separately from unset state (extra boolean, distinct enum, non-zero sentinel). Decrement path cannot transition into the meaning of "unlimited".


**325. Mapping Default Value State Ambiguity**

- **D:** Mapping default values (`0`, `false`, empty struct) are treated as "never initialized", but those same values are also valid initialized states. Attackers reset or route execution through the default state to re-trigger initialization, bypass one-time checks, or claim resources repeatedly.
- **FP:** Initialization tracked with an explicit boolean / version field. Default value is never used as the sole signal for state existence. Distinct-state collision tests cover `never set` vs `set to zero`.


**326. Self-Transfer Accounting / Delegation Distortion**

- **D:** Code assumes `src != dst` during transfer or delegation accounting. When sender and receiver are the same address, before/after balance reconstruction, voting checkpoints, or fee logic updates both sides asymmetrically, minting phantom voting power or corrupting accounting.
- **FP:** Self-transfer is an explicit no-op or has dedicated logic. Tests cover `from == to` for token, staking, and delegation flows. Balance / checkpoint deltas collapse to zero in the self-transfer case.


**327. Dual ETH/WETH Input Path Ambiguity**

- **D:** Function accepts native ETH (`msg.value > 0`) and also accepts WETH / wrapped-asset transfers on the same path. If both are provided simultaneously, accounting may credit both, wrap twice, or process inconsistent slippage / refund branches.
- **FP:** ETH and WETH paths are mutually exclusive (`require(msg.value == 0 || tokenIn != WETH)`, etc.). Native path wraps exactly once, ERC20 path rejects non-zero `msg.value`, and tests cover both-supplied input.


**328. Swap-and-Pop Moved Index Stale Reference**

- **D:** List deletion uses swap-and-pop, but auxiliary state still points to the moved element's old index. Subsequent reads, deletes, or authorization checks operate on the wrong record, enabling corruption or unauthorized access to the moved item.
- **FP:** Every swap-and-pop updates both the removed item's metadata and the moved item's index mapping atomically. No external references depend on unstable indices, or stable IDs are used instead.


**329. ERC-1363 transferAndCall Reentrancy**

- **D:** ERC-1363 tokens implement `transferAndCall`/`transferFromAndCall` which invoke `onTransferReceived` on the recipient. Protocols guarding only against ERC-777 hooks remain vulnerable to the same reentrancy surface via ERC-1363 callbacks, as the token standard is distinct and its hooks are not covered by ERC-777-specific guards.
- **FP:** Protocol does not accept arbitrary ERC-20 tokens. `nonReentrant` covers all state-changing paths regardless of callback source. CEI pattern followed throughout.


**330. Silent Modifier (if-Without-Revert Access Control Bypass)**

- **D:** A Solidity modifier uses `if (msg.sender == admin) { _; }` instead of `require(msg.sender == admin)`. When a non-admin calls the function, the modifier body is skipped but the function still executes because there is no revert — the `_;` is simply never reached for non-admins, but execution continues past the modifier, silently granting access to anyone.
- **FP:** Modifier uses `require`/`revert` instead of a bare `if`. The `if` pattern is intentionally used for optional side-effects (not access control).


**331. Tautology in Require (Self-Comparison Validation Bypass)**

- **D:** A `require()` statement compares a variable to itself (`require(sourceAddressesRoot == sourceAddressesRoot)`), which always evaluates to true. This is a copy-paste or typo error where the right-hand side should be a different variable (e.g., the computed/expected root). The validation is completely bypassed, allowing arbitrary inputs to pass proof verification.
- **FP:** The comparison is intentionally tautological as a placeholder. The function is not security-critical.


**332. Existence Check Misused as Ownership Check**

- **D:** A function calls `_requireOwned(tokenId)` or similar to validate authorization, but the internal function only checks whether the token exists (has a non-zero owner), not whether `msg.sender` is that owner. Any user can call the function for any existing token, enabling unauthorized operations like splitting, burning, or transferring other users' positions.
- **FP:** The existence check is intentional (function is meant to be callable by anyone for any token). A separate ownership check exists elsewhere in the call path.


**333. Partial-Claim Timestamp Advance (Unclaimed Reward Forfeiture)**

- **D:** When a claim/harvest function caps the claimed amount (via allowance, balance, or rate limit), the timestamp/checkpoint for future claims advances to `block.timestamp` even when `claimed < owed`. The unclaimed portion is permanently forfeited because the protocol believes it was already distributed. This silently burns user entitlements whenever a rate limit is hit.
- **FP:** Protocol explicitly documents that unclaimed amounts above the cap are forfeited by design. The checkpoint only advances proportionally to the amount actually claimed.


**334. Keeper Under-Incentivization (Maintenance Function Gas Economics)**

- **D:** Protocol depends on external keepers calling maintenance functions (`accrueInterest()`, `updateRewards()`, `liquidate()`, `performUpkeep()`), but the gas cost of calling these functions exceeds the keeper's reward. When gas prices spike, keepers go offline, causing state to become stale. In lending protocols, this leads to bad debt accumulation from unliquidated positions during high-gas periods.
- **FP:** Protocol has its own subsidized keeper network. Keeper incentives dynamically scale with gas costs. The maintenance function is not time-critical.


**335. Timelock Queue Observation Exploit Window**

- **D:** When a governance proposal to change a parameter (interest rate, collateral factor, oracle source, fee rate) is queued in a timelock, the window between queuing and execution is exploitable. Attackers monitor the timelock queue and front-run the parameter change — borrowing at the old collateral factor before it tightens, or depositing before a fee increase. Timelock queue flooding can also delay legitimate governance actions.
- **FP:** The timelock delay is shorter than practical front-running opportunity. The parameter change has no user-exploitable arbitrage. Affected positions are automatically settled at new parameters upon execution.


**336. Permit Nonce-DoS (Nonce Consumption Attack)**

- **D:** An attacker front-runs a user's EIP-2612 permit by submitting the same signature first. The nonce is consumed on the attacker's transaction (no approval is granted to the attacker — the nonce just gets burned). The user's bundled `permit + action` transaction then reverts because the nonce is stale. Distinct from permit front-run DoS (#225) because no approval is granted — pure nonce consumption to grief time-sensitive operations like liquidation prevention or deadline-bound swaps.
- **FP:** The `permit()` call is wrapped in `try/catch` with a fallback to pre-existing allowance. The nonce check has been removed or modified.


**337. Short Address Attack (ABI Padding Exploit)**

- **D:** An attacker uses an Ethereum address shorter than 20 bytes in raw transaction calldata. Solidity's ABI decoder right-pads the short address with zeros, causing it to consume bytes from the next parameter (typically the amount). This effectively multiplies the transfer amount by 256 for each missing byte.
- **FP:** Transaction is sent through a standard wallet/library that validates address length before encoding. Contract uses Solidity ≥0.5 with strict ABI decoding that rejects malformed calldata at the decoder level.


**338. Zero-Value Token Transfer Phishing (Address Poisoning)**

- **D:** Attacker calls `transferFrom(victim, spoofedAddress, 0)` on an ERC-20 token, which succeeds without approval because the amount is zero. This injects a fake transaction into the victim's history showing a transfer to a vanity address that closely resembles a legitimate recipient (same first/last characters). Victims who copy-paste addresses from transaction history send funds to the attacker's lookalike address.
- **FP:** Token reverts on zero-amount transfers (see vector #10). Wallet/explorer filters zero-value `transferFrom` events from transaction history display.


**339. Unenforced Safety Mechanism (Written-But-Not-Read State)**

- **D:** Safety-critical state variables (circuit breaker flags, pause states, emergency mode booleans, rate limit counters) are correctly written/updated but never checked in the code paths they are supposed to gate. The protocol has a `setPaused(true)` function that updates a `paused` storage variable, but the critical functions that should check `require(!paused)` simply do not read it — the safety mechanism exists in code but provides zero protection.
- **FP:** The safety state is checked via a modifier that is applied but not visible in the immediate function body (e.g., inherited modifier). The variable is read via an assembly/low-level path that is not immediately obvious.


**340. Override/Extension Mismatch (Inherited Security Property Loss)**

- **D:** When a contract overrides or wraps a base contract's function, the override may preserve explicit guards (`require`, `revert`, access control) but silently drop implicit structural properties (storage key schemes, ordering assumptions, aggregation granularity). For example, a base contract uses composite storage keys `keccak256(user, epoch)` for isolation, but the override switches to `mapping(user => value)`, losing epoch isolation. Explicit checks all pass but the structural security property is gone.
- **FP:** Override was intentionally designed to change the structural property (documented). The structural property is not security-relevant in the derived context.


**341. Comment-Formula Divergence (Code-Comment Mismatch in Financial Math)**

- **D:** Inline comments describe one formula (e.g., `// fee = amount * rate / total`) but the adjacent code implements a different formula (e.g., `fee = amount / total * rate`). When the comment was written as specification and the code was written differently due to refactoring or copy-paste error, the divergence creates a precision loss or economic bug. The comment serves as evidence of developer intent, making the code provably wrong.
- **FP:** The comment is outdated documentation that was never updated (informational, not a bug). Both formulas produce identical results for all valid inputs.
