---
description: Apple security frameworks: Keychain Services, CryptoKit, biometric auth, Secure Enclave, and credential storage patterns
user_invocable: true
---

# Skill 16 — Swift Security

**Source:** Apple Security Framework, CryptoKit, LocalAuthentication, OWASP MASVS
**Applies to:** Credential storage, encryption, authentication, data protection
**Minimum target:** iOS 15+ (features noted where iOS 16+ or iOS 17+ required)

---

## Keychain Fundamentals

The Keychain is the **only** acceptable place to store secrets (tokens, passwords, keys) on Apple platforms. It persists across app reinstalls (by default), encrypts data at rest, and integrates with iCloud sync and biometric protection.

### Adding an Item

```swift
import Security

func saveToKeychain(account: String, data: Data, service: String) throws {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: account,
        kSecAttrService as String: service,
        kSecValueData as String: data,
        kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly
    ]

    let status = SecItemAdd(query as CFDictionary, nil)

    switch status {
    case errSecSuccess:
        return
    case errSecDuplicateItem:
        try updateKeychainItem(account: account, data: data, service: service)
    default:
        throw KeychainError.unhandled(status)
    }
}
```

### Reading an Item

```swift
func readFromKeychain(account: String, service: String) throws -> Data {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: account,
        kSecAttrService as String: service,
        kSecReturnData as String: true,
        kSecMatchLimit as String: kSecMatchLimitOne
    ]

    var result: AnyObject?
    let status = SecItemCopyMatching(query as CFDictionary, &result)

    guard status == errSecSuccess, let data = result as? Data else {
        throw KeychainError.itemNotFound
    }

    return data
}
```

### Updating an Item

```swift
func updateKeychainItem(account: String, data: Data, service: String) throws {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: account,
        kSecAttrService as String: service
    ]

    let attributes: [String: Any] = [
        kSecValueData as String: data
    ]

    let status = SecItemUpdate(query as CFDictionary, attributes as CFDictionary)

    guard status == errSecSuccess else {
        throw KeychainError.unhandled(status)
    }
}
```

### Deleting an Item

```swift
func deleteFromKeychain(account: String, service: String) throws {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: account,
        kSecAttrService as String: service
    ]

    let status = SecItemDelete(query as CFDictionary)

    guard status == errSecSuccess || status == errSecItemNotFound else {
        throw KeychainError.unhandled(status)
    }
}
```

### Add-or-Update Pattern (Preferred)

Always use the add-or-update pattern. Calling `SecItemAdd` when an item already exists returns `errSecDuplicateItem` — you must handle it.

```swift
func saveOrUpdate(account: String, data: Data, service: String) throws {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: account,
        kSecAttrService as String: service
    ]

    let attributes: [String: Any] = [
        kSecValueData as String: data,
        kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly
    ]

    let status = SecItemUpdate(query as CFDictionary, attributes as CFDictionary)

    if status == errSecItemNotFound {
        var addQuery = query
        addQuery.merge(attributes) { _, new in new }
        let addStatus = SecItemAdd(addQuery as CFDictionary, nil)
        guard addStatus == errSecSuccess else {
            throw KeychainError.unhandled(addStatus)
        }
    } else if status != errSecSuccess {
        throw KeychainError.unhandled(status)
    }
}
```

### Error Handling

**Never ignore `OSStatus` from Keychain calls.** Always check the return value and map it to a meaningful error.

```swift
enum KeychainError: Error, LocalizedError {
    case itemNotFound
    case duplicateItem
    case authenticationFailed
    case unhandled(OSStatus)

    var errorDescription: String? {
        switch self {
        case .itemNotFound:
            return "Keychain item not found"
        case .duplicateItem:
            return "Keychain item already exists"
        case .authenticationFailed:
            return "Biometric or passcode authentication failed"
        case .unhandled(let status):
            return "Keychain error: \(status) - \(SecCopyErrorMessageString(status, nil) as String? ?? "Unknown")"
        }
    }
}
```

