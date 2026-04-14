---
name: whatsapp-notify
description: >
  WhatsApp Business Cloud API patterns for SiteOps. Load when generating notification
  code. Contains API request structure, template parameter mapping, phone number
  formatting rules, and error handling for the Meta Graph API.
user-invocable: false
---

# WhatsApp Notify — Meta Business Cloud API Patterns

## API Endpoint

```
POST https://graph.facebook.com/v18.0/{WHATSAPP_PHONE_NUMBER_ID}/messages
Authorization: Bearer {WHATSAPP_ACCESS_TOKEN}
Content-Type: application/json
```

## Template Message Request Structure

```json
{
  "messaging_product": "whatsapp",
  "recipient_type": "individual",
  "to": "+919876543210",
  "type": "template",
  "template": {
    "name": "worker_monthly_settlement",
    "language": {
      "code": "en_IN"
    },
    "components": [
      {
        "type": "body",
        "parameters": [
          { "type": "text", "text": "Ramesh Kumar" },
          { "type": "text", "text": "November 2024" },
          { "type": "text", "text": "22" },
          { "type": "text", "text": "2" },
          { "type": "text", "text": "16,800" },
          { "type": "text", "text": "3,000" },
          { "type": "text", "text": "13,800" },
          { "type": "text", "text": "Balaji Construction" }
        ]
      }
    ]
  }
}
```

## Java WebClient Implementation

```java
public void sendTemplateMessage(
        String recipientPhone,
        WhatsAppTemplate template,
        List<String> parameters,
        Long contractorId,
        String recipientType,
        Long recipientId) {

    String formattedPhone = PhoneNumberFormatter.toE164India(recipientPhone);

    // Build parameters array
    List<Map<String, String>> params = parameters.stream()
        .map(p -> Map.of("type", "text", "text", p))
        .collect(Collectors.toList());

    Map<String, Object> requestBody = Map.of(
        "messaging_product", "whatsapp",
        "recipient_type", "individual",
        "to", formattedPhone,
        "type", "template",
        "template", Map.of(
            "name", template.getTemplateName(),
            "language", Map.of("code", "en_IN"),
            "components", List.of(
                Map.of("type", "body", "parameters", params)
            )
        )
    );

    // Save notification as PENDING first
    Notification notif = notificationRepo.save(Notification.builder()
        .contractorId(contractorId)
        .recipientType(recipientType)
        .recipientId(recipientId)
        .recipientPhone(formattedPhone)
        .channel("WHATSAPP")
        .templateName(template.getTemplateName())
        .status("PENDING")
        .build());

    // Send via WebClient
    whatsappWebClient.post()
        .uri("/{phoneNumberId}/messages", phoneNumberId)
        .bodyValue(requestBody)
        .retrieve()
        .bodyToMono(Map.class)
        .subscribe(
            response -> {
                String messageId = extractMessageId(response);
                notif.setStatus("SENT");
                notif.setExternalMessageId(messageId);
                notif.setSentAt(LocalDateTime.now());
                notificationRepo.save(notif);
                log.info("WhatsApp sent: {} to {}", template.getTemplateName(), formattedPhone);
            },
            error -> {
                notif.setStatus("FAILED");
                notif.setErrorMessage(error.getMessage());
                notificationRepo.save(notif);
                log.error("WhatsApp failed: {} to {} — {}", template.getTemplateName(), formattedPhone, error.getMessage());
                // DO NOT rethrow — notifications are non-critical
            }
        );
}
```

## Phone Number Formatting Rules

```java
// Indian mobile numbers: 10 digits starting with 6,7,8,9
// Input formats to handle:
//   9876543210    → +919876543210
//   09876543210   → +919876543210
//   919876543210  → +919876543210
//   +919876543210 → +919876543210

public static String toE164India(String rawPhone) {
    if (rawPhone == null) throw new IllegalArgumentException("Phone cannot be null");
    String digits = rawPhone.replaceAll("[^0-9]", "");  // strip non-digits

    if (digits.length() == 10) return "+91" + digits;
    if (digits.length() == 11 && digits.startsWith("0")) return "+91" + digits.substring(1);
    if (digits.length() == 12 && digits.startsWith("91")) return "+" + digits;
    if (digits.length() == 13 && digits.startsWith("091")) return "+" + digits.substring(1);

    throw new IllegalArgumentException("Cannot parse Indian phone number: " + rawPhone);
}

public static boolean isValidIndianMobile(String phone) {
    try {
        String e164 = toE164India(phone);
        return e164.matches("\\+91[6-9]\\d{9}");
    } catch (IllegalArgumentException e) {
        return false;
    }
}
```

## Template Parameter Reference

| Template | Parameters (ordered) |
|----------|----------------------|
| `worker_monthly_settlement` | workerName, monthYear, presentDays, halfDays, totalEarned, totalWithdrawn, netPayable, contractorName |
| `invoice_sent_to_builder` | builderName, invoiceNumber, contractorName, siteName, totalAmount, dueDate, paymentLink, contractorPhone |
| `invoice_overdue_alert` | invoiceNumber, siteName, builderName, amountPending, dueDate |
| `invoice_payment_received` | invoiceNumber, amountPaid, remainingBalance, paymentMode |

## Error Handling

Meta API returns HTTP 400 for:
- Invalid phone number format → log and mark FAILED, do not throw
- Template not approved → log and mark FAILED, do not throw
- Rate limit (HTTP 429) → log and retry after 60 seconds (max 3 retries)
- Invalid access token (HTTP 401) → log critical alert, mark FAILED

**Principle:** Notifications are best-effort. A failed WhatsApp message must NEVER
cause the primary operation (invoice send, wage settlement) to fail.
Always catch notification errors and log them without rethrowing.
