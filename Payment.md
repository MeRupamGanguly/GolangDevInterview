## Overview of Payment System
Key Objectives:

        Allow merchants to easily integrate payment methods into their platforms.
        Ensure that financial transactions are secure and protect sensitive data.
        Handle millions of transactions, especially during high-traffic periods like sales, holidays, or promotions.
        Ensure high uptime and fault tolerance for a mission-critical service.
        Adhere to global standards like PCI-DSS (Payment Card Industry Data Security Standard).


The Payment Gateway acts as the intermediary between the merchant's website (or app like Amazon/Netflix) and the payment processor. 
- The customer chooses a payment method (e.g., credit card, debit card, UPI, wallet).
- The customer enters their payment details (card number, expiry date, CVV, etc.) in the payment form on the merchant's website.
- The payment information is encrypted using SSL/TLS to ensure that the data is securely transmitted over the internet.
- The encrypted payment details are sent from the merchant's website to the Payment Gateway.
- The Payment Gateway tokenizes (or encrypts) the sensitive card data before sending it further to avoid direct exposure of sensitive information.
- The Payment Gateway forwards the encrypted payment information to the Payment Processor for further action (validation and authorization).

The Payment Processor is responsible for communicating with the payment network, validating the transaction, and facilitating the fund transfer from the customer's bank (issuer) to the merchant's bank (acquirer). 
- The Payment Processor receives the request from the Payment Gateway and starts the authorization process.
- It checks the merchant’s account details to ensure that the merchant is allowed to accept payments.
- The Payment Processor forwards the transaction request (along with customer details) to the Card Network (e.g., Visa, MasterCard, American Express). The Payment Processor adds important transaction data, such as the merchant’s account, payment amount, and customer details (card number, etc.), and sends it to the Card Network for authorization.

The Card Network is responsible for ensuring the transaction is routed from the payment processor to the issuer, and it also provides security layers like fraud detection.
- The Card Network routes the transaction request to the Issuer Bank for authorization. The Issuer Bank is the financial institution that issued the card to the customer.
- The Issuer Bank checks whether the transaction is legitimate and if the customer has sufficient funds or credit to complete the purchase.
- It also checks for fraud detection (e.g., verifying the transaction with 3D Secure, checking the customer’s location, card status, etc.).
- If the transaction is authorized, the Issuer Bank sends an authorization response (approved or declined) back through the Card Network to the Payment Processor.

Once the Payment Processor receives the authorization result from the Card Network then The Payment Processor forwards the authorization result (approved or declined) to the Payment Gateway. If the transaction is approved, the Payment Processor prepares to capture the payment later. If declined, the processor returns the error (e.g., insufficient funds, card expired, etc.).

- The Payment Gateway receives the authorization response from the Payment Processor and sends it to the merchant's website or app.
- If approved, the customer receives a confirmation of the payment. If declined, the customer is prompted with an error message (e.g., insufficient funds, invalid card).
- After the authorization is successful, the merchant (or the Payment Processor) can initiate the capture request to collect the funds from the customer's account. This can happen immediately or after a certain period (depending on the merchant's settings, such as "Delayed Capture").
- The Payment Processor sends the capture request to the Card Network, which routes it to the Issuer Bank for the transfer of funds.
- The Issuer Bank transfers the funds to the Acquirer Bank (merchant’s bank), minus any transaction fees. The payment is then credited to the merchant’s account, and the transaction is complete.

Important Points:
- For secure login, merchants and customers authenticate via OAuth tokens, ensuring a secure method for third-party integrations (e.g., login via Google, Facebook, etc.).
- Merchants and admins can have varying levels of access to the system. Admins can access the entire dashboard, whereas merchants can only manage their transactions. RBAC can handle this.
- Each transaction moves through different states like Pending, Authorized, Captured, Failed, Refunded, and Chargeback.
- Every transaction is logged with complete details such as transaction ID, user ID, amount, payment method, timestamp, status, etc.
- After each payment, a reconciliation process ensures that the transaction data in the gateway matches the bank’s records.
- The system can use machine learning or heuristics to detect unusual spending patterns, such as multiple failed login attempts, rapid transaction attempts, or transactions from suspicious regions. This is an additional layer of authentication for card payments. The customer might be prompted for a code (OTP) or additional verification step when making a purchase. Each transaction can be assigned a risk score based on factors like transaction amount, geographic location, frequency, and previous transaction history. High-risk transactions can trigger manual review or additional verification.
- Send SMS, email, or push notifications to customers and merchants. For example, after a transaction, customers receive a notification with the payment status (approved or declined). The system can provide webhook endpoints to notify merchants about events like payment success, payment failure, or refund initiation.
- Merchants can access detailed reports about their daily, weekly, or monthly sales. These reports include total revenue, number of successful/failed transactions, chargebacks, etc. Data-driven insights into transaction trends, customer behavior, and product popularity. Generate reports for tax purposes, including GST breakdowns (if applicable).

