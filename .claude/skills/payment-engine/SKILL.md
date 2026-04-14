---
name: payment-engine
description: >
  Razorpay integration patterns for SiteOps invoice payment links, webhook verification,
  and payment status reconciliation. Load when generating payment-related backend code.
user-invocable: false
---

# Payment Engine — Razorpay Patterns for SiteOps

## Payment Link Creation Flow

```
Contractor clicks "Send Invoice"
        │
        ▼
InvoiceService.send(invoiceId)
        │
        ├── 1. Validate invoice is DRAFT
        ├── 2. Call RazorpayService.createPaymentLink(invoice, contractor)
        │         └── POST https://api.razorpay.com/v1/payment_links
        ├── 3. Save razorpay_payment_link_id + razorpay_payment_link_url to invoice
        ├── 4. Update invoice status → SENT, sent_at → now
        ├── 5. Call WhatsAppService.sendInvoiceToBuilder(invoice, paymentLinkUrl)
        └── 6. Return updated InvoiceResponse
```

## Razorpay Payment Link Request Body

```java
JSONObject request = new JSONObject();
request.put("amount", invoiceBalance.multiply(BigDecimal.valueOf(100)).intValue()); // paise
request.put("currency", "INR");
request.put("accept_partial", true);  // allow partial payments
request.put("first_min_partial_amount",
    invoiceBalance.multiply(BigDecimal.valueOf(100)).multiply(BigDecimal.valueOf(0.25)).intValue()); // min 25%
request.put("description", "Payment for " + invoice.getInvoiceNumber() + " - " + site.getName());
request.put("expire_by", Instant.now().plus(30, ChronoUnit.DAYS).getEpochSecond());
request.put("send_sms", false);  // we handle WhatsApp ourselves
request.put("send_email", false);
request.put("reminder_enable", true);

JSONObject customer = new JSONObject();
customer.put("name", invoice.getBuilderName());
customer.put("contact", "+91" + invoice.getBuilderPhone().replaceFirst("^0", ""));
request.put("customer", customer);

JSONObject notify = new JSONObject();
notify.put("sms", false);
notify.put("whatsapp", true);  // Razorpay also sends WhatsApp (backup)
request.put("notify", notify);

JSONObject notes = new JSONObject();
notes.put("invoice_id", invoice.getId().toString());
notes.put("contractor_id", contractor.getId().toString());
notes.put("site_name", site.getName());
request.put("notes", notes);

// Callback URL for status updates
request.put("callback_url", appBaseUrl + "/invoices/" + invoice.getId());
request.put("callback_method", "get");
```

## Webhook Signature Verification

```java
// Razorpay sends X-Razorpay-Signature header
// Verify: HMAC-SHA256(payload, webhookSecret) == signature

public void verifyWebhookSignature(String payload, String signature) {
    try {
        Mac mac = Mac.getInstance("HmacSHA256");
        SecretKeySpec keySpec = new SecretKeySpec(
            webhookSecret.getBytes(StandardCharsets.UTF_8), "HmacSHA256"
        );
        mac.init(keySpec);
        byte[] hash = mac.doFinal(payload.getBytes(StandardCharsets.UTF_8));
        String computed = HexFormat.of().formatHex(hash);
        if (!computed.equals(signature)) {
            throw new WebhookVerificationException("Invalid Razorpay webhook signature");
        }
    } catch (NoSuchAlgorithmException | InvalidKeyException e) {
        throw new WebhookVerificationException("Signature verification failed: " + e.getMessage());
    }
}
```

## Webhook Event Handling

```java
// Relevant webhook events for SiteOps:
switch (event) {
    case "payment_link.paid":
        // Extract: payload.payment_link.entity (payment link object)
        //          payload.payment.entity (payment object)
        String paymentLinkId = entity.getString("id");
        String razorpayPaymentId = paymentEntity.getString("id");
        BigDecimal amount = new BigDecimal(paymentEntity.getInt("amount"))
            .divide(BigDecimal.valueOf(100));  // convert from paise
        invoicePaymentService.recordRazorpayPayment(paymentLinkId, razorpayPaymentId, amount, LocalDate.now());
        break;

    case "payment_link.expired":
        log.info("Payment link expired: {}", paymentLinkId);
        // Optionally: regenerate link or notify contractor
        break;
}
```

## Invoice Status State Machine

```
DRAFT ──► SENT ──► PARTIALLY_PAID ──► PAID
            │                           ▲
            └──────────────────────────┘
                  (full payment)
SENT ──► OVERDUE (when due_date < today, via scheduler)
Any non-PAID state ──► CANCELLED (by contractor)
```

## Amount Reconciliation Logic

```java
// After recording any payment:
BigDecimal newAmountPaid = invoice.getAmountPaid().add(payment.getAmount());
invoice.setAmountPaid(newAmountPaid);

BigDecimal balance = invoice.getTotalAmount().subtract(newAmountPaid);
if (balance.compareTo(BigDecimal.ZERO) <= 0) {
    invoice.setStatus(InvoiceStatus.PAID);
} else if (newAmountPaid.compareTo(BigDecimal.ZERO) > 0) {
    invoice.setStatus(InvoiceStatus.PARTIALLY_PAID);
}
// If status was OVERDUE and payment received → keep PARTIALLY_PAID or PAID
```

## Invoice Number Generation

```java
// Format: INV-{YYYY}-{NNNN} (zero-padded, sequential per contractor)
// Example: INV-2024-0001, INV-2024-0042

@Transactional
public String generateInvoiceNumber(Long contractorId) {
    int year = LocalDate.now().getYear();
    int nextSeq = invoiceRepository.countByContractorIdAndYear(contractorId, year) + 1;
    return String.format("INV-%d-%04d", year, nextSeq);
}
```
