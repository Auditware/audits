# **Review Summary**

	This report outlines the operational security of \<REDACTED\>. The review  
content has been organized by multiple security domains, each with an evaluation  
of the security posture of the organization, security risks that have been investigated  
for each domain, and recommended action items.

As part of our review, Auditware has prepared several security guides that describe our recommendations in detail for securing operations, assets, and services. They can be accessed from the shared Notion repo.

All content of this report is based on comments provided by \<REDACTED\>. This review is only as precise and accurate as that information. This report should be reviewed in tandem with the accompanying security guides to ensure you are maximizing your security posture and following the guidance on an ongoing basis to maintain a high bar of security.

## Endpoint Security

Company assets are only as secure as the endpoint machines used to access them. An “endpoint” refers to any device connected to a user (employee), that is used to access company resources (internal email, shared files, wikis, etc.).

Endpoint security is largely in the hands of users, which supports the need for good standardized security training, in the form of text or video-based periodic training, and/or setup requirements and guidelines. Beyond training, it is important to secure and manage employee devices to enforce secure configurations and provide baseline security controls.

\<REDACTED\> organization member endpoints are member-provided, personal devices that are not managed by \<REDACTED\>. Insecure endpoints present significant risk to the organization.

### Risks:

* \<REDACTED\> has X organization members, half of which are in privileged or dev roles. **Malware infection** of developers especially is a particularly high risk.  
* \<REDACTED\> members currently **provide their own devices** with no segregation from personal usage enforced. This significantly increases the likelihood of malware infection or other social engineering compromises impacting \<REDACTED\> assets.  
* **Public usage** of devices presents a significant risk to password and information leakage. Organization members should be made aware of these risks and their mitigations: passwordless login (biometrics, pass keys, etc.), privacy screens, VPN usage on public wifi, turning screen facing a wall, and full-disk encryption with a quick account lock in the case of theft.  
* **External files** are a major risk vector for delivery of malware. It is highly recommended to prohibit opening external files except when sanitized properly  
  * Google Docs can be used to securely view files. For any that are unsupported, a tool like [Dangerzone](https://dangerzone.rocks/) should be used. You can also use virtual machine instances to open untrusted files.

### Action Items:

* Provide **dedicated devices** for members (new laptops, used only for work purposes)  
  * Any brand-new devices will work for this purpose, but we recommend that you select devices with good security properties (fingerprint reader, secure boot, remote wipe capabilities). Macbooks are our standard recommendation as they fit these properties nicely.  
  * Ordering devices online and/or mailing them presents minor transit risk and enables specific targeting for supply chain attacks. We recommend each member buy their devices off the shelf at a reputable, local store.  
  * When setting up devices, active network monitoring (Little Snitch) should be the first thing installed to prevent other installed applications from potentially downloading malware.  
  * Only the bare minimum set of applications for work tasks should be installed. We recommend not installing any meeting software such as Zoom, and to use [development containers](https://www.notion.so/Web3-Developer-Security-Guide-20f77e2280418131b96bffecc2689a2b?source=copy_link#20f77e2280418184860fc65cf28743d6) to prevent common malware delivery methods.  
  * See more detailed guidance in the Endpoints section of our OpSec guide.  
* Require a **password manager** be used by all members  
  * 1Password was mentioned as being partially adopted. This should be made a requirement for all members to store all work related credentials.  
  * 2FA OTP seeds should not be stored in password managers. If account sharing is required, we recommend setting up 2FA apps together in person to preserve the “something you have” property of this second login factor. Sharing OTP seeds/QR codes over encrypted Signal chat that is deleted immediately after is also acceptable.  
* Mandate [**Little Snitch**](https://www.obdev.at/products/littlesnitch/index.html) (or equivalent) be installed on all member laptops  
* Consider using an **EDR solution** such as [Crowdstrike](https://www.crowdstrike.com/en-us/cybersecurity-101/endpoint-security/endpoint-detection-and-response-edr/) to secure employee laptops

## Insider Threats

	Insider threats are threats that present themselves through the vector of an employee (insider), allowing a malicious party to abuse the trust and permissions afforded to that insider. This includes the standard “rogue employee” risk model, which is often overlooked, but also includes the risk of an attacker compromising an employee.

Employees can be compromised in many different ways: Endpoint compromise through malware or poor physical security, physical duress (threats to safety), financial incentives (large black market bounties), social engineering, and through accidental information leakage. It is highly recommended to reduce unnecessary employee permissions and put in place detective measures to detect potential compromises to mitigate the impact of potential insider threats.

Insider threat risks can generally be mitigated with proper security controls:

* **Multi-party approvals** and time-locks on deployments and admin actions  
  * This can be accomplished by using automated system accounts to perform these actions, and only granting minimum necessary access on an as-needed basis with approvals from one or more other team members.  
* **Robust monitoring** and alerting  
  * Alerts are only as effective as those watching them and administrating them.  
  * It is recommended to duplicate alert channels and to make them immutable (e.g. any admin change would trigger out of band alerts, with no way to suppress such an alert to ALL channels at once).  
  * We also recommend that a member be dedicated to monitoring and reviewing alerts, such as an on-call engineer, with explicit confirmation of review required.

### Risks:

* Some \<REDACTED\> insiders have significant privileges within the organization. It is recommended to evaluate their permissions and reduce them as much as possible.  
  * **Admin access** to cloud console, GitHub repo, and deployment processes are the primary sensitive assets highlighted, see the Infrastructure & Development section for further details on those.  
* There is no plan for rapid **access control revocation**. In the event of a compromised account or device, all account credentials should be revocable with haste.

Action Items:

* Restrict **admin access** to cloud assets and code/deployment management to a single admin breakglass account that is separate from the user accounts of admins.  
  * Just-In-Time(JIT) access control should be used in combination with a peer review ticketing system to provision access to the minimum set of temporary permissions for user accounts when changes need to be made. All users should otherwise only have read access privileges.  
  * In the case of emergency, the breakglass account could be used directly, but accessing these credentials/account should trigger alarms to the entire team.  
* Enforce **2FA** on all accounts for all members (SMS-based 2FA should not be used)  
* Create an **incident response plan** to allow you to quickly respond to any account or device compromises.  
  * This plan should include quick revocation of access to all accounts, instructions to isolate and image the affected device for analysis, and reaching out to us to aid in investigation and recovery efforts.

## Communications & Social Engineering

	Communications security is the number one point of failure for companies. Phishing attacks, employee impersonation, insider threats, and insecure communications transport methods all add significant security risk.

Social engineering is the simplest path to compromise for an attacker, and attack vectors such as phishing are difficult to mitigate at scale. Research shows that those in technical roles are no less likely than those in non-technical roles to fall for phishing attacks. Everyone is vulnerable.

### Risks:

* Malware delivery via **email phishing**  
  * This is largely an individual responsibility problem, but providing members clear guidelines for handling external communications and files helps set a standard for mitigating phishing attempts.  
  * This should be included in ongoing security training, but the primary controls for mitigating this risk are:  
    * Prohibiting opening externally received files or downloading any file at the direction of an external party (or training them on usage of file sanitizers)  
    * Requiring a dedicated browser for risky browsing (no wallet connections, no cookies stored, not used for any authenticated sessions), which can be used to open urls from external parties.  
* **Impersonation** of organization members  
  * This is largely mitigated by service configurations and security training on authenticating organization members on platforms like Telegram and Discord.

### Action Items:

* Set up **DMARC** with your email provider to prevent spoofing of organization members via email.  
* Provide basic security and **anti-phishing training** to organization members. [Unphishable](https://unphishable.io/) is a decent option for web3-tailored security training.

## Infrastructure & Development

	Cloud assets and development pipelines are significant risk areas. Deployment permissions can be abused or compromised, trusted code could be infected with malware, and deployment secrets could be leaked. Developers, by necessity, have a high level of privilege to modify code and often infrastructure assets. Modifying back-end and front-end code can allow a malicious or compromised dev to attack users and organization assets directly.

	\<REDACTED\> uses Render for infrastructure management, which abstracts away fine-grained configurations and management of cloud assets. This simplifies security, but also presents itself as a black box, with no way to be certain that Render has properly configured cloud assets that back Render instances. That said, we could assume Render has made a best effort to provide adequate security, but accept that it presents residual risk as an unknown configuration.

### Risks:

* **Malware infection** of git repo could go undetected.  
  * Signed commits are not enforced, allowing an attacker to impersonate other users and overwrite old commits with malicious content without being noticed.  
  * Force pushing PRs could allow a compromised privileged user to deploy malware to production servers.  
* Automated deployments occur through GitHub pushes to main and staging branches, with the main branch requiring only **one approval**. This prevents a single compromised or malicious dev from deploying an unauthorized commit, but does not protect against compromise of multiple devs (a likely occurrence if the repo becomes infected by malware).

### Action Items:

* Enforce **signed commits** on git repos to prevent spoofing.  
* Disable **force pushing** of branches in GitHub.  
* Set up [monitoring](https://docs.github.com/en/code-security/secret-scanning/introduction/about-secret-scanning) for **accidentally checked in secrets**  
* Enable **Dependabot** security alerts in GitHub  
* It is recommended to add up to a **one-week delay** to automated deployments, which would allow you time to respond to a potential compromise and revert any unauthorized deployment attempts.  
* Adjust deployment and infrastructure management to be **entirely automated** with no permissions granted to or intervention required by humans.  
  * For any necessary high-privilege permissions. Ensure immutable alerts are in place to monitor for anomalies.  
  * A breakglass manual deployment mechanism should be made available in case needed. This should trigger an alert to multiple org members on usage and should only be used in emergency situations where the configured delay could not be tolerated.  
  * If any devs other than the single admin/breakglass account have privileges to either perform a manual deployment or to alter the configuration of GitHub or cloud infrastructure, those permissions should be removed. All regular deployments should occur via automated channels.

## Wallet Management & On-Chain Operations

	Wallet security, individually and at the organization level, is vital to ensuring safety of custodied assets and managed on-chain contracts. Securing these involves individual responsibility to follow best practices, strong multi-party approval systems (multi-sigs), and robust monitoring and alerting. Technical controls must be supplemented by diligent signing processes and attentive monitoring of transactions with rapid response before they execute, if needed.

	The \<REDACTED\> treasury is managed by a N/M multi-sig (the founders plus an external advisor), and there are plans to operate a multi-sig to manage the ownership and upgrades of future deployed smart contracts. The treasury multi-sig is reasonably configured, but the absence of a time-lock period on confirmed transactions precludes \<REDACTED\> from recovering from potential compromise of multiple signers. Further, a time-lock period is only valuable when paired with effective transaction monitoring, alerting, and review scrutiny \- all of which need improvement.

### Risks:

* Malicious proposals could be blind signed  
  * Increasing transaction review scrutiny with a formal process will prevent degradation of individual responsibility to review transaction details  
* Undetected malicious or erroneous transaction proposals  
  * Transaction activity is lightly monitored, but lack of a robust oversight mechanism could allow some undesired transactions to slip through the cracks.

### Action Items:

* Require **hardware wallets**  
  * A hardware wallet is the absolute best defense possible against tampered impersonated/tampered transactions and malware. This should be a hard requirement for all organization members that participate in on-chain operations (e.g. multi-sig signers)  
  * Hardware wallets are not created equal. A wallet with a large screen to read transaction data is best, and allows signers to confirm that signed transactions are authentic and have not been tampered with by malware.  
* Set up out-of-band **confirmations**  
  * Transaction data signed by the multi-sig participants should be confirmed on a separate device, using a different source of truth (e.g. on a ticket or internal chat compared to transaction details on the signing wallet).  
    * Signal is currently used to coordinate signing operations. In addition to verifying details in Signal, each signer should also reference the transaction details via another device, on another platform (e.g. on Trello, Notion, or Slack).  
    * These details should then be compared to the transaction details displayed on the hardware wallets of each signer.  
    * Tenderly simulation is performed for these transactions, and should be further verified against the expected outcome. It is recommended to duplicate this effort so at least 2 signers are independently verifying transaction data.  
  * Likewise for operations like contract deployments, a hash should be generated of the bytecode before deployment is executed, and this hash should be compared to the deployed bytecode to ensure its authenticity. It is also advised to compare contract initialization parameters in the same way.  
* Implement **challenge periods** with timelocks  
  * Transactions (multi-sig or otherwise), unless for emergency actions via an override mechanism, should have a mandatory time-lock delay before their execution. This allows time for transactions to be thoroughly reviewed for unauthorized actions before they execute.  
  * Use a [Delay Modifier](https://www.zodiac.wiki/documentation/delay-modifier) or similar mechanism to add a time delay to Treasury transactions  
  * It was mentioned that this may be intolerable for the multi-sig that manages smart contracts. In that case, we suggest to simply increase the number of required signers and to add a timelock contract (callable by any signer in the quorum) as a signer. This would allow you to continue with the current 2-signer threshold in addition to the timelock contract, or to override the timelock with a 3rd and/or 4th signer in case of emergency.  
* Provide additional **“air-gapped” laptops** to multi-sig signers that are used ONLY for signing operations and transaction data verification. These should be additional to the laptops dedicated for general work tasks.  
* Increase transaction **review scrutiny**  
  * On-chain activity alerts are vital to security when combined with a time lock challenge period. With diligent review, they allow a compromised multi-sig to recover before an attacker transaction could be executed. It is recommended to configure an on-chain monitoring service for multi-sig transactions. Using redundant monitoring services is also recommended (also with separate admins)  
  * It is recommended to duplicate these alerts in at least two channels, and to split up admin permissions for them to mitigate the risk that any one admin becomes compromised and is able to both execute an unauthorized transaction, and to suppress monitoring so it goes undetected and unchallenged throughout the timelock period.  
* Put in place **active monitoring** of multi-sig wallet transactions.  
  * Alerts from the monitors should be published to a channel visible to the entire team, to increase the likelihood of catching any anomalous actions.  
  * We recommend electing one or two organization members (who are not signers) to scrutinize all pending transactions as they are approved and waiting for the timelock to lapse.  
  * Hyperactive is used for monitoring. This is adequate, but we recommend ensuring that admins are not able to disable or edit these monitors without triggering alerts that are impossible to suppress. This way, a compromised admin account could not disable alerts on future nefarious actions without being detected.

This report was produced by Auditware for \<REDACTED\>, based solely on the information provided by \<REDACTED\>. Auditware provides no guarantees as to the accuracy of the contents of this report and does not make any guarantees that following the advice within will prevent security incidents or issues.

Auditware is available for questions or comments about any of the contents of this report. We can be reached at [https://auditware.io/](https://auditware.io/) or by email at [joe@auditware.io](mailto:joe@auditware.io).