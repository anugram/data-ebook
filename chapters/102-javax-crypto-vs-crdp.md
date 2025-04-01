# Simplifying Crypto with Thales CipherTrust RESTful Data Protection APIs

Thales CipherTrust RESTful Data Protection or CRDP simplifies the Data Protection by offloading the crypto settings and configuration to security administrators while developers focusing on calling simple PROTECT and REVEAL APIs to mask or redact the sensitive data.
Static Data Masking masks the data that doesn’t need to be accessed, revealing only the data relevant to the group – this is especially valuable for data analysts (pseudonymization) and customer service (no calls to Protect/Reveal).
Dynamic Data Masking is preferred when users with different access levels will be accessing the data and have rights to access different parts of the data. Redaction provides an absolute mask that shows a field with redacted data or hides the field.

And best part is all of this is abstracted away in just two API calls i.e. PROTECT and REVEAL.

## Protection and Access policy in CipherTrust Manager
Define policy on CipherTrust Manager as an [example here](https://github.com/anugram/data-ebook/blob/main/chapters/UC001-credit-card-protection.md).

## Impact to development, QA, and OPs team
### Replace complex crypto code to simple API calls
As a developer, the **code to maintain reduces** from 
```java
// Encrypt data
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
    return result;
}
```
```java
// Key generation
public byte[] generateAESKey() throws Exception {
    KeyGenerator keyGen = KeyGenerator.getInstance("AES");
    SecureRandom random = new SecureRandom();
    keyGen.init(256, random); // 256-bit key
    return keyGen.generateKey().getEncoded();
}
```
**to a simple API call** i.e.
```java
CrdpProtectRequest requestPayload = new CrdpProtectRequest(policyName, dataToProtect);
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create(CRDP_BASE_URL + PROTECT_ENDPOINT))
    .timeout(Duration.ofSeconds(20))
    .header("Content-Type", "application/json")
    .header("Accept", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString(jsonRequestBody))
    .build();
HttpResponse<String> response = httpClient.send(request, HttpResponse.BodyHandlers.ofString());
```
**Changes at high level**:
| Without CRDP | With CRDP |
|--------------|-----------|
| Define crypto specifications | Crypto specifications abstarcted within Protection and Access Policy |
| Define Crypto algo | Call PROTECT API |
| Define IV | |
| Define HMAC | |
| Create and manage keys | |
| Define data access and visibility rules | Call REVEAL API |

### Subsequent updates
What are the possible updates
- Key need to be rotated
- Key compromised and need to be replaced
- Crypto algorithm is made obsolete

Updates are required at every place keys are being generated or at every place data is being encrypted and decrypted
In case of CRDP, the policy shall be updated on CipherTrust Manager and application code need not change.
