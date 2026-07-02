# aes-encryption-practice
Built a browser-based AES-256-GCM encryption tool using the Web Crypto API to protect sensitive HR records stored locally. No libraries, no server — just JavaScript and real cryptography.

# AES-256-GCM Encryption Implementation
### Yesenia Requena  July 2026

---

## Overview

This project implements AES-256-GCM symmetric encryption using the browser-native Web Crypto API in a standalone HTML file. The goal was to build a foundational encryption layer that could be applied to a sensitive HR compliance tool (workplace investigation tracker) storing personally identifiable information (PII) and protected employee records locally on a device.

No external libraries were used. All cryptographic operations are performed by the browser's built-in `crypto.subtle` API.

---

## Why AES-256-GCM

| Property | Detail |
|---|---|
| Algorithm | AES — Advanced Encryption Standard |
| Key length | 256 bits |
| Mode | GCM — Galois/Counter Mode |
| Classification | Symmetric encryption |
| Standard | NIST FIPS 197, SP 800-38D |

**AES-256** was selected because it is the gold standard for data at rest encryption. It is used by the U.S. government for classified information, by financial institutions, and is recommended by NIST for sensitive data protection.

**GCM mode** was selected over other AES modes (CBC, CTR) because it provides both **confidentiality** (encrypts the data) and **authenticity** (detects tampering). If the data is modified or the wrong password is used, GCM throws a hard error rather than returning corrupted data silently. This is critical for legal recordkeeping integrity.

**Security+ SY0-701 Alignment:**
- Domain 3.1 — Cryptographic algorithms
- Domain 3.2 — Key management (PBKDF2, salt, IV)
- Domain 3.3 — Symmetric vs asymmetric encryption
- Domain 3.5 — Protecting data at rest

---

## Architecture

### Three Core Functions

#### 1. `getKey(password, salt)`
Converts a human-readable password into a cryptographically strong 256-bit AES key using PBKDF2 (Password-Based Key Derivation Function 2).

```javascript
async function getKey(password, salt) {
  const keyMaterial = await crypto.subtle.importKey(
    "raw",
    new TextEncoder().encode(password),
    { name: "PBKDF2" },
    false,
    ["deriveKey"]
  );
  return crypto.subtle.deriveKey(
    {
      name: "PBKDF2",
      salt: salt,
      iterations: 100000,
      hash: "SHA-256"
    },
    keyMaterial,
    { name: "AES-GCM", length: 256 },
    false,
    ["encrypt", "decrypt"]
  );
}
```

**Why 100,000 iterations?**
Each password guess requires running PBKDF2 100,000 times. On modern hardware this takes approximately 1 second per guess, making brute-force attacks computationally impractical. A billion guesses would take ~30 years.

**Input:** password (string) + salt (16 random bytes)
**Output:** AES-256-GCM CryptoKey object

---

#### 2. `encryptData()`
Generates random salt and IV, derives the encryption key, encrypts the plaintext, and bundles all components into a single base64 string.

```javascript
async function encryptData() {
  const salt = crypto.getRandomValues(new Uint8Array(16));
  const iv   = crypto.getRandomValues(new Uint8Array(12));
  const key  = await getKey(password, salt);

  const encrypted = await crypto.subtle.encrypt(
    { name: "AES-GCM", iv: iv },
    key,
    new TextEncoder().encode(plaintext)
  );

  // Bundle: [salt 16 bytes][IV 12 bytes][encrypted data]
  const bundle = new Uint8Array(16 + 12 + encryptedArray.byteLength);
  bundle.set(salt, 0);
  bundle.set(iv, 16);
  bundle.set(encryptedArray, 28);

  return btoa(String.fromCharCode(...bundle));
}
```

**Bundle structure:**

```
| salt (16 bytes) | IV (12 bytes) | encrypted data (n bytes) |
| offset 0        | offset 16     | offset 28                |
```

The salt and IV are not secret and are stored with the encrypted data. The password is never stored.

---

#### 3. `decryptData()`
Unpacks the bundle, re-derives the same key using the stored salt and the user's password, and decrypts the data. GCM authentication automatically rejects wrong passwords or tampered data.

```javascript
async function decryptData() {
  const bundle    = Uint8Array.from(atob(base64), c => c.charCodeAt(0));
  const salt      = bundle.slice(0, 16);
  const iv        = bundle.slice(16, 28);
  const encrypted = bundle.slice(28);

  const key = await getKey(password, salt);

  try {
    const decrypted = await crypto.subtle.decrypt(
      { name: "AES-GCM", iv: iv },
      key,
      encrypted
    );
    return new TextDecoder().decode(decrypted);
  } catch (e) {
    return "Decryption failed - wrong password or data was tampered with.";
  }
}
```

---

## Key Concepts

### What is stored vs. what is secret

| Component | Stored | Secret |
|---|---|---|
| Salt | ✅ Yes — in bundle | ❌ No |
| IV (Initialization Vector) | ✅ Yes — in bundle | ❌ No |
| Encrypted data | ✅ Yes — in bundle | ❌ No |
| Encryption key | ❌ Never stored | ✅ Yes — derived at runtime |
| Password | ❌ Never stored | ✅ Yes — user provides |

### Why a random IV matters
A new random IV is generated for every encryption. This means encrypting the same message twice with the same password produces completely different ciphertext. This prevents pattern analysis attacks.

