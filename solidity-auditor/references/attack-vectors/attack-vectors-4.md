328 total attack vectors

**126. Missing chainId (Cross-Chain Replay)**

- **D:** Signed payload omits `chainId`. Valid signature replayable on forks/other EVM chains. Or `chainId` hardcoded at deployment instead of `block.chainid`.
- **FP:** EIP-712 domain separator includes dynamic `chainId: block.chainid` and `verifyingContract`.


**127. Non-Standard ERC20 Return Values (USDT-style)**

- **D:** `require(token.transfer(to, amount))` reverts on tokens returning nothing (USDT, BNB). Or return value ignored (silent failure).
- **FP:** OZ `SafeERC20.safeTransfer()`/`safeTransferFrom()` used throughout.


**128. Front-Running Zero Balance Check with Dust Transfer**

- **D:** `require(token.balanceOf(address(this)) == 0)` gates a state transition. Dust transfer makes balance non-zero, DoS-ing the function.
- **FP:** Threshold check (`<= DUST_THRESHOLD`) instead of `== 0`. Access-controlled function. Internal accounting ignores direct transfers.


**129. Diamond Proxy Facet Selector Collision**

- **D:** EIP-2535 Diamond where two facets register same 4-byte selector. Malicious facet via `diamondCut` hijacks calls to critical functions.
- **FP:** `diamondCut` validates no selector collisions. `DiamondLoupeFacet` enumerates/verifies selectors post-cut.


**130. Flash Loan Governance Attack**

- **D:** Voting uses `token.balanceOf(msg.sender)` or `getPastVotes(account, block.number)` (current block). Borrow tokens, vote, repay in one tx.
- **FP:** `getPastVotes(account, block.number - 1)` used. Timelock between snapshot and vote.


**131. Hardcoded Network-Specific Addresses**

- **D:** Literal `address(0x...)` constants for external dependencies in deployment scripts/constructors. Wrong contracts on different chains.
- **FP:** Per-chain config file keyed by chain ID. Script asserts `block.chainid`. Addresses passed as constructor args.


**132. ERC4626 Round-Trip Profit Extraction**

- **D:** `redeem(deposit(a)) > a` or inverse — rounding errors in both directions favor the user, yielding net profit per round-trip.
- **FP:** Rounding per EIP-4626: deposit/mint round down, withdraw/redeem round up (vault-favorable). OZ ERC4626 with `_decimalsOffset()`.


**133. ERC1155 ID-Based Role Access Control With Publicly Mintable Role Tokens**

- **D:** Access control via `require(balanceOf(msg.sender, ROLE_ID) > 0)` where `mint` for those IDs is not separately gated. Role tokens transferable by default.
- **FP:** Minting role-token IDs gated behind separate access control. Role tokens non-transferable.


**134. Off-By-One in Bounds or Range Checks**

- **D:** (1) `i <= arr.length` in loop (accesses OOB index). (2) `arr[arr.length - 1]` in `unchecked` without length > 0 check. (3) `>=` vs `>` confusion in financial logic.
- **FP:** Loop uses `<`. Last-element access preceded by length check. Financial boundaries demonstrably correct.


**135. ERC4626 Caller-Dependent Conversion Functions**

- **D:** `convertToShares()`/`convertToAssets()` branches on `msg.sender`-specific state (per-user fees, whitelist). EIP-4626 requires caller-independence. Downstream aggregators get wrong sizing.
- **FP:** Implementation reads only global vault state.

---

---


**136. Multi-Block TWAP Oracle Manipulation**

- **D:** TWAP observation window < 30 minutes. Post-Merge validators controlling consecutive blocks can hold manipulated AMM state across blocks.
- **FP:** TWAP window >= 30 min. Chainlink/Pyth as price source. Max-deviation circuit breaker.


**137. Merkle Proof Reuse — Leaf Not Bound to Caller**

- **D:** Merkle leaf doesn't include `msg.sender`. Proof can be front-run from different address.
- **FP:** Leaf encodes `msg.sender`. Proof recorded as consumed after first use.


**138. Uninitialized Implementation Takeover**

- **D:** Implementation behind proxy has `initialize()` but constructor lacks `_disableInitializers()`. Attacker calls `initialize()` on implementation directly. Ref: Wormhole (2022), Parity (2017).
- **FP:** Constructor contains `_disableInitializers()`. OZ `Initializable` correctly gates the function.


**139. Missing chainId / Message Uniqueness in Bridge**

- **D:** Bridge processes messages without `processedMessages[hash]` replay check, `destinationChainId == block.chainid` validation, or source chain ID in hash.
- **FP:** Unique nonce per sender. Hash of `(sourceChain, destChain, nonce, payload)` stored and checked.


