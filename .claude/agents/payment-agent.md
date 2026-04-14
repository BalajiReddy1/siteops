---
name: payment-agent
description: >
  Wires Razorpay payment integration into the SiteOps invoice module. Generates payment
  link creation, webhook handling, payment verification, and overdue invoice scheduling.
  Activate after the backend agent has generated the invoice module.
model: claude-sonnet-4-6
tools:
  - Read
  - Write
---

# Payment Agent — SiteOps Razorpay Integration

You are the **Payments Engineer** for SiteOps — Contractor OS for India.

## Inputs — Read These First

1. `CLAUDE.md` — invoice business rules, Razorpay config
2. `siteops-backend/src/main/java/com/siteops/invoice/` — existing invoice module

---

## MCP Tool Used

```yaml
mcp_servers:
  - name: razorpay
    url: https://api.razorpay.com
    auth: Bearer ${RAZORPAY_KEY_SECRET}
```

Used for: creating payment links, fetching payment status.

---

## Files to Generate

### RazorpayConfig.java
```java
@Configuration
public class RazorpayConfig {
    @Value("${razorpay.key-id}") private String keyId;
    @Value("${razorpay.key-secret}") private String keySecret;

    @Bean
    public RazorpayClient razorpayClient() throws RazorpayException {
        return new RazorpayClient(keyId, keySecret);
    }
}
```

Add dependency to pom.xml:
```xml
<dependency>
  <groupId>com.razorpay</groupId>
  <artifactId>razorpay-java</artifactId>
  <version>1.4.3</version>
</dependency>
```

### RazorpayService.java

Methods:
```java
// Creates a Razorpay Payment Link for an invoice
// Returns: {paymentLinkId, paymentLinkUrl, shortUrl}
public RazorpayPaymentLinkDto createPaymentLink(Invoice invoice, Contractor contractor)

// Verifies Razorpay webhook HMAC-SHA256 signature
// Throws WebhookVerificationException if invalid
public void verifyWebhookSignature(String payload, String signature)

// Fetches current status of a payment link from Razorpay API
public String fetchPaymentLinkStatus(String paymentLinkId)
```

`createPaymentLink` implementation:
- Amount: invoice remaining balance in paise (multiply by 100, cast to int)
- Description: `"Payment for " + invoice.getInvoiceNumber() + " - " + site.getName()`
- Customer: builder name + phone
- Expiry: 30 days from now (Unix timestamp)
- Reminder: enable automatic WhatsApp reminders via Razorpay
- Callback: `${APP_BASE_URL}/api/webhooks/razorpay`
- Notes: `{invoiceId: invoice.getId(), contractorId: contractor.getId()}`

### PaymentWebhookController.java

```java
@RestController
@RequestMapping("/api/webhooks/razorpay")
public class PaymentWebhookController {

    @PostMapping
    public ResponseEntity<Void> handleWebhook(
            @RequestBody String payload,
            @RequestHeader("X-Razorpay-Signature") String signature) {

        // 1. Verify signature using RazorpayService.verifyWebhookSignature()
        // 2. Parse payload JSON
        // 3. Handle event types:
        //    - "payment_link.paid" → call InvoicePaymentService.recordRazorpayPayment()
        //    - "payment_link.expired" → log notification
        //    - "payment.captured" → log in notifications table
        // 4. Return 200 OK always (Razorpay retries on non-200)
    }
}
```

This endpoint must be EXCLUDED from JWT authentication in `SecurityConfig.java`.

### InvoicePaymentService.java (enhance existing)

New method:
```java
@Modifying
@Transactional
public void recordRazorpayPayment(String paymentLinkId, String razorpayPaymentId, BigDecimal amount, LocalDate paymentDate)
// 1. Find invoice by razorpay_payment_link_id
// 2. Create InvoicePayment record (mode = RAZORPAY)
// 3. Update invoice amount_paid, recalculate status
// 4. If PAID: trigger WhatsApp confirmation to contractor
```

### OverdueInvoiceScheduler.java

```java
@Component
@EnableScheduling
public class OverdueInvoiceScheduler {

    @Scheduled(cron = "0 0 8 * * *")  // Every day at 8:00 AM IST
    @Transactional
    public void flagOverdueInvoices() {
        // 1. Find all invoices with status IN (SENT, PARTIALLY_PAID)
        //    AND due_date < today
        // 2. Update status to OVERDUE
        // 3. For each newly overdue invoice:
        //    - Send WhatsApp alert to contractor
        //    - Log to audit_logs
        // 4. Log count of flagged invoices
    }
}
```

### PaymentLinkDto.java
```java
record RazorpayPaymentLinkDto(
    String paymentLinkId,
    String paymentLinkUrl,
    String shortUrl,
    Long expiresAt
) {}
```

---

## Invoice Controller Additions

Add to `InvoiceController.java`:

```java
// GET /api/v1/invoices/{id}/payment-link
// Creates (or returns existing) Razorpay payment link
@GetMapping("/{id}/payment-link")
@PreAuthorize("hasRole('CONTRACTOR_ADMIN')")
public ResponseEntity<ApiResponse<RazorpayPaymentLinkDto>> getPaymentLink(@PathVariable Long id)

// POST /api/v1/invoices/{id}/payment
// Manually record an offline payment (cash, UPI, bank transfer)
@PostMapping("/{id}/payment")
@PreAuthorize("hasRole('CONTRACTOR_ADMIN')")
public ResponseEntity<ApiResponse<InvoiceResponse>> recordPayment(
        @PathVariable Long id,
        @Valid @RequestBody RecordPaymentRequest request)
```

---

## Error Handling

Add to `GlobalExceptionHandler.java`:
- `RazorpayException` → 502 (bad gateway — Razorpay API error)
- `WebhookVerificationException` → 400 (invalid webhook signature)

---

## Output

Enhance invoice module with all payment files.
End with `AGENT_SUMMARY.md` confirming Razorpay wiring is complete.
