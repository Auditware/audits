# **Review Summary**

The Auditware team has performed a security review on Conduct Protocol, which is a mobile-first, decentralized blockchain that lets you run a full node on your phone.

It uses proof-of-stake, where staking power comes from Bitcoin transaction fees, and is designed to complement Bitcoin. It supports smart contracts, decentralized apps, and uses a messenger server to enable peer-to-peer communication on mobile. The goal is to give users full control of their digital assets with future plans for decentralization and ecosystem growth.

The scope consisted of the following code folders:

* **crates/protocol**  
* **crates/sdk**

Audit commit hash: [e3e6603](https://github.com/ConductProtocol/Conduct/tree/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c) 

The focus of the audit was to evaluate potential security risks associated with the programs in scope, looking to identify vulnerabilities and recommend measures to mitigate identified risks.

Although best practices were applied and the contract was written with security in mind, several High severity issues and unrecommended design patterns were found. Moreover, a lot of TODO’s and developer notes exist across the codebase, indicating a known situation of the code still being in progress. It’s important to address all work-in-progress features in full attention. Detailed explanations for each item can be found in the report.

As with all applications, it is highly recommended to:

* Conduct regular and thorough audits to identify and mitigate vulnerabilities.  
* Achieve the highest possible test coverage, ideally approaching 100%.  
* Maintain comprehensive and up-to-date documentation.  
* Participate in auditing contests and/or a bug bounty program.  
* Implement on-chain monitoring for real-time threat detection and mitigation.

# **Findings Overview**

| Finding | Component | Type | Risk Level |
| :---: | :---: | :---: | :---: |
| **[AW-H-01: Secret Key Stored in Wallet Struct](#aw-h-01:-secret-key-stored-in-wallet-struct)** | sdk | Key Management | ***High*** |
| **[AW-H-02: Reliance on Local System Time for Global Timestamp](#aw-h-02:-reliance-on-local-system-time-for-global-timestamp)** | Block production | Unexpected Behavior | ***High*** |
| **[AW-H-03: Single Point of Failure in Messenger Server](#aw-h-03:-single-point-of-failure-in-messenger-server)** | p2p | DoS | ***High*** |
| **[AW-H-04: Incorrect Chain Selection Due to Misordered VRF Test Hash Comparison](#aw-h-04:-incorrect-chain-selection-due-to-misordered-vrf-test-hash-comparison)** | Chain selection | Integrity Violation | ***High*** |
| **[AW-H-05: Incorrect Chain Selection Due to Favoring of a Shorter Chain](#aw-h-05:-incorrect-chain-selection-due-to-favoring-of-a-shorter-chain)** | Chain selection | Integrity Violation | ***High*** |
| **[AW-H-06: Missing Staking Rewards SQLite Table Breaks Reward Functionality](#aw-h-06:-missing-staking-rewards-sqlite-table-breaks-reward-functionality)** | Blockchain | DoS | ***High*** |
| **[AW-H-07: Lack of Integrity Check for Blockchain Data](#aw-h-07:-lack-of-integrity-check-for-blockchain-data)** | Blockchain | Integrity Violation | ***High*** |
| **[AW-M-01: Strict Bitcoin Confirmation Epoch Check Can Prevent Staker Registration and Renewal](#aw-m-01:-strict-bitcoin-confirmation-epoch-check-can-prevent-staker-registration-and-renewal)** | Consensus | DoS | ***Medium*** |
| **[AW-M-02: Transaction Race Condition in Mempool Validation](#aw-m-02:-transaction-race-condition-in-mempool-validation)** | Mempool | Insufficient Validation | ***Medium*** |
| **[AW-M-03: Insecure Random Number Generation](#aw-m-03:-insecure-random-number-generation)** | sdk | Bad Practice | ***Medium*** |
| **[AW-M-04: Connection Spam To Fill Max Peers](#aw-m-04:-connection-spam-to-fill-max-peers)** | p2p | DoS | ***Medium*** |
| **[AW-M-05: Incorrect Change Allocation in Transaction Payment](#aw-m-05:-incorrect-change-allocation-in-transaction-payment)** | Wallet | Unexpected Behavior | ***Medium*** |
| **[AW-L-01: Silent Failure in Block Ancestry Validation](#aw-l-01:-silent-failure-in-block-ancestry-validation)** | Block tree | Unexpected Behavior | ***Low*** |
| **[AW-L-02: Hardcoded Minimum Timelock Quantity for Staker Registration](#heading=h.erbkhcsnzr3h)** | Consensus | Integrity Violation | ***Low*** |
| **[AW-L-03: No Transaction Fee Priority in Mempool](#aw-l-03:-no-transaction-fee-priority-in-mempool)** | Mempool | Insufficient Validation | ***Low*** |
| **[AW-L-04: Mutex Lock Contention Could Cause Mempool Processing Delays](#aw-l-04:-mutex-lock-contention-could-cause-mempool-processing-delays)** | Mempool | Integrity Violation | ***Low*** |
| **[AW-L-05: No Mempool Transaction Expiry](#aw-l-05:-no-mempool-transaction-expiry)** | Mempool | DoS | ***Low*** |
| **[AW-L-06: Incorrect Minimum Active Stakers Validation](#aw-l-06:-incorrect-minimum-active-stakers-validation)** | Consensus | DoS | ***Low*** |
| **[AW-L-07: Misleading Success Response for Failed Transaction Addition](#aw-l-07:-misleading-success-response-for-failed-transaction-addition)** | Mempool | Error Handling | ***Low*** |
| **[AW-L-08: Duplicate Functions for Shifting Staker Fees](#aw-l-08:-duplicate-functions-for-shifting-staker-fees)** | Blockchain | Redundant Code | ***Low*** |
| **[AW-L-9: Inefficient Database v.s. State Management](#aw-l-9:-inefficient-database-v.s.-state-management)** | Blockchain | Bad Practice | ***Low*** |
| **[AW-L-10: read\_remaining Does Not Update Position Field](#aw-l-10:-read_remaining-does-not-update-position-field)** | sdk | Unexpected Behavior | ***Low*** |
| **[AW-L-11: Improper Error Handling in read\_vec](#aw-l-11:-improper-error-handling-in-read_vec)** | sdk | Unexpected Behavior | ***Low*** |

# 

# **AW-H-01: Secret Key Stored in Wallet Struct** {#aw-h-01:-secret-key-stored-in-wallet-struct}

**Severity:** *High*									**Status:** *Unmitigated*

**Code:**

* [crates/sdk/src/models.rs\#L24-L26](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/sdk/src/models.rs#L24-L26)

**Description:**  
The Wallet struct is instantiated within the **BlockchainApp::create** method using the provided **wallet\_credentials**. These credentials include a **SecretKey**, which is critical for cryptographic operations.

During the audit it was found that these credentials are stored unencrypted, meaning that processes on the running device might access the memory containing the secrets and steal the user’s credentials.

Other than defying best practices of key management, this insecure storage can lead to unauthorized access and compromise of the user's wallet. 

**Recommendations:**

* Use platform-specific secure storage mechanisms to store the **SecretKey** and other sensitive credentials. Both Android and iOS provide secure storage solutions that are designed to protect sensitive information (e.g. Android Keystore and iOS Keychain).

# **AW-H-02: Reliance on Local System Time for Global Timestamp** {#aw-h-02:-reliance-on-local-system-time-for-global-timestamp}

**Severity:** *High*									**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/clock.rs\#L32-L37](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/clock.rs#L32-L37)

* [crates/protocol/src/production/block\_production.rs\#L75-L75](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/production/block_production.rs#L75-L75)

**Description:**  
The **global\_timestamp** function in **clock.rs** relies on the local system time to generate a global timestamp.

This timestamp is used in the block production process in **block\_production.rs**.

If the local system time changes unexpectedly or is manipulated, it can alter the global timestamp, affecting the timing of block production and potentially disrupting the entire blockchain's consensus mechanism.

**Recommendations:**

* Use a trusted time source, such as NTP (Network Time Protocol) servers, to obtain the current time.  
* Implement checks to detect significant deviations in the local system time and raise alerts or correct the time automatically.  
* Consider using a consensus-based approach to determine the current time across multiple nodes in the network.

# **AW-H-03: Single Point of Failure in Messenger Server** {#aw-h-03:-single-point-of-failure-in-messenger-server}

**Severity:** *High*									**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/p2p/messenger/server.rs\#L89-L103](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/p2p/messenger/server.rs#L89-L103)

* [crates/protocol/src/p2p/messenger/client.rs\#L36-L36](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/p2p/messenger/client.rs#L36-L36)  

**Description:**  
During the audit it was found that the **MessengerServer** implementation uses a single **TcpListener** to accept incoming connections.

This creates both a centralization concern and a single-point-of-failure, appearing as a bad implementation decision for the protocol.

An attacker flooding the server with connection requests should trivially be able to lead to a DoS attack and disrupt the entire p2p protocol.

**Recommendations:**

* Consider using a distributed architecture to eliminate the single point of failure.  
* Implement load balancing to distribute incoming connections across multiple instances of the server.  
* Implement rate limiting to prevent a single client from overwhelming the server with connection requests.  
* Use a Content Delivery Network (CDN) like Cloudflare to help mitigate DoS attacks by distributing traffic across multiple servers and providing additional security features.

# **AW-H-04: Incorrect Chain Selection Due to Misordered VRF Test Hash Comparison** {#aw-h-04:-incorrect-chain-selection-due-to-misordered-vrf-test-hash-comparison}

**Severity:** *High* 									**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/consensus/chain\_selection.rs\#L184-L186](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/consensus/chain_selection.rs#L184-L186) 

**Description:**  
The **standard\_chain\_selection** function is part of the chain selection process in a blockchain protocol. It determines which of two competing block headers should be selected based on predefined rules.

Other than height and slot comparisons, the function compares the VRF test hash values of the block headers.

As stated in [this comment](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/consensus/chain_selection.rs#L155-L155), If slots are equal, the function should choose the chain with the lower VRF test hash value. However, during the audit it was found that when **y\_i** is bigger than **x\_i** (effectively meaning **x\_i** has the lower VRF hash value), **Y\_STANDARD** is being chosen instead of **X\_STANDARD**.

Incorrect chain selection can lead to network instability, as the chain with the higher VRF test hash value may be selected over the intended chain. This can cause forks and reduce the overall security and reliability of the blockchain network.

**Recommendations:**

* Ensure that the intended chain (the one with a lower VRF hash value) is being selected on **standard\_chain\_selection**:  
     if y\_i \> x\_i {  
   \-      return Ok(ChainSelectionOutcome::Y\_STANDARD);  
   \+      return Ok(ChainSelectionOutcome::X\_STANDARD);  
     }

# **AW-H-05: Incorrect Chain Selection Due to Favoring of a Shorter Chain** {#aw-h-05:-incorrect-chain-selection-due-to-favoring-of-a-shorter-chain}

**Severity:** *Medium*								**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/consensus/chain\_selection.rs\#L97-L99](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/consensus/chain_selection.rs#L97-L99) 

**Description:**  
The **compare** function in the **ChainSelection** struct is responsible for selecting the correct chain when two competing chains diverge from a common ancestor.

In one of the conditions it checks if the parent block ID of **y\_head** is equal to the block ID of **x\_head**. If this condition is true, the function returns **X\_STANDARD**, indicating that the chain ending with **x\_head** should be selected as the valid chain. However, during the audit we found this logic to be flawed and most like a development error.

If **x\_head** is the parent of **y\_head**, **y\_head** should be favored as it represents the longer chain.

By not favoring the longer chain, it can lead to a situation where the network cannot reach consensus on the state of the blockchain and undermine the security and reliability of the entire system.

**Recommendations:**

* Correct the faulty condition to favor the longer chain out of the two like so:  
     if y\_head.parent\_block\_id \== x\_id {  
   \-     return Ok(ChainSelectionOutcome::X\_STANDARD);  
   \+     return Ok(ChainSelectionOutcome::Y\_STANDARD);  
     }

# **AW-H-06: Missing Staking Rewards SQLite Table Breaks Reward Functionality** {#aw-h-06:-missing-staking-rewards-sqlite-table-breaks-reward-functionality}

**Severity:** *High*									**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/db/blockchain.rs\#L105-L105](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/db/blockchain.rs#L105-L105) 

**Description:**  
The following functions:

* **init\_staker\_rewards**  
* **shift\_staker\_fee\_reward**  
* **remove\_staker**  
* **exit\_staker**  
* **unexit\_staker**  
* **shift\_staker\_accumulated\_fees**  
* **get\_staker\_rewards**

attempt to read from or write staking reward data to the **staker\_rewards** table. However, the **staker\_rewards** table has not been created in the database initialization process, which will cause these functions to fail.

Triggering these functions will cause the application to fail or behave unexpectedly, potentially leading to data inconsistency or denial of service on the staking rewards protocol.

**Recommendations:**

* Add the creation of the **staker\_rewards** table to the database initialization process.

# **AW-H-07: Lack of Integrity Check for Blockchain Data** {#aw-h-07:-lack-of-integrity-check-for-blockchain-data}

**Severity:** *High*									**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/db/blockchain.rs\#L1438-L1438](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/db/blockchain.rs#L1438-L1438) (example)

**Description:**  
The conduct protocol uses SQLITE as the underlying database engine to store blockchain data on the device. However, there are no explicit cryptographic integrity checks (e.g., hashing, digital signatures) implemented in the provided code. 

In cases where sqlite is stored in a shared application folder on the device, or the user had been compromised by a 3rd malicious app that was installed to his device, there’s a risk that a malicious actor might modify the data in the SQLite database and affect any part of the system that’s reliant on its integrity.

**Recommendations:**

* Implement cryptographic integrity checks (e.g., hashing) for all critical blockchain data before writing to the database.  
* Verify the integrity of the data when reading from the database to ensure it has not been tampered with.

# **AW-M-01: Strict Bitcoin Confirmation Epoch Check Can Prevent Staker Registration and Renewal** {#aw-m-01:-strict-bitcoin-confirmation-epoch-check-can-prevent-staker-registration-and-renewal}

**Severity:** *Medium*								**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/consensus/header\_validation.rs\#L291-L293](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/consensus/header_validation.rs#L291-L293) 

**Description:**  
The **verify\_bitcoin\_confirmation** function is designed to ensure that a given Bitcoin header was created long enough ago to be confident that it won't be re-organized. Specifically, it checks that the Bitcoin header was created within the N-2 epoch by comparing timestamps.

During the audit it was found that the function might not correctly handle edge cases where the epoch boundaries are crossed, leading to incorrect validation results.

For example, if the Bitcoin header timestamp is near the boundary of an epoch, the conversion to slot and epoch might not be accurate, causing the function to fail the validation check incorrectly making valid stakers to be incorrectly rejected if their Bitcoin header timestamp is near the boundary of an epoch.

**Recommendations:**

* Consider adding an additional checks to handle edge cases where the Bitcoin header timestamp is near the boundary of an epoch, for example:  
   \-  if bitcoin\_header\_epoch \!= current\_epoch \- 2 {  
   \+  if bitcoin\_header\_epoch \!= current\_epoch \- 2 && bitcoin\_header\_epoch \!= current\_epoch \- 1 {  
         return Ok(Some(HeaderValidationError::StakerNotConfirmedOnBitcoin));  
     }

# **AW-M-02: Transaction Race Condition in Mempool Validation** {#aw-m-02:-transaction-race-condition-in-mempool-validation}

**Severity:** *Medium*								**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/ledger/mempool.rs\#L67-L122](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/ledger/mempool.rs#L67-L122) 

**Description:**

During the audit it was found that the mempool insecurely implements a retry mechanism, allowing transaction order manipulation.

The implementation retries for up to 5 seconds when encountering a **MissingInput** error, which allows an attacker to delay broadcasting a required dependency transaction and then release it strategically within this window.

To abuse this issue, an attacker might follow these steps:

* Attacker submits a transaction referencing a yet-to-be-broadcast dependency.  
* The mempool encounters a **MissingInput** error and retries for 5 seconds.  
* The attacker delays broadcasting the dependency transaction.  
* At a strategic moment, the attacker introduces the dependency transaction.  
* The mempool accepts the original transaction, but now it was already ordered strategically relative to others.

This opens up risks of manipulating transaction ordering, fee sniping and more.

**Recommendations:**

* Remove Fixed-Time Retries, Use Dependency Tracking Instead  
  * Instead of blindly retrying, maintain a dependency-aware mempool  
  * Track missing input dependencies and only include transactions once all dependencies exist  
  * If a transaction has missing inputs, store it in **pending\_transactions**  
  * When the dependency appears, automatically retry

# **AW-M-03: Insecure Random Number Generation** {#aw-m-03:-insecure-random-number-generation}

**Severity:** *Medium*								**Status:** *Unmitigated*

**Code:**

* [crates/sdk/src/cryptography.rs\#L173-L178](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/sdk/src/cryptography.rs#L173-L178)   
* [crates/sdk/src/cryptography.rs\#L94-L99](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/sdk/src/cryptography.rs#L94-L99) 

**Description:**

During the audit it was found that the **generate\_random\_bytes** function uses **rand::thread\_rng()** to generate random bytes.

This method returns a thread-local generator backed by **StdRng** \- a deterministic pseudo-random number that is seeded once from **OsRng**.

The following risks arise from this

* It’s not guaranteed to be cryptographically secure \- **StdRng** is not guaranteed to meet the requirements for cryptographic randomness \- even though it’s seeded securely.  
* No **CryptoRng** guarantee \- The random number generator is not explicitly constrained to implement the **CryptoRng** trait (from **rand\_core**), which is a marker for random number generators intended for cryptographic use.  
* Using **OsRng** or constraining your generator to **CryptoRng** makes your intent explicit and guards against regressions or subtle changes in how **thread\_rng()** behaves across versions.

This can lead to predictable random numbers, which can compromise cryptographic operations that rely on secure randomness.

**Recommendations:**

* Consider switching to use **rand::rngs::OsRng**, which draws directly from the operating system's CSPRNG (Cryptographically Secure Pseudorandom Number Generator):  
     pub fn generate\_random\_bytes(len: u32) \-\> Vec\<u8\> {  
   **\-**    let mut rng \= rand::thread\_rng();  
        let mut ret \= vec\!\[0u8; len as usize\];  
   **\-**    rng.fill\_bytes(&mut ret);  
   **\+**    OsRng.fill\_bytes(&mut ret);  
        ret  
    }  
* For further reference: [https://github.com/rust-random/rand/issues/1358](https://github.com/rust-random/rand/issues/1358) 

# **AW-M-04: Connection Spam To Fill Max Peers** {#aw-m-04:-connection-spam-to-fill-max-peers}

**Severity:** *Medium*								**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/p2p/network.rs\#L89-L89](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/p2p/network.rs#L89-L89) 

**Description:**

The **MAX\_PEERS** constant is hardcoded to **20**, which may allow attackers to DOS peers by targeting a peer node with 20 attacker-controlled nodes so the attacker could control the peer’s view of the network.

An attacker can continuously initiate discovery requests until they pair with a desired peer and flood the target node, allowing them to censor transactions, delay confirmations, consume resources, and even manipulate the victim’s view of the ledger.

**Recommendations:**

* Add a rate limit and/or backoff mechanism to the peer discovery function  
* Periodically prune connected peers  
* Enforce that at least some of the connected peers on a single node must be from connection requests originating from that node (prevents an attacker from controlling 100% of the peer nodes)  
* Bundle the peer discovery and peer connection/communication functions so that a node can only establish communication with a node that they have been randomly assigned to (preventing an attacker from specifying a node they want to connect to)

# **AW-M-05: Incorrect Change Allocation in Transaction Payment** {#aw-m-05:-incorrect-change-allocation-in-transaction-payment}

**Severity:** *Medium*								**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/wallet.rs\#L251-L258](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/wallet.rs#L251-L258) 

**Description:**

The **pay** function in the **Wallet** struct is responsible for constructing a new transaction that pays for some transaction. It ensures that the transaction has enough inputs to cover the outputs and the required fees. If there is any leftover amount (change), it should be returned to the sender's address.

It also calculates the change output and adds it to the transaction outputs if the change is greater than or equal to the required minimum quantity.

However, if the change is greater than or equal to the required minimum quantity, this amount is not correctly deducted from the sender. Since the **required\_minimum\_quantity** function ensures that the outputs include a minimum cost based on metadata such as assets, labels, and additional data, failing to account for this deduction can result in a transaction where the sender does not actually cover the required minimum storage cost, potentially leading to inconsistencies in fee calculations.

**Recommendations:**

* Modify the change assignment logic to first deduct the required minimum quantity before assigning the remaining amount to the change output.

# **AW-L-01: Silent Failure in Block Ancestry Validation** {#aw-l-01:-silent-failure-in-block-ancestry-validation}

**Severity:** *Low*									**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/block\_tree.rs\#L43-L50](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/block_tree.rs#L43-L50) 

**Description:**  
The **trace\_to\_genesis** function in the **BlockTree** struct is responsible for traversing the blockchain from a given block ID back to the genesis block, constructing a path of block IDs. This function is critical for ensuring the integrity of the blockchain's structure, as it validates the ancestry of blocks and ensures they are properly connected to the chain.

During the audit we found that **trace\_to\_genesis** does not handle the case where a block's parent cannot be found in the database, potentially returning an incomplete or empty path without error.

This can lead to invalid or orphaned blocks being mistakenly assumed to be connected to the chain.

While it does not directly lead to a loss of funds, it can cause significant data inconsistencies and incorrect state transitions, which can have cascading effects on the system's integrity.

**Recommendations:**

* To handle the case where a block's parent cannot be found, you can modify the **trace\_to\_genesis** function to return an error if the parent block is not found in the database. This ensures that the function does not silently return an incomplete or empty path, for example:  
   \+ if head \!= Genesis::PARENT\_ID {  
   \+     return Err(CndtError::new("Parent block not found in the database"));  
   \+ }  
     Ok(path.into\_iter().collect())

# **AW-L-02: Hardcoded Minimum Timelock Quantity for Staker Registration**

**Severity:** *Low*									**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/consensus/header\_validation.rs\#L347-L357](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/consensus/header_validation.rs#L347-L357) 

**Description:**  
During the audit it was found that the value for the minimum timelock quantity required for staker registration is hardcoded and set to **5000000**, which may not be sufficient under certain network conditions.

An attacker could exploit this by creating a Bitcoin transaction with a timelock quantity just above the hardcoded value, potentially registering as a staker with insufficient stake. This could undermine the security and integrity of the staking process.

**Recommendations:**

* Replace the hardcoded value with a configurable parameter that can be adjusted based on network conditions.  
* Implement a mechanism to dynamically adjust the minimum timelock quantity required for staker registration.

# 

# **AW-L-03: No Transaction Fee Priority in Mempool** {#aw-l-03:-no-transaction-fee-priority-in-mempool}

**Severity:** *Low* 								**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/ledger/mempool.rs\#L75-L81](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/ledger/mempool.rs#L75-L81)

**Description:**  
During the audit it was found that the mempool does not prioritize transactions based on fees.

It simply rejects any new transaction that conflicts with an existing one, even if the new transaction offers a higher fee.

This allows an attacker to submit a low-fee transaction first, blocking a higher-fee transaction from replacing it, effectively manipulating transaction inclusion.

**Recommendations:**

* Implement Replace-by-Fee (RBF) to allow higher-fee transactions to replace lower-fee ones, ensuring that the mempool prioritizes transactions based on the fees offered.

# **AW-L-04: Mutex Lock Contention Could Cause Mempool Processing Delays** {#aw-l-04:-mutex-lock-contention-could-cause-mempool-processing-delays}

**Severity:** *Low*									**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/ledger/mempool.rs\#L72-L72](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/ledger/mempool.rs#L72-L72)  
* [crates/protocol/src/ledger/mempool.rs\#L129-L129](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/ledger/mempool.rs#L129-L129)  
* [crates/protocol/src/ledger/mempool.rs\#L146-L146](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/ledger/mempool.rs#L146-L146)

**Description:**  
During the audit it was found that the **Mempool** structure uses a **Mutex** to protect **MempoolState**, but multiple asynchronous functions (**add**, **read**, **update\_to**) acquire locks on it.

**Mutex** allows only one thread to access the protected data at a time, which can lead to significant wait times when multiple threads are trying to acquire the lock concurrently.

If transaction submission volume is high, contention on the mutex could introduce performance bottlenecks and processing delays. 

**Recommendations:**

* Consider replacing **Mutex** with **RwLock** to allow concurrent reads.  
* Release the lock before calling asynchronous functions to prevent holding the lock for extended periods.

# **AW-L-05: No Mempool Transaction Expiry** {#aw-l-05:-no-mempool-transaction-expiry}

**Severity:** *Low*									**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/ledger/mempool.rs\#L127-L127](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/ledger/mempool.rs#L127-L127)

**Description:**  
During the audit it was found that the mempool implementation does not have a proper expiration mechanism for transactions that never get confirmed.

This means attackers can flood the mempool with transactions that have no chance of being included in a block, leading to unnecessary memory consumption and potential denial of service.

**Recommendations:**

* Implement a TTL (time-to-live) mechanism for unconfirmed transactions.  
* Remove transactions that remain unconfirmed for too long.

# **AW-L-06: Incorrect Minimum Active Stakers Validation** {#aw-l-06:-incorrect-minimum-active-stakers-validation}

**Severity:** *Low*									**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/consensus/header\_validation.rs\#L166-L173](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/consensus/header_validation.rs#L166-L173) 

**Description:**  
During the audit we discovered a discrepancy between the documentation and the implementation.

The current implementation condition checks for less than 3 stakers (**staker\_count \< 3**). This does not prevent a scenario where exactly three stakers remain, and one exits, reducing the count to two.

This oversight allows a situation where two remaining validator stakers could collude to produce non-legitimate blocks, potentially disrupting the consensus standards.

**Recommendations:**

* Update the condition to check for less than or equal to 3 stakers:  
   **\-** if staker\_count \< 3 {  
   **\+** if staker\_count \<= 3 {  
        return Ok(Some(HeaderValidationError::InsufficientStakers));  
     }


# **AW-L-07: Misleading Success Response for Failed Transaction Addition** {#aw-l-07:-misleading-success-response-for-failed-transaction-addition}

**Severity:** *Low*									**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/ledger/mempool.rs\#L99-L99](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/ledger/mempool.rs#L99-L99) 

**Description:**  
The add function in the **Mempool** struct returns **Ok(())** after retrying to add a transaction with missing inputs up to 5 times. This return value can be misleading as it suggests that the transaction was successfully added to the mempool, even though it was not.

Improper handling is extra important in sensitive locations in the code and should always be thought of thoroughly. This specific oversight can lead to confusion and incorrect assumptions about the state of the mempool, potentially affecting the reliability of the system.

**Recommendations:**

* Return a specific error indicating that the transaction could not be added due to missing inputs after the maximum number of retries.  
     if i \== 4 {  
   **\-**    return Ok(());  
   **\+**    return Err(CndtError::new(format\!("Failed to add transaction due to  
         missing input: {:?}", reference)));  
     }  
  


# **AW-L-08: Duplicate Functions for Shifting Staker Fees** {#aw-l-08:-duplicate-functions-for-shifting-staker-fees}

**Severity:** *Low*									**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/db/blockchain.rs\#L1302-L1310](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/db/blockchain.rs#L1302-L1310)   
* [crates/protocol/src/db/blockchain.rs\#L1254-L1264](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/db/blockchain.rs#L1254-L1264) 

**Description:**  
During the audit we discovered two almost identical functions, **shift\_staker\_fee\_reward** and **shift\_staker\_accumulated\_fees**, which perform the same operation of updating the **accumulated\_fees** field in the **staker\_rewards** table.

Both use this SQL statement:

* **UPDATE staker\_rewards SET accumulated\_fees \+= ? WHERE vrf\_vk \= ?**

Even if not a direct security vulnerability at time of writing this report, having two functions to pose the same purposes could create serious instability in the code over time.

For example, consider a case where both functions are randomly used by different functionalities, and an error or update needs to be done \- that could create a room for major mistakes as one function will be updated and not both, creating overseen instability across the system.

**Recommendations:**

* Remove one of the duplicate functions to avoid redundancy and potential maintenance issues.  
* Ensure that the remaining function is used consistently throughout the codebase.

# **AW-L-9: Inefficient Database v.s. State Management** {#aw-l-9:-inefficient-database-v.s.-state-management}

**Severity:** *Low*									**Status:** *Unmitigated*

**Code:**

* [crates/protocol/src/db/blockchain.rs\#L211-L220](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/db/blockchain.rs#L211-L220) (SQLite usage)  
* [crates/protocol/src/consensus/staker\_tracker.rs\#L286-L298](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/protocol/src/consensus/staker_tracker.rs#L286-L298) (In-memory usage)

**Description:**  
During the audit we found a bad pattern evolving working with the database.

For example, the current implementation of the **StakerTracker** and **TrackerState** structs manages staking information in-memory while other related information is stored in a persistent database. This inconsistency can lead to potential data loss or corruption if the state is not properly synchronized between the two storage mechanisms. Specifically, the **insert\_staker** function in the **TrackerState** struct inserts a **staker** into the epochs’ HashMap without ensuring that this state is consistently reflected in the database.

In other cases, data is managed and updated in the database. This fact, with the addition of a not clear enough database design documentation, can result in incorrect staking information being used for eligibility calculations, reward distributions, and other critical operations. In the worst case, it could lead to data loss or corruption, affecting the integrity and reliability of the staking system.

**Recommendations:**

* Consider moving all state management to the database to avoid potential inconsistencies.

# **AW-L-10: read\_remaining Does Not Update Position Field** {#aw-l-10:-read_remaining-does-not-update-position-field}

**Severity:** *Low*									**Status:** *Unmitigated*

**Code:**

* [crates/sdk/src/codecs.rs\#L190-L192](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/sdk/src/codecs.rs#L190-L192) 

**Description:**  
The **read\_remaining** method in the **BytesCursor** struct does not update the position field after consuming the remaining bytes.

This can lead to logical inconsistencies, as subsequent calls to other read methods will incorrectly assume that the bytes are still available, potentially leading to denial-of-service (DoS) or incorrect state transitions in critical parts of the protocol.

**Recommendations:**

* Update the position field in the **read\_remaining** method to reflect that all bytes have been consumed. For example:  
     pub fn read\_remaining(&mut self) \-\> DecodeResult\<Vec\<u8\>\> {  
   **\-**     Ok(self.bytes\[self.position..\].to\_vec())  
   **\+**     let result \= self.bytes\[self.position..\].to\_vec();  
   **\+**     self.position \= self.bytes.len();  
   **\+**     Ok(result)  
     }


# **AW-L-11: Improper Error Handling in read\_vec** {#aw-l-11:-improper-error-handling-in-read_vec}

**Severity:** *Low*									**Status:** *Unmitigated*

**Code:**

* [crates/sdk/src/codecs.rs\#L488-L496](https://github.com/ConductProtocol/Conduct/blob/e3e6603d43db84d4be05cf1ab29c9cd111f6b18c/crates/sdk/src/codecs.rs#L488-L496) 

**Description:**  
The **read\_vec** function does not properly handle errors during the decoding process. If the provided decode function returns an error (e.g., due to invalid or corrupted input), **read\_vec** continues decoding subsequent elements instead of halting immediately.

This behavior can lead to undefined behavior, incorrect data processing, or partial results being returned, which may compromise the integrity of the system.

**Recommendations:**

* Modify **read\_vec** to stop decoding immediately upon encountering an error. For example:  
     pub fn read\_vec\<T, F\>(&mut self, decode: F) \-\> DecodeResult\<Vec\<T\>\>  
     where  
         F: Fn(&mut Self) \-\> DecodeResult\<T\>,  
     {  
         let count \= self.read\_u32()?;  
   **\-**     let results: Vec\<DecodeResult\<T\>\> \= (0..count).map(|\_| decode(self)).collect();  
   **\-**     results.into\_iter().collect()  
   **\+**     let mut results \= Vec::with\_capacity(count as usize);  
   **\+**     for \_ in 0..count {  
   **\+**         results.push(decode(self)?);  
   **\+**     }  
   **\+**     Ok(results)  
     }




This report was produced by Auditware for Conduct based solely on the   
information provided by Conduct. Auditware provides no guarantees as to the accuracy  
 of the contents of this report and does not make any guarantees that following the advice   
within will prevent security incidents or issues.

Auditware is available for questions or comments about any of the contents of this report. We can be reached at [https://auditware.io/](https://auditware.io/) or by email at [joe@auditware.io](mailto:joe@auditware.io).
