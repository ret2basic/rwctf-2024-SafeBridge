# rwctf-6th-safebridge

A cross-chain bridge vulnerability challenge from Real World CTF 6th.

## Docker Setup Fix

The original Docker setup had a critical issue with the Foundry installation. The problem was that the base image `ghcr.io/foundry-rs/foundry:latest` is no longer Alpine-based, causing the `apk` package manager commands to fail with "apk: not found" errors.

### Solution
Changed the Dockerfile to use a Debian-based image and appropriate package managers:

```dockerfile
FROM node:18-slim AS builder

# Install foundry
RUN apt-get update && apt-get install -y curl git && rm -rf /var/lib/apt/lists/*
RUN curl -L https://foundry.paradigm.xyz | bash
ENV PATH="/root/.foundry/bin:${PATH}"
RUN foundryup
```

Key changes:
- Base image: `ghcr.io/foundry-rs/foundry:latest` → `node:18-slim`
- Package manager: `apk` → `apt-get`
- Added proper Foundry installation via the official installer script

## Challenge Vulnerability: L1/L2 Token Accounting Mismatch

### The Bug

The vulnerability exists in the `L1ERC20Bridge.sol` contract's withdrawal finalization logic. The contract has an accounting mismatch between:
1. **What it tracks**: Uses `_l2Token` parameter for internal accounting (`deposits[_l1Token][_l2Token]`)
2. **What it verifies**: Hardcodes `L2_WETH` address in cross-domain messages

### Vulnerable Code
```solidity
function finalizeERC20Withdrawal(
    address _l1Token,
    address _l2Token,  // Used for accounting
    address _from,
    address _to,
    uint256 _amount
) external {
    deposits[_l1Token][_l2Token] -= _amount;  // Tracks using _l2Token
    
    // But message verification uses hardcoded L2_WETH
    require(
        msg.sender == address(MESSENGER),
        "L1ERC20Bridge: function can only be called from the messenger"
    );
    require(
        MESSENGER.xDomainMessageSender() == Predeploys.L2_ERC20_BRIDGE,
        "L1ERC20Bridge: function can only be called from the L2 bridge"
    );
}
```

### Exploitation Strategy

1. **Deploy Malicious Token on L2**: Create a token contract that:
   - Returns the L1 WETH address when `l1Token()` is called
   - Has a no-op `burn()` function that doesn't actually burn tokens

2. **Deposit WETH**: Bridge WETH from L1 to L2 using the malicious token address as the L2 token
   - This credits `deposits[WETH][maliciousToken]`

3. **Double Withdrawal**:
   - First: Withdraw using the malicious token address
     - Decrements `deposits[WETH][maliciousToken]`
     - Doesn't burn L2 WETH (due to no-op burn)
   - Second: Withdraw using the actual L2 WETH address
     - Uses the preserved L2 WETH balance
     - Bridge processes this as a valid withdrawal

4. **Result**: Extract 2x the deposited amount from the bridge

### Root Cause
The bridge fails to enforce consistency between the token used for accounting and the token validated in cross-domain messages. This allows attackers to manipulate the accounting system while bypassing the burn mechanism that should prevent double-spending.