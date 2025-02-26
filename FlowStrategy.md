# Overview of the Flow Strategy Contracts

This repository implements a token ecosystem called **FlowStrategy**, combined with Dutch Auctions, bond/ATM auctions, a deposit mechanism, and a governance contract. Below is a summary of each file, how they interact, and initial considerations for an audit.

---

## 1. AllowableAccounts.sol

**Purpose**  
Manages a whitelist of allowed addresses using `EnumerableSet`.

**Key Points**  
- `_addWhitelist` and `_removeWhitelist` revert on invalid addresses or duplication/removal errors.  
- `isWhitelisted` and `getWhitelist` provide read access to the set.

**Audit Notes**  
- Straightforward storage logic.  
- Pay attention to access control: only specific roles or contract logic should call these internal functions.

---

## 2. AtmAuction.sol

**Inheritance**  
Extends `DutchAuction`.

**Key Function**  
`_fill(uint128 amount, uint128 price, uint64 startTime, uint64 duration)`

**Logic**  
- Uses `TokenPriceLib._normalize` to calculate the payment amount.  
- Transfers `paymentToken` from the buyer to `owner()`.  
- Mints tokens to the buyer via `IFlowStrategy(flowStrategy).mint`.

**Audit Notes**  
- Similar structure to `BondAuction` but simpler.  
- Confirm the math in `_normalize` is correct for the specified decimals.

---

## 3. BondAuction.sol

**Inheritance**  
Extends `DutchAuction`.

**Structure**  
- Introduces a `Bond` struct with `amount`, `price`, and `startRedemption`.
- Uses a redemption window (`REDEMPTION_WINDOW = 1 days`).

**Logic**  
- `_fill` places funds in the contract instead of sending them to the owner immediately.  
- Buyers redeem within a specific time window (`redeem()`) to get the tokens, or `withdraw()` to get a refund.

**Audit Notes**  
- Watch for potential edge cases around the redemption window.  
- Buyers cannot hold multiple unredeemed bonds.

---

## 4. Deposit.sol

**Purpose**  
Receives native currency and mints FlowStrategy tokens at a fixed `CONVERSION_RATE`.

**Key Points**  
- Maintains a global `depositCap` that decreases per deposit.  
- Respects `MIN_DEPOSIT` and `MAX_DEPOSIT` limits.  
- Optionally enforces a whitelist.  
- Sends deposited funds to `owner()`.

**Audit Notes**  
- Uses `ReentrancyGuard`.  
- Ensure correct handling of `operator` role, whitelist logic, and deposit caps.  
- Transfers `msg.value` to `owner()` and mints tokens for the sender.

---

## 5. DutchAuction.sol

**Purpose**  
Generic Dutch auction contract.

**Struct**  
`Auction { startTime, duration, startPrice, endPrice, amount }`

**Key Functions**  
- `startAuction` sets up auction parameters.  
- `fill` handles user participation, calculates the current price, and calls the `_fill` hook (overridden by child contracts).

**Audit Notes**  
- Confirms `startPrice > endPrice`.  
- Uses a linear price drop from `startPrice` to `endPrice`.  
- Check for integer overflows, reentrancy, and correct boundary checks.

---

## 6. FlowStrategy.sol

**Implementation**  
Extends `ERC20Votes` (Solady).  
Implements `IFlowStrategy`.

**Roles**  
- `MINTER_ROLE` can mint.  
- `PAUSER_ROLE` can pause token transfers.  
- Defaults to `isTransferPaused = true`, so transfers are disabled unless unpaused.

**Audit Notes**  
- Verify that minting and pausing logic works as intended.  
- Ensure only trusted addresses/roles can mint or unpause transfers.

---

## 7. FlowStrategyGovernor.sol

**Purpose**  
A governance contract using OpenZeppelin’s Governor framework.

**Key Points**  
- Extends `GovernorVotesQuorumFraction` and uses the FlowStrategy token for voting power.  
- Configurable `votingDelay_`, `votingPeriod_`, and `proposalThreshold_`.

**Audit Notes**  
- Standard governance flow.  
- Verify parameters align with project needs.

---

## 8. IFlowStrategy.sol

**Interface**  
Defines functions for minting and retrieving token information.

**Used By**  
Contracts like `AtmAuction`, `BondAuction`, and `Deposit` that need to mint FlowStrategy tokens.

---

## 9. TokenPriceLib.sol

**Purpose**  
Handles decimal normalization for token prices and amounts.

**Logic**  
- Adjusts `_price` and `_amount` based on `priceDecimals`, `paymentTokenDecimals`, and `purchaseTokenDecimals`.  
- Final result accounts for these differences and divides by `10^(purchaseTokenDecimals)`.

**Audit Notes**  
- Potential source of rounding errors or off-by-one issues.  
- Ensure `_price` is correctly scaled, especially if `PRICE_DECIMALS` is 6 vs. token decimals of 18 or other values.

---

## Interactions and Flow

1. **FlowStrategy**  
   Main ERC20 token (paused transfers by default) with voting extensions.  

2. **FlowStrategyGovernor**  
   Manages on-chain governance using that token’s voting power.  

3. **DutchAuction**  
   Base for auctions with a linear price drop.  
   - **AtmAuction** transfers funds directly to `owner()`.  
   - **BondAuction** locks funds, allowing redemption or withdrawal later.

4. **Deposit**  
   Allows deposits of native currency, with optional whitelisting.  
   Mints FlowStrategy tokens and forwards funds to `owner()`.

5. **AllowableAccounts**  
   Maintains a whitelist for use in the Deposit contract.

6. **TokenPriceLib**  
   Normalizes prices and token amounts, used by the auctions.

---

## Initial Audit Considerations

- **Access Control**: Check ownership, roles (`MINTER_ROLE`, `ADMIN_ROLE`, `operator`), and whitelist logic.  
- **Decimal Arithmetic**: Verify `TokenPriceLib` calculations, especially if `PRICE_DECIMALS` and token decimals differ.  
- **Auction Edge Cases**: Confirm time-based logic, linear price drops, and redemption windows.  
- **Deposit Flow**: Ensure reentrancy is handled, deposit caps are correct, and roles are respected.  
- **Paused Transfers**: Confirm that tokens can be minted and only transferred if unpaused.  
- **Governance Parameters**: Confirm `votingDelay`, `votingPeriod`, and `proposalThreshold` settings.