DUMMY GOLANG EXAMPLE: 

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"log"
	"math/rand"
	"net/http"
    "net/smtp"
	"strings"
	"sync"
	"time"

	"github.com/golang-jwt/jwt/v5"
	"github.com/gorilla/mux"
	"github.com/go-redis/redis/v8"
	"github.com/segmentio/kafka-go"
	"golang.org/x/net/context"
)

// -------------------- Redis Setup --------------------

// Redis client setup for managing rate limiting
var redisClient *redis.Client

// Initialize Redis client connection
func initRedis() {
	redisClient = redis.NewClient(&redis.Options{
		Addr: "localhost:6379", // Redis server address
	})
}

// -------------------- Kafka Setup --------------------

// Kafka setup
var kafkaWriter *kafka.Writer
var kafkaReader *kafka.Reader

// Initialize Kafka writer connection
func initKafka() {
	kafkaWriter = &kafka.Writer{
		Addr:     kafka.TCP("localhost:9092"), // Kafka server address
		Topic:    "payment-events",            // Kafka topic for payment events
		Balancer: &kafka.LeastBytes{},         // Kafka message distribution strategy
	}
	kafkaReader = kafka.NewReader(kafka.ReaderConfig{
		Brokers:   []string{"localhost:9092"},
		GroupID:   "payment-event-consumer-group",
		Topic:     "payment-events",
		Partition: 0,
	})
}

// -------------------- JWT Setup --------------------

// JWT secret key used to sign and validate tokens
var jwtSecret = []byte("supersecretkey")

// Validate JWT token and retrieve merchant ID from the token's claims
func validateJWT(tokenStr string) (string, error) {
	token, err := jwt.Parse(tokenStr, func(token *jwt.Token) (interface{}, error) {
		return jwtSecret, nil
	})
	if err != nil || !token.Valid {
		return "", fmt.Errorf("invalid token")
	}
	claims, ok := token.Claims.(jwt.MapClaims)
	if !ok {
		return "", fmt.Errorf("invalid token claims")
	}
	merchantID, ok := claims["merchant_id"].(string)
	if !ok {
		return "", fmt.Errorf("merchant_id not found in token")
	}
	return merchantID, nil
}

// -------------------- Rate Limiting --------------------

// Rate limiter using Redis to track requests from each merchant
func rateLimitRedis(merchantID string, limit int, duration time.Duration) bool {
	ctx := context.Background()
	key := fmt.Sprintf("rate_limit:%s", merchantID)
	count, err := redisClient.Get(ctx, key).Int()
	if err != nil {
		if errors.Is(err, redis.Nil) {
			count = 0
		} else {
			log.Println("Redis error:", err)
			return false
		}
	}
	if count >= limit {
		return false
	}
	_, err = redisClient.Incr(ctx, key).Result()
	if err != nil {
		log.Println("Error incrementing Redis key:", err)
		return false
	}
	if count == 0 {
        redisClient.Expire(ctx, key, duration)
    }
	return true
}


// -------------------- Email Sending --------------------
var smtpHost = "smtp.gmail.com"
var smtpPort = "587"
var smtpUser = "your-email@gmail.com" // Replace with your email
var smtpPassword = "your-email-password" // Replace with your email password or app password
// Send email notification to the merchant
func sendEmail(to, subject, body string) error {
	auth := smtp.PlainAuth("", smtpUser, smtpPassword, smtpHost)

	msg := []byte("To: " + to + "\r\n" +
		"Subject: " + subject + "\r\n" +
		"\r\n" +
		body + "\r\n")

	err := smtp.SendMail(smtpHost+":"+smtpPort, auth, smtpUser, []string{to}, msg)
	if err != nil {
		return fmt.Errorf("failed to send email: %v", err)
	}

	return nil
}

// -------------------- Kafka Consumer and Email --------------------
func consumeAndSendEmail() {
	for {
		msg, err := kafkaReader.ReadMessage(context.Background())
		if err != nil {
			log.Println("Error reading from Kafka:", err)
			continue
		}

		var paymentEvent map[string]interface{}
		if err := json.Unmarshal(msg.Value, &paymentEvent); err != nil {
			log.Println("Error unmarshalling Kafka message:", err)
			continue
		}

		merchantID := paymentEvent["merchant_id"].(string)
		status := paymentEvent["status"].(string)
		amount := paymentEvent["amount"].(float64)
		currency := paymentEvent["currency"].(string)

		subject := "Payment Event Notification"
		body := fmt.Sprintf("Merchant ID: %s\nStatus: %s\nAmount: %.2f %s", merchantID, status, amount, currency)

		err = sendEmail("merchant@example.com", subject, body) // Replace with merchant email
		if err != nil {
			log.Println("Error sending email:", err)
		} else {
			log.Println("Email sent successfully for payment event:", paymentEvent)
		}
	}
}

