283 total attack vectors

**43. Self-Liquidation Profit Extraction**

- **D:** Borrower liquidates their own undercollateralized position from a second address, collecting the liquidation bonus/discount on their own collateral. Profitable whenever liquidation incentive exceeds the cost of being slightly undercollateralized.
- **FP:** `require(msg.sender != borrower)` on liquidation. Liquidation penalty exceeds any collateral bonus. Liquidation incentive is small enough that self-liquidation is net-negative after gas.


**44. State-Time Lag Exploitation (lzRead Stale State)**

- **D:** `lzRead` queries state on a remote chain, but there is a latency window between query and delivery of the result via `lzReceive`. During this window, the queried state may change (token transferred, position closed, price moved). Protocol makes irreversible decisions based on the stale read result.
- **FP:** Read targets immutable or slowly-changing state (contract code, historical data). Read result treated as a hint with on-chain re-validation. Time-sensitive operations require fresh on-chain state, not cross-chain reads.


**45. Integer Overflow / Underflow**

- **D:** Arithmetic in `unchecked {}` (>=0.8) without prior bounds check: subtraction without `require(amount <= balance)`, large multiplications. Any arithmetic in <0.8 without SafeMath.
- **FP:** Range provably bounded by earlier checks in same function. `unchecked` only for `++i` loop increments where `i < arr.length`.


**46. OFT Shared Decimals Truncation (uint64 Overflow)**

- **D:** OFT converts between local decimals and shared decimals (typically 6). `_toSD()` casts to `uint64`. If `sharedDecimals >= localDecimals` (both 18), no decimal conversion occurs but the `uint64` cast silently truncates amounts exceeding ~18.4e18. Also: custom fee logic applied BEFORE `_removeDust()` produces incorrect fee calculations.
- **FP:** Standard OFT with `sharedDecimals = 6` and `localDecimals = 18` (default). Fee logic applied after dust removal. Transfer amounts validated against `uint64.max` before conversion.


**47. UUPS Upgrade Logic Removed in New Implementation**

- **D:** New UUPS implementation doesn't inherit `UUPSUpgradeable` or removes `upgradeTo`/`upgradeToAndCall`. Proxy permanently loses upgrade capability. Pattern: V2 missing `_authorizeUpgrade` override.
- **FP:** Every version inherits `UUPSUpgradeable`. Tests verify `upgradeTo` works after each upgrade. OZ upgrades plugin checks in CI.


**48. ERC721 onERC721Received Arbitrary Caller Spoofing**

- **D:** `onERC721Received` uses parameters (`from`, `tokenId`) to update state without verifying `msg.sender` is the expected NFT contract. Anyone calls directly with fabricated parameters.
- **FP:** `require(msg.sender == address(nft))` before state update. Function is view-only or reverts unconditionally.


**49. ERC1155 totalSupply Inflation via Reentrancy Before Supply Update**

- **D:** `totalSupply[id]` incremented AFTER `_mint` callback. During `onERC1155Received`, `totalSupply` is stale-low, inflating caller's share in any supply-dependent formula. Ref: OZ GHSA-9c22-pwxw-p6hx (2021).
- **FP:** OZ >= 4.3.2 (patched ordering). `nonReentrant` on all mint functions. No supply-dependent logic callable from mint callback.


**50. Missing Nonce (Signature Replay)**

- **D:** Signed message has no per-user nonce, or nonce present but never stored/incremented after use. Same signature resubmittable.
- **FP:** Monotonic per-signer nonce in signed payload, checked and incremented atomically. Or `usedSignatures[hash]` mapping.


**51. Single-Function Reentrancy**

- **D:** External call (`call{value:}`, `safeTransfer`, etc.) before state update — check-external-effect instead of check-effect-external (CEI).
- **FP:** State updated before call (CEI followed). `nonReentrant` modifier. Callee is hardcoded immutable with known-safe receive/fallback.

---

---


**52. Diamond Proxy Cross-Facet Storage Collision**

