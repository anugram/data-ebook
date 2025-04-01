# Problem Statement
Protecting credit card data is critical due to its sensitive nature and regulatory requirements like **PCI-DSS (Payment Card Industry Data Security Standard)**. 
Below are some of the commonly recommended techniques, algorithms, and parameters to secure such data:

## 1. Encryption (At Rest & In Transit)
**Recommended Algorithms & Parameters:**

- **Symmetric Encryption (for storage & high-speed encryption):**
  - **AES-256** (Advanced Encryption Standard)
    - Mode:
      - **GCM (Galois/Counter Mode)** (provides authenticated encryption)
      - **CBC (Cipher Block Chaining)**
    - Key Size: **256-bit** (PCI-DSS recommends at least 128-bit, but 256 is stronger)
    - **PKCS Padding**
    - IV (Initialization Vector): **Randomly generated per encryption**

- **Asymmetric Encryption (for secure key exchange):**
  - **RSA** (3072-bit or 4096-bit keys)
  - **Elliptic Curve Cryptography (ECC)** (e.g., ECDSA with secp384r1 curve)

- **TLS for Data in Transit:**
  - **Use TLS 1.2 or 1.3** (PCI-DSS mandates disabling TLS 1.0 & 1.1)
  - Cipher Suites: Prefer **AES-GCM, ChaCha20-Poly1305, ECDHE for key exchange**

## 2. Tokenization (Replace Sensitive Data with Tokens)
- **Use Format-Preserving Encryption (FPE) or Vault-Based Tokenization**
- Example:
  - Original: 4111 1111 1111 1111 → Token: tok_4x12a9b7e3f8c2d1
- **Benefits**: Reduces PCI-DSS scope since tokens are useless if stolen.

## 3. Masking (Display Protection)
- **Partial Masking (for displaying only last 4 digits):**
  - Example: **** **** **** 1111
- **Algorithm**: Simple string manipulation (no cryptographic strength needed).

## 4. Key Management (Critical for Encryption)
- **HSM (Hardware Security Module)** for storing encryption keys.
- **Key Rotation Policy:**
  - **AES keys**: Rotate every **1-2 years** (or per PCI-DSS requirements).
  - **RSA keys**: Rotate every **2-3 years**.
- Use Key Derivation Functions (KDF):
  - HKDF (HMAC-based Extract-and-Expand Key Derivation Function)

## 5. Access Control & Monitoring
- **Role-Based Access Control (RBAC)**: Only authorized personnel should access raw data.
- Audit Logs: Log all access to cardholder data (PCI-DSS Requirement 10).
- Multi-Factor Authentication (MFA) for database/API access.

## 6. Data Minimization & Retention Policies
- PCI-DSS Requirement 3.2: Do not store sensitive authentication data (CVV, full track data) after authorization.
- Automated Deletion: Delete unnecessary card data after a set period.

# Using CipherTrust Data Security Platform as a solution
If using CipherTrust Data Security Platform, you can offload some of the above challanges.

## 1. Using Symmetric Encryption for Data Protection
![image](https://github.com/user-attachments/assets/80256d21-afed-49f5-8992-eaf5052131ae)
- **Algorithm:** AES/CBC/PKCS5Padding
- **Key:** AES 256 bit
- **IV:** 16 bytes

## 2. Format Preserving Encryption and 3. Masking
![image](https://github.com/user-attachments/assets/4873df15-dbca-43cc-8258-9f431e56c623)
- **Algorithm:** FPE/FF3-1
- **Key:** AES 256 bit
- **Masking Format:** Show_Last_Four
- **IV:** 16 bytes
- **Tweak Algo**: SHA-256

## 4. Key Management
Keys on CipherTrust Manager can be backed by HSM
Key rotation policy can be defined on CM and connectors like CipherTrust RESTful Data Protection work with CM to work with latest keys

## 5. Access Control via Access Policies
![image](https://github.com/user-attachments/assets/c0f91b37-c2ad-412c-b6f5-c2d0e2687b47)
- Users in userset masked will see only last 4 digits in plaintext
- Users in userset plaintext will see all digits in plaintext
- All other users will see all digits in CipherText format
