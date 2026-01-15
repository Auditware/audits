# Threat Model

## Approach

- Identify sensitive wallet assets, find usages of them and evaluate how they might be manipulated
- Trace data inputs to find any adverse effects of user inputs
- Identify external call returns, evaluate how they are handled and if input is trusted/sanitized
  - These are probably out of scope, though

## Code Review

> The communication with the backend, authentication, and authorization for Firebase operations, and everything related to user preferences is out of scope

- src/
  - analytics/
    - analytics.ts
      - track() could leak data to the logs.
        - When searching tokens, the search term is logged
        - When updating wallet metadata, the wallet name is logged, but the wallet name is unbounded user input
        - No other track() usages included significant user-specified data or leaked anything apparently sensitive
    - events.ts
      - Shows the metadata structure of all of the analytics tracking calls
  - ethereum/
    - eip681.ts
      - Parses EIP 681 URIs for sending native tokens or transferring ERC20 tokens
      - These links are poorly trusted; regardless of their authenticity, the user is expected to verify the parameters of these transactions.
  - hooks/
    - useApproveToken.ts
      - Executes approve token transactions
    - useBalance.ts
      - Fetches the balance of a wallet with `wallet.provider.getBalance()` for native balance, and
      - `Erc20Abi__factory.*connect*(address, provider).balanceOf(account)` for erc20
    - useBalances.ts
      - Fetches the balances of a wallet using `firestore().collection('wallets').doc(account).collection('balances')` for the tokenIDs provided
    - useFirebaseAuth.ts
      - Sets the firebase user id that is used throughout the code as a reference to the user’s account when ***`auth***().onAuthStateChanged()`
      - OUT OF SCOPE - Firebase
    - useHandleLinks.ts
      - Handles links to open views for Wallet Connect, sends (EIP 681, QR scans), trades, and deep links
      - THREAT: What happens if the EIP 681 is not a send transaction?
    - useLocalAuthentication.ts
      - Uses `expo-local-authentication` to perform biometric auth, otherwise performs PIN auth. Also manages biometrics status.
      - THREAT: Could the biometrics be disabled without PIN entry?
    - useLogout.ts
      - Logs out from Firebase, kills wallet connect sessions, clears pending transactions, etc.
    - usePortfolioHistory.ts
      - Loads balance data from Firebase.
      - OUT OF SCOPE - Firebase
    - useRecipient.ts
      - Maps a transaction recipient string to a useable address
    - useRecipients.ts
      - Loads a list of known potential recipients from own wallets and persisted recipients (ones that have already been used as a recipient)
    - useReplacementTransaction.ts
      - Used to craft replacement transactions (speed up or cancel)
    - useSearchTokens.ts
      - Performs a search on tokens from the Firebase store
      - OUT OF SCOPE - Firebase
    - useTrade.ts
      - Performs trades (creates, executes, and tracks/saves transactions)
    - useUnlock.ts
      - Handles logic flow for unlocking the app (biometrics with passcode as a fallback)
    - useUpdateWallet.ts
      - Updates wallet metadata (account, name, color, etc.)
      - OUT OF SCOPE - Firebase
    - useWallets.ts
      - Interfaces with the WalletManager to load and create wallets
      - This is the primary chunk of code for wallet management
    - useWalletsBalances.ts
      - Fetches balances from Firebase store
      - OUT OF SCOPE - Firebase
  - screens/
    - onboarding/
      - SetPasscodeScreen.tsx
        - View for setting passcode
    - send/
      - machine.ts
        - Validates send request parameters and format
      - useSend.ts
        - Creates and performs send transactions (`mutationFn()` probably belongs in hooks)
        - Provides data for send transaction displaying
    - swap/
      - machine.ts
        - Validates swap request parameters and format
      - useOrder.ts
        - Provides data for swap transaction displaying
    - TransactionUnlock/
      - TransactionUnlock.tsx
        - Handles unlock process on transaction initiation
    - unlock/
      - Unlock.tsx
        - View for unlock process of app
    - walletconnect/
      - handlers.ts
        - Contains various handlers for data for this view
        - `dappNameOverride()` uses an includes() call that could be falsely triggered by inserting into into the url path or query params
      - Request.tsx
        - View for a Wallet Connect request
      - Session.tsx
        - View for a Wallet Connect session
      - Sessions.tsx
        - View for multiple Wallet Connect Sessions (across networks)
        - useSignRequest.ts
          - Handles approval for Wallet Connect transaction requests
    - webview/
      - CoinbasePayScreen.tsx
        - WebView for Coinbase Pay
          - It loads the CBP url and waits for a message response
      - WebViewScreen.tsx
        - View for loading a multi-pupose WebView (used for displaying agreements)
  - services/
    - api.ts
      - This file includes read calls to the slingshot API.
      - `fetchSlingshotProxyContractAddresses()` - could potentially be hijacked
      - Is the cert for slingshot pinned?
      - `fetchQuote()` - could potentially be impersonated? - that could trick a user into buying or selling a token when price looks different than it really is
      - OUT OF SCOPE - slingshot service has been reviewed separately
    - pin-store.ts
      - Handles logic for PIN validation, setup, and storage
    - transactions-manager.ts
      - Watches for transaction and send events for status changes
      - Also stores the latest local nonce per token and account
  - utils/
    - auth.ts
      - Firebase auth
      - OUT OF SCOPE - Firebase
    - DeepLinking.ts
      - Tags raw URLs based on pattern matching, also converts EIP 681 links to transactions
    - Gas.ts
      - Calculates gas fees and limits
    - Logger.ts
      - Contains handlers for global logging
  - wallet/
    - EVMWallet.ts
      - EVM implementation of the Wallet interface, a wrapper around EthersWallet from the ethers library
    - Wallet.ts
      - Wallet object interface
    - WalletManager.ts
      - Manages wallet loading and handling
    - WalletStore.ts
      - Performs storage and loading operations for wallets
  - walletconnect/
    - hooks/
      - useSession.ts
        - Manages the Wallet Connect session
    - WalletConnect.ts
      - Manages session approvals and termination
      - Holds listeners for Wallet Connect calls
  - App.tsx
    - App entry point
    - Sets up various listeners
    - Establishes base component routing
  - QueryClient.ts
    - Manages TanStack Query client

