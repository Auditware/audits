# **Review Summary**

An audit was performed on Rarible’s Solana-based NFT marketplace that allows  
minting of editions and items, listing NFTs, creating collections and more.

The scope consisted of 3 Anchor programs:

* **rarible\_editions** \-  functionality for creation and management of NFT editions  
* **rarible\_editions\_controls** \- interface for managing NFT editions, importing from rarible\_editions  
* **rarible\_marketplace** \- functionality for creation of NFT markets, seemingly inspired by [wen-program-library](https://github.com/wen-community/wen-program-library)

Audit commit hash: [60eeaad](https://github.com/rarible/protocol-contracts-svm-private/tree/60eeaad48d436a064c96ccb8168dbea3eb57c4b8)

The focus of the audit was to evaluate potential security risks associated with the programs in scope, looking to identify vulnerabilities and recommend measures to mitigate identified risks.

Although best practices were applied and the contract was written with security in mind, several issues were identified during the audit, for example, partial loss of funds due to premature closure of an order account. Detailed explanations per item can be found in the report.

As with most smart contracts, it is highly recommended to:

* Conduct regular and thorough audits to identify and mitigate vulnerabilities.  
* Achieve the highest possible test coverage, ideally approaching 100%.  
* Maintain comprehensive and up-to-date documentation.  
* Participate in auditing contests and/or a bug bounty program.  
* Implement on-chain monitoring for real-time threat detection and mitigation.

# **Findings Overview**

| Finding | Component | Type | Risk Level |
| :---: | :---: | :---: | :---: |
| **[AW-C-01: Loss of Funds Due to Premature Closure of Partially-filled Orders](#aw-c-01:-loss-of-funds-due-to-premature-closure-of-partially-filled-orders)** | rarible\_marketplace | Loss of Funds | *Critical* |
| **[AW-H-01: Missing Market Closure Functionality](#aw-h-01:-missing-market-closure-functionality)** | rarible\_marketplace | Emergency Response | *High* |
| **[AW-M-01: Incorrect Fee Calculation](#aw-m-01:-incorrect-fee-calculation)** | rarible\_editions\_controls | Loss of Funds | *Medium* |
| **[AW-M-02: Fee Only Sent to First Recipient](#aw-m-02:-fee-only-sent-to-first-recipient)** | rarible\_editions\_controls | Loss of Funds | *Medium* |
| **[AW-M-03: Inconsistent NFT Fee Deduction](#aw-m-03:-inconsistent-nft-fee-deduction)** | rarible\_marketplace | Loss of Funds | *Medium* |
| **[AW-M-04: Unlimited NFT Minting Setting Can Become Unchangeable](#aw-m-04:-unlimited-nft-minting-setting-can-become-unchangeable)** | rarible\_editions | DoS | *Medium* |
| **[AW-M-05: Missing Share Sum Validation](#aw-m-05:-missing-share-sum-validation)** | rarible\_editions\_controls | DoS | *Medium* |
| **[AW-M-06: Incorrect Account Mutability](#aw-m-06:-incorrect-account-mutability)** | rarible\_editions | DoS | *Medium* |
| **[AW-L-01: Improper Handling of NFT Listing Order Size](#aw-l-01:-improper-handling-of-nft-listing-order-size)** | rarible\_marketplace | Unexpected Behavior | *Low* |
| **[AW-L-02: Missing Address Validation In Update Functionalities](#aw-l-02:-missing-address-validation-in-update-functionalities)** | rarible\_editions\_controls | Unexpected Behavior | *Low* |
| **[AW-L-03: No Maximum Market Fee](#aw-l-03:-no-maximum-market-fee)** | rarible\_marketplace | Centralization | *Low* |
| **[AW-L-04: Unchecked Copied Arrays](#aw-l-04:-unchecked-copied-arrays)** | rarible\_editions\_controls | Validation | *Low* |
| **[AW-L-05: Overusing UncheckedAccount](#aw-l-05:-overusing-uncheckedaccount)** | All programs | Validation | *Low* |
| **[AW-L-06: Lack of Documentation](#aw-l-06:-lack-of-documentation)** | All programs | Documentation | *Low* |
| **[AW-I-01: Comment Mismatches Implementation](#aw-i-01:-comment-mismatches-implementation)** | rarible\_editions | Documentation | *Informational* |
| **[AW-I-02: mint\_with\_controls Optimizations](#aw-i-02:-mint_with_controls-optimizations)** | rarible\_editions\_controls | Optimization | *Informational* |

# **AW-C-01: Loss of Funds due to Premature Closure of Partially-filled Orders** {#aw-c-01:-loss-of-funds-due-to-premature-closure-of-partially-filled-orders}

**Severity:** *Critical*							**Status:** *Unmitigated*

**Code:**

* [rarible\_marketplace/src/instructions/order/fill.rs\#L59](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_marketplace/src/instructions/order/fill.rs#L59)

**Description:**  
During the audit it was found that an order account on the **rarible\_marketplace** program can be prematurely closed, leading to loss of user funds.

Using [BidNFT](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_marketplace/src/instructions/order/bid.rs#L73-L73) a user (maker) can create a buy order in the marketplace, which initiates a token transfer to the Order account [order\_payment\_ta](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_marketplace/src/instructions/order/bid.rs#L57-L57).

On [FillOrder](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_marketplace/src/instructions/order/fill.rs#L36-L36) a taker can then fulfill the placed order. In intended functionality, a taker can perform a partial fill, in which the order size is reduced by the amount filled, but the order should remain active until its size reaches zero or is explicitly canceled (intended to allow the taker to fill the remaining portion later).

However, it was found that due to the [close constraint](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_marketplace/src/instructions/order/fill.rs#L59) used on **FillOrder**, what actually happens on a partially filled order is an automatic closure of the order account when the transaction ends, meaning that the NFTs remain under the **order\_payment\_ta** account but become inaccessible to the user, resulting in the user losing the unfilled portion.

**Recommendations:**

* Since the logic in the **FillOrder** handler seems to be designed to explicitly handle partial fills, simply remove the `close = maker` constraint from the account attribute and instead use the remaining existing functionality to close the account programmatically when the condition `new_size == 0` occurs.

# **AW-H-01: Missing Market Closure Functionality** {#aw-h-01:-missing-market-closure-functionality}

**Severity:** *High*							**Status:** *Unmitigated*

**Code:**

* [rarible\_marketplace/src/state/market.rs\#L70-L117](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_marketplace/src/state/market.rs#L70-L117)

**Description:**  
During the audit it was found that the **Market** struct does not have an implementation to update the market state to “closed”, despite having checks to determine if it’s closed or not (e.g. [is\_active](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_marketplace/src/state/market.rs#L97-L99)).

This creates a security and operational risk, as the market owner is unable to react to security incidents, misconfigurations, or emergency shutdown scenarios.

Without the ability to close the market, any discovered vulnerabilities or active exploits remain live, leading to possible financial loss, data corruption, or system compromise.

**Recommendations:**

* Implement a Market closure functionality within the **Market** struct, for example:  
  \+   /// close the market  
  \+   pub fn close(&mut self) {  
  \+       self.state \= MarketState::Closed.into();  
  \+   }


# **AW-M-01: Incorrect Fee Calculation** {#aw-m-01:-incorrect-fee-calculation}

**Severity:** *Medium*							**Status:** *Unmitigated*

**Code:**

* [rarible\_editions\_controls/src/instructions/mint\_with\_controls.rs\#L235-L237](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions_controls/src/instructions/mint_with_controls.rs#L235-L237)

**Description:**  
During the audit it was found that when **is\_fee\_flat** is true, **total\_fee** is not being properly deducted from **price\_amount**.

This results in the **remaining\_amount** being miscalculated, causing the treasury not to receive the correct fee amount leading to loss of funds for the treasury and damage overall program integrity.

**Recommendations:**

* Ensure **total\_fee** is being properly deducted from **price\_amount**, for example:  
  remaining\_amount \= price\_amount  
  \+ .checked\_sub(total\_fee)  
  \+ .ok\_or(EditionsControlsError::FeeCalculationError)?;

# **AW-M-02: Fee Only Sent to First Recipient** {#aw-m-02:-fee-only-sent-to-first-recipient}

**Severity:** *Medium*							**Status:** *Unmitigated*

**Code:**

* [rarible\_editions\_controls/src/instructions/mint\_with\_controls.rs\#L282-L282](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions_controls/src/instructions/mint_with_controls.rs#L282-L282)

**Description:**  
In the **mint\_with\_controls** function, platform fees are distributed to recipients in a loop like so:

   // Distribute fees to recipients  
   for (i, recipient\_struct) in recipients.iter().enumerate() {  
	..

During the audit, it was found that there is a [break](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions_controls/src/instructions/mint_with_controls.rs#L282-L282) statement inside of the loop, that causes the loop to exit after processing only the first recipient.

As a result, only the first recipient receives the platform fee, and subsequent ones are ignored. This causes other fee recipients to not receive their fees, and damages the program’s integrity.

**Recommendations:**

* To ensure that platform fees are correctly distributed to all intended recipients, the **break** statement should be removed, and the loop should continue to process all recipients.  
* Additionally, ensure you handle multiple recipient accounts correctly by matching the index to the appropriate account.

# **AW-M-03: Inconsistent NFT Fee Deduction** {#aw-m-03:-inconsistent-nft-fee-deduction}

**Severity:** *Medium*							**Status:** *Unmitigated*

**Code:**

* [rarible\_marketplace/src/instructions/order/fill.rs\#L629-L812](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_marketplace/src/instructions/order/fill.rs#L629-L812)

**Description:**

During the audit, an inconsistency was found between the fee deduction done on WNS versus non-WNS NFTs, following is the relevant part of the inconsistent logic:

if \*nft\_program\_key \== WNS\_PID {

   // ...

   seller\_received\_amount \= seller\_received\_amount

   	.checked\_sub(royalties).unwrap()

   	.checked\_sub(fee\_amount).unwrap();

} else {

   // ...

   seller\_received\_amount \= seller\_received\_amount

.checked\_sub(royalties).unwrap();

}

When `nft_program_key == WNS_PID`, the **fee\_amount** is subtracted from **seller\_received\_amount** alongside royalties before proceeding to the payment transfer. In the else case for other NFT program keys, the **fee\_amount** is not deducted from **seller\_received\_amount** prior to transferring the payment.

However, the code still attempts to process the fee payment later via `ctx.accounts.transfer_fee`.

This inconsistency can lead to incorrect fee calculations and potential financial discrepancies. Sellers might receive different amounts depending on the NFT program key, which could result in unexpected behavior and financial losses.

**Recommendations:**

* Ensure that the **fee\_amount** is consistently deducted from **seller\_received\_amount** in all cases before proceeding to the payment transfer. This will standardize the fee deduction process and prevent any discrepancies.  
  if \*nft\_program\_key \== WNS\_PID {  
     // ...  
  } else {  
     // ...  
  \-   seller\_received\_amount \= seller\_received\_amount.checked\_sub(royalties\_amount).unwrap();  
    
  \+   seller\_received\_amount \= seller\_received\_amount.checked\_sub(royalties).unwrap().checked\_sub(fee\_amount).unwrap();  
  }


# **AW-M-04: Unlimited NFT Minting Setting Can Become Unchangeable** {#aw-m-04:-unlimited-nft-minting-setting-can-become-unchangeable}

**Severity:** *Medium*							**Status:** *Unmitigated*

**Code:**

* [rarible\_editions/src/instructions/modify.rs\#L54-L57](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions/src/instructions/modify.rs#L54-L57)

**Description:**

During the audit it was found that the unlimited minting option, defined when [max\_number\_of\_tokens](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions/src/state.rs#L23-L24) is set to 0, can not be activated after tokens are minted.

This occurs due to the following enforcement on **modify**:

require\!(

    max\_number\_of\_tokens \>= editions\_deployment.number\_of\_tokens\_issued,

    EditionsError::InvalidMaxNumberOfTokens

 );

After tokens are minted, **number\_of\_tokens\_issued** is no longer 0, meaning that the value 0 used as the setting of “unlimited” can never be updated, as the enforcement would return an error and stop execution before the update is performed.

This will prevent the unlimited option from ever being usable after tokens are issued. Other than harming user experience, in a theoretical edition launch, if admins planned to depend on the unlimited functionality, for example for a promotional post-launch period, this could even impair the success of the launch.

**Recommendation:**

* Add logic to handle a case where **max\_number\_of\_tokens** needs to be set to 0, like so:  
  require\!(  
  \-   max\_number\_of\_tokens \>= editions\_deployment.number\_of\_tokens\_issued,  
  \+   max\_number\_of\_tokens \== 0 || max\_number\_of\_tokens \>= editions\_deployment.number\_of\_tokens\_issued,  
     EditionsError::InvalidMaxNumberOfTokens  
  );

# **AW-M-05: Missing Share Sum Validation** {#aw-m-05:-missing-share-sum-validation}

**Severity:** *Medium*							**Status:** *Unmitigated*

**Code:**

* [rarible\_editions\_controls/src/instructions/update\_platform\_fee.rs\#L68-L68](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions_controls/src/instructions/update_platform_fee.rs#L68-L68)  
* [rarible\_editions\_controls/src/instructions/mint\_with\_controls.rs\#L226-L230](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions_controls/src/instructions/mint_with_controls.rs#L226-L230)

**Description:**

During the audit it was found that the **recipients\_array** filled when calling **update\_platform\_fee** does not validate that the added shares of the recipients equals to 100\.

Due to the fact that **process\_platform\_fees** relies on shares being [exactly 100](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions_controls/src/instructions/mint_with_controls.rs#L226-L230), this could lead to a temporary DoS in the minting process, by causing an error to be triggered whenever **process\_platform\_fees** is called, leading to service disruption and preventing legitimate minting operations. This could result in financial losses and degraded user experience.

**Recommendations:**

* Add a check on **update\_platform\_fee** to ensure the sum of the recipient shares is equal to 100, to prevent DoS when **process\_platform\_fees** is triggered.  
  \+   let total\_share: u32 \= recipients\_array.iter().map(|r| r.share).sum();  
  \+   if total\_share \!= 100 {  
  \+       return Err(ProgramError::InvalidArgument.into());  
  \+   }

# **AW-M-06: Incorrect Account Mutability** {#aw-m-06:-incorrect-account-mutability}

**Severity:** *Medium*							**Status:** *Unmitigated*

**Code:**

* [rarible\_editions/src/instructions/metadata/remove.rs\#L33-L33](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions/src/instructions/metadata/remove.rs#L33-L33) 

**Description:**

During the audit it was found that an account intended for mutation is not marked as mutable with the **mut** keyword.

The **mint** account under the [RemoveMetadata](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions/src/instructions/metadata/remove.rs#L20-L36) struct is not marked as mutable, but sent as the **metadata** argument to the anchor [remove\_key](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions/src/instructions/metadata/remove.rs#L51-L51) instruction.

**remove\_key** is implemented like so:

pub fn remove\_key(  
   program\_id: &Pubkey,  
   metadata: &Pubkey,  
   update\_authority: &Pubkey,  
   key: String,  
   idempotent: bool,  
) \-\> Instruction {  
   let data \= TokenMetadataInstruction::RemoveKey(RemoveKey { key, idempotent });  
   Instruction {  
       program\_id: \*program\_id,  
       accounts: vec\!\[  
           AccountMeta::new(\*metadata, false),  
           AccountMeta::new\_readonly(\*update\_authority, true),  
       \],  
       data: data.pack(),  
   }  
}

This results in the creation of an **AccountMeta** instance with **is\_writable** set to **true**.

This oversight can lead to unintended protocol behavior or failure when the program will try to modify the immutable **mint** account, resulting in a DoS.

**Recommendations:**

* Mark the **mint** account as mutable in the **RemoveMetadata** struct to ensure it can be correctly modified by the **remove\_key** instruction.  
  \+  \#\[account(mut)\]  
     pub mint: Box\<InterfaceAccount\<'info, Mint\>\>,


# **AW-L-01: Improper Handling of NFT Listing Order Size**  {#aw-l-01:-improper-handling-of-nft-listing-order-size}

**Severity:** *Low*									**Status:** *Unmitigated*

**Code:**

* [rarible\_marketplace/src/instructions/order/list.rs\#L114-L127](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_marketplace/src/instructions/order/list.rs#L114-L127)

**Description:**

The `data.size` parameter in the handler function of the **list.rs** file is not properly validated or enforced to be 1, even though it is assumed to be so by the comments.

   Order::init(  
...  
       data.size, // always 1  
       data.price,  
       OrderState::Ready.into(),  
       true,  
   );

Since **data** is user-controlled, size can be set to any value contradicting the expectation.

This would mean users could potentially list multiple NFTs in a single order. This could disrupt the intended functionality of the marketplace, where each order is supposed to represent a single NFT.

Moreover, if in the future other parts of the program will depend on an NFT listing always being one, violating that truth might result in unexpected, potentially harmful behavior.

**Recommendations:**

* Enforce the `data.size` parameter to be 1 within the handler function. This ensures that each order only lists a single NFT, maintaining the intended functionality and preventing potential misuse.  
     Order::init(  
  ...  
  \-      data.size, // always 1  
  \+      1, // always 1  
         data.price,  
         OrderState::Ready.into(),  
         true,  
     );

# 

# **AW-L-02: Missing Address Validation In Update Functionalities** {#aw-l-02:-missing-address-validation-in-update-functionalities}

**Severity:** *Low*									**Status:** *Unmitigated*

**Code:**

* [rarible\_editions\_controls/src/instructions/transfer\_ownership.rs\#L28-L33](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions_controls/src/instructions/transfer_ownership.rs#L28-L33) 

* [rarible\_editions\_controls/src/instructions/update\_platform\_fee\_secondary\_admin.rs\#L30-L38](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions_controls/src/instructions/update_platform_fee_secondary_admin.rs#L30-L38)

**Description:**

During the audit, two instances were found in which an address is being updated, without sufficient validations that ensure the destination addresses are as expected.

For example, in [update\_platform\_fee\_secondary\_admin](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions_controls/src/instructions/update_platform_fee_secondary_admin.rs#L30-L30), where the **platform\_fee\_secondary\_admin** is set to **input.new\_admin**, if the admin mistakenly sets an incorrect or invalid address, it could result in a potential DoS or unexpected behavior.

**Recommendations:**

* On every update involving an address, perform validations to ensure the address is in the expected format, for example:  
  * Ensure it’s not a program address or PDA  
  * Ensure the address exists  
  * Ensure the address can receive token

# **AW-L-03: No Maximum Market Fee** {#aw-l-03:-no-maximum-market-fee}

**Severity:** *Low*									**Status:** *Unmitigated*

**Code:**

* [rarible\_marketplace/src/instructions/market/modify.rs\#L30-L40](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_marketplace/src/instructions/market/modify.rs#L30-L40)

**Description:**

The handler function in **modify.rs** allows the initializer of a market to modify the market fee without upper limits.

This could lead to a centralization situation where the market initializer sets excessively which fees, on the expense of the market users.

**Recommendations:**

* Enforce a maximum fee limit within the handler function, to ensure fees cannot be set above a reasonable threshold to be determined to rarible’s discretion  
  \#\[inline(always)\]  
  pub fn handler(ctx: Context\<ModifyMarket\>, params: ModifyMarketParams) \-\> Result\<()\> {  
     // Define a maximum fee limit (e.g., 10000 basis points \= 100%)  
  \+  const MAX\_FEE\_BPS: u64 \= 10000;  
    
     // Check if the provided fee is within the allowed limit  
  \+  if params.fee\_bps \> MAX\_FEE\_BPS {  
  \+      return Err(ProgramError::InvalidArgument.into());  
  \+  }  
    
     msg\!("Modify existing market");  
     Market::modify\_fee(  
         &mut ctx.accounts.market,  
         params.fee\_recipient,  
         params.fee\_bps,  
     );  
     Ok(())  
  }

# **AW-L-04: Unchecked Copied Arrays** {#aw-l-04:-unchecked-copied-arrays}

**Severity:** *Low*									**Status:** *Unmitigated*

**Code:**

* [rarible\_editions\_controls/src/instructions/update\_platform\_fee.rs\#L63-L65](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions_controls/src/instructions/update_platform_fee.rs#L63-L65)

* [rarible\_editions\_controls/src/instructions/initialise.rs\#L160-L162](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions_controls/src/instructions/initialise.rs#L160-L162) 

**Description:**

In two instances within **rarible\_editions\_controls** there are attempts to copy elements from one array to another without verifying if the source array's size exceeds the destination array's size.

This can lead to an out-of-bounds access, causing a panic and potentially crashing the program, creating a DoS, or even lead to data corruption.

**Recommendations:**

* In every instance in the code where an array is copied to another, add a check to ensure the size of the source array does not exceed the size of the destination array. For example:  
  \+ if input.platform\_fee.recipients.len() \> recipients\_array.len() {  
  \+    return Err(ProgramError::InvalidArgument.into());  
  \+ }  
  for (i, recipient) in input.platform\_fee.recipients.iter().enumerate() {  
     recipients\_array\[i\] \= recipient.clone();  
  }

# **AW-L-05: Overusing UncheckedAccount** {#aw-l-05:-overusing-uncheckedaccount}

**Severity:** *Low*									**Status:** *Unmitigated*

**Code:**

* [rarible\_editions\_controls/src/instructions/update\_token\_name.rs\#L29-L31](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions_controls/src/instructions/update_token_name.rs#L29-L31)

* Codebase-wide

**Description:**

The **UncheckedAccount** type refers to an account that is passed into an Anchor program but does not undergo the typical security checks that Anchor normally enforces (wrapper of AccountInfo with the security emphasis that it’s not being checked).

This means the account's properties (such as whether it is writable, signer, or owned by a specific program) are not automatically verified by Anchor's deserialization logic. Anchor enforces use of `/// CHECK:` comments to encourage awareness while using this type.

During the audit we counted around 64 references of **UncheckedAccount** (and around 73 of **AccountInfo**) with several of them lacking detailed enough CHECK explanations, and [one](http://rarible_editions_controls/src/instructions/update_token_name.rs#L29-L31) usage without any description.

Despite the decrease in general safety, in a lot of scenarios, such as certain CPI’s, optimization cases, and flexibility needs, **UncheckedAccount** is useful and may be used. However, overly relying on this type is considered a bad practice and should be limited to the minimal amount possible.

**Recommendations:**

* Where possible, reduce the number of instances where **UncheckedAccount** is used.  
* Ensure `/// CHECK:` comments are thorough, and clearly explain where the accommodating checks are being made.

# **AW-L-06: Lack of Documentation** {#aw-l-06:-lack-of-documentation}

**Severity:** *Low*									**Status:** *Unmitigated*

**Code:**

* Codebase-wide

**Description:**

The audited protocol lacks sufficient documentation, making it difficult for developers, auditors, and users to understand its intended functionality, edge cases, and security considerations.

Without clear documentation, security reviews become less effective, increasing the likelihood of overlooked attack vectors and unintended behaviors.

Furthermore, the absence of detailed specifications can lead to more misconfiguration, incorrect assumptions or inconsistencies in implementations, making it easier for attackers to exploit ambiguities in the protocol.Recommendation:

**Recommendations:**

* Create and constantly update a clear source of documentation explaining the protocol’s architecture, intended functionality, and expected behavior in different scenarios.


# **AW-I-01: Comment Mismatches Implementation** {#aw-i-01:-comment-mismatches-implementation}

**Severity:** *Informational*							**Status:** *Unmitigated*

**Code:**

* [rarible\_editions/src/instructions/royalties/add.rs\#L82-L86](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions/src/instructions/royalties/add.rs#L82-L86) 

**Description:**

The comment in the code states that the **fee\_basis\_point** should be less than 10000 (100%), but the implementation checks if it is less than **or equal** to 10000\.

This discrepancy could lead to confusion or potential bugs if the comment is used as a reference for understanding the code.

**Recommendations:**

* Update the comment to accurately reflect the implementation or adjust the implementation to match the comment.

# **AW-I-02: mint\_with\_controls Optimizations** {#aw-i-02:-mint_with_controls-optimizations}

**Severity:** *Informational*							**Status:** *Unmitigated*

**Description:**

As per Rarible request, our team had tried to suggest several optimization options for **mint\_with\_controls** to improve stack/heap usage:

1. **Repeated Indexing:**  
   **Code**: [rarible\_editions\_controls/src/instructions/mint\_with\_controls.rs\#L135-L135](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions_controls/src/instructions/mint_with_controls.rs#L135-L135)   
   **Issue**: The code repeatedly accesses `editions_controls.phases[mint_input.phase_index as usize]` which is creating a mutable burrow every time, affecting stack usage.  
   **Optimization**: Store the phase in a variable to avoid repeated lookups.  
   **Example**:  
   \- check\_phase\_constraints(  
   \-     &editions\_controls.phases\[mint\_input.phase\_index as usize\],  
   \-     minter\_stats,  
   \-     minter\_stats\_phase,  
   \-     editions\_controls,  
   \- )?;  
   \- let mut price\_amount \= editions\_controls.phases\[mint\_input.phase\_index as usize\].price\_amount;  
     
   \+ let phase \= &editions\_controls.phases\[mint\_input.phase\_index as usize\];  
   \+ check\_phase\_constraints(phase, minter\_stats, minter\_stats\_phase, editions\_controls)?;  
   \+ let mut price\_amount \= phase.price\_amount;

2. **Combine Conditions:**  
   **Code**: [rarible\_editions\_controls/src/instructions/mint\_with\_controls.rs\#L145-L161](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions_controls/src/instructions/mint_with_controls.rs#L145-L161)   
   **Issue**: The code uses nested conditions to check for the presence of **merkle\_proof** and whether the phase is private. Every time a new condition is evaluated, the program may need to temporarily store intermediate variables on the stack. Each nested level increases the depth accordingly.  
   **Optimization**: Use `if let` to combine conditions and reduce nesting.  
   **Example**:  
   \- if mint\_input.merkle\_proof.is\_some() {  
   \-     check\_allow\_list\_constraints(  
   \-         &editions\_controls.phases\[mint\_input.phase\_index as usize\],  
   \-         &minter.key(),  
   \-         minter\_stats\_phase,  
   \-         mint\_input.merkle\_proof,  
   \-         mint\_input.allow\_list\_price,  
   \-         mint\_input.allow\_list\_max\_claims,  
   \-     )?;  
   \-     price\_amount \= mint\_input.allow\_list\_price.unwrap\_or(0);  
   \- } else {  
   \-     if editions\_controls.phases\[mint\_input.phase\_index as usize\].is\_private {  
   \-         return Err(EditionsControlsError::PrivatePhaseNoProof.into());  
   \-     }  
   \- }  
     
   \+ if let Some(merkle\_proof) \= mint\_input.merkle\_proof {  
   \+     check\_allow\_list\_constraints(  
   \+         phase,  
   \+         &minter.key(),  
   \+         minter\_stats\_phase,  
   \+         Some(merkle\_proof),  
   \+         mint\_input.allow\_list\_price,  
   \+         mint\_input.allow\_list\_max\_claims,  
   \+     )?;  
   \+     price\_amount \= mint\_input.allow\_list\_price.unwrap\_or(0);  
   \+ } else if phase.is\_private {  
   \+     return Err(EditionsControlsError::PrivatePhaseNoProof.into());  
   \+ }  
3. **Destructure Context:**  
   **Code**: [rarible\_editions\_controls/src/instructions/mint\_with\_controls.rs\#L125-L128](https://github.com/rarible/protocol-contracts-svm-private/blob/60eeaad48d436a064c96ccb8168dbea3eb57c4b8/programs/rarible_editions_controls/src/instructions/mint_with_controls.rs#L125-L128)   
   **Issue**: The context is accessed multiple times for different accounts. Each line burrows from `ctx.accounts` separately, creating multiple intermediate stack frames.  
   **Optimization**: Destructure the context to make the code cleaner.  
   **Example**:  
   \- let editions\_controls \= &mut ctx.accounts.editions\_controls;  
   \- let minter\_stats \= &mut ctx.accounts.minter\_stats;  
   \- let minter\_stats\_phase \= &mut ctx.accounts.minter\_stats\_phase;  
   \- let minter \= &ctx.accounts.payer;  
     
   \+ let Context { accounts: Accounts { editions\_controls, minter\_stats, minter\_stats\_phase, payer: minter, .. }, .. } \= ctx;

**Recommendations:**

* Overview suggested optimization suggestions, and if applicable consider using them to optimize the performance of **mint\_with\_controls**.

# **Appendix: How To Read Findings**

**Severity**  
The findings in this report are prioritized through a severity score, determined by the AuditWare auditing team based on various factors, including team discussions, personal expertise, and a contextual understanding of the protocol and its team. The prioritization does not follow a standardized generic format but is instead tailored to the auditors' assessment of the specific project

* Critical \- The issue poses an immediate and severe risk to the protocol, potentially leading to loss of funds, contract failure, or unauthorized access. Exploitation could have serious consequences and requires urgent mitigation.  
* High \- A significant vulnerability that, if exploited, could result in substantial financial loss or operational disruption. While not as severe as a critical issue, it still demands prompt attention.  
* Medium \- A moderate security concern that may require specific conditions to be exploited. While it does not pose an immediate threat, it could still impact functionality, efficiency, or create indirect financial risks if left unaddressed.  
* Low \- The issue has minimal security impact but could affect contract efficiency, best practices, or usability. It is recommended to address such issues to enhance code quality and maintainability.  
* Informational \- The issue does not pose any security or functional risk but serves as a recommendation for best practices, optimization, or improving the clarity of the codebase.

**Status**  
The mitigation status of the security issue

* Unmitigated \- The issue is not mitigated, and risk is significant; mitigation is necessary.  
* Partially mitigated \- Some measures have been implemented to reduce the risk, but additional security controls are required for full mitigation.  
* Acknowledged \- The issue remains unmitigated; however, the team has recognized the risk and either mitigated it through alternative mechanisms or deemed it an acceptable risk based on cost-benefit considerations.  
* Mitigated \- The issue was found to be sufficiently mitigated.

This report was produced by Auditware for Rarible based solely on the   
information provided by Rarible. Auditware provides no guarantees as to the accuracy  
 of the contents of this report and does not make any guarantees that following the advice   
within will prevent security incidents or issues.

Auditware is available for questions or comments about any of the contents of this report. We can be reached at [https://auditware.io/](https://auditware.io/) or by email at [joe@auditware.io](mailto:joe@auditware.io).