- **D:** EIP-2535 Diamond facets declare storage variables without EIP-7201 namespaced storage. Multiple facets independently start at slot 0, writing to same slots.
- **FP:** All facets use single `DiamondStorage` struct at namespaced position (EIP-7201). No top-level state variables in facets.


**53. Force-Feeding ETH via selfdestruct / Coinbase / CREATE2 Pre-Funding**

- **D:** Contract uses `address(this).balance` for accounting or gates logic on exact balance (e.g., `require(balance == totalDeposits)`). `selfdestruct(target)`, coinbase rewards, or pre-computed `CREATE2` deposits force ETH in without calling `receive()`/`fallback()`, breaking invariants.
- **FP:** Internal accounting only (`totalDeposited` state variable, never reads `address(this).balance`). Contract designed to accept arbitrary ETH (e.g., WETH wrapper).


**54. Wrong Price Feed for Derivative or Wrapped Asset**

- **D:** Protocol uses ETH/USD feed to price stETH collateral, or BTC/USD feed for WBTC. During normal conditions the error is small, but during depeg events the mispricing enables undercollateralized borrows or incorrect liquidations.
- **FP:** Dedicated feed for the actual derivative asset (e.g., stETH/USD, WBTC/BTC). Deviation check against secondary oracle. Protocol documentation explicitly accepts depeg risk.


**55. ERC4626 Preview Rounding Direction Violation**

- **D:** `previewDeposit` returns more shares than `deposit` mints, or `previewMint` charges fewer assets than `mint`. Custom `_convertToShares`/`_convertToAssets` with wrong `Math.mulDiv` rounding direction.
- **FP:** OZ ERC4626 base without overriding conversion functions. Custom impl explicitly uses `Floor` for share issuance, `Ceil` for share burning.


**56. Block Number as Timestamp Approximation**

- **D:** Time computed as `(block.number - startBlock) * 13` assuming fixed block times. Variable across chains/post-Merge. Wrong interest/vesting/rewards.
- **FP:** `block.timestamp` used for all time-sensitive calculations.


**57. Transparent Proxy Admin Routing Confusion**

- **D:** Admin address also used for regular protocol interactions. Calls from admin route to proxy admin functions instead of delegating — silently failing or executing unintended logic.
- **FP:** Dedicated `ProxyAdmin` contract used exclusively for admin calls. OZ `TransparentUpgradeableProxy` enforces separate admin.


**58. Cross-Chain Address Ownership Variance**

- **D:** Same address has different owners on different chains (EOA private key not used on all chains, or `CREATE`-deployed contract at same nonce but different deployer). Cross-chain logic that assumes `address(X) on Chain A == address(X) on Chain B` implies same owner enables impersonation.
- **FP:** `CREATE2`-deployed contracts with same factory + salt are safe. Peer mapping explicitly binds (chainId, address) pairs. Authorization uses cross-chain messaging (not address equality) to prove ownership.


**59. Read-Only Reentrancy**

- **D:** Protocol calls `view` function (`get_virtual_price()`, `totalAssets()`) on external contract from within a callback. External contract has no reentrancy guard on view functions — returns transitional/manipulated value mid-execution.
- **FP:** External view functions are `nonReentrant`. Chainlink oracle used instead. External contract's reentrancy lock checked before calling view.


**60. Bytecode Verification Mismatch**

- **D:** Verified source doesn't match deployed bytecode behavior: different compiler settings, obfuscated constructor args, or `--via-ir` vs legacy pipeline mismatch. No reproducible build (no pinned compiler in config).
- **FP:** Deterministic build with pinned compiler/optimizer in committed config. Verification in deployment script (Foundry `--verify`). Sourcify full match. Constructor args published.


**61. Scratch Space Corruption Across Assembly Blocks**

- **D:** Data written to scratch space (`0x00`–`0x3f`) in one assembly block is expected to persist and be read in a later assembly block, but intervening Solidity code (or compiler-generated code for `keccak256`, `abi.encode`, etc.) overwrites scratch space between the two blocks.
- **FP:** All scratch space reads occur within the same contiguous assembly block as the writes. Developer explicitly rewrites scratch space before each use.

---

---


**62. ERC1155 Custom Burn Without Caller Authorization**