## Threat List

We identified the following potential threats against the security of the app.

| Threat                                                                          | Notes                                                                                                                                                                                                          | Findings                                                                                                                                                                                                                                                                                                                                        |
|:-------------------------------------------------------------------------------:| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Lingering keychain/keystore data                                                | https://github.com/OWASP/owasp-mstg/blob/master/Document/0x06d-Testing-Data-Storage.md#keychain-data-persistence                                                                                               | According to the devs, and our understanding, the keychain data is deleted when the app is reinstalled. This is not a significant risk, either way; the key is only persisted to allow future decrypts and re-encrypts, confidentiality or integrity could not be violated.                                                                     |
| Keychain/keystore permission misconfiguration                                   | keychain/keystore access has to be explicitly granted                                                                                                                                                          | Keystore is not configured to be shared. Expo is used as the library to manage this, and its inner workings are out of scope of this audit.                                                                                                                                                                                                     |
| Unencrypted keychain/keystore data                                              | Only the PIN and encryption key are stored in the keystore                                                                                                                                                     | The data is stored here unencrypted, but this is the best place to store unencrypted secrets.                                                                                                                                                                                                                                                   |
| Calls to eval() with unsanitized/unvalidated user input                         | https://eslint.org/docs/latest/rules/no-eval                                                                                                                                                                   | eval() is not used anywhere in the code.                                                                                                                                                                                                                                                                                                        |
| Dependency risk                                                                 | Socket.dev is used by the team to migate this risk                                                                                                                                                             | There is residual dependency risk, but no more than any project.                                                                                                                                                                                                                                                                                |
| CPRNG determinism                                                               | Look at crypto library in comparison to references                                                                                                                                                             | `encryptionKey = Buffer.from(crypto.randomBytes(16)).toString('hex')` is used. This is the proper way to generate a key.                                                                                                                                                                                                                        |
| Poor crypto configuration                                                       | (e.g. cbc mode instead of gcm)                                                                                                                                                                                 | Key generation is fine, encryption is not vulnerable, but is not resistent to integrity violations or chosen ciphertext attacks.                                                                                                                                                                                                                |
| Broken auth / authentication bypass                                             | We should note this one in the report that the risk is mitigated, even though it deviates from how other wallets work                                                                                          | The PIN/biometrics do a good job of unlocking the app, itself, but are not used to safegaurd the encrypted values, themselves. This risk is mitigated by random IVs+keys, and provides a better UX to not reprompt for PIN on every transaction.                                                                                                |
| Weak device detection (jailbroken, compromised, old/ unsupported)               | Important because compromised OSes allow attackers to read memory. We need to research a good method for doing this                                                                                            | This is not implemented.                                                                                                                                                                                                                                                                                                                        |
| Defense in depth measures                                                       | Covered mostly in other threats                                                                                                                                                                                | Redundant encryption is recommended, user phishing/scam protection statements are recommended, larger PIN option is recommended.                                                                                                                                                                                                                |
| Salt/nonce/IV leakage due to exceptions/errors                                  | https://github.com/Tencent/MMKV/issues/288                                                                                                                                                                     | IV is unique (it is reset on construction of MMKV crypto library) - I am not satisfied that it is unique for every message, though. This would need in-depth testing, but it does not appear that the IV is unique to every message. This can be included into a larger 'weak auth' risk.                                                       |
| Lingering secrets in memory                                                     | Secrets could linger in memory even after the app is closed/deleted                                                                                                                                            | The secrets are stored in keychain, or encrypted by the keychain, which is clean up on removal of the app.                                                                                                                                                                                                                                      |
| PIN entry could potentially be bypassed                                         | PIN entry uses a fixed entry array, with custom number inputs, not keyboard.                                                                                                                                   | PIN entry is build robustly, and was not found to have any feasible bypass.                                                                                                                                                                                                                                                                     |
| Token price in ui could be manipulated                                          | Firebase/slingshot API provides this data                                                                                                                                                                      | This is out of scope. It is a risk, but the transaction data is pulled directly from on-chain.                                                                                                                                                                                                                                                  |
| Gas price could be manipulated                                                  |                                                                                                                                                                                                                | Gas price *could* be manipulated, but it is verified by the user upon transaction confirmation.                                                                                                                                                                                                                                                 |
| Slingshot endpoint impersonation                                                | This is a pretty low risk, and slingshot just provides read data                                                                                                                                               | Certs are not pinned (for anything). The certs for Slingshot are suggested to be pinned (2-3 redundant certs stored separately).                                                                                                                                                                                                                |
| Wallet Connect endpoint impersonation                                           | Endpoint doesn't really matter too much, as the transactions are not entirely trusted anyway (Wallet Connect link could be valid and still malicious)                                                          | Out of scope. Wallet Connect is invoked via the deep linking mechanism.                                                                                                                                                                                                                                                                         |
| ethers endpoint impersonation (for gas estimation)                              | This is also unlikely due to HTTPS usage                                                                                                                                                                       | This is technically possible, but gas is verified upon transaction confirmation. Also, ethers uses a quorum of provider endpoints, so it would be very difficult to successfully attack enough of the providers, and such an attack would have much more far reaching consequences than just this app: https://docs.ethers.io/v5/api/providers/ |
| Transaction variable compromise (recipient, gas fee, price)                     |                                                                                                                                                                                                                | Transaction data is validated using Yup, and verified by the user before confirming transactions                                                                                                                                                                                                                                                |
| Overflow/underflow of send amounts                                              |                                                                                                                                                                                                                | Overflow/underflow is detectable by user when validating transaction details                                                                                                                                                                                                                                                                    |
| Deep linking hijacking                                                          |                                                                                                                                                                                                                | Deep links are untrusted by default, the routing is done only by protocol predicate in the URI, not by the URI contents or params.                                                                                                                                                                                                              |
| Mnemonic/private key leakage                                                    |                                                                                                                                                                                                                | These values are stored securely.                                                                                                                                                                                                                                                                                                               |
| Coinbase pay impersonation/hijacking                                            | Coinbase Pay view allows the user to onramp with fiat - Buy(.ts) screen - two ways to access it (depending on the version) - Deposit/Buy button                                                                | The coinbase SDK is used for generating the CBP url that is used                                                                                                                                                                                                                                                                                |
| Liquidity zone manipulation                                                     | useLiquidityZones() runs the liquidityZones query to fetch rpc and transaction nodes. If it is manipulated, the rpc url could be redirected by an attacker and DoS a user, or redirect to a mev-heavy rpc node | This is out of scope, this data comes from Firebase.                                                                                                                                                                                                                                                                                            |
| semaphore/rwlock could have issues with multiple wallet writes at the same time |                                                                                                                                                                                                                | Lock does not appear to have any security concerns.                                                                                                                                                                                                                                                                                             |
| Local database write before encrypt (write THEN encrypt)                        | WalletStore.ts writes to MMKV, THEN encrypts (#102-103)                                                                                                                                                        | Dedicated encryption should be used.                                                                                                                                                                                                                                                                                                            |
| Log injection                                                                   | Log injection potentially allows for forged log messages or malicious code to be submitted, that could break out of the log view context                                                                       | User input logging of urls was found: useHandleLinks.ts:105 - "Logger.log('📫', `Unhandled url: ${rawURL}`)"                                                                                                                                                                                                                                    |
| Malicious code submission in bug report/request feature inputs                  | Malicious code could be inserted that could potentially break out of the context of viewing user comments                                                                                                      | There is no mitigation against this, but it is more of a note for this audit, and more of a backend problem.                                                                                                                                                                                                                                    |
| Liquidity zone swap                                                             | Included in transaction metadata                                                                                                                                                                               | Again, this is another untrusted source of data that the user is responsible for verifying.                                                                                                                                                                                                                                                     |
| Crypto patch vulnerability                                                      | There is a patch to replace to replace the javascript implementation of a crypto operation                                                                                                                     | The patch replaces the ethers custom kdf function with the node 'crypto' library version of the function.                                                                                                                                                                                                                                       |
| Sensitive data entry in bug report/request feature inputs                       | This is a good idea to avoid logging sensitive data                                                                                                                                                            | No mitigation present.                                                                                                                                                                                                                                                                                                                          |
| Transaction PIN disablement does not require PIN entry                          | Does this apply to biometrics as well?                                                                                                                                                                         | PIN prompt on transaction can be disabled without re-entry. Biometrics does re-prompt on disable.                                                                                                                                                                                                                                               |
| Dependency vulns                                                                | See reference links                                                                                                                                                                                            | Socket.dev is used for dependency vuln detection                                                                                                                                                                                                                                                                                                |
| Local wallet address overwrite                                                  | The metadata of the local wallets could be altered                                                                                                                                                             | No conceivable method for an attacker to overwrite the addresses was found                                                                                                                                                                                                                                                                      |
| WalletConnect/QR transaction redress                                            | Could transactions be marked as swap or something, but actually have different transaction data? - The transaction confirmation should cover this, but still is a threat regardless                            | These details are untrusted, but adding a user advisory on these flows is recommended                                                                                                                                                                                                                                                           |
| Lack of security tips in transaction confirmation                               | This should at least be a tip in the onboarding process, or a toast on first send                                                                                                                              | Not present                                                                                                                                                                                                                                                                                                                                     |
| Unexpected/useless params on crypto functions                                   | See references for example                                                                                                                                                                                     | Crypto is handled by MMKV and expo, no crypto params are supplied to them                                                                                                                                                                                                                                                                       |
| Reinstalling app allows accessing wallet data without the PIN                   | The unlock PIN needs to be part of the encrypt/decrypt operations for the wallet keys                                                                                                                          | Data store is wiped clean on reinstall.                                                                                                                                                                                                                                                                                                         |
| PIN stored in unencrypted space                                                 |                                                                                                                                                                                                                | PIN is stored in secure storage.                                                                                                                                                                                                                                                                                                                |
| Unauthenticated encryption                                                      | Encryption could be tampered with. There is no integrity assurance                                                                                                                                             | CFB mode is used, recommend to use GCM mode or add an HMAC verification step.                                                                                                                                                                                                                                                                   |

## Findings

- MMKV used for encryption (poor crypto library)
  
  - MMKV does rotate the IV on initialization, it is unclear if the IV is unique to every message
  
  - AES-CFB-128 is used, but integrity validation is not present
  
  - recrypt() is performed AFTER storage, meaning the values are written to storage potentially unencrypted (the docs make it seem as though this is the proper method)
  
  - Custom encryption with an authenticated encryption mode is suggested

- Weak wallet pins allowed
  
  - This is likely an accepted risk, but it is recommended to enforce a certain amount of minimum entropy
  
  - It is also recommended to allow users to set larger PINs

- No PIN brute force lockout (10^6 combinations is feasibly brute forceable)

- PIN auth for transactions can be disabled without requiring re-entry of PIN

- Unbounded wallet names allowed (more of a UI hinderance, but you users cannot delete these wallets, as their view does not render if the name is large enough, these names are also stored in the analytics server)

- Users could potentially enter sensitive information in bug reports and feature requests (e.g. mnemonics or passcodes)
  
  - There is no mitigation against this, but it is more of a note for this audit, and more of a backend problem

- Endpoint certificates not pinned

- Endpoints (Firebase, Ethers, Coinbase Pay, Slingshot backend, etc.) are a potential risk vector. They were left out of scope of this review, but it is noted that the high trust level of these data sources presents the potential for vulnerability

- Weak device detection (jailbroken, compromised, old/ unsupported) is not implemented beyond what expo performs

- User protection statements are recommended (e.g. verify transaction parameters before confirming)
  
  - Emphasis on QR code and WalletConnect flows

- Potential log injection via user input logging of URLs is present: useHandleLinks.ts:105 - "Logger.log('📫', `Unhandled url: ${rawURL}`)"

## References

- [https://www.cossacklabs.com/blog/react-native-libraries-security/#react-native-guides-recommend-insecure-libraries](https://www.cossacklabs.com/blog/react-native-libraries-security/#react-native-guides-recommend-insecure-libraries)
- [https://www.cossacklabs.com/blog/crypto-wallets-security/#dependency-issues-with-crypto-wallets](https://www.cossacklabs.com/blog/crypto-wallets-security/#dependency-issues-with-crypto-wallets)
- [https://www.cossacklabs.com/blog/react-native-app-security/](https://www.cossacklabs.com/blog/react-native-app-security/)
- [https://owasp.org/www-project-mobile-top-10/](https://owasp.org/www-project-mobile-top-10/)
