# **Review Summary**

The Auditware team has performed a security review on Blobhouse Arena, a Rock-Paper-Scissors wagering protocol that solves the fundamental problem of  
creating provably fair games on a transparent blockchain where all transactions are  
visible.

The scope consisted of the Arena contract:

* [**src/Arena.sol**](https://github.com/numbergroup/blobhouse-contracts/blob/bd91d53c8f26590cee10092c4c3ef7bb02c4c3b4/src/Arena.sol) 

Audit commit hash: [bd91d53](https://github.com/numbergroup/blobhouse-contracts/tree/bd91d53c8f26590cee10092c4c3ef7bb02c4c3b4)

A **retest** was conducted on September 3d, 2025 on commit hash [ac54ba](https://github.com/numbergroup/blobhouse-contracts/commit/ac54baa8582fa95b8c39bec6a02380f68c1d7ec4), fix statuses are updated on a per-finding basis.

## Contract Overview

Arena uses a commit-reveal scheme where players first submit cryptographic hashes of their weapon choices, then an authorized matcher pairs players with equal stakes, and finally both players reveal their actual weapons within a block window to determine the winner through best-of-three.

### Example Operation Sequence:

* Players enter queue with ETH wager (0.005-2.5 ETH across 4 tiers) \+ secret weapon commitment  
* Authorized matcher pairs players with equal wagers  
* Both players reveal weapons within 1,800 blocks (\~6.5 hours)  
* Best-of-three RPS determines winner, with a 1% instant win case and a tiebreaker  
* Winner gets 96% of pot, 4% goes to protocol fees

### Business Model:

The protocol generates revenue through a 4% fee structure:

* 2% Protocol fee for development and maintenance  
* 1% Current season fee for ongoing rewards  
* 1% Next season fee for future incentives

## Security Overview

### Access Model:

* **Owner**: Protocol parameter control, fee recipients, matcher assignment. An owner can modify fee recipients without a timelock  
* **Matcher**: Player pairing authority, critical for game flow. controls all game initiation  
* **Players**: Entry, reveal, and forfeit capabilities

### Commit-Reveal Security

The protocol employs a commit-reveal scheme where players submit:

* Weapon array hash: keccak256(weapon0, weapon1, weapon2, salt, player, contract, chainid)  
* Time-locked commitment: 1,800 block reveal window (\~6.5 hours)  
* Cross-validation: Both players must reveal for game resolution

### Randomness Generation

The protocol implements a two-stage randomness system:

* Primary: blockhash(matchBlock \+ 1\) for recent matches  
* Fallback: keccak256(matchBlock) when blockhash unavailable

The seed is derived from multiple entropy sources:

* Player addresses (deterministic ordering)  
* Commit hashes from both players  
* Salt values from both players  
* Block-based randomness

As with all smart contracts, it is highly recommended to:

* Conduct regular and thorough audits to identify and mitigate vulnerabilities.  
* Achieve the highest possible test coverage, ideally approaching 100%.  
* Maintain comprehensive and up-to-date documentation.  
* Participate in auditing contests and/or a bug bounty program.  
* Implement on-chain monitoring for real-time threat detection and mitigation.

Protocol-specific monitoring recommendations:

* Job Completion Rates: Unusual completion patterns may indicate attacks  
* Large Deposits: Monitor for unusually large job amounts  
* Proxy Authorization Changes: Alert on new proxy authorizations  
* Failed Transactions: High failure rates may indicate attack attempts  
* Contract Balance: Unexpected balance changes warrant investigation

# **Findings Overview**

| Finding | Type | Risk Level | Retest (Sep 3rd, 2025\) |
| :---: | :---: | :---: | :---: |
| [**AW-C-01: Fund Appropriation via Unrestricted Forfeit Declaration**](#aw-c-01:-fund-appropriation-via-unchecked-forfeit-winner) | **Front Running** | *Critical* | ***RESOLVED*** |
| [**AW-L-01: Potential Fund Lockup in Credit System Edge Case**](#aw-l-01:-potential-fund-lockup-in-credit-system-edge-case) | **Locked Funds** | *Low* | ***RESOLVED*** |
| [**AW-L-02: Payment Failures via Insufficient Gas Limits in \_safeSend**](#aw-l-02:-payment-failures-via-insufficient-gas-limits-in-_safesend) | **Insecure Limitations** | *Low* | ***RESOLVED*** |
| **[AW-L-03: Centralized Control Risks in Owner and Matcher Roles](#aw-l-03:-centralized-control-risks-in-owner-and-matcher-roles)** | **Centralization** | *Low* | ***ACKNOWLEDGED*** |

# **AW-C-01: Fund Appropriation via Unchecked Forfeit Winner** {#aw-c-01:-fund-appropriation-via-unchecked-forfeit-winner}

**Severity:** *Critical*								**Status:** *Resolved*

**Retest:**  
During the retest, this issue was found to be mitigated by the new revealIndividual() function ([/src/Arena.sol\#L130](https://github.com/numbergroup/blobhouse-contracts/blob/1457a8a00221a10a8cf32b1a8a585133720ff6d2/src/Arena.sol#L130)), which allows players to individually reveal. The revealed flag is then checked in the forfeit function to ensure that an honest player cannot be griefed, any attacker that withholds their reveal will be the only allowed loser of a forfeit.

**Locations:**

* [src/Arena.sol\#L199-L221](https://github.com/numbergroup/blobhouse-contracts/blob/bd91d53c8f26590cee10092c4c3ef7bb02c4c3b4/src/Arena.sol#L199-L221) 

**Description:**  
The **forfeitUnrevealed** function lacks proper validation of the game winner when the opponent’s commit has been revealed, allowing a player that would normally lose to claim the funds via forfeit of the opponent. The losing player can delay their commit reveal until the game window passes to DOS finalization within the reveal deadline, then call the **fortfeitUnrevealed** function, declaring themselves the winner. There is no validation that the declared loser has not indeed revealed in time and should thus be subject to forfeit.

The forfeit mechanism allows either player to call **forfeitUnrevealed**, specifying themselves as the winner, as long as they provide their valid commit proof.

function forfeitUnrevealed(address winner, address loser, WeaponType\[3\] calldata winnerWeapons, bytes32 winnerSalt)  
   external  
   nonReentrant  
{  
   // MISSING: Validation that declared loser has not revealed  
   \_validateCommit(winner, winnerWeapons, winnerSalt, entryW.commitHash);  
   \_distributePayout(winner, pot);  
}

While the **\_validateCommit** function provides validation by including the player address, contract address, and chain ID in the commit hash verification, preventing attackers from guessing or brute-forcing secrets to directly impersonate winners, when legitimate players attempt to finalize games by revealing their moves, their opponent could refuse to finalize the game, allowing them to claim the funds via forfeit.  
**Recommendation:**

* Allow players to independently reveal their commits on-chain, so that a check that the declared loser of the forfeit function has not submitted their commit reveal could be added:  
  // MISSING: Validation that declared loser has not revealed  
  if (entryL.revealed) revert("LoserAlreadyRevealed");  
    
* Alternative solution: Implement a challenge period where the declared "loser" can dispute illegitimate forfeits

#   

# **AW-L-01: Potential Fund Lockup in Credit System Edge Case** {#aw-l-01:-potential-fund-lockup-in-credit-system-edge-case}

**Severity:** *Low*									**Status:** *Resolved*

**Retest:**   
During the retest it was found that a recovery option was implemented ([/src/Arena.sol\#L384-L398](https://github.com/numbergroup/blobhouse-contracts.git/blob/c0aa5c4eb28e9d59a5f40d69dc7ab15f8283e55e/src/Arena.sol#L384-L398)) addressing the fund lockup by letting the owner recover stuck funds after 365 days. This does come with a centralization that changes the trust model, it’s recommended to ensure this is aligns with intended design principles.

**Locations:**

* [src/Arena.sol\#L223-L231](https://github.com/numbergroup/blobhouse-contracts/blob/bd91d53c8f26590cee10092c4c3ef7bb02c4c3b4/src/Arena.sol#L233-L242)  
* [src/Arena.sol\#L333-L339](https://github.com/numbergroup/blobhouse-contracts/blob/bd91d53c8f26590cee10092c4c3ef7bb02c4c3b4/src/Arena.sol#L244-L250)

**Description:**  
The Arena protocol's credit system may allow funds to become permanently locked when recipient contracts become non-callable after accumulating credits.

The credit claiming mechanism relies entirely on recipients being able to call the **claim()** function:  
function claim() external nonReentrant {  
   uint256 amount \= credits\[msg.sender\];  
   if (amount \== 0) revert NoCredits();

   credits\[msg.sender\] \= 0;

   (bool success,) \= msg.sender.call{ value: amount }("");  
   if (\!success) revert TransferFailed();  
}

Credits may accumulate when the **\_safeSend** function fails to deliver payments:  
function \_safeSend(address to, uint256 amount) private {  
   if (amount \== 0) return;

   (bool success,) \= to.call{ value: amount, gas: 30000 }("");  
   if (\!success) {  
       credits\[to\] \+= amount;  
       emit Credited(to, amount);  
   }  
}  
This may lead to funds being locked in scenarios like the following examples:

* A contract accumulates credits through failed transfers, then self-destructs using **selfdestruct()**. The credits remain assigned to the destroyed address but become permanently unclaimable.  
* Contracts that become non-callable due to state changes (e.g., ownership transfers, emergency stops, or logic errors) after accumulating credits.  
* Contracts with buggy receive/fallback functions that work during credit accumulation but fail during claiming attempts.  
* Upgradeable contracts that are upgraded to incompatible implementations after accumulating credits.

The protocol has no administrative recovery mechanism for these scenarios, meaning funds become permanently locked in the Arena contract with no possibility of retrieval.

**Recommendation:**

* Add owner-controlled function to recover unclaimed credits after extended periods (e.g., 1 year)  
* Alternatively, implement automatic credit expiration with funds returning to the protocol treasury.

# **AW-L-02: Payment Failures via Insufficient Gas Limits in \_safeSend** {#aw-l-02:-payment-failures-via-insufficient-gas-limits-in-_safesend}

**Severity:** *Low*									**Status:** *Resolved*

**Retest:**   
During the retest it was found the finding was fixed by increasing the gas limit to **100,000** ([src/Arena.sol\#L372-L372](https://github.com/numbergroup/blobhouse-contracts/blob/c0aa5c4eb28e9d59a5f40d69dc7ab15f8283e55e/src/Arena.sol#L372-L372)).

**Locations:**

* [src/Arena.sol\#L332-L337](https://github.com/numbergroup/blobhouse-contracts/blob/bd91d53c8f26590cee10092c4c3ef7bb02c4c3b4/src/Arena.sol#L332-L337) 

**Description:**  
The payment distribution mechanism uses a hardcoded 30,000 gas limit that may be insufficient for legitimate contract recipients with complex receive functions:  
function \_safeSend(address to, uint256 amount) private {  
   if (amount \== 0) return;  
    
   (bool success,) \= to.call{ value: amount, gas: 30000 }("");  
   if (\!success) {  
       credits\[to\] \+= amount;  
       emit Credited(to, amount);  
   }  
}

Modern smart wallets, multisig contracts, and DeFi-integrated wallets often require more than 30,000 gas for incoming ETH processing.

This results in systematic payment failures for contract users, forcing them into the credit system and requiring additional transactions to claim winnings.

While the credit system may prevent fund loss, the degraded experience for contract wallet users justifies handling.

**Recommendation:**

* Increase gas limit to 100,000 to accommodate common contract patterns  
* Alternatively, implement address type detection with variable gas limits or add owner-settable gas limits for different recipient categories.  
    
  


# **AW-L-03: Centralized Control Risks in Owner and Matcher Roles** {#aw-l-03:-centralized-control-risks-in-owner-and-matcher-roles}

**Severity:** *Low*									**Status:** *Acknowledged*

**Retest:**   
Finding was acknowledged by the protocol team and classified as intended functionality.

**Locations:**

* [src/Arena.sol\#L233-L242](https://github.com/numbergroup/blobhouse-contracts/blob/bd91d53c8f26590cee10092c4c3ef7bb02c4c3b4/src/Arena.sol#L233-L242)  
* [src/Arena.sol\#L244-L250](https://github.com/numbergroup/blobhouse-contracts/blob/bd91d53c8f26590cee10092c4c3ef7bb02c4c3b4/src/Arena.sol#L244-L250)  
* [src/Arena.sol\#L252-L258](https://github.com/numbergroup/blobhouse-contracts/blob/bd91d53c8f26590cee10092c4c3ef7bb02c4c3b4/src/Arena.sol#L252-L258)

**Description:**

The Arena protocol has centralization risks through privileged roles with immediate, unrestricted control over critical protocol parameters:

* The owner has unrestricted control over fee recipients with no timelock delays:  
  function setFeeRecipients(address protocol, address seasonCurr, address seasonNext) external onlyOwner {  
     if (protocol \== address(0) || seasonCurr \== address(0) || seasonNext \== address(0)) {  
         revert InvalidAddress();  
     }  
     if (protocol \== address(this) || seasonCurr \== address(this) || seasonNext \== address(this)) {  
         revert InvalidAddress();  
     }  
     feeProtocol \= protocol;  
     feeSeasonCurrent \= seasonCurr;  
     feeSeasonNext \= seasonNext;  
     emit FeeRecipientsSet(protocol, seasonCurr, seasonNext);  
  }

* The matcher has complete discretion over player pairing without algorithmic constraints:  
  function matchPlayers(address a, address b) external onlyMatcher nonReentrant {  
     if (a \== b) revert OpponentRequired();  
    
     Entry storage entryA \= queue\[a\];  
     Entry storage entryB \= queue\[b\];  
    
     if (entryA.wager \== 0 || entryB.wager \== 0) revert InactiveEntry();  
     if (entryA.wager \!= entryB.wager) revert WagerMismatch();  
     if (entryA.opponent \!= address(0) || entryB.opponent \!= address(0)) revert AlreadyMatched();  
    
     uint64 currentBlock \= uint64(block.number);  
     entryA.opponent \= b;  
     entryA.matchBlock \= currentBlock;  
     entryB.opponent \= a;  
     entryB.matchBlock \= currentBlock;  
    
     emit Matched(a, b, currentBlock);  
  }


The protocol's trust model requires users to trust that:

* The owner will not redirect 4% of all wagering volume to malicious addresses  
* The matcher will pair players fairly without favoritism or collusion  
* Administrative changes will be made in good faith without exploitation

This represents a significant departure from typical DeFi trustlessness and creates single points of failure that could be exploited or compromised.

**Recommendation:**

* Implement timelocks, for example,  add a 48 hour delay on critical parameter changes to allow community response  
* Require multiple signatures for administrative functions  
* For the matcher role, consider replacing manual matching with algorithmic pairing (FIFO, random, etc.)

This report was produced by Auditware for ETHCF based solely on the   
information provided by them. Auditware provides no guarantees as to the accuracy  
 of the contents of this report and does not make any guarantees that following the advice   
within will prevent security incidents or issues.

Auditware is available for questions or comments about any of the contents of this report. We can be reached at [https://auditware.io/](https://auditware.io/) or by email at [joe@auditware.io](mailto:joe@auditware.io).
