# Chapter 1: Securing Credit Card Data in a Java Application using CipherTrust RESTful Data Protection APIs

## The Scenario: Processing Payments

Imagine a common scenario: a Java-based e-commerce platform or service application needs to process customer payments. This involves accepting credit card details (Cardholder Name, Primary Account Number (PAN), Expiry Date, CVV) and potentially storing some of this information for recurring billing or user convenience.

## The Problem: A Flawed, High-Risk Approach

A naive or legacy implementation might handle this sensitive data directly within the Java application code and store it, often insecurely, in the application's database.

**Example (Conceptual - Illustrating Bad Practice):**

```java
// WARNING: Insecure approach - DO NOT use in production!
public class InsecurePaymentService {

    private DatabaseConnection dbConnection; // Assume DB connection exists

    public void processPayment(String cardNumber, String expiry, String cvv, String cardHolderName) {
        // 1. Send card details to a payment gateway (potentially insecurely)
        boolean paymentSuccess = callPaymentGateway(cardNumber, expiry, cvv);

        if (paymentSuccess) {
            // 2. Store sensitive data directly in the application database
            // THIS IS A MAJOR PCI DSS VIOLATION AND SECURITY RISK
            String sql = "INSERT INTO UserPayments (userId, full_card_number, expiry_date, card_holder) VALUES (?, ?, ?, ?)";
            dbConnection.execute(sql, getCurrentUserId(), cardNumber, expiry, cardHolderName);

            // 3. Log sensitive data (another common mistake)
            System.out.println("Processed payment for card: " + cardNumber);
        }
    }
    // ... other methods
}
```

### Why This is Bad

1. **Direct PCI DSS Violations:**
  * **Requirement 3 (Protect stored cardholder data):** Storing the full PAN (Primary Account Number) after authorization is generally prohibited unless essential for a specific business function, and even then, it *must* be rendered unreadable (e.g., using strong encryption, truncation, tokenization). Storing CVV after authorization is *never* allowed.
  * **Requirement 6 (Develop and maintain secure systems and applications):** This approach likely violates secure coding guidelines.
  * **Requirement 10 (Track and monitor all access):** Logging sensitive data like the full PAN makes logs highly sensitive and difficult to manage securely.

2. **Increased PCI Scope:** Any system that stores, processes, or transmits cardholder data falls within the scope of a PCI DSS audit. This insecure approach dramatically increases the scope, complexity, and cost of achieving and maintaining compliance. Every component touching this data needs to be secured and audited.

3. **Massive Security Risk:** If the application server or database is compromised, attackers gain direct access to raw credit card numbers. This leads to:
    * Significant financial loss for customers.
    * Severe reputational damage to the business.
    * Large fines and legal liabilities.
    * Potential loss of the ability to process credit card payments.

4. **In-House Complexity:** Implementing and managing secure encryption, key management, and access controls correctly is extremely difficult and error-prone.

### The Solution: Offloading Security with CRDP
Instead of handling raw sensitive data, the application can leverage CRDP to protect the data before it's stored or potentially even before it's transmitted internally. CRDP takes the sensitive data and returns a non-sensitive token or encrypted blob, based on a defined policy.

**Implementation with CRDP API:**

The core idea is to intercept the credit card data within the application, send it to CRDP for protection, and then store the resulting token (or encrypted data) instead of the original PAN.

Here's how the application would call the CRDP API:
```bash
# Example API Call to protect a credit card number

curl <YOUR_CRDP_IP>:32082/v1/protect -X POST \
-H "Content-Type: application/json" \
-d '{
  "protection_policy_name": "protect-credit-card",
  "data": "4929123456789012" # The actual PAN to protect
}'

# Expected Response (Example - structure may vary)
# {
#   "protected_data": "tkn_live_abcdef12345EXAMPLETOKEN"
# }
```
**Explanation:**

* **<YOUR_CRDP_IP>:** The IP address or hostname where your CRDP service is accessible.
* **/v1/protect:** The API endpoint for data protection operations.
* **"protection_policy_name":** "protect-credit-card": This tells CRDP how to protect the data. This policy would be pre-configured within CRDP to perform specific actions like:
    * Tokenization: Replace the PAN with a non-sensitive token. CRDP securely stores the original PAN mapped to the token.
    * Encryption: Encrypt the PAN using strong, managed cryptographic keys.
    * (The policy also define format preservation, role based access control, etc.)
* **"data": "4929...":** The sensitive payload – the credit card number.