// -------------------- Payment Data Models --------------------

// PaymentMethod represents the type of payment (CreditCard, UPI, Wallet, etc.)
type PaymentMethod string

const (
	CreditCard PaymentMethod = "credit_card"
	UPI        PaymentMethod = "upi"
	Wallet     PaymentMethod = "wallet"
	NetBanking PaymentMethod = "net_banking"
)

// PaymentInput is the struct containing payment data from the merchant
type PaymentInput struct {
	Method     PaymentMethod `json:"method"`
	Card       *CardDetails  `json:"card,omitempty"`
	UPI        *UPIInfo      `json:"upi,omitempty"`
	Wallet     *WalletInfo   `json:"wallet,omitempty"`
	NetBanking *NetBankingInfo `json:"net_banking,omitempty"`
	MerchantID string         `json:"merchant_id"`
	Amount     float64        `json:"amount"`
	Currency   string         `json:"currency"`
}

// CardDetails for storing credit card details
type CardDetails struct {
	Number string `json:"number"`
	Expiry string `json:"expiry"`
	CVV    string `json:"cvv"`
}

// UPIInfo for storing UPI information
type UPIInfo struct {
	VPA string `json:"vpa"`
}

// WalletInfo for storing wallet information
type WalletInfo struct {
	WalletID string `json:"wallet_id"`
}

// NetBankingInfo for storing net banking information
type NetBankingInfo struct {
	BankCode string `json:"bank_code"`
	Account  string `json:"account"`
}
type AuthResponse struct {
	Status string `json:"status"`
	Reason string `json:"reason,omitempty"`
	AuthID string `json:"auth_id,omitempty"`
}

// -------------------- Payment Gateway --------------------

// PaymentGateway interface defines methods for tokenizing and forwarding payment info to the processor
type PaymentGateway interface {
	Tokenize(input PaymentInput) (string, error)
	ForwardToProcessor(token string, input PaymentInput) AuthResponse
}

type GatewayService struct{}

// Tokenize function creates a token for the payment data
func (g *GatewayService) Tokenize(input PaymentInput) (string, error) {
	var data string
	switch input.Method {
	case CreditCard:
		if input.Card != nil {
			data = input.Card.Number + input.Card.CVV
		}
	case UPI:
		if input.UPI != nil {
			data = input.UPI.VPA
		}
	case Wallet:
		if input.Wallet != nil {
			data = input.Wallet.WalletID
		}
	case NetBanking:
		if input.NetBanking != nil {
			data = input.NetBanking.BankCode + input.NetBanking.Account
		}
	default:
		return "", fmt.Errorf("unsupported payment method")
	}

	hash := sha256.Sum256([]byte(data))
	return hex.EncodeToString(hash[:]), nil
}

// ForwardToProcessor sends tokenized payment data to the payment processor for authorization
func (g *GatewayService) ForwardToProcessor(token string, input PaymentInput) AuthResponse {
	processor := ProcessorService{}
	return processor.AuthorizeTransaction(TransactionRequest{
		MerchantID: input.MerchantID,
		Amount:     input.Amount,
		Token:      token,
		Method:     input.Method,
		Currency:   input.Currency,
	})
}

// -------------------- Payment Processor --------------------

// PaymentProcessor interface defines methods for processing payments
type PaymentProcessor interface {
	AuthorizeTransaction(req TransactionRequest) AuthResponse
	Capture(authID string, amount float64) error
}

type ProcessorService struct{}

// AuthorizeTransaction processes and authorizes the transaction
func (p *ProcessorService) AuthorizeTransaction(req TransactionRequest) AuthResponse {
	if req.MerchantID == "" {
		return AuthResponse{Status: "declined", Reason: "invalid_merchant"}
	}

	if req.Amount > 10000 {
		return AuthResponse{Status: "declined", Reason: "limit_exceeded"}
	}

	if req.Token == "" {
		return AuthResponse{Status: "declined", Reason: "invalid_token"}
	}

	if rand.Intn(100) < 5 {
		return AuthResponse{Status: "declined", Reason: "fraud_suspected"}
	}

	authID := fmt.Sprintf("AUTH-%d", rand.Intn(100000))
	return AuthResponse{Status: "approved", AuthID: authID}
}

