# PCI Data Protection in Golang: Plaintext vs CipherTrust RESTful Data Protection API

## Database Operations Comparison

### 1. Insecure Approach (Storing Raw PCI Data)

```go
package main

import (
	"database/sql"
	"fmt"
	"log"
	_ "github.com/mattn/go-sqlite3" // SQLite driver
)

type Payment struct {
	Name        string
	CardNumber  string // Plaintext PAN - PCI violation
	Expiry      string // Plaintext expiry - sensitive
	Amount      float64
}

func initDB() *sql.DB {
	db, err := sql.Open("sqlite3", "insecure_payments.db")
	if err != nil {
		log.Fatal(err)
	}
	
	_, err = db.Exec(`
		CREATE TABLE IF NOT EXISTS payments (
			id INTEGER PRIMARY KEY,
			name TEXT,
			card_number TEXT,  // Storing raw PAN - PCI violation!
			expiry TEXT,       // Raw expiry - sensitive
			amount REAL
		)`)
	if err != nil {
		log.Fatal(err)
	}
	return db
}

func storePayment(db *sql.DB, p Payment) {
	_, err := db.Exec(
		"INSERT INTO payments (name, card_number, expiry, amount) VALUES (?, ?, ?, ?)",
		p.Name, p.CardNumber, p.Expiry, p.Amount)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Stored INSECURE payment for %s\n", p.Name)
}

func main() {
	db := initDB()
	defer db.Close()
	
	payment := Payment{
		Name:       "John Doe",
		CardNumber: "4111111111111111", // Raw PAN
		Expiry:     "12/25",
		Amount:     100.00,
	}
	
	storePayment(db, payment)
}
```

**Security Issues:**

🚫 Entire database in PCI DSS scope
🚫 Backups contain sensitive cardholder data
🚫 Violates PCI DSS Requirement 3.4 (render PAN unreadable)
🚫 Memory may retain plaintext values

### 2. Secure Approach (Using CipherTrust API)

```go
package main

import (
	"bytes"
	"database/sql"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	_ "github.com/mattn/go-sqlite3"
)

const cipherTrustURL = "http://<IP>:32082/v1/protect"

type PaymentSecure struct {
	Name          string
	ProtectedPAN  string // Tokenized PAN
	ProtectedExp  string // Encrypted expiry
	Amount        float64
}

type ProtectRequest struct {
	PolicyName string `json:"protection_policy_name"`
	Data       string `json:"data"`
}

type ProtectResponse struct {
	ProtectedData string `json:"protected_data"`
}

func protectData(data string, policy string) (string, error) {
  // Create the JSON payload for CRDP PROTECT API call
	reqBody := ProtectRequest{
		PolicyName: policy,
		Data:       data,
	}
	
	jsonBody, _ := json.Marshal(reqBody)
  // Make the CRDP PROTECT API call
	resp, err := http.Post(cipherTrustURL, "application/json", bytes.NewBuffer(jsonBody))
	if err != nil {
		return "", err
	}
	defer resp.Body.Close()
	
	var result ProtectResponse
	json.NewDecoder(resp.Body).Decode(&result)
	return result.ProtectedData, nil
}

func initDBSecure() *sql.DB {
	db, err := sql.Open("sqlite3", "secure_payments.db")
	if err != nil {
		log.Fatal(err)
	}
	
	_, err = db.Exec(`
		CREATE TABLE IF NOT EXISTS payments (
			id INTEGER PRIMARY KEY,
			name TEXT,
			protected_pan TEXT,  // Tokenized PAN
			protected_exp TEXT,   // Encrypted expiry
			amount REAL
		)`)
	if err != nil {
		log.Fatal(err)
	}
	return db
}

func storePaymentSecure(db *sql.DB, p PaymentSecure) {
	_, err := db.Exec(
		"INSERT INTO payments (name, protected_pan, protected_exp, amount) VALUES (?, ?, ?, ?)",
		p.Name, p.ProtectedPAN, p.ProtectedExp, p.Amount)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Stored SECURE payment for %s\n", p.Name)
}

func main() {
	db := initDBSecure()
	defer db.Close()
	
	// Protect data before storage
	pan, err := protectData("4111111111111111", "protect-credit-card")
	if err != nil {
		log.Fatal(err)
	}
	
	exp, err := protectData("12/25", "protect-expiry")
	if err != nil {
		log.Fatal(err)
	}
	
	payment := PaymentSecure{
		Name:         "John Doe",
		ProtectedPAN: pan,
		ProtectedExp: exp,
		Amount:       100.00,
	}
	
	storePaymentSecure(db, payment)
}
```

### Security Comparison Table

| Aspect              | Insecure Approach                    | Secure Approach                      |
|---------------------|--------------------------------------|--------------------------------------|
| Data Storage        | 🚫 Raw PAN in database               | ✅ Only tokenized values             |
| PCI Scope           | 🔴 Entire application                | 🟢 Only API calls in scope           |
| Memory Exposure     | 🚫 Plaintext in memory               | ✅ Only protected data in process    |
| Database Backups    | 🔴 Contains sensitive data           | 🟢 Safe to backup anywhere           |
| Compliance Evidence | 🔴 Difficult to prove                | ✅ Automatic audit trails            |
| Key Management      | 🚫 Your responsibility              | ✅ Fully managed by CipherTrust      |

**Key to Symbols:**
- ✅ = Secure/Compliant
- 🟢 = Positive security attribute
- 🚫 = Security risk
- 🔴 = Compliance violation

### Key Advantages
1. **Memory Safety:**

```
// Insecure - plaintext remains in memory
card := "4111111111111111" 

// Secure - only protected version persists
protected, _ := protectData(card, "policy")
card = "" // Original cleared from memory
```

2. **Concurrent Protection:**

```
// Safe to use across goroutines
func processPayment(wg *sync.WaitGroup, card string) {
    defer wg.Done()
    protected, _ := protectData(card, "policy")
    // Store protected version...
}
```

3. **Zero Sensitive Data in Logs:**

```
// Insecure
log.Printf("Processing card: %s", card) // 🚫 PAN in logs

// Secure
log.Printf("Processing token: %s", protected) // ✅ Safe
```

### Implementation Checklist
1. Replace all raw PAN storage with API calls
2. Ensure no card data appears in logs/debug outputs
3. Update database schemas to remove plaintext fields
4. Verify all error paths don't expose raw data
5. Implement secure memory wiping for temporary values
