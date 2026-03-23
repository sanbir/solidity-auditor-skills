283 total attack vectors

**84. Assembly Arithmetic Silent Overflow and Division-by-Zero**

- **D:** Arithmetic inside `assembly {}` (Yul) does not revert on overflow/underflow (wraps like `unchecked`) and division by zero returns 0 instead of reverting.
- **FP:** Manual overflow checks in assembly (`if gt(result, x) { revert(...) }`). Denominator checked before `div`. Assembly block is read-only.


**85. Flash Loan-Assisted Price Manipulation**

- **D:** Function reads price from on-chain source (AMM reserves, vault `totalAssets()`) manipulable atomically via flash loan + swap in same tx.
- **FP:** TWAP with >= 30min window. Multi-block cooldown between price reads. Separate-block enforcement.


**86. Non-Standard Approve Behavior (Zero-First / Max-Approval Revert)**

- **D:** (a) USDT-style: `approve()` reverts when changing from non-zero to non-zero allowance. (b) Some tokens (UNI, COMP) revert on `approve(type(uint256).max)`.
- **FP:** OZ `SafeERC20.forceApprove()` or `safeIncreaseAllowance()` used. Allowance always set from zero. Token whitelist excludes non-standard tokens.


**87. Missing Chain ID Validation in Deployment Configuration**

- **D:** Deploy script reads `$RPC_URL` from `.env` without `eth_chainId` assertion. Foundry script without `--chain-id` flag or `block.chainid` check.
- **FP:** `require(block.chainid == expectedChainId)` at script start. CI validates chain ID before deployment.


**88. Array `delete` Leaves Zero-Value Gap Instead of Removing Element**

- **D:** `delete array[index]` resets element to zero but does not shrink the array or shift subsequent elements. Iteration logic treats the zeroed slot as a valid entry.
- **FP:** Swap-and-pop pattern used. Iteration skips zero entries explicitly. EnumerableSet or similar library used.


**89. Governance Flash-Loan Upgrade Hijack**

- **D:** Proxy upgrades via governance using current-block vote weight (`balanceOf` or `getPastVotes(block.number)`). No voting delay or timelock.
- **FP:** `getPastVotes(block.number - 1)`. Timelock 24-72h. High quorum thresholds. Staking lockup required.


**90. Write to Arbitrary Storage Location**

- **D:** (1) `sstore(slot, value)` where `slot` derived from user input without bounds. (2) Solidity <0.6: direct `arr.length` assignment + indexed write at crafted large index wraps slot arithmetic.
- **FP:** Assembly is read-only. Slot is compile-time constant or non-user-controlled. Solidity >= 0.6 used.


**91. Insufficient Return Data Length Validation**

- **D:** Assembly `staticcall`/`call` writes return data into a fixed-size buffer then reads `mload(ptr)` without checking `returndatasize() >= 32`. If target is EOA or non-compliant contract, `mload` reads stale memory.
- **FP:** `if lt(returndatasize(), 32) { revert(0,0) }` checked before reading return data. `extcodesize(target)` verified > 0. Safe ERC20 pattern used.


**92. Chainlink Feed Deprecation / Wrong Decimal Assumption**

- **D:** (a) Chainlink aggregator address hardcoded with no update path — deprecated feed returns stale/zero price. (b) Assumes `feed.decimals() == 8` without runtime check.
- **FP:** Feed address updatable via governance. `feed.decimals()` called and used for normalization. Secondary oracle deviation check.


**93. Upgrade Race Condition / Front-Running**

- **D:** `upgradeTo(V2)` and post-upgrade config calls are separate txs in public mempool. Window for front-running or sandwiching.
- **FP:** `upgradeToAndCall()` bundles upgrade + init. Private mempool. V2 safe with V1 state from block 0.

---

---


**94. Missing or Expired Deadline on Swaps**

- **D:** `deadline = block.timestamp` (always valid), `deadline = type(uint256).max`, or no deadline. Tx holdable in mempool indefinitely.
- **FP:** Deadline is calldata parameter validated as `require(deadline >= block.timestamp)`, not derived from `block.timestamp` internally.


**95. Deployment Transaction Front-Running (Ownership Hijack)**

- **D:** Deployment tx sent to public mempool. Attacker extracts bytecode and deploys first or front-runs initialization. Pattern: constructor sets `owner = msg.sender`.
- **FP:** Private relay used. Owner passed as constructor arg, not `msg.sender`. CREATE2 salt tied to deployer.


**96. Duplicate Items in User-Supplied Array**

- **D:** Function accepts array parameter without checking for duplicates. User passes same ID multiple times, claiming rewards/voting/withdrawing repeatedly.
- **FP:** Duplicate check via mapping (`require(!seen[id]); seen[id] = true`). Sorted-unique input enforced. State zeroed on first claim.


**97. Transient Storage Low-Gas Reentrancy (EIP-1153)**

