# **Review Summary**

The Auditware team has performed a security review on BETH, a simple ERC20  
wrapper contract of ETH with a burn mechanic.

The token rewards users who burn ETH with corresponding BETH tokens.

The scope of this audit consisted of:

* [BETH.sol](https://github.com/ETHCF/beth/blob/7e136fa6765c41be126c52fd0037c9ba98f53d50/src/BETH.sol)

Audit commit hash: [7e136fa](https://github.com/ETHCF/beth/tree/7e136fa6765c41be126c52fd0037c9ba98f53d50) 

## Contract Overview

BETH adds two main functions to the basic OpenZeppelin ERC20 implementation, forwards three functions that accept ETH transfers to the **\_depositTo()** function, and defines a burn address to send received funds to.

The added functionality was found to introduce no security risk, with simple logic and proper checks in place.

**\_depositTo()**:

##     /// @notice Internal helper to forward ETH then mint BETH 1:1

##     /// @dev Forwards ETH to the burn address, then mints to \`recipient\` on success.

##     /// @param recipient The address to receive newly minted BETH

##     /// @return mintedAmount The amount of BETH minted, in wei

##     function \_depositTo(address recipient) private returns (uint256 mintedAmount) {

##         uint256 amount \= msg.value;

##         if (amount \== 0) revert ZeroDeposit();

## 

##         // Forward ETH to the burn address

##         (bool success,) \= payable(ETH\_BURN\_ADDRESS).call{ value: amount }("");

##         if (\!success) revert ForwardFailed();

## 

##         // Mint BETH after successful forwarding

##         \_mint(recipient, amount);

##         \_totalBurned \+= amount;

## 

##         emit Burned(msg.sender, amount);

##         emit Minted(recipient, amount);

## 

##         return amount;

##     }

**flush()**:

##     /// @notice Forward any stray ETH balance to the burn address without minting

##     /// @dev Handles ETH force-sent to the contract (e.g., via selfdestruct). Increases totalBurned only.

##     function flush() external {

##         uint256 amount \= address(this).balance;

##         if (amount \== 0) return;

##         // Effects first to satisfy static analyzers; revert will roll back on failure below

##         \_totalBurned \+= amount;

##         emit Burned(address(this), amount);

##         (bool success,) \= payable(ETH\_BURN\_ADDRESS).call{ value: amount }("");

##         if (\!success) revert ForwardFailed();

##     }

**ETH\_BURN\_ADDRESS**:  
    /// @notice Canonical Ethereum burn address used as the ETH sink  
    /// @dev This is a constant well-known address to which ETH is forwarded  
    address public constant ETH\_BURN\_ADDRESS \= 0x0000000000000000000000000000000000000000;

## 

# **Findings Overview**

| Finding | Type | Risk Level |
| :---: | :---: | :---: |
| [**AW-I-01: Nonstandard Burn Address**](#aw-i-01:-nonstandard-burn-address) | Centralization | *INFO* |

# **AW-I-01: Nonstandard Burn Address** {#aw-i-01:-nonstandard-burn-address}

**Severity:** *Informational*							**Status:** *Mitigated*

**Locations:**

* [beth/src/BETH.sol\#L15](https://github.com/ETHCF/beth/blob/7e136fa6765c41be126c52fd0037c9ba98f53d50/src/BETH.sol#L15) 

address public constant ETH\_BURN\_ADDRESS \= 0x000000000000000000000000000000000000dEaD;

**Description:**  
The BETH token contract defines a non-standard burn address ending in “dEaD”. It is recommended to use the typical zero address to conform to best practices.

**Recommendation:**

* Update address to the standard 0x0000000000000000000000000000000000000000 burn address

**Fixed:**  
[https://github.com/ETHCF/beth/commit/d3cad6a60d1a4ee0a0c41356d2e41508ba8b7b96](https://github.com/ETHCF/beth/commit/d3cad6a60d1a4ee0a0c41356d2e41508ba8b7b96)

This report was produced by Auditware for ETHCF based solely on the   
information provided by them. Auditware provides no guarantees as to the accuracy  
 of the contents of this report and does not make any guarantees that following the advice   
within will prevent security incidents or issues.

Auditware is available for questions or comments about any of the contents of this report. We can be reached at [https://auditware.io/](https://auditware.io/) or by email at [joe@auditware.io](mailto:joe@auditware.io).