---

## Keychain Item Classes

| Class Constant | Purpose | Common Use |
|---|---|---|
| `kSecClassGenericPassword` | App-specific secrets | OAuth tokens, API keys, user credentials |
| `kSecClassInternetPassword` | Server-bound credentials | Website passwords (includes URL attributes) |
| `kSecClassKey` | Cryptographic keys | Encryption keys, Secure Enclave keys |
| `kSecClassCertificate` | X.509 certificates | Certificate pinning, client certs |
| `kSecClassIdentity` | Certificate + private key pair | Client TLS authentication |

### Generic Password vs. Internet Password

Use `kSecClassGenericPassword` for most app secrets. Use `kSecClassInternetPassword` only when you need URL-specific attributes (`kSecAttrServer`, `kSecAttrProtocol`, `kSecAttrPort`).

```swift
// Generic password — app-level token
let genericQuery: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrAccount as String: "user-auth-token",
    kSecAttrService as String: "com.myapp.auth",
    kSecValueData as String: tokenData
]

// Internet password — server-specific credential
let internetQuery: [String: Any] = [
    kSecClass as String: kSecClassInternetPassword,
    kSecAttrAccount as String: "user@example.com",
    kSecAttrServer as String: "api.example.com",
    kSecAttrProtocol as String: kSecAttrProtocolHTTPS,
    kSecValueData as String: passwordData
]
```

---

## Keychain Access Control and Accessibility

### Accessibility Levels

**Always set `kSecAttrAccessible` explicitly.** The default behavior varies across OS versions.

| Constant | When Accessible | Survives Backup | Use Case |
|---|---|---|---|
| `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` | After first unlock, until reboot | No (device-only) | **Default choice for most secrets** |
| `kSecAttrAccessibleAfterFirstUnlock` | After first unlock, until reboot | Yes (migrates) | Tokens that sync via backup |
| `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` | Only while device unlocked | No (device-only) | Highly sensitive data |
| `kSecAttrAccessibleWhenUnlocked` | Only while device unlocked | Yes (migrates) | Sensitive data that syncs |
| `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly` | While unlocked + passcode exists | No (device-only) | Data requiring passcode protection |

**Recommendation:** Use `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` as the default. It allows background access (needed for background fetch, push notifications) while preventing data migration to other devices.

### SecAccessControl — Biometric and Passcode Gates

`SecAccessControl` adds authentication requirements on top of accessibility levels.

```swift
import Security
import LocalAuthentication

func saveBiometricProtectedItem(account: String, data: Data, service: String) throws {
    var error: Unmanaged<CFError>?

    guard let accessControl = SecAccessControlCreateWithFlags(
        nil,
        kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
        .biometryCurrentSet,  // invalidates if biometry changes
        &error
    ) else {
        throw error!.takeRetainedValue() as Error
    }

    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: account,
        kSecAttrService as String: service,
        kSecValueData as String: data,
        kSecAttrAccessControl as String: accessControl
    ]

    let status = SecItemAdd(query as CFDictionary, nil)
    guard status == errSecSuccess else {
        throw KeychainError.unhandled(status)
    }
}
```

### Access Control Flags

| Flag | Behavior |
|---|---|
| `.biometryCurrentSet` | Requires biometric auth; invalidated if enrollments change |
| `.biometryAny` | Requires biometric auth; survives enrollment changes |
| `.devicePasscode` | Requires device passcode |
| `.userPresence` | Biometric OR passcode (fallback) |
| `.or` / `.and` | Combine constraints (e.g., `.biometryCurrentSet` OR `.devicePasscode`) |

**Use `.biometryCurrentSet` for maximum security** — if a new fingerprint is enrolled (potentially by an attacker with physical access), existing items become inaccessible.

**Important:** Do NOT set both `kSecAttrAccessControl` and `kSecAttrAccessible` in the same query. The accessibility level is embedded inside `SecAccessControl`. Setting both causes `errSecParam`.