**140. Missing Oracle Price Bounds (Flash Crash / Extreme Value)**

- **D:** Oracle returns a technically valid but extreme price (e.g., ETH at $0.01 during a flash crash). No min/max sanity bound.
- **FP:** Circuit breaker: `require(price >= MIN_PRICE && price <= MAX_PRICE)`. Deviation check against secondary oracle.


**141. DVN Collusion or Insufficient DVN Diversity**

- **D:** OApp configured with a single DVN (`1/1/1` security stack) or multiple DVNs controlled by the same entity. Compromising one entity approves fraudulent messages.
- **FP:** Diverse DVN set with `2/3+` threshold. DVNs use independent verification methods. Protocol runs its own required DVN.


**142. Missing Cross-Chain Rate Limits / Circuit Breakers**

- **D:** Bridge or OFT contract has no per-transaction or time-window transfer caps. A single exploit can drain the entire locked asset pool. Ref: Ronin hack.
- **FP:** Per-tx and per-window rate limits. `whenNotPaused` modifier. Guardian/emergency multisig can freeze.


**143. Staking Reward Front-Run by New Depositor**

- **D:** Reward checkpoint (`rewardPerTokenStored`) updated AFTER new stake recorded: `_balances[user] += amount` before `updateReward()`. New staker earns rewards for unstaked period.
- **FP:** `updateReward(account)` executes before any balance update. `rewardPerTokenPaid[user]` tracks per-user checkpoint.


**144. L2 Sequencer Uptime Not Checked**

- **D:** Contract on L2 uses Chainlink feeds without querying L2 Sequencer Uptime Feed. Stale data during downtime triggers wrong liquidations.
- **FP:** Sequencer uptime feed queried with grace period after restart.


**145. Insufficient Gas Forwarding / 63/64 Rule**

- **D:** External call without minimum gas budget. 63/64 rule leaves subcall with insufficient gas. In relayer patterns, subcall silently fails but outer tx marks request as "processed."
- **FP:** `require(gasleft() >= minGas)` before subcall. Return value + returndata both checked.

---

---


**146. Accrued Interest Omitted from Health Factor or LTV Calculation**

- **D:** Health factor or LTV computed from principal debt without adding accrued interest. Understates actual debt, delays necessary liquidations.
- **FP:** `getDebt()` includes accrued interest. Interest accrual function called before health check.


**147. ERC1155 setApprovalForAll Grants All-Token-All-ID Access**

- **D:** Protocol requires `setApprovalForAll(protocol, true)` for deposits/staking. No per-ID or per-amount granularity.
- **FP:** Protocol uses direct `safeTransferFrom` with user as `msg.sender`. Operator is immutable contract with escrow-only logic.


**148. Storage Layout Collision Between Proxy and Implementation**

- **D:** Proxy declares state variables at sequential slots (not EIP-1967). Implementation also starts at slot 0. Ref: Audius (2022).
- **FP:** EIP-1967 slots. OZ Transparent/UUPS pattern. No state variables in proxy contract.


**149. validateUserOp Missing EntryPoint Caller Restriction**

- **D:** `validateUserOp` is `public`/`external` without `require(msg.sender == entryPoint)`.
- **FP:** `require(msg.sender == address(_entryPoint))` or `onlyEntryPoint` modifier present.


**150. Stale Cached ERC20 Balance from Direct Token Transfers**

- **D:** Contract tracks holdings in state variable updated only through protocol functions. Direct `token.transfer(contract)` inflates real balance beyond cached value.
- **FP:** Accounting reads `balanceOf(this)` live. Cached value reconciled at start of every state-changing function.


**151. Cross-Function Reentrancy**

- **D:** Two functions share state variable. Function A makes external call before updating shared state; Function B reads that state. `nonReentrant` on A but not B.
- **FP:** Both functions share same contract-level mutex. Shared state updated before any external call.


**152. Slippage Enforced at Intermediate Step, Not Final Output**

- **D:** Multi-hop swap checks `minAmountOut` on the first hop or intermediate step, but final received amount has no independent slippage bound.
- **FP:** `minAmountOut` validated against user's final received balance. Single-hop swap. User specifies per-hop minimums.


**153. Non-Atomic Proxy Deployment Enabling CPIMP Takeover**

- **D:** Non-atomic deploy+init where attacker inserts malicious middleman implementation that persists across upgrades by restoring itself in ERC-1967 slot.
- **FP:** Atomic init calldata in constructor. `_disableInitializers()` in implementation constructor.


**154. Cross-Chain Reentrancy via Safe Transfer Callbacks**

- **D:** Cross-chain receive function calls `_safeMint`/`_safeTransfer` before updating supply/ownership counters. Callback re-enters to initiate another cross-chain send.
- **FP:** State updates committed before any safe transfer. `nonReentrant` on receive path. `_mint` used instead of `_safeMint`.

