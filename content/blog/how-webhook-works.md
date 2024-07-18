---
title: "How Webhook Works?"
date: 2024-04-12T15:18:25+05:30
description: "Let's understand how webhook works under the hood."
tags: ["how-it-works", "webhook"]
cover:
  image: images/how-webhook-works/cover.png
---
A few years ago, my colleagues and I were tasked with implementing webhooks in one of our products. I immersed myself in numerous blogs and documentation until the concept of webhooks clicked. Excited to share my newfound understanding, I explained it to my colleagues using a simple analogy:

> _"Imagine you're hungry and order food at a restaurant. Instead of constantly asking the waiter every few mins if your food is ready, the waiter brings it to you the moment it's prepared. That's exactly what a webhook does. Instead of repeatedly requesting information, a webhook sends you data in real-time as soon as an event happens."_

### Breaking It Down: What exactly is a Webhook?

In simpler terms, a webhook is a way for one application to send real-time data to another application. It's like getting a text message or a call when new mail arrives in your mailbox—instead of you checking the mailbox repeatedly, the postman informs you directly when there's something new. While webhooks provide instant updates and are more efficient, they can be more complex to set up compared to polling. But one might wonder why not simply poll the data, so let's understand that.

#### Polling vs Webhook

Polling is a technique where one application (client) checks another application (server) at regular intervals to detect any updates. It's like repeatedly checking your mailbox to see if there's any new mail. You keep going back to the mailbox at regular intervals to check for updates.

**Pros of Polling:**

- Simple to implement.
- Works with any server that can handle regular requests.

**Cons of Polling:**

- Inefficient, as it keeps sending requests even if there's no new data.
- Can create unnecessary load on the server and consume more bandwidth.

**How does Polling work?**

1. `Application A` sends a request to `Application B` to check for updates.
2. `Application B` responds with either the new update or indicates no new update is available.
3. `Application A` waits for a specified interval (e.g., N seconds or minutes) before repeating the process.

<img class="toggleDarkMode" src="/images/how-webhook-works/polling-vs-webhook.svg" alt="polling-vs-webhook" title="polling-vs-webhook" border="2"/>

Webhooks provide a more efficient solution by notifying the application directly when new data is available, reducing unnecessary requests and ensuring real-time updates.

**Pros of Web Hooks:**

- More efficient, as the server only sends data when there's something new.
- Reduces unnecessary requests, saving bandwidth and reducing server load.
- Provides real-time updates.

**Cons of Web Hooks:**

- Can be more complex to set up compared to polling.
- Your application needs to be accessible to the server (i.e., it must have a public-facing URL).
- Security considerations are crucial to ensure that only trusted servers can send data to your application.

### How Webhooks Actually Work

Here’s a step-by-step explanation of how webhooks work:

1. **Event Occurs:** An event triggers the webhook request in `Application A`. This event could be anything from a new user signing up, a payment being processed, or a GitHub action being completed.
2. **Webhook URL:** `Application A`, where the event occurred, sends a payload of data to a specified webhook URL.
3. **Receiving Data:** The receiving `Application B` captures the data and processes it accordingly.
4. **Response:** The receiving `Application B` sends a response back, confirming whether the data was received successfully or with an error.

### Webhook Security

Ensuring the security of webhooks is crucial to prevent unauthorized access and data breaches. Here are effective strategies to secure webhooks:

| **Threat**                                      | **Solution**                                   |
|-------------------------------------------------|------------------------------------------------|
| **Payload Exposure**                            | Use HTTPS for webhook URLs to ensure SSL encryption. |
| **Attacks from Unknown Webhook Sources**        | Implement authentication tokens and whitelist webhook source IPs. |
| **Webhook Interception and Redirection**        | Employ Mutual TLS for client verification.     |
| **Webhook Payload Corruption**                  | Verify messages using HMAC signatures.         |
| **Replay Attacks**                              | Use timestamped messages to prevent replay attacks. |

For more detailed explanations of these security threats and their solutions, you can refer to <a href="https://hookdeck.com/webhooks/guides/webhooks-security-checklist" target="_blank">this blog on webhook security</a>.

### A simple Webhook implementation in Go

Here, **Application A** sends a webhook request to **Application B** whenever an event occurs. **Application B** acts as the webhook recipient and both applications share a common secret key for HMAC signature verification, ensuring data integrity and authenticating the source in the webhook implementation.

#### Step-0

Let's create a new directory `webhook` and navigate into it:

```bash
mkdir webhook
cd webhook
```

#### Step-1: Create Webhook Server