---

## Biometric Authentication

### LAContext — The Authentication Gateway

`LAContext` evaluates biometric policies. It can work standalone or integrate with Keychain access.

```swift
import LocalAuthentication

func authenticateWithBiometrics() async throws -> Bool {
    let context = LAContext()
    context.localizedCancelTitle = "Use Password"

    var authError: NSError?
    guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &authError) else {
        throw BiometricError.notAvailable(authError)
    }

    return try await context.evaluatePolicy(
        .deviceOwnerAuthenticationWithBiometrics,
        localizedReason: "Authenticate to access your account"
    )
}
```

### Biometric Policies

| Policy | Behavior |
|---|---|
| `.deviceOwnerAuthenticationWithBiometrics` | Biometric only, no passcode fallback |
| `.deviceOwnerAuthentication` | Biometric with automatic passcode fallback |

### LAContext + Keychain Integration (Correct Pattern)

**Never use `LAContext.evaluatePolicy()` alone as an auth gate.** An attacker can bypass the local check. Instead, tie authentication to Keychain access — the Secure Enclave enforces the policy at the hardware level.

```swift
// CORRECT: Keychain-enforced biometric protection
func readBiometricProtectedItem(account: String, service: String) throws -> Data {
    let context = LAContext()
    context.localizedReason = "Access your secure data"

    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: account,
        kSecAttrService as String: service,
        kSecReturnData as String: true,
        kSecMatchLimit as String: kSecMatchLimitOne,
        kSecUseAuthenticationContext as String: context
    ]

    var result: AnyObject?
    let status = SecItemCopyMatching(query as CFDictionary, &result)

    guard status == errSecSuccess, let data = result as? Data else {
        if status == errSecAuthFailed || status == errSecUserCanceled {
            throw KeychainError.authenticationFailed
        }
        throw KeychainError.itemNotFound
    }

    return data
}
```

```swift
// WRONG: standalone LAContext as auth gate — can be bypassed
func getSecretDangerously() async throws -> String {
    let context = LAContext()
    let success = try await context.evaluatePolicy(
        .deviceOwnerAuthenticationWithBiometrics,
        localizedReason: "Authenticate"
    )
    if success {
        return loadSecretFromUserDefaults()  // double wrong: UserDefaults + bypassable gate
    }
    throw BiometricError.failed
}
```

### Biometric Availability Checks

```swift
func biometricType() -> BiometricType {
    let context = LAContext()
    var error: NSError?
    guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) else {
        return .none
    }

    switch context.biometryType {
    case .faceID: return .faceID
    case .touchID: return .touchID
    case .opticID: return .opticID
    case .none: return .none
    @unknown default: return .none
    }
}

enum BiometricType {
    case faceID, touchID, opticID, none
}
```

**Remember:** Add `NSFaceIDUsageDescription` to Info.plist. Without it, Face ID authentication silently fails.

---

## Secure Enclave

The Secure Enclave is a hardware security processor that generates and stores keys. Keys **never leave** the Secure Enclave — cryptographic operations happen inside it.

### Key Generation

```swift
import CryptoKit

func generateSecureEnclaveKey() throws -> SecureEnclave.P256.Signing.PrivateKey {
    let key = try SecureEnclave.P256.Signing.PrivateKey()
    return key
}
```

### Persisting a Secure Enclave Key

Secure Enclave keys are ephemeral by default. To persist across app launches, store the `dataRepresentation` in the Keychain.

```swift
func createAndStoreSecureEnclaveKey(tag: String) throws -> SecureEnclave.P256.Signing.PrivateKey {
    let key = try SecureEnclave.P256.Signing.PrivateKey()

    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: tag,
        kSecAttrService as String: "com.myapp.secure-enclave-keys",
        kSecValueData as String: key.dataRepresentation,
        kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlockedThisDeviceOnly
    ]

    let status = SecItemAdd(query as CFDictionary, nil)
    guard status == errSecSuccess else {
        throw KeychainError.unhandled(status)
    }

    return key
}

func loadSecureEnclaveKey(tag: String) throws -> SecureEnclave.P256.Signing.PrivateKey {
    let storedData = try readFromKeychain(account: tag, service: "com.myapp.secure-enclave-keys")
    return try SecureEnclave.P256.Signing.PrivateKey(dataRepresentation: storedData)
}
```

