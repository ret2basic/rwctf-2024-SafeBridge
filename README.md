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

The vulnerability exists in the `L1ERC20Bridge.sol` contract where there's a critical mismatch between:
1. **What gets tracked**: The `deposits` mapping uses the user-controlled `_l2Token` parameter
2. **What actually happens**: When `_l1Token == WETH`, the bridge hardcodes `L2_WETH` in the cross-chain message

### Vulnerable Code

In `depositERC20()`:
```solidity
if (_l1Token == weth) {
    // Records deposit with user-controlled _l2Token
    deposits[_l1Token][_l2Token] = deposits[_l1Token][_l2Token] + _amount;
    
    // But sends message with hardcoded L2_WETH!
    bytes memory message = abi.encodeWithSelector(
        IL2ERC20Bridge.finalizeDeposit.selector,
        _l1Token,
        Predeploys.L2_WETH,  // <-- Hardcoded, ignores _l2Token
        _from,
        _to,
        _amount
    );
}
```

This means:
- Accounting tracks: `deposits[WETH][maliciousToken] += amount`
- But L2 receives: Real L2 WETH tokens

### Exploitation Strategy

1. **Deploy Malicious Token on L2**: 
   ```solidity
   contract L2MaliciousToken {
       function l1Token() returns (address) { 
           return L1_WETH_ADDRESS; 
       }
       function burn(address, uint256) external {} // No-op
   }
   ```

2. **Bridge 2 WETH to L2** with `_l2Token = maliciousToken`:
   - Credits: `deposits[WETH][maliciousToken] = 2 ether`
   - Mints: 2 real L2 WETH to attacker (due to hardcoded address)

3. **First Withdrawal - Using Malicious Token**:
   - Withdraw with `_l2Token = maliciousToken`
   - Debits: `deposits[WETH][maliciousToken] -= 2 ether`
   - Burns: Nothing (malicious token's burn is no-op)
   - Receives: 2 WETH on L1
   - **Still has**: 2 L2 WETH (preserved)

4. **Second Withdrawal - Using Real L2 WETH**:
   - Withdraw the preserved 2 L2 WETH legitimately
   - Burns: 2 L2 WETH (real burn)
   - Receives: 2 more WETH on L1

### Result
- **Deposited**: 2 WETH
- **Withdrew**: 4 WETH (2 WETH × 2 withdrawals)
- **Net profit**: 2 WETH stolen from bridge

### Root Cause
The vulnerability stems from the inconsistency between the accounting system (which trusts user input) and the actual token operations (which use hardcoded addresses). This allows an attacker to:
- Make one deposit that credits their malicious token in accounting
- But receive real L2 WETH due to the hardcoded address
- Withdraw twice: once using the accounting credit (without burning L2 WETH), and once using the actual L2 WETH