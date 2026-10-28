# Threat model: jeverpro/multibit (MultiBit Classic)

_Status: built from the code plus the user's answers: personal/internal use, only the offline wallet/backup thief matters, no code areas excluded._

## System overview
MultiBit Classic (v0.5.19) is a desktop **SPV Bitcoin wallet**: Java 7, Swing UI, and a bundled fork of bitcoinj (`0.11.2-MB-ALICE`). Upstream deprecated it in 2017 (see README.md). Everything runs on the end user's machine: there are no servers and no infrastructure code. Its only remote dependencies are Bitcoin P2P peers and a few HTTPS endpoints (exchange rate APIs, multibit.org).

### Trust boundaries
1. **Bitcoin P2P network → app**: blocks, transactions, peer addresses and checkpoints come from untrusted peers (`org/multibit/network/*`, `com/google/bitcoin/core/MultiBitBlockChain.java`).
2. **Internet HTTPS → app**: exchange tickers via XChange (`org/multibit/exchange/TickerTimerTask.java`, `CurrencyConverter.java`), the multibit.org version and help pages, and HTML help shown in the in-app browser (`viewsystem/swing/browser/*`). On startup `MultiBit.java` calls `ConnectHttps.trustAllCerts()`, which installs a JVM-wide trust-all `SSLSocketFactory` and hostname verifier (`model/exchange/ConnectHttps.java`).
3. **External apps or websites → app**: `bitcoin:` URIs from the OS URI handler (`platform/handler/*`, `controller/core/CoreController.java`, `controller/bitcoin/BitcoinController.java` with the `OPEN_URI_SHOW_DIALOG` / `OPEN_URI_USE_URI` preferences), QR codes (`qrcode/*`), and pasted addresses.
4. **Other local processes → app**: the single-instance listener on `127.0.0.1` (`ApplicationInstanceManager.java`) accepts a raw URI from any local process, with no authentication.
5. **Filesystem → app**: wallet files (`.wallet`, `.info`), backups, imported/exported private key files (`file/PrivateKeysHandler.java`), `multibit.properties` (looked up first in the current working directory; see `ApplicationDataDirectoryLocator.java`), and checkpoint files.

## Components
| Component | Code |
|---|---|
| **Wallet & Keys** | `org/multibit/file/*` (FileHandler, BackupManager, PrivateKeysHandler, SecureFiles), `org/multibit/crypto/KeyCrypterOpenSSL.java`, `org/multibit/model/bitcoin/*`, password actions (`viewsystem/swing/action/*Password*`, `*PrivateKeys*`, `SignMessage*`, `VerifyMessage*`) |
| **Bitcoin Network** | `org/multibit/network/*`, `org/multibit/controller/bitcoin/*`, `com/google/bitcoin/core/*`, send actions (`SendBitcoin*Action.java`) |
| **Payment Requests** | `bitcoin:` URI handling (`platform/handler/*`, `controller/core/CoreController.java`, `ShowOpenUri*Action.java`), `ApplicationInstanceManager.java`, `qrcode/*` |
| **Exchange & Web Services** | `org/multibit/exchange/*`, `org/multibit/model/exchange/*` (incl. `ConnectHttps.java`), `controller/exchange/*`, version check in `MultiBit.java` |
| **Desktop UI** | `org/multibit/viewsystem/**` (except the actions listed above), `viewsystem/swing/browser/*`, `utils/SafeDesktop.java`, `utils/HtmlUtils.java`, `platform/builder/*` |
| **Build & Dependencies** | `pom.xml` (incl. the Bitcoinj Enforcer digest rules), `src/assemble/*` |

## Assets (most sensitive first)
1. **Private keys and funds**: wallet files, encrypted with AES-256-CBC under an OpenSSL-style PBE key (`KeyCrypterOpenSSL`, 1024 iterations); unencrypted wallets are possible; exported key files; backups.
2. **Wallet password**, as entered in the UI and while it sits in memory.
3. **Transaction integrity**: the payee address and amount the user sends, and the fee.
4. **Privacy**: addresses, balances, transaction history and labels (CSV export, logs).
5. **Availability** of the wallet and its sync.

## Entry points and who can reach them
- P2P messages: any Bitcoin peer, or a network MITM (peers can be pinned with `peers=` in `multibit.properties`).
- HTTPS responses (tickers, version, help pages): anyone who can MITM the connection, because certificate checks are disabled globally.
- `bitcoin:` URIs and QR codes: any website, email or document the user opens.
- Local socket on 127.0.0.1: any local process or user on the same host.
- Wallet, key and properties files: whoever can write to the app data directory, the working directory, or a file the user imports.

## Usage context (user)
- **Personal/internal use only.** The wallet is not shipped to other users.

## Attacker profiles (user-confirmed)
- **A4 Offline thief (in scope; the attacker that matters)**: obtains a copy of the wallet, backup or exported key files (stolen laptop, disk, USB stick, cloud copy) and tries to recover keys offline, e.g. by cracking the password. Relevant code: wallet encryption (`KeyCrypterOpenSSL`, 1024 PBE iterations), unencrypted wallet and backup handling, exported key files (`PrivateKeysHandler`), secure delete (`SecureFiles`), plaintext data left on disk (logs, CSV exports, `.info` files).
- **Not selected by the user** (the user did not pick these as attackers that matter, so they count as lower priority): A1 network attacker (MITM, malicious peers, spoofed HTTPS servers); A2 malicious `bitcoin:` URIs, QR codes or web content; A3 malware or other users on the same machine.

## Existing controls
- Wallet encryption with AES-256 and a per-wallet salt; secure delete of old files (`SecureFiles.secureDelete`).
- Send confirmation dialog; optional confirmation dialog for `bitcoin:` URIs (user preference).
- The single-instance socket binds to loopback only.
- The Maven Enforcer `DigestRule` pins dependency hashes (`pom.xml`).
- SPV header and proof-of-work validation in bitcoinj; checkpoints.

## Out of scope / accepted risk
- No code areas are excluded (the user said everything is in scope, including the bitcoinj fork, dependencies, the exchange ticker and build tooling). Findings that only matter to attackers A1–A3 are deprioritised, not dismissed outright.

## Open questions
- Does any attacker other than the offline thief matter in practice, e.g. connecting from untrusted Wi-Fi (A1)?
- There is no infrastructure repository: this is a desktop application, so none applies.