// Capture function processes the capture of funds after authorization
func (p *ProcessorService) Capture(authID string, amount float64) error {
	log.Printf("Captured %.2f for AuthID %s", amount, authID)
	return nil
}

// -------------------- Auth Response --------------------

// AuthResponse struct defines the response after payment authorization
type AuthResponse struct {
	Status string `json:"status"`
	Reason string `json:"reason,omitempty"`
	AuthID string `json:"auth_id,omitempty"`
}

// -------------------- HTTP Middleware --------------------

// JWT middleware for validating JWT tokens
func jwtMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		tokenHeader := r.Header.Get("Authorization")
		if tokenHeader == "" || !strings.HasPrefix(tokenHeader, "Bearer ") {
			http.Error(w, "Missing or invalid Authorization header", http.StatusUnauthorized)
			return
		}

		token := strings.TrimPrefix(tokenHeader, "Bearer ")
		merchantID, err := validateJWT(token)
		if err != nil {
			http.Error(w, "Unauthorized: "+err.Error(), http.StatusUnauthorized)
			return
		}

		// Attach the merchantID to the context
		ctx := r.Context()
		ctx = context.WithValue(ctx, "merchant_id", merchantID)
		r = r.WithContext(ctx)

		next.ServeHTTP(w, r)
	})
}

// Rate Limit middleware using Redis
func rateLimitMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		merchantID := r.Context().Value("merchant_id").(string)

		if !rateLimitRedis(merchantID, 5, time.Second) {
			http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
			return
		}

		next.ServeHTTP(w, r)
	})
}

// -------------------- Sign-In Route --------------------

// User sign-in route to generate JWT tokens for authenticated users
func signInHandler(w http.ResponseWriter, r *http.Request) {
	var user struct {
		Username string `json:"username"`
		Password string `json:"password"`
	}

	// Parse the JSON body
	if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
		http.Error(w, "Invalid input", http.StatusBadRequest)
		return
	}

	// In a real-world scenario, validate the user's credentials from a database
	if user.Username == "merchant1" && user.Password == "password" {
		// Generate JWT token
		token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
			"merchant_id": "merchant1",
			"exp":         time.Now().Add(time.Hour * 24).Unix(),
		})

		// Sign the token with the secret key
		tokenString, err := token.SignedString(jwtSecret)
		if err != nil {
			http.Error(w, "Failed to generate token", http.StatusInternalServerError)
			return
		}

		// Send the token back in the response
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{"token": tokenString})
	} else {
		http.Error(w, "Invalid username or password", http.StatusUnauthorized)
	}
}

// -------------------- Payment Handler --------------------
func paymentHandler(w http.ResponseWriter, r *http.Request) {
	var paymentInput PaymentInput

	if err := json.NewDecoder(r.Body).Decode(&paymentInput); err != nil {
		http.Error(w, "Invalid input", http.StatusBadRequest)
		return
	}

	gateway := GatewayService{}
	token, err := gateway.Tokenize(paymentInput)
	if err != nil {
		http.Error(w, "Error tokenizing payment", http.StatusInternalServerError)
		return
	}
	authResponse := gateway.ForwardToProcessor(token, paymentInput)
	// Send the response back to the client
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(authResponse)
	// Publish payment event to Kafka
	event := map[string]interface{}{
		"merchant_id": paymentInput.MerchantID,
		"amount":      paymentInput.Amount,
		"currency":    paymentInput.Currency,
		"status":      authResponse.Status,
		"reason":      authResponse.Reason,
		"auth_id":     authResponse.AuthID,
		"timestamp":   time.Now().Unix(),
	}
	eventData, _ := json.Marshal(event)

	err = kafkaWriter.WriteMessages(context.Background(),
		kafka.Message{
			Value: eventData,
		},
	)

	if err != nil {
		log.Println("Error sending event to Kafka:", err)
	}
}

// -------------------- Main Function --------------------

func main() {
	rand.Seed(time.Now().UnixNano())

	// Initialize Redis and Kafka
	initRedis()
	initKafka()
    go consumeAndSendEmail()
	// Create a new router using mux
	r := mux.NewRouter()

	// Public sign-in route
	r.HandleFunc("/signin", signInHandler).Methods("POST")

	// Payment route with JWT and rate limit middlewares
	r.Handle("/pay", rateLimitMiddleware(jwtMiddleware(http.HandlerFunc(paymentHandler)))).Methods("POST")

	// Start the server
	log.Println("Payment Gateway running on port 8080...")
	log.Fatal(http.ListenAndServe(":8080", r))
}

```










