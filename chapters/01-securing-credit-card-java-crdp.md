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