**Revised Application Flow:**
```java
// Secure approach using CRDP
public class SecurePaymentService {

    private DatabaseConnection dbConnection;
    private CrdpApiClient crdpClient; // Client to interact with CRDP API

    public void processPayment(String cardNumber, String expiry, String cvv, String cardHolderName) {

        // Assume payment gateway call happens first (ideally without logging raw data)
        boolean paymentSuccess = callPaymentGateway(cardNumber, expiry, cvv);

        if (paymentSuccess) {
            // PROTECT the sensitive data using CRDP
            String protectedCardData = crdpClient.protectData("protect-credit-card", cardNumber);
            // Note: CVV is typically NOT stored after authorization.
            // Expiry and Cardholder Name might be stored directly if needed,
            // or also protected depending on requirements.

            // Store the PROTECTED data (token or encrypted blob)
            String sql = "INSERT INTO UserPayments (userId, protected_card_info, expiry_date, card_holder) VALUES (?, ?, ?, ?)";
            dbConnection.execute(sql, getCurrentUserId(), protectedCardData, expiry, cardHolderName);

            // Log only non-sensitive information
            System.out.println("Processed payment using protected card data: " + protectedCardData);
        }
    }
    // ... other methods
}
```

**Benefits of Using CRDP**
1. **Dramatically Reduced PCI Scope:** Since the Java application no longer stores, processes, or transmits the raw PAN (it only handles the token/encrypted data from CRDP), many parts of the application and its infrastructure may be removed from PCI DSS scope. This simplifies audits and reduces compliance costs significantly.
2. **Enhanced Security:** The actual PAN is vaulted securely within CRDP (in case of tokenization) or strongly encrypted using centrally managed keys. A compromise of the application database no longer exposes usable card numbers.
3. **Simplified Compliance:** Helps directly meet PCI DSS requirements like Req 3.4 (render PAN unreadable wherever it is stored). CRDP handles the complexities of secure storage, encryption, and key management.
4. **Reduced Risk:** Minimizes the impact of a potential data breach within the application environment.
5. **Centralized Policy Management:** Security policies (how data is tokenized/encrypted, who can access it) are managed within CRDP, not scattered across application code.
6. **Developer Productivity:** Developers don't need to be crypto experts. They integrate with a simple API, allowing them to focus on business logic.

### Conclusion
By integrating CRDP via its API, the Java application transforms from a high-risk system struggling with PCI compliance into a more secure and compliant solution. It effectively outsources the most critical data security functions for handling credit card information, leading to reduced scope, lower risk, and simplified development.

