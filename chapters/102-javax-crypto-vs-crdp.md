# Simplifying Crypto with Thales CipherTrust RESTful Data Protection APIs

Thales CipherTrust RESTful Data Protection or CRDP simplifies the Data Protection by offloading the crypto settings and configuration to security administrators while developers focusing on calling simple PROTECT and REVEAL APIs to mask or redact the sensitive data.
Static Data Masking masks the data that doesn’t need to be accessed, revealing only the data relevant to the group – this is especially valuable for data analysts (pseudonymization) and customer service (no calls to Protect/Reveal).
Dynamic Data Masking is preferred when users with different access levels will be accessing the data and have rights to access different parts of the data. Redaction provides an absolute mask that shows a field with redacted data or hides the field.

And best part is all of this is abstracted away in just two API calls i.e. PROTECT and REVEAL.

## Protection and Access policy in CipherTrust Manager
Define policy on CipherTrust Manager as an [example here](https://github.com/anugram/data-ebook/blob/main/chapters/UC001-credit-card-protection.md).