### Biometric-Protected Secure Enclave Key

```swift
func createBiometricSecureEnclaveKey() throws -> SecureEnclave.P256.Signing.PrivateKey {
    var error: Unmanaged<CFError>?
    guard let accessControl = SecAccessControlCreateWithFlags(
        nil,
        kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
        [.privateKeyUsage, .biometryCurrentSet],
        &error
    ) else {
        throw error!.takeRetainedValue() as Error
    }

    let authContext = LAContext()
    authContext.localizedReason = "Create secure key"

    return try SecureEnclave.P256.Signing.PrivateKey(
        accessControl: accessControl,
        authenticationContext: authContext
    )
}
```

### Signing with Secure Enclave

```swift
func signData(_ data: Data, with key: SecureEnclave.P256.Signing.PrivateKey) throws -> Data {
    let signature = try key.signature(for: data)
    return signature.derRepresentation
}

func verifySignature(_ signature: Data, for data: Data, publicKey: P256.Signing.PublicKey) -> Bool {
    guard let ecdsaSignature = try? P256.Signing.ECDSASignature(derRepresentation: signature) else {
        return false
    }
    return publicKey.isValidSignature(ecdsaSignature, for: data)
}
```

### Secure Enclave Constraints

- Only supports **P256** (NIST) curves — no Curve25519, no RSA
- Only supports **signing** and **key agreement** — no direct encryption
- Keys are **device-bound** — they do not sync via iCloud or backup
- Check availability: `SecureEnclave.isAvailable` (false on Simulator)
- Must use `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` or `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly`

---

## CryptoKit — Symmetric Cryptography

### Hashing

```swift
import CryptoKit

let data = "Hello, World!".data(using: .utf8)!

let sha256 = SHA256.hash(data: data)     // 32 bytes
let sha384 = SHA384.hash(data: data)     // 48 bytes
let sha512 = SHA512.hash(data: data)     // 64 bytes

// Convert to hex string
let hexString = sha256.compactMap { String(format: "%02x", $0) }.joined()
```

### HMAC (Message Authentication)

```swift
let key = SymmetricKey(size: .bits256)
let authCode = HMAC<SHA256>.authenticationCode(for: data, using: key)

// Verify
let isValid = HMAC<SHA256>.isValidAuthenticationCode(authCode, authenticating: data, using: key)
```

### AES-GCM (Authenticated Encryption — Default Choice)

```swift
// Encrypt
func encrypt(data: Data, using key: SymmetricKey) throws -> Data {
    let sealedBox = try AES.GCM.seal(data, using: key)
    guard let combined = sealedBox.combined else {
        throw CryptoError.encryptionFailed
    }
    return combined  // nonce + ciphertext + tag (all in one)
}

// Decrypt
func decrypt(data: Data, using key: SymmetricKey) throws -> Data {
    let sealedBox = try AES.GCM.SealedBox(combined: data)
    return try AES.GCM.open(sealedBox, using: key)
}
```

### ChaChaPoly (Alternative Authenticated Encryption)

ChaChaPoly is functionally equivalent to AES-GCM. Prefer AES-GCM on Apple hardware (hardware-accelerated). Use ChaChaPoly for interoperability with systems that prefer it.

```swift
// Encrypt
let sealedBox = try ChaChaPoly.seal(data, using: key)
let encrypted = sealedBox.combined

// Decrypt
let box = try ChaChaPoly.SealedBox(combined: encrypted)
let decrypted = try ChaChaPoly.open(box, using: key)
```

### Symmetric Key Management