- **D:** Contract uses `transfer()`/`send()` (2300-gas) as reentrancy guard + uses `TSTORE`/`TLOAD`. Post-Cancun, `TSTORE` succeeds under 2300 gas. Also: transient reentrancy lock not cleared at call end — persists for entire tx, DoS via multicall.
- **FP:** `nonReentrant` backed by regular storage slot (or transient mutex properly cleared). CEI followed unconditionally.


**98. Calldataload / Calldatacopy Out-of-Bounds Read**

- **D:** `calldataload(offset)` where `offset` exceeds actual calldata length. Reading past `calldatasize()` returns zero-padded bytes silently.
- **FP:** `calldatasize()` validated before assembly block. Static offsets into known fixed-length function signatures.


**99. Banned Opcode in Validation Phase (Simulation-Execution Divergence)**

- **D:** `validateUserOp`/`validatePaymasterUserOp` references `block.timestamp`, `block.number`, `block.coinbase`, etc. Per ERC-7562, banned in validation.
- **FP:** Banned opcodes only in execution phase. Entity is staked under ERC-7562 reputation system.


**100. Deployer Privilege Retention Post-Deployment**

- **D:** Deployer EOA retains owner/admin/minter/pauser/upgrader after deployment script completes. Pattern: `Ownable` sets `owner = msg.sender` with no `transferOwnership()`.
- **FP:** Script includes `transferOwnership(multisig)`. Admin role granted to timelock/governance, deployer renounces. `Ownable2Step` with pending owner set to multisig.


**101. Non-Atomic Multi-Contract Deployment (Partial System Bootstrap)**

- **D:** Deployment script deploys interdependent contracts across separate transactions. Midway failure leaves half-deployed state.
- **FP:** Single `vm.startBroadcast()`/`vm.stopBroadcast()` block. Factory deploys+wires all in one tx. Script is idempotent.


**102. CREATE / CREATE2 Deployment Failure Silently Returns Zero**

- **D:** Assembly `create(v, offset, size)` or `create2(v, offset, size, salt)` returns `address(0)` on failure but code does not check for zero.
- **FP:** Immediate check: `if iszero(addr) { revert(0, 0) }` after create/create2. Address validated downstream.


**103. ERC721 / ERC1155 Type Confusion in Dual-Standard Marketplace**

- **D:** Shared `buy`/`fill` function uses type flag for ERC721/ERC1155. `quantity` accepted for ERC721 without requiring == 1. `price * quantity` with `quantity = 0` yields zero payment. Ref: TreasureDAO (2022).
- **FP:** ERC721 branch `require(quantity == 1)`. Separate code paths for ERC721/ERC1155.

---

---


**104. Cross-Contract Reentrancy**

- **D:** Two contracts share logical state (balances in A, collateral check in B). A makes external call before syncing state B reads. A's `ReentrancyGuard` doesn't protect B.
- **FP:** State B reads is synchronized before A's external call. No re-entry path from A's callee into B.


**105. Non-Atomic Proxy Initialization (Front-Running `initialize()`)**

- **D:** Proxy deployed in one tx, `initialize()` called in separate tx. Uninitialized proxy front-runnable. Ref: Wormhole (2022).
- **FP:** Proxy constructor receives init calldata atomically. OZ `deployProxy()` used.


**106. EIP-2981 Royalty Signaled But Never Enforced**

- **D:** `royaltyInfo()` implemented and `supportsInterface(0x2a55205a)` returns true, but transfer/settlement logic never calls `royaltyInfo()` or routes payment.
- **FP:** Settlement contract reads `royaltyInfo()` and transfers royalty on-chain. Royalties intentionally zero and documented.


**107. Paymaster Gas Penalty Undercalculation**

- **D:** Paymaster prefund formula omits 10% EntryPoint penalty on unused execution gas (`postOpUnusedGasPenalty`). Large `executionGasLimit` with low usage drains paymaster deposit.
- **FP:** Prefund explicitly adds unused-gas penalty. Conservative overestimation covers worst case.


**108. ERC721 Approval Not Cleared in Custom Transfer Override**

- **D:** Custom `transferFrom` override skips `super._transfer()`, missing the `delete _tokenApprovals[tokenId]` step. Previous approval persists under new owner.
- **FP:** Override calls `super.transferFrom` or `super._transfer` internally. Or explicitly deletes approval.


**109. DoS via Push Payment to Rejecting Contract**

- **D:** ETH distribution in a single loop via `recipient.call{value:}("")`. Any reverting recipient blocks entire loop.
- **FP:** Pull-over-push pattern. Loop uses `try/catch` and continues on failure.


**110. Weak On-Chain Randomness**

- **D:** Randomness from `block.prevrandao`, `blockhash(block.number - 1)`, `block.timestamp`, `block.coinbase`, or combinations. Validator-influenceable or visible before inclusion.
- **FP:** Chainlink VRF v2+. Commit-reveal with future-block reveal.


**111. Delegatecall to Untrusted Callee**