- **D:** Public `burn(address from, uint256 id, uint256 amount)` callable by anyone without verifying `msg.sender == from` or operator approval. Any caller burns another user's tokens.
- **FP:** `require(from == msg.sender || isApprovedForAll(from, msg.sender))` before `_burn`. OZ `ERC1155Burnable` used.


**63. ERC721 Unsafe Transfer to Non-Receiver**

- **D:** `_transfer()`/`_mint()` used instead of `_safeTransfer()`/`_safeMint()`, sending NFTs to contracts without `IERC721Receiver`. Tokens permanently locked.
- **FP:** All paths use `safeTransferFrom`/`_safeMint`. Function is `nonReentrant`.


**64. ERC1155 Fungible / Non-Fungible Token ID Collision**

- **D:** ERC1155 represents both fungible and unique items with no enforcement: missing `require(totalSupply(id) == 0)` before NFT mint, or no cap preventing additional copies of supply-1 IDs.
- **FP:** `require(totalSupply(id) + amount <= maxSupply(id))` with `maxSupply=1` for NFTs. Fungible/NFT ID ranges disjoint and enforced.


**65. ERC4626 Deposit/Withdraw Share-Count Asymmetry**

- **D:** `_convertToShares` uses `Rounding.Floor` for both deposit and withdraw paths. `withdraw(a)` burns fewer shares than `deposit(a)` minted, manufacturing free shares.
- **FP:** `deposit` uses `Floor`, `withdraw` uses `Ceil` (vault-favorable both directions). OZ ERC4626 without custom conversion overrides.


**66. ERC4626 Mint/Redeem Asset-Cost Asymmetry**

- **D:** `redeem(s)` returns more assets than `mint(s)` costs — cycling yields net profit. Root cause: `_convertToAssets` rounds up in `redeem` and down in `mint` (opposite of EIP-4626 spec).
- **FP:** `redeem` uses `Math.Rounding.Floor`, `mint` uses `Math.Rounding.Ceil`. OZ ERC4626 without custom conversion overrides.


**67. ERC1155 Batch Transfer Partial-State Callback Window**

- **D:** Custom batch mint/transfer updates `_balances` and calls `onERC1155Received` per ID in loop, instead of committing all updates first then calling `onERC1155BatchReceived` once.
- **FP:** All balance updates committed before any callback (OZ pattern). `nonReentrant` on all transfer/mint entry points.


**68. Chainlink Staleness / No Validity Checks**

- **D:** `latestRoundData()` called but missing checks: `answer > 0`, `updatedAt > block.timestamp - MAX_STALENESS`, `answeredInRound >= roundId`, fallback on failure.
- **FP:** All four checks present. Circuit breaker or fallback oracle on failure.


**69. Unsafe Downcast / Integer Truncation**

- **D:** `uint128(largeUint256)` without bounds check. Solidity >= 0.8 silently truncates on downcast (no revert). Dangerous in price feeds, share calculations, timestamps.
- **FP:** `require(x <= type(uint128).max)` before cast. OZ `SafeCast` used.


**70. Missing `enforcedOptions` — Insufficient Gas for lzReceive**

- **D:** OApp does not call `setEnforcedOptions()` to mandate minimum gas for destination execution. User-supplied `_options` can specify insufficient gas, causing `lzReceive` to revert on destination.
- **FP:** `enforcedOptions` configured with tested gas limits for each message type. `lzReceive` logic is simple (single mapping update) requiring minimal gas.


**71. Nonce Gap from Reverted Transactions (CREATE Address Mismatch)**

- **D:** Deployment script uses `CREATE` and pre-computes addresses from deployer nonce. Reverted/extra tx advances nonce — subsequent deployments land at wrong addresses.
- **FP:** `CREATE2` used (nonce-independent). Script reads nonce from chain before computing. Addresses captured from deployment receipts.

---

---


**72. Fee-on-Transfer Token Accounting**