Create a new file `recipient/server.go` to set up an `Application B` which acts as a web server with signature verification to ensure some basic security

```go
// server.go (Application B)
package main

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"io"
	"log"
	"net/http"
)

const secretKey = "shared_secret_key"

func verifySignature(payload []byte, signature string, secret string) bool {
	mac := hmac.New(sha256.New, []byte(secret))
	mac.Write(payload)
	expectedMAC := mac.Sum(nil)
	expectedSignature := hex.EncodeToString(expectedMAC)
	return hmac.Equal([]byte(expectedSignature), []byte(signature))
}

func webhookHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		log.Println("Invalid request method")
        http.Error(w, "Invalid request method", http.StatusBadRequest)
		return
	}

	signature := r.Header.Get("X-Signature")
	if signature == "" {
		log.Println("Missing signature")
        http.Error(w, "Missing signature", http.StatusBadRequest)
		return
	}

	payload, err := io.ReadAll(r.Body)
	if err != nil {
        log.Println("Invalid payload")
		http.Error(w, "Invalid payload", http.StatusBadRequest)
		return
	}

	if !verifySignature(payload, signature, secretKey) {
		log.Println("Invalid signature")
        http.Error(w, "Invalid signature", http.StatusBadRequest)
		return
	}

	log.Println("Received payload:", string(payload))
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	w.Write([]byte(`{"status": "success"}`))
}

func main() {
	http.HandleFunc("/webhook", webhookHandler)
	log.Println("Webhook Server started at port 8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

#### Step-2: Create Webhook Client

Create a new file `origin/client.go`, to set up an `Application A` which will compute the signature based on common shared secret key, and then sends the request to webhook URL when an event occurs with `X-Signature` request headers.

```go
// client.go (Application A)
package main

import (
	"bytes"
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"io"
	"net/http"
)

const webhookURL = "http://localhost:8080/webhook"
const secretKey = "shared_secret_key"

func computeSignature(payload []byte, secret string) string {
	mac := hmac.New(sha256.New, []byte(secret))
	mac.Write(payload)
	expectedMAC := mac.Sum(nil)
	return hex.EncodeToString(expectedMAC)
}

func sendRequest(url string, payload []byte, signature string) error {
	req, err := http.NewRequest("POST", url, bytes.NewBuffer(payload))
	if err != nil {
		return fmt.Errorf("error creating request: %v", err)
	}

	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("X-Signature", signature)

	client := &http.Client{}
	resp, err := client.Do(req)
	if err != nil {
		return fmt.Errorf("error sending request: %v", err)
	}
	defer resp.Body.Close()

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		return fmt.Errorf("error reading response: %v", err)
	}

	fmt.Println("Response status:", resp.Status)
	fmt.Println("Response body:", string(body))
	return nil
}

func main() {
	eventOccurred := true

	if eventOccurred {
		payload := []byte(`{"event": "test", "data": "sample"}`)
		signature := computeSignature(payload, secretKey)

		err := sendRequest(webhookURL, payload, signature)
		if err != nil {
			fmt.Println("Error:", err)
		}
	} else {
		fmt.Println("No event occurred.")
	}
}
```

#### Step-3: Run the Webhook Server

```bash
➜  go run recipient/server.go
2024/07/14 15:57:44 Webhook Server started at port 8080
2024/07/14 15:57:50 Received payload: {"event": "test", "data": "sample"}
```

#### Step-4: Run the Webhook Client

To test the web hook with the client program, ensure your webhook server is running and then run the client program:

```bash
➜  go run origin/client.go
Response status: 200 OK
Response body: {"status": "success"}
```

We can also send the webhook request using cURL or any programming language, as REST APIs are language-agnostic. We just need to compute the `X-Signature` request header using the shared secret key.

```bash
# Simple cURL example
curl -X POST http://localhost:8080/webhook \
     -H "Content-Type: application/json" \
     -H "X-Signature: <computed_signature>" \
     -d '{"event": "test", "data": "sample"}'
```

### Real-Life Usecases

Some real-life implementations include:

1. When a GitHub action, such as a CI/CD pipeline, completes, it can notify a Slack channel using a webhook URL set up through Slack's workflow.

2. Payment services like Stripe or Razorpay use webhooks to instantly update merchant systems about successful payments, refunds, ensuring real-time transaction status updates.

3. E-commerce platforms utilize webhooks to notify warehouses or logistics systems about new orders, updates in order status, or cancellations, ensuring efficient processing and timely delivery. 

I realized the value of this knowledge and felt motivated to share it through this blog. Webhooks ability to automate workflows and deliver timely updates makes them indispensable in modern digital products.
