To protect credit card information in a Java application while meeting Payment Card Industry Data Security Standard (PCI DSS) requirements, you need to implement strong cryptographic algorithms, appropriate key lengths, and secure parameters. 
Below, I’ll outline recommendations based on PCI DSS guidelines (specifically PCI DSS v4.0 as of March 31, 2025) and cryptographic best practices suitable for a Java environment.

## General PCI DSS Cryptography Requirements
PCI DSS mandates the use of strong cryptography to protect sensitive cardholder data (e.g., Primary Account Number - PAN) during transmission and storage. 

**Key requirements include:**
- Use of industry-standard algorithms (e.g., AES, RSA).
- Adequate key lengths (e.g., minimum 128-bit symmetric keys, 2048-bit asymmetric keys).
- Secure key management (generation, storage, rotation, etc.).
- Avoidance of deprecated or weak algorithms (e.g., DES, 3DES, MD5, SHA-1).

### Example Cryptographic Algorithms and Parameters
- Here’s a breakdown of algorithms and settings you can implement in Java:

#### Symmetric Encryption (for Data at Rest)**
* **Algorithm:** AES (Advanced Encryption Standard)
  * **Mode:** GCM (Galois/Counter Mode) for authenticated encryption or CBC (Cipher Block Chaining) with proper padding.
  * **Key Length:** 256 bits (preferred for future-proofing; 128 bits is minimum acceptable).
  * **Parameters:**
    * For GCM: Use a 96-bit nonce (randomly generated, never reused with the same key).
    * For CBC: Use a random Initialization Vector (IV) for each encryption.
  * **Why:** AES is FIPS 140-3 compliant and widely accepted by PCI DSS.
  * **Java Implementation:** Use javax.crypto.Cipher with "AES/GCM/NoPadding" or "AES/CBC/PKCS5Padding".
 
**Example Code (AES-GCM):**
```java
import javax.crypto.Cipher;
import javax.crypto.Mac;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import java.security.SecureRandom;
import java.util.Arrays;

public byte[] encrypt(byte[] plaintext, byte[] encryptionKey, byte[] hmacKey) throws Exception {
    byte[] iv = new byte[16];
    new SecureRandom().nextBytes(iv);

    // AES-CBC Encryption
    Cipher cipher = Cipher.getInstance(AES_CBC_PKCS5);
    cipher.init(Cipher.ENCRYPT_MODE, new SecretKeySpec(encryptionKey, "AES"), new IvParameterSpec(iv));
    byte[] ciphertext = cipher.doFinal(plaintext);

    // Compute HMAC-SHA256 of (IV + Ciphertext)
    Mac hmac = Mac.getInstance(HMAC_SHA256);
    hmac.init(new SecretKeySpec(hmacKey, HMAC_SHA256));
    hmac.update(iv);
    byte[] hmacDigest = hmac.doFinal(ciphertext);

    // Combine IV + Ciphertext + HMAC
    byte[] result = new byte[iv.length + ciphertext.length + hmacDigest.length];
    System.arraycopy(iv, 0, result, 0, iv.length);
    System.arraycopy(ciphertext, 0, result, iv.length, ciphertext.length);
    System.arraycopy(hmacDigest, 0, result, iv.length + ciphertext.length, hmacDigest.length);

    return result;
}
```

**Developer pain points:**
- 🚫 **Too many new concepts to learn:** Need to understand different ciphers, relevant parameters and there meanings
- 🚫 **Things can get obsolete or just change:** Algorithms can get deprecated and thus need to be continously updated. This would also mean regular security patches
- 🚫 **Sensitive assets sprawl:** Sensitive data is processed at different places with different type of sensitive info requiring different ciphers
- 🚫 **Delayed CI/CD:** Frequent updates mean frequent releases and thus running CI/CD cycles everytime 

#### Key Management
* **Key Generation:** Use java.security.SecureRandom for random key and nonce generation.
* **Key Storage:** Store keys in a secure keystore (e.g., Java KeyStore - JKS or PKCS12) or a Hardware Security Module (HSM) for PCI compliance.
* **Key Rotation:** Rotate keys periodically (e.g., annually) and re-encrypt data as required by PCI DSS Requirement 3.6.

**Example Code (Key Generation):**
```java
import javax.crypto.KeyGenerator;
import java.security.SecureRandom;

public byte[] generateAESKey() throws Exception {
    KeyGenerator keyGen = KeyGenerator.getInstance("AES");
    SecureRandom random = new SecureRandom();
    keyGen.init(256, random); // 256-bit key
    return keyGen.generateKey().getEncoded();
}
```

**Developer pain points:**
- 🚫 Key generation locally
- 🚫 Key can get compromised, so regular rotation is needed
- 🚫 Multiple keys are employed for multiple data points

#### PCI DSS Compliance Notes

* **Requirement 3:** Protect stored cardholder data with encryption (AES-256 recommended).
* **Requirement 4:** Encrypt transmission of cardholder data over public networks (e.g., use TLS 1.3 with strong ciphers).
* **Requirement 6:** Avoid vulnerabilities by using only supported algorithms and libraries (e.g., ensure Java version is up-to-date).
* **Avoid Deprecated Algorithms:** Do not use DES, 3DES (except in limited legacy cases), or RSA with PKCS#1 v1.5 padding.

## What's next
With the above examples, it is evident that developer working with crypto in Java need to understand cryptography operations in details as well as keep the code and the keys updated to ensure future risks are covered. In the next section we will see how using policies on CipherTrust Manager and everaging CRDP APIs can offload this work.
* [Simplify this with CRDP APIs and CM Policies](../chapters/102-javax-crypto-vs-crdp.md)

## Revisit
* [Securing Credit Card Data in Java using CRDP](../chapters/01-securing-credit-card-java-crdp.md)
