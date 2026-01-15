# **Review Summary**

The **AuraSoulsV1** smart contract is designed to facilitate the buying and selling of  
"souls" associated with a specific subject using a quadratic bonding curve  
mechanism.

The contract incorporates a fee structure that includes protocol fees, subject fees, and LP bucket fees, which are distributed to designated addresses.

Users can purchase and sell souls, with the contract handling supply management, price calculations, and fee distributions.

The focus of this audit was to evaluate potential security risks associated with the contract's functionality, including fee configuration, bonding curve calculations, input validation, access control, the accurate updating of internal mappings and balances, and more.

The audit revealed several issues that need addressing, for example, the contract does not properly enforce access control on the initial soul acquisition, allowing unauthorized users to purchase the first soul (see [**AW-H-01: Insufficient Enforcement on Initial Soul Acquisition**](#aw-m-01:-lack-of-fee-destination-update-functionality) **).**

As an added value, our team also took the initiative to add suggested unit tests (see [**Appendix A**](#appendix-a---added-unit-tests))  to core functionalities in the contract, we recommend to constantly maintain and improve the coverage of tests of this nature, especially in future code changes.

# **Findings Overview**

| Finding | Risk Level | Status |
| :---: | :---: | :---: |
| **[AW-H-01: Insufficient Enforcement on Initial Soul Acquisition](#aw-m-01:-lack-of-fee-destination-update-functionality)**  | ***High*** | ***Unmitigated*** |
| **[AW-M-01: Lack of Fee Destination Update Functionality](#aw-m-01:-lack-of-fee-destination-update-functionality)** | ***Medium*** | ***Unmitigated*** |
| **[AW-M-02: Insufficient Enforcement on Final Soul Sale](#aw-m-02:-insufficient-enforcement-on-final-soul-sale)** | ***Medium*** | ***Unmitigated*** |
| **[AW-L-01: Missing Require Statements](#aw-l-01:-missing-require-statements)** | ***Low*** | ***Unmitigated*** |

# **AW-H-01: Insufficient Enforcement on Initial Soul Acquisition**

**Severity:** *High*								**Status:** *Unmitigated*

**Locations:**

* [https://github.com/Auditware/AuraSoulsV1Foundry/blob/main/src/AuraSoulsV1.sol\#L112](https://github.com/Auditware/AuraSoulsV1Foundry/blob/main/src/AuraSoulsV1.sol#L112)

**Description:**

The **buySouls** function lacks sufficient access control to ensure that only the intended soul subject can purchase the first soul. This creates an opportunity for malicious actors to front-run the transaction and claim the first soul for themselves, bypassing the intended exclusivity.

   function buySouls(address soulsSubject, uint256 amount)  
       public  
       payable  
       nonReentrant  
   {  
       uint256 price \= getPrice(soulsSupply\[soulsSubject\], amount);  
       uint256 totalPrice \= applyFees(price, true);  
       require(msg.value \== totalPrice, "Incorrect payment amount");

       // Update state  
       soulsBalance\[soulsSubject\]\[msg.sender\] \+= amount;  
       soulsSupply\[soulsSubject\] \+= amount;  
       creatorEarnings\[soulsSubject\] \+= (price \* subjectFeePercent) / 1 ether;

In order to abuse this issue, a potential threat actor might follow these steps:

* The attacker monitors the blockchain for transactions or events indicating the potential initialization of a soul subject.  
* The attacker sends a transaction to the **buySouls** function with the required **soulsSubject** and amount (likely 1\) to claim the free soul before the rightful owner does.  
* This exploit denies the soul subject their intended first soul and potentially **disrupts further interactions or creates financial opportunities for resale.**

The absence of a mechanism to restrict initial purchases to the soul subject allows unauthorized users to exploit this function.

**Recommendation:**

* Implement an access control check to ensure that only the **soulsSubject** can purchase the first soul. For example:  
  * **require(msg.sender \== soulsSubject, “Only the soul subject can buy the first soul”);**

# **AW-M-01: Lack of Fee Destination Update Functionality** {#aw-m-01:-lack-of-fee-destination-update-functionality}

**Severity:** *Medium*							**Status:** *Unmitigated*

**Locations:**

* [https://github.com/Auditware/AuraSoulsV1Foundry/blob/main/src/AuraSoulsV1.sol\#L41](https://github.com/Auditware/AuraSoulsV1Foundry/blob/main/src/AuraSoulsV1.sol#L41)  
* [https://github.com/Auditware/AuraSoulsV1Foundry/blob/main/src/AuraSoulsV1.sol\#L42](https://github.com/Auditware/AuraSoulsV1Foundry/blob/main/src/AuraSoulsV1.sol#L42) 

**Description:**

During the assessment we discovered that the contract lacks functionality to allow the owner to update the protocolFeeDestination and lpBucketFeeDestination addresses after initialization.  
   constructor(  
       address \_owner,  
       address \_protocolFeeDestination,  
       address \_lpBucketFeeDestination  
   ) Ownable(\_owner) ReentrancyGuard() {  
       protocolFeeDestination \= \_protocolFeeDestination;  
       lpBucketFeeDestination \= \_lpBucketFeeDestination;  
   }

If either of these addresses becomes compromised or inaccessible, the contract's fee mechanism may be exploited or rendered ineffective, leading to financial loss or an operational bottleneck.

**Recommendation:**

* Add onlyOwner functions to update the **protocolFeeDestination** and **lpBucketFeeDestination** addresses. [For further information](https://docs.openzeppelin.com/contracts/2.x/access-control).

# **AW-M-02: Insufficient Enforcement on Final Soul Sale** {#aw-m-02:-insufficient-enforcement-on-final-soul-sale}

**Severity:** *Medium*							**Status:** *Unmitigated*

**Locations:**

* [https://github.com/Auditware/AuraSoulsV1Foundry/blob/main/src/AuraSoulsV1.sol\#L147](https://github.com/Auditware/AuraSoulsV1Foundry/blob/main/src/AuraSoulsV1.sol#L147)


**Description:**

During the assessment we discovered that a subject's **soulSupply** can be reduced back to 0, after their souls are put on the market.

When a soul subject (creator) buys their own, first, free soul they put their souls on the open market. The decision for a subject (creator) to sell their souls is irreversible, and sale of their souls should never be refused in the future.

function sellSouls(address soulsSubject, uint256 amount)  
       public  
       nonReentrant  
   {  
       uint256 price \= getPrice(soulsSupply\[soulsSubject\] \- amount, amount);  
       uint256 totalPayout \= applyFees(price, false);

       require(  
           soulsBalance\[soulsSubject\]\[msg.sender\] \>= amount,  
           "Insufficient souls"  
       );

Given the intended logic that only a subject (creator) can initiate trade of their own souls, a later reduction of the subject’s soul supply to zero effectively removes their souls from the open market.  
There is no check ensuring that a subject’s final, valueless soul cannot be sold when their soul supply is one.

**Recommendation:**

* Add a **require** statement to **sellSouls** verifying that **soulsSupply\[soulsSubject\] \- amount \> 0**

# **AW-L-01: Missing Require Statements** {#aw-l-01:-missing-require-statements}

**Severity:** *Low*								**Status:** *Unmitigated*

**Locations:**

* [https://github.com/Auditware/AuraSoulsV1Foundry/blob/main/src/AuraSoulsV1.sol\#L65](https://github.com/Auditware/AuraSoulsV1Foundry/blob/main/src/AuraSoulsV1.sol#L65) (Missing check against unreasonably high supply)  
* [https://github.com/Auditware/AuraSoulsV1Foundry/blob/main/src/AuraSoulsV1.sol\#L147](https://github.com/Auditware/AuraSoulsV1Foundry/blob/main/src/AuraSoulsV1.sol#L147) (Require should be moved to before the **getPrice** and **applyFees** operations)  
* [https://github.com/Auditware/AuraSoulsV1Foundry/blob/main/src/AuraSoulsV1.sol\#L112](https://github.com/Auditware/AuraSoulsV1Foundry/blob/main/src/AuraSoulsV1.sol#L112) (Missing check that amount is not 0\)

**Description:**

The contract lacks **require** statements in several locations where inputs or conditions should ideally be validated explicitly.  
For example, **getPrice** does not validate against overly large supply:  
   function getPrice(uint256 supply, uint256 amount)  
       public  
       pure  
       returns (uint256)  
   {  
       uint256 start \= supply \== 0 ? 0 : supply \- 1;  
       uint256 end \= supply \+ amount \- 1;  
       uint256 sum1 \= cumulativeSum(start);  
       uint256 sum2 \= cumulativeSum(end);  
       uint256 summation \= sum2 \- sum1;  
       return (summation \* 1 ether) / 16000;  
   }

While the contract might still revert and not work without the **require** additions due to internal Solidity checks or subsequent logic failures, adding **require** statements improves code clarity, prevents unintended behavior, and simplifies debugging and auditing.  
**Recommendation:**

* Introduce **require** statements for explicit input validation and state enforcement in all functions that are deemed important.

# **Appendix A \- Added Unit Tests** {#appendix-a---added-unit-tests}

This report provides an analysis of the tests added for the AuraSoulsV1 protocol.

The tests aim to verify the correctness, robustness, and security of the contract by covering various scenarios, including edge cases and error handling.

The results of running these tests have been analyzed to identify any issues or areas for improvement.

**testInvalidProtocolFee**

* **Purpose**: Ensure that setting the protocol fee percent to a value greater than 100% (1 ether) reverts with the correct error message.  
* **Result**: **Pass**  
* **Comments**: The test correctly verifies that the contract enforces the maximum allowed fee percentage for the protocol fee.

**testInvalidSubjectFee**

* **Purpose**: Similar to the previous test but for the subject fee percent.  
* **Result**: **Pass**  
* **Comments**: Confirms that setting an invalid subject fee percent is properly rejected.

**testInvalidLpBucketFee**

* **Purpose**: Ensure that setting the LP bucket fee percent above 100% reverts with the correct error message.  
* **Result**: **Pass**  
* **Comments**: Validates that all fee percentages are constrained appropriately.

**testExtremelyLargeInputForQuadraticBondingCurve**

* **Purpose**: Test the getPrice function with extremely large input values to ensure it handles them without errors.  
* **Result**: **Pass**  
* **Comments**: The function correctly calculates the price for large inputs, indicating that the bonding curve handles large numbers as expected.

**testOnlySoulsSubjectCanBuyFirstSoul**

* **Purpose**: Verify that only the subject can buy the first soul.  
* **Result**: **Fail**  
* **Reason**: The call did not revert as expected.  
* **Comments**: The test expected a revert when a non-subject attempts to buy the first soul, but it did not occur. This is due to a flaw in access control within the buySouls function. ( See [**AW-H-01: Insufficient Enforcement on Initial Soul Acquisition**](#aw-m-01:-lack-of-fee-destination-update-functionality) )

**testZeroAmountForBuy**

* **Purpose**: Ensure that attempting to buy zero souls reverts with the correct error message.  
* **Result**: **Fail**  
* **Reason**: Error message did not match the expected error ("Incorrect payment amount").  
* **Comments**: The test expects a specific revert message, but since there isn’t a require statement the test fails. (See [**AW-L-01: Missing Require Statements**](#aw-l-01:-missing-require-statements) )

**testSupplyMappingUpdatedCorrectly**

* **Purpose**: Verify that the total supply and buyer's soul balance are updated correctly after a purchase.  
* **Result**: **Pass**  
* **Comments**: The contract correctly updates the soulsSupply and soulsBalance mappings.

**testCreatorEarningsMappingUpdatedCorrectly**

* **Purpose**: Check that the creator's earnings are updated appropriately after a soul purchase.  
* **Result**: **Pass**  
* **Comments**: The creatorEarnings mapping is accurately updated, reflecting the correct earnings.

**testFeesDistributedCorrectly**

* **Purpose**: Ensure that protocol, LP bucket, and subject fees are correctly distributed upon soul purchase.  
* **Result**: **Pass**  
* **Comments**: Fee distribution to respective destinations works as intended.

**testCannotSellMoreSoulsThanOwned**

* **Purpose**: Verify that users cannot sell more souls than they own.  
* **Result**: **Fail**  
* **Comments**: The contract correctly reverts when attempting to sell more souls than the user's balance due to the require statement implemented after getPrice and applyFees operations instead of before of them (See [**AW-L-01: Missing Require Statements**](#aw-l-01:-missing-require-statements) ).

**testCannotSellMoreThanTotalSupply**

* **Purpose**: Ensure that users cannot sell more souls than the total supply.  
* **Result**: **Fail**  
* **Reason**: Error message did not match the expected error ("Unable to send payout" vs. expected).  
* **Comments**: The test failed due to a lack of revert when trying to sell the last soul ( See [**AW-M-02: Insufficient Enforcement on Final Soul Sale**](#aw-m-02:-insufficient-enforcement-on-final-soul-sale) ).

**testInvalidFeesAreRejected**

* **Purpose**: Verify that setting any fee percent above 100% is rejected.  
* **Result**: **Pass**  
* **Comments**: The contract enforces maximum fee percentages across all fee types.

**testGetPrice**

* **Purpose**: Test the getPrice function with various inputs, including extremely large supply and amount values.  
* **Result**: **Fail**  
* **Reason**: Encountered arithmetic overflow (panic: arithmetic underflow or overflow).  
* **Comments**: The function fails with large input values due to integer overflow, indicating a lack of input validation or safe arithmetic operations. (See [**AW-L-01: Missing Require Statements**](#aw-l-01:-missing-require-statements) ).

**testFuzzBondingCurve**

* **Purpose**: Fuzz tests the \`getPrice\` function to ensure it behaves as expected across a wide range of inputs.  
* **Result**: **Pass**  
* **Comments**: Checks that the same math acts in the same way with different supply and amount inputs.

# **Appendix B \- Static Analysis Results**

# 

This report was produced by Auditware for Aura based solely on the   
information provided by Aura. Auditware provides no guarantees as to the accuracy  
 of the contents of this report and does not make any guarantees that following the advice   
within will prevent security incidents or issues.

Auditware is available for questions or comments about any of the contents of this report. We can be reached at [https://auditware.io/](https://auditware.io/) or by email at [joe@auditware.io](mailto:joe@auditware.io).