### Annexure - Sample implementation of CrdpApiClient
```java
import com.fasterxml.jackson.annotation.JsonProperty;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.io.IOException;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;

public class CrdpApiClient {

    // Configure your CRDP service location
    private static final String CRDP_BASE_URL = "http://<YOUR_CRDP_IP>:32082"; // Replace <YOUR_CRDP_IP>
    private static final String PROTECT_ENDPOINT = "/v1/protect";

    // Reusable HttpClient and ObjectMapper instances
    private final HttpClient httpClient;
    private final ObjectMapper objectMapper;

    public CrdpApiClient() {
        this.httpClient = HttpClient.newBuilder()
                .version(HttpClient.Version.HTTP_1_1) // Or HTTP_2 if supported
                .connectTimeout(Duration.ofSeconds(10))
                .build();
        this.objectMapper = new ObjectMapper(); // Jackson's JSON handler
    }

    // --- Inner classes to represent JSON request and response ---

    // Represents the JSON request body for the /v1/protect endpoint
    private static class CrdpProtectRequest {
        @JsonProperty("protection_policy_name") // Maps Java field to JSON field name
        public String protectionPolicyName;
        public String data; // Jackson maps this to "data" JSON field automatically

        public CrdpProtectRequest(String protectionPolicyName, String data) {
            this.protectionPolicyName = protectionPolicyName;
            this.data = data;
        }
    }

    // Represents the JSON response body
    private static class CrdpProtectResponse {
        @JsonProperty("protected_data")
        public String protectedData;
        // Add other fields if your API returns more information
    }

    // --- Method to call the CRDP Protect API ---

    /**
     * Calls the CRDP /v1/protect API to protect sensitive data.
     *
     * @param policyName The name of the protection policy configured in CRDP.
     * @param dataToProtect The sensitive data string to protect.
     * @return The protected data (e.g., token or encrypted string) returned by CRDP.
     * @throws IOException If there's a network or communication error.
     * @throws InterruptedException If the request is interrupted.
     * @throws CrdpApiException If CRDP returns an error status code.
     * @throws JsonProcessingException If there's an issue processing JSON.
     */
    public String protectData(String policyName, String dataToProtect)
            throws IOException, InterruptedException, CrdpApiException {

        // 1. Create the request body object
        CrdpProtectRequest requestPayload = new CrdpProtectRequest(policyName, dataToProtect);

        // 2. Serialize the request body object to a JSON string
        String jsonRequestBody;
        try {
            jsonRequestBody = objectMapper.writeValueAsString(requestPayload);
        } catch (JsonProcessingException e) {
            // Handle JSON serialization error (shouldn't typically happen with simple objects)
            System.err.println("Error serializing request payload: " + e.getMessage());
            throw e; // Re-throw or handle more gracefully
        }

        // 3. Build the HTTP Request
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(CRDP_BASE_URL + PROTECT_ENDPOINT))
                .timeout(Duration.ofSeconds(20)) // Set a request timeout
                .header("Content-Type", "application/json")
                .header("Accept", "application/json") // Optional: Specify expected response type
                .POST(HttpRequest.BodyPublishers.ofString(jsonRequestBody))
                .build();

        // 4. Send the request and get the response
        HttpResponse<String> response = httpClient.send(request, HttpResponse.BodyHandlers.ofString());

        // 5. Check the response status code
        int statusCode = response.statusCode();
        if (statusCode >= 200 && statusCode < 300) { // Success range (e.g., 200 OK, 201 Created)
            // 6. Deserialize the JSON response body to our response object
            try {
                CrdpProtectResponse crdpResponse = objectMapper.readValue(response.body(), CrdpProtectResponse.class);
                return crdpResponse.protectedData;
            } catch (JsonProcessingException e) {
                System.err.println("Error deserializing successful response body: " + e.getMessage());
                System.err.println("Response Body: " + response.body());
                throw e; // Re-throw or handle
            }
        } else {
            // Handle error response
            String errorMessage = String.format("CRDP API Error: Received status code %d. Response Body: %s",
                    statusCode, response.body());
            System.err.println(errorMessage);
            throw new CrdpApiException(errorMessage, statusCode, response.body());
        }
    }

    // --- Custom Exception Class for API Errors ---
    public static class CrdpApiException extends Exception {
        private final int statusCode;
        private final String responseBody;

        public CrdpApiException(String message, int statusCode, String responseBody) {
            super(message);
            this.statusCode = statusCode;
            this.responseBody = responseBody;
        }

        public int getStatusCode() {
            return statusCode;
        }

        public String getResponseBody() {
            return responseBody;
        }
    }


    // --- Example Usage ---
    public static void main(String[] args) {
        // IMPORTANT: Replace with your actual CRDP IP address in CRDP_BASE_URL constant

        CrdpApiClient client = new CrdpApiClient();
        String creditCard = "4929123456789012"; // Example sensitive data
        String policy = "protect-credit-card";

        try {
            System.out.println("Sending data to CRDP for protection...");
            String protectedToken = client.protectData(policy, creditCard);
            System.out.println("Successfully protected data.");
            System.out.println("Original Data: " + creditCard);
            System.out.println("Policy Used:   " + policy);
            System.out.println("Protected Data (Token/Ciphertext): " + protectedToken);

            // Now you would store 'protectedToken' in your database instead of 'creditCard'

        } catch (JsonProcessingException e) {
            System.err.println("JSON Error: " + e.getMessage());
        } catch (CrdpApiException e) {
            System.err.println("CRDP API failed with status " + e.getStatusCode());
            System.err.println("Response: " + e.getResponseBody());
        } catch (IOException e) {
            System.err.println("Network/IO Error: " + e.getMessage());
        } catch (InterruptedException e) {
            System.err.println("Request Interrupted: " + e.getMessage());
            Thread.currentThread().interrupt(); // Restore interruption status
        }
    }
}
```

**Explanation:**
1. **Dependencies:** Make sure Jackson is included.
2. **Constants:** CRDP_BASE_URL and PROTECT_ENDPOINT define where to send the request. Remember to replace <YOUR_CRDP_IP>.
3. **HttpClient & ObjectMapper:** Instances are created in the constructor. These are generally thread-safe and can be reused.
4. **POJOs (Plain Old Java Objects):** CrdpProtectRequest and CrdpProtectResponse map directly to the JSON structure. @JsonProperty is used when the Java field name doesn't exactly match the JSON key name (like protectionPolicyName vs protection_policy_name).
5. **protectData Method:**
    * Takes the policy name and sensitive data as input.
    * Creates the CrdpProtectRequest object.
    * Uses objectMapper.writeValueAsString to convert the Java object into a JSON string.
    * Builds an HttpRequest specifying the URI, timeout, headers (Content-Type), and the POST method with the JSON body (BodyPublishers.ofString).
    * Sends the request synchronously using httpClient.send. You could use sendAsync for non-blocking operations.
    * Checks the HTTP status code. Success is typically 2xx.
    * If successful, it uses objectMapper.readValue to parse the JSON response body string into the CrdpProtectResponse object and returns the protectedData.
    * If not successful, it prints an error and throws a custom CrdpApiException.
6. **Error Handling:** Basic error handling for JSON processing, network issues (IOException), interruption (InterruptedException), and API errors (non-2xx status codes) is included.
7. **main Method:** Provides a simple example of how to instantiate the client and call the protectData method.