- **D:** `address(target).delegatecall(data)` where `target` is user-provided or unconstrained.
- **FP:** `target` is hardcoded immutable verified library address.


**112. UUPS `_authorizeUpgrade` Missing Access Control**

- **D:** `function _authorizeUpgrade(address) internal override {}` with empty body and no access modifier. Anyone can call `upgradeTo()`. Ref: CVE-2021-41264.
- **FP:** `_authorizeUpgrade()` has `onlyOwner` or equivalent. Multisig/governance controls owner role.


**113. Insufficient Block Confirmations / Reorg Double-Spend**

- **D:** DVN relays cross-chain message before source chain reaches finality. Attacker deposits on source, gets minted on destination, then reorg reverses the deposit.
- **FP:** Confirmation count matches chain-specific finality guarantees. Chain has fast finality. DVN waits for finalized blocks.

---

---


**114. Nested Mapping Inside Struct Not Cleared on `delete`**

- **D:** `delete myMapping[key]` on struct containing `mapping` or dynamic array. `delete` zeroes primitives but not nested mappings. Reused key exposes stale values.
- **FP:** Nested mapping manually cleared before `delete`. Key never reused after deletion.


**115. ERC721A Lazy Ownership — ownerOf Uninitialized in Batch Range**

- **D:** ERC721A batch mint: only first token has ownership written. `ownerOf(id)` for mid-batch IDs may return `address(0)` before any transfer.
- **FP:** Explicit transfer initializes packed slot before ownership check. Standard OZ `ERC721` writes per mint.


**116. Cross-Chain Message Spoofing (Missing Endpoint/Peer Validation)**

- **D:** Receiver contract accepts cross-chain messages without verifying `msg.sender == endpoint` and `_origin.sender == registeredPeer[srcChainId]`. Ref: CrossCurve bridge exploit (Jan 2026).
- **FP:** `onlyPeer` modifier checks both endpoint and peer. Standard `OAppReceiver._acceptNonce` validates origin.


**117. Proxy Admin Key Compromise**

- **D:** `ProxyAdmin.owner()` returns EOA, not multisig/governance; no timelock on `upgradeTo`. Ref: PAID Network (2021), Ankr (2022).
- **FP:** Multisig (threshold >= 2) + timelock (24-72h). Admin role separate from operational roles.


**118. Unauthorized Peer Initialization (Fake Peer Attack)**

- **D:** `setPeer()` sets the remote peer address that a cross-chain contract trusts. If `setPeer` lacks proper access control, attacker registers fraudulent peer. Ref: GAIN token exploit (Sep 2025).
- **FP:** `setPeer` protected by multisig + timelock. `allowInitializePath()` properly implemented.


**119. Rounding in Favor of the User**

- **D:** `shares = assets / pricePerShare` rounds down for deposit but up for redeem. First-depositor donation amplifies the error.
- **FP:** `Math.mulDiv` with explicit `Rounding.Up` where vault-favorable. OZ ERC4626 `_decimalsOffset()`. Dead shares at init.


**120. Arbitrary External Call with User-Supplied Target and Calldata**

- **D:** `target.call{value: v}(data)` where `target` or `data` are caller-supplied. Attacker crafts calldata to invoke `transferFrom` on approved ERC20, stealing assets.
- **FP:** Target restricted to hardcoded/whitelisted address. Calldata function selector restricted to known-safe set.


**121. Paymaster ERC-20 Payment Deferred to postOp Without Pre-Validation**

- **D:** `validatePaymasterUserOp` doesn't transfer/lock tokens — payment deferred to `postOp`. User can revoke allowance between validation and execution.
- **FP:** Tokens transferred/locked during `validatePaymasterUserOp`. `postOp` only refunds excess.


**122. Minimal Proxy (EIP-1167) Implementation Destruction**

- **D:** EIP-1167 clones `delegatecall` a fixed implementation. If implementation is destroyed, all clones become no-ops with locked funds.
- **FP:** No `selfdestruct` in implementation. `_disableInitializers()` in constructor. Post-Dencun: code not destroyed.

---


**123. Spot Price Oracle from AMM**

- **D:** Price from AMM reserves: `reserve0 / reserve1`, `getAmountsOut()`, `getReserves()`. Flash-loan exploitable atomically.
- **FP:** TWAP >= 30 min window. Chainlink/Pyth as primary source.

---


**124. Missing Slippage Protection (Sandwich Attack)**

- **D:** Swap/deposit/withdrawal with `minAmountOut = 0`, or `minAmountOut` computed on-chain from current pool state.
- **FP:** `minAmountOut` set off-chain by user and validated on-chain.


**125. NFT Staking Records msg.sender Instead of ownerOf**

- **D:** `depositor[tokenId] = msg.sender` without checking `nft.ownerOf(tokenId)`. Approved operator credited as depositor.
- **FP:** Reads `nft.ownerOf(tokenId)` before transfer and records actual owner.