### GCM Authentication
GCM mode appends a 128-bit authentication tag to the ciphertext. On decryption, this tag is verified before any data is returned. If verification fails — due to wrong password or data tampering — the operation throws an error. This is called **Authenticated Encryption with Associated Data (AEAD)**.

---

## Troubleshooting Log

### Issue 1 — File displaying as code instead of rendering
**Symptom:** Chrome showed raw HTML code as text instead of rendering the page.
**Cause:** TextEdit saved the file as `.html.txt` instead of `.html` — the `.txt` extension caused Chrome to treat it as plain text.
**Resolution:** Disabled "Add .txt extension to plain text files" in TextEdit Settings → Open and Save. Verified file extension was `.html` only.

---

### Issue 2 — Encrypt button produced no output and no visible error
**Symptom:** Clicking Encrypt did nothing — no output, no alert.
**Diagnosis:** Opened Chrome DevTools (Command + Option + J) → Console tab → called `encryptData()` manually.
**Error returned:**
```
Uncaught (in promise) RangeError: offset is out of bounds
at Uint8Array.set (<anonymous>)
at encryptData (aes-practical.html:78:10)
```
**Cause:** The `encrypted` ArrayBuffer was being passed directly to `Uint8Array.set()` without first converting it to a `Uint8Array`. The `.byteLength` calculation was also referencing the raw ArrayBuffer instead of the typed array.
**Resolution:**
```javascript
// Before (broken)
const bundle = new Uint8Array(salt.byteLength + iv.byteLength + encrypted.byteLength);
bundle.set(new Uint8Array(encrypted), salt.byteLength + iv.byteLength);

// After (fixed)
const encryptedArray = new Uint8Array(encrypted);
const bundle = new Uint8Array(16 + 12 + encryptedArray.byteLength);
bundle.set(encryptedArray, 28);
```

---

### Issue 3 — TypeError: Constructor Uint8Array requires 'new'
**Symptom:** New error in console after fixing Issue 2.
**Error:**
```
TypeError: Constructor Uint8Array requires 'new'
at Uint8Array (<anonymous>)
at encryptData (aes-practical.html:77:26)
```
**Cause:** Missing `new` keyword before `Uint8Array(encrypted)`.
**Resolution:** Added `new` keyword:
```javascript
const encryptedArray = new Uint8Array(encrypted);
```

---

### Issue 4 — Decryption failing with correct password
**Symptom:** Encrypt produced output but Decrypt returned "Decryption failed" even with the correct password.
**Cause:** TextEdit's autocorrect silently replaced straight quotes `"` with curly/smart quotes `"` `"` and converted hyphens to em-dashes `–`. JavaScript requires straight ASCII quotes and standard hyphens. Curly quotes cause silent syntax corruption that doesn't always show as an error but breaks string parsing.
**Resolution:** Downloaded clean file generated without TextEdit. Confirmed TextEdit is not suitable for writing JavaScript due to autocorrect interference. Recommend VS Code for all future code editing.

---

### Issue 5 — Wrong password test
**Symptom (expected):** Changing one character of the password caused decryption to fail.
**Result:** "Decryption failed - wrong password or data was tampered with."
**Explanation:** PBKDF2 with a different password produces a completely different 256-bit key. AES-GCM authentication tag verification fails, throwing a DOMException. The `catch` block handles this gracefully. This confirmed the encryption is working correctly — even a one-character difference in the password makes the data permanently unreadable.

---

## Security Considerations

**Strengths of this implementation:**
- AES-256 with GCM mode provides both confidentiality and integrity
- PBKDF2 with 100,000 iterations and random salt prevents brute-force and rainbow table attacks
- Random IV per encryption prevents ciphertext pattern analysis
- No server, no network, no third party — data never leaves the device
- Web Crypto API is a browser-native, audited cryptographic implementation

**Limitations and recommendations:**
- Password management is the responsibility of the user — a forgotten password means permanent data loss
- localStorage can be accessed by any script running on the same origin — for maximum security, use encrypted file export rather than localStorage alone
- This implementation is appropriate for single-user local storage of sensitive HR records, not for multi-user or networked environments
- For networked or cloud environments, consider a dedicated secrets manager (HashiCorp Vault, AWS KMS)

---

## Application to HR Compliance

This encryption implementation was built specifically to protect a workplace investigation tool containing:
- Complainant and respondent PII
- Complaint descriptions and witness names
- Investigation findings and disciplinary actions
- Electronic signatures

Under NYC Human Rights Law, these records must be maintained for a minimum of 3-7 years after an investigation or until the statute of limitations for potential lawsuits has expired. Encrypting them at rest ensures that even if the device is lost, stolen, or accessed without authorization, the records remain protected.

The same encryption pattern can be applied to:
- Fair Workweek compliance records
- Protected leave tracking
- Candidate interview scoring data

---

## Files

| File | Description |
|---|---|
| `aes_practice.html` | Standalone AES-256-GCM encryption practice tool |
| `CFA_Investigation_Tool.html` | Workplace investigation tracker (encryption integration pending) |

---

## Next Steps

- [ ] Integrate AES-256-GCM into investigation tool
- [ ] Add password prompt modal on tool open
- [ ] Add auto-lock after inactivity timeout
- [ ] Add encrypted backup export function
- [ ] Apply same pattern to Fair Workweek compliance tool

---

*Built as part of cybersecurity portfolio development*