```swift
// Generate a new random key
let key = SymmetricKey(size: .bits256)

// Derive a key from a password using HKDF
let salt = Data("my-app-salt".utf8)
let inputKeyMaterial = SymmetricKey(data: Data("user-password".utf8))
let derivedKey = HKDF<SHA256>.deriveKey(
    inputKeyMaterial: inputKeyMaterial,
    salt: salt,
    info: Data("encryption-key".utf8),
    outputByteCount: 32
)

// Store a key in Keychain — convert to Data first
func storeKey(_ key: SymmetricKey, account: String) throws {
    let keyData = key.withUnsafeBytes { Data($0) }
    try saveOrUpdate(account: account, data: keyData, service: "com.myapp.keys")
}

// Load a key from Keychain
func loadKey(account: String) throws -> SymmetricKey {
    let keyData = try readFromKeychain(account: account, service: "com.myapp.keys")
    return SymmetricKey(data: keyData)
}
```

---

## CryptoKit — Public Key Cryptography

### Algorithm Selection

| Algorithm | Use Case | Key Size |
|---|---|---|
| `P256` (NIST P-256) | ECDSA signing, ECDH key agreement | 256-bit |
| `P384` (NIST P-384) | Higher security signing/agreement | 384-bit |
| `P521` (NIST P-521) | Maximum security signing/agreement | 521-bit |
| `Curve25519` | Signing (Ed25519), key agreement (X25519) | 256-bit |

### ECDSA Signing (P256)

```swift
// Generate key pair
let signingKey = P256.Signing.PrivateKey()
let verifyingKey = signingKey.publicKey

// Sign
let signature = try signingKey.signature(for: data)

// Verify
let isValid = verifyingKey.isValidSignature(signature, for: data)
```

### Ed25519 Signing (Curve25519)

```swift
let signingKey = Curve25519.Signing.PrivateKey()
let publicKey = signingKey.publicKey

let signature = try signingKey.signature(for: data)
let isValid = publicKey.isValidSignature(signature, for: data)
```

### ECDH Key Agreement

Two parties derive a shared secret without transmitting private keys.

```swift
// Alice generates her key pair
let alicePrivate = P256.KeyAgreement.PrivateKey()
let alicePublic = alicePrivate.publicKey

// Bob generates his key pair
let bobPrivate = P256.KeyAgreement.PrivateKey()
let bobPublic = bobPrivate.publicKey

// Both derive the same shared secret
let aliceShared = try alicePrivate.sharedSecretFromKeyAgreement(with: bobPublic)
let bobShared = try bobPrivate.sharedSecretFromKeyAgreement(with: alicePublic)
// aliceShared == bobShared

// Derive a symmetric key from the shared secret
let symmetricKey = aliceShared.hkdfDerivedSymmetricKey(
    using: SHA256.self,
    salt: Data("app-salt".utf8),
    sharedInfo: Data("encryption".utf8),
    outputByteCount: 32
)
```

### Curve25519 Key Agreement (X25519)

```swift
let alicePrivate = Curve25519.KeyAgreement.PrivateKey()
let bobPrivate = Curve25519.KeyAgreement.PrivateKey()

let sharedSecret = try alicePrivate.sharedSecretFromKeyAgreement(with: bobPrivate.publicKey)
let key = sharedSecret.hkdfDerivedSymmetricKey(
    using: SHA256.self,
    salt: Data(),
    sharedInfo: Data("key-agreement".utf8),
    outputByteCount: 32
)
```

### Key Serialization

```swift
// Export public key
let publicKeyData = signingKey.publicKey.compactRepresentation  // 32 bytes
let publicKeyDER = signingKey.publicKey.derRepresentation       // DER-encoded
let publicKeyPEM = signingKey.publicKey.pemRepresentation       // PEM string

// Import public key
let importedKey = try P256.Signing.PublicKey(compactRepresentation: publicKeyData!)
let importedFromPEM = try P256.Signing.PublicKey(pemRepresentation: pemString)

// Export private key (handle with care)
let privateKeyData = signingKey.rawRepresentation  // raw bytes
```