---

---


**155. abi.encodePacked Hash Collision with Dynamic Types**

- **D:** `keccak256(abi.encodePacked(a, b))` where two+ args are dynamic types. No length prefix means different inputs produce identical hashes. Affects permits, access control keys, nullifiers.
- **FP:** `abi.encode()` used instead. Only one dynamic type arg. All args fixed-size.


**156. Signed Integer Mishandling (signextend / sar / slt Confusion)**

- **D:** Assembly performs arithmetic on signed integers but uses unsigned opcodes. `shr` instead of `sar` loses the sign bit. `lt`/`gt` instead of `slt`/`sgt` treats negative numbers as huge positives.
- **FP:** Code consistently uses `sar`/`slt`/`sgt` for signed operations. `signextend` applied after loading sub-256-bit signed values.


**157. Missing `_debit` / `_debitFrom` Authorization in OFT**

- **D:** Custom OFT override of `_debit` omits authorization check. Anyone can bridge tokens from any holder's balance.
- **FP:** Standard LayerZero OFT implementation used without override. Custom `_debit` includes proper authorization.


**158. Default Message Library Hijack (Configuration Drag-Along)**

- **D:** OApp does not explicitly pin its send/receive library version. It relies on the endpoint's mutable default. A malicious default library update silently applies to all unpinned OApps.
- **FP:** OApp explicitly sets library versions in constructor or initialization. Configuration is immutable or governance-controlled.


**159. ERC-1271 isValidSignature Delegated to Untrusted Module**

- **D:** `isValidSignature(hash, sig)` delegated to externally-supplied contract without whitelist check. Malicious module always returns `0x1626ba7e`.
- **FP:** Delegation only to owner-controlled whitelist. Module registry has timelock/guardian approval.


**160. Counterfactual Wallet Initialization Parameters Not Bound to Deployed Address**

- **D:** Factory `createAccount` uses CREATE2 but salt doesn't incorporate all init params (especially owner). Attacker deploys wallet they control to same counterfactual address.
- **FP:** Salt derived from all init params. Factory reverts if account exists. Initializer called atomically.


**161. Oracle Price Update Front-Running**

- **D:** On-chain oracle update tx visible in mempool. Attacker front-runs a favorable price update by opening a position at the stale price.
- **FP:** Pull-based oracle (user submits price update atomically). Private mempool. Price-update-and-action in single tx.


**162. Metamorphic Contract via CREATE2 + SELFDESTRUCT**

- **D:** `CREATE2` deployment where deployer can `selfdestruct` and redeploy different bytecode at same address. Post-Dencun (EIP-6780): largely mitigated except same-tx create-destroy-recreate.
- **FP:** Post-Dencun: `selfdestruct` no longer destroys code unless same tx as creation. `EXTCODEHASH` verified at execution time.


**163. Free Memory Pointer Corruption**

- **D:** Assembly block writes to memory at fixed offsets without reading or updating the free memory pointer at `0x40`. Subsequent Solidity code allocates from stale pointer, overwriting assembly data.
- **FP:** Assembly block reads `mload(0x40)`, writes above that offset, updates `mstore(0x40, newFreePtr)`. Or only uses scratch space. Block annotated `memory-safe` and verified.

---

---


**164. ERC4626 Inflation Attack (First Depositor)**

- **D:** `shares = assets * totalSupply / totalAssets`. When `totalSupply == 0`, deposit 1 wei + donate inflates share price, victim's deposit rounds to 0 shares.
- **FP:** OZ ERC4626 with `_decimalsOffset()`. Dead shares minted to `address(0)` at init.


**165. Storage Layout Shift on Upgrade**

- **D:** V2 inserts new state variable in middle of contract instead of appending. Subsequent variables shift slots, corrupting state.
- **FP:** New variables only appended. OZ storage layout validation in CI. Variable types unchanged between versions.


**166. Hardcoded Calldataload Offset Bypass via Non-Canonical ABI Encoding**

- **D:** Assembly reads a field at hardcoded calldata offset assuming standard ABI layout. Attacker crafts non-canonical encoding so a different value sits at the expected position.
- **FP:** Field decoded via `abi.decode()`. No hardcoded `calldataload` offsets. `calldatasize() >= expected` validated.


**167. Calldata Input Malleability**

- **D:** Contract hashes raw calldata for uniqueness. Dynamic-type ABI encoding uses offset pointers — multiple distinct layouts decode to identical values. Attacker bypasses dedup.
- **FP:** Uniqueness check hashes decoded parameters: `keccak256(abi.encode(decodedParams))`. Nonce-based replay protection.