- **D:** Deposit records `deposits[user] += amount` then `transferFrom(..., amount)`. Fee-on-transfer tokens cause contract to receive less than recorded.
- **FP:** Balance measured before/after: `uint256 before = token.balanceOf(this); transferFrom(...); received = balanceOf(this) - before;` and `received` used for accounting.


**73. Assembly Delegatecall Missing Return/Revert Propagation**

- **D:** Proxy fallback written in assembly performs `delegatecall` but omits one or more required steps: (1) not copying full calldata, (2) not copying return data, (3) not branching on result to return/revert.
- **FP:** Complete proxy pattern with calldatacopy → delegatecall → returndatacopy → switch. OZ Proxy.sol used.


**74. Merkle Tree Second Preimage Attack**

- **D:** `MerkleProof.verify(proof, root, leaf)` where leaf derived from user input without double-hashing or type-prefixing. 64-byte input passes as intermediate node.
- **FP:** Leaves double-hashed or include type prefix. Input length enforced != 64 bytes. OZ MerkleProof >= v4.9.2.


**75. Dirty Higher-Order Bits on Sub-256-Bit Types**

- **D:** Assembly loads a value as a full 32-byte word (`calldataload`, `sload`, `mload`) but treats it as a smaller type without masking upper bits. Dirty bits cause incorrect comparisons or storage corruption.
- **FP:** Explicit bitmask applied: `and(value, mask)` immediately after load. `shr(96, calldataload(offset))` pattern for addresses.


**76. Griefing via Dust Deposits Resetting Timelocks or Cooldowns**

- **D:** Timelock/cooldown resets on any deposit with no minimum: `lastActionTime[user] = block.timestamp` inside `deposit(uint256 amount)` without `require(amount >= MIN)`. Attacker calls `deposit(1)` to reset victim's lock indefinitely.
- **FP:** Minimum deposit enforced unconditionally. Cooldown resets only for depositing user. Lock assessed independently of deposit amounts.


**77. Returndatasize-as-Zero Assumption**

- **D:** Assembly uses `returndatasize()` as a gas-cheap substitute for `push 0` (saves 1 gas). If a prior `call` returned data, `returndatasize()` is nonzero, corrupting the intended zero value.
- **FP:** `returndatasize()` used as zero only at the very start of execution before any external calls. Used as actual size measurement.


**78. tx.origin Authentication**

- **D:** `require(tx.origin == owner)` used for auth. Phishable via intermediary contract.
- **FP:** `tx.origin == msg.sender` used only as anti-contract check, not auth.


**79. ERC20 Non-Compliant: Return Values / Events**

- **D:** Custom `transfer()`/`transferFrom()` doesn't return `bool` or always returns `true` on failure. `mint()`/`burn()` missing `Transfer` events.
- **FP:** OZ `ERC20.sol` base with no custom overrides of transfer/approve/event logic.


**80. ERC721Enumerable Index Corruption on Burn or Transfer**

- **D:** Override of `_beforeTokenTransfer` (OZ v4) or `_update` (OZ v5) without calling `super`. Index structures become stale.
- **FP:** Override always calls `super` as first statement. Contract doesn't inherit `ERC721Enumerable`.


**81. Block Stuffing / Gas Griefing on Subcalls**

- **D:** Time-sensitive function blockable by filling blocks. For relayer gas-forwarding griefing (63/64 rule), see Vector 146.
- **FP:** Function not time-sensitive or window long enough that block stuffing is economically infeasible.

---

---


**82. ERC777 tokensToSend / tokensReceived Reentrancy**

- **D:** Token `transfer()`/`transferFrom()` called before state updates on a token that may implement ERC777 (ERC1820 registry). ERC777 hooks fire on ERC20-style calls, enabling reentry.
- **FP:** CEI — all state committed before transfer. `nonReentrant` on all entry points. Token whitelist excludes ERC777.


**83. Rebasing / Elastic Supply Token Accounting**

- **D:** Contract holds rebasing tokens (stETH, AMPL, aTokens) and caches `balanceOf(this)`. After rebase, cached value diverges from actual balance.
- **FP:** Rebasing tokens blocked at code level (revert/whitelist). Accounting reads `balanceOf` live. Wrapper tokens (wstETH) used.