---

## Credential Storage Patterns

### OAuth Token Storage

```swift
struct TokenStore {
    private let service = "com.myapp.auth"
    private let accessTokenKey = "access-token"
    private let refreshTokenKey = "refresh-token"

    func saveTokens(accessToken: String, refreshToken: String) throws {
        try saveOrUpdate(
            account: accessTokenKey,
            data: Data(accessToken.utf8),
            service: service
        )
        try saveOrUpdate(
            account: refreshTokenKey,
            data: Data(refreshToken.utf8),
            service: service
        )
    }

    func accessToken() throws -> String? {
        guard let data = try? readFromKeychain(account: accessTokenKey, service: service) else {
            return nil
        }
        return String(data: data, encoding: .utf8)
    }

    func refreshToken() throws -> String? {
        guard let data = try? readFromKeychain(account: refreshTokenKey, service: service) else {
            return nil
        }
        return String(data: data, encoding: .utf8)
    }

    func clearTokens() throws {
        try deleteFromKeychain(account: accessTokenKey, service: service)
        try deleteFromKeychain(account: refreshTokenKey, service: service)
    }
}
```

### Token Refresh Flow

```swift
protocol AuthServiceProtocol: Sendable {
    func refreshAccessToken() async throws -> String
    func authenticatedRequest(_ request: URLRequest) async throws -> (Data, URLResponse)
}

actor AuthService: AuthServiceProtocol {
    private let tokenStore: TokenStore
    private let urlSession: URLSession
    private var refreshTask: Task<String, Error>?

    init(tokenStore: TokenStore = TokenStore(), urlSession: URLSession = .shared) {
        self.tokenStore = tokenStore
        self.urlSession = urlSession
    }

    func authenticatedRequest(_ request: URLRequest) async throws -> (Data, URLResponse) {
        var authorizedRequest = request
        let token = try tokenStore.accessToken() ?? refreshAccessToken()
        authorizedRequest.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")

        let (data, response) = try await urlSession.data(for: authorizedRequest)

        if let httpResponse = response as? HTTPURLResponse, httpResponse.statusCode == 401 {
            // Token expired — refresh and retry once
            let newToken = try await refreshAccessToken()
            authorizedRequest.setValue("Bearer \(newToken)", forHTTPHeaderField: "Authorization")
            return try await urlSession.data(for: authorizedRequest)
        }

        return (data, response)
    }

    func refreshAccessToken() async throws -> String {
        // Coalesce concurrent refresh requests into a single network call
        if let existingTask = refreshTask {
            return try await existingTask.value
        }

        let task = Task { () -> String in
            defer { refreshTask = nil }

            guard let refreshToken = try tokenStore.refreshToken() else {
                throw AuthError.noRefreshToken
            }

            let newTokens = try await performTokenRefresh(refreshToken: refreshToken)
            try tokenStore.saveTokens(
                accessToken: newTokens.accessToken,
                refreshToken: newTokens.refreshToken
            )
            return newTokens.accessToken
        }

        refreshTask = task
        return try await task.value
    }

    private func performTokenRefresh(refreshToken: String) async throws -> TokenPair {
        // Build and execute the refresh request
        var request = URLRequest(url: URL(string: "https://api.example.com/auth/refresh")!)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = try JSONEncoder().encode(["refresh_token": refreshToken])

        let (data, _) = try await urlSession.data(for: request)
        return try JSONDecoder().decode(TokenPair.self, from: data)
    }
}

struct TokenPair: Codable {
    let accessToken: String
    let refreshToken: String
}
```

### Logout — Secure Credential Cleanup

```swift
func logout() throws {
    try tokenStore.clearTokens()

    // Clear any cached user data
    URLCache.shared.removeAllCachedResponses()
    HTTPCookieStorage.shared.removeCookies(since: .distantPast)

    // Clear UserDefaults of non-sensitive preferences if needed
    // Do NOT rely on UserDefaults having stored secrets — but clean up anyway
}
```

---

## Anti-Patterns and Common Mistakes

### 1. Storing Secrets in UserDefaults

```swift
// WRONG: UserDefaults is a plain plist file — trivially readable
UserDefaults.standard.set(accessToken, forKey: "token")
UserDefaults.standard.set(password, forKey: "password")

// CORRECT: Use Keychain
try saveOrUpdate(account: "access-token", data: Data(accessToken.utf8), service: "com.myapp.auth")
```

**Why:** UserDefaults writes to an unencrypted plist in the app sandbox. On jailbroken devices, or via device backup extraction, anyone can read it.

### 2. Hardcoded Secrets in Source Code

```swift
// WRONG: hardcoded API key in source
let apiKey = "sk-1234567890abcdef"

// WRONG: hidden in a plist or xcconfig — still in the app bundle
// APIKeys.plist { "StripeKey": "sk_live_..." }

// CORRECT: fetch secrets from a secure backend at runtime
// or use environment-specific server-side configuration
```

**Why:** The app binary is trivially decompiled. Strings are extracted in seconds. Hardcoded keys are the #1 credential leak vector in mobile apps.

### 3. Ignoring OSStatus Return Values

```swift
// WRONG: fire and forget
SecItemAdd(query as CFDictionary, nil)  // no error check

// CORRECT: always check status
let status = SecItemAdd(query as CFDictionary, nil)
guard status == errSecSuccess else {
    throw KeychainError.unhandled(status)
}
```

### 4. Using LAContext Alone as an Auth Gate

```swift
// WRONG: LAContext result is a local boolean — no hardware enforcement
let authenticated = try await context.evaluatePolicy(.deviceOwnerAuthentication, localizedReason: "...")
if authenticated {
    showSecretData()  // an attacker can patch this check
}

// CORRECT: tie biometric auth to Keychain access (see Biometric Authentication section)
```

### 5. Missing kSecAttrAccessible

```swift
// WRONG: no accessibility level set — behavior varies by OS version
let query: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrAccount as String: "token",
    kSecValueData as String: data
]

// CORRECT: always set explicitly
let query: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrAccount as String: "token",
    kSecValueData as String: data,
    kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly
]
```

### 6. Calling SecItem APIs on the Main Thread

Keychain operations can block for hundreds of milliseconds, especially with biometric prompts or Secure Enclave operations.

```swift
// WRONG: blocking the main thread
@MainActor
func loadToken() -> String? {
    // SecItemCopyMatching can block UI
    let data = try? readFromKeychain(account: "token", service: "auth")
    return data.flatMap { String(data: $0, encoding: .utf8) }
}

// CORRECT: move to a background context
func loadToken() async throws -> String? {
    let data = try await Task.detached {
        try readFromKeychain(account: "token", service: "auth")
    }.value
    return String(data: data, encoding: .utf8)
}
```

### 7. Storing Raw Passwords

```swift
// WRONG: storing the password directly
try saveOrUpdate(account: "password", data: Data(password.utf8), service: "auth")

// CORRECT: if you must verify locally, store a hash
let passwordData = Data(password.utf8)
let hash = SHA256.hash(data: passwordData)
let hashData = Data(hash)
try saveOrUpdate(account: "password-hash", data: hashData, service: "auth")

// BEST: don't store passwords at all — use token-based auth
```

### 8. Not Clearing Secrets from Memory

```swift
// For highly sensitive operations, zero out buffers when done
var sensitiveData = Data(password.utf8)
defer {
    sensitiveData.resetBytes(in: 0..<sensitiveData.count)
}
// use sensitiveData...
```

---

## OWASP Mobile Top 10 Alignment

How this skill maps to [OWASP Mobile Top 10 (2024)](https://owasp.org/www-project-mobile-top-10/):

| OWASP Risk | Mitigation in This Skill |
|---|---|
| **M1: Improper Credential Usage** | Keychain storage, token refresh flow, no hardcoded secrets |
| **M2: Inadequate Supply Chain Security** | Not covered (build pipeline concern) |
| **M3: Insecure Authentication/Authorization** | LAContext + Keychain integration, biometric-protected items |
| **M4: Insufficient Input/Output Validation** | Not covered (see skill 06 Error Handling) |
| **M5: Insecure Communication** | Certificate pinning via `SecTrust` (use `URLSession` delegate) |
| **M6: Inadequate Privacy Controls** | `ThisDeviceOnly` accessibility, Secure Enclave |
| **M7: Insufficient Binary Protections** | No hardcoded secrets, runtime key fetching |
| **M8: Security Misconfiguration** | Explicit `kSecAttrAccessible`, proper access control flags |
| **M9: Insecure Data Storage** | Keychain over UserDefaults, encrypted CryptoKit storage |
| **M10: Insufficient Cryptography** | CryptoKit (AES-GCM, P256, HKDF), no deprecated algorithms |

---

## Quick Reference — Algorithm Selection

| Need | CryptoKit API | Minimum iOS |
|---|---|---|
| Hash data | `SHA256`, `SHA384`, `SHA512` | iOS 13 |
| Authenticate a message (HMAC) | `HMAC<SHA256>` | iOS 13 |
| Encrypt data (symmetric) | `AES.GCM` | iOS 13 |
| Encrypt data (alternative) | `ChaChaPoly` | iOS 13 |
| Sign data (NIST) | `P256.Signing` | iOS 13 |
| Sign data (modern) | `Curve25519.Signing` | iOS 13 |
| Key agreement (NIST) | `P256.KeyAgreement` | iOS 13 |
| Key agreement (modern) | `Curve25519.KeyAgreement` | iOS 13 |
| Derive key from password | `HKDF<SHA256>` | iOS 13 |
| Hardware-backed key | `SecureEnclave.P256` | iOS 13 |

---

## Key Rules

- [ ] Store all secrets in the Keychain — never in UserDefaults, plists, or source code
- [ ] Always set `kSecAttrAccessible` explicitly on every Keychain item
- [ ] Always handle `OSStatus` from every `SecItem*` call
- [ ] Use the add-or-update pattern for Keychain writes (handle `errSecDuplicateItem`)
- [ ] Use `SecAccessControl` + Keychain for biometric auth — never `LAContext` alone
- [ ] Use `.biometryCurrentSet` for biometric-protected items (invalidates on enrollment change)
- [ ] Prefer `AES.GCM` for symmetric encryption (hardware-accelerated on Apple devices)
- [ ] Use `SecureEnclave.P256` for hardware-backed keys when available
- [ ] Never call `SecItem*` on `@MainActor` — Keychain operations can block
- [ ] Add `NSFaceIDUsageDescription` to Info.plist when using Face ID
- [ ] Derive keys from passwords with HKDF — never use raw passwords as keys
- [ ] Clear tokens and cached data completely on logout

## Mistakes to Avoid

- [ ] Storing tokens, passwords, or API keys in `UserDefaults`
- [ ] Hardcoding secrets in source code, plists, or xcconfig files
- [ ] Ignoring `OSStatus` return values from Keychain operations
- [ ] Using `LAContext.evaluatePolicy()` as a standalone security gate
- [ ] Setting both `kSecAttrAccessControl` and `kSecAttrAccessible` on the same item
- [ ] Omitting `kSecAttrAccessible` from Keychain queries (undefined default behavior)
- [ ] Running Keychain operations on the main thread (blocks UI, especially with biometrics)
- [ ] Using deprecated `CC_SHA256` / CommonCrypto when CryptoKit is available
- [ ] Storing raw passwords instead of using token-based authentication
- [ ] Assuming Secure Enclave is always available (check `SecureEnclave.isAvailable`)
- [ ] Using `kSecAttrAccessibleAlways` (deprecated since iOS 12)
- [ ] Forgetting to invalidate tokens server-side on logout
