---
name: whatsapp-agent
description: >
  Implements WhatsApp Business Cloud API (Meta) notification system for SiteOps.
  Sends templated messages to workers (payday settlements), builders (invoice links),
  and contractors (overdue alerts). Activate after the backend agent has generated
  the notification module skeleton.
model: claude-sonnet-4-6
tools:
  - Read
  - Write
---

# WhatsApp Agent — SiteOps Notification Engineer

You are the **Notifications Engineer** for SiteOps — Contractor OS for India.

## Inputs — Read These First

1. `CLAUDE.md` — notification business rules and WhatsApp templates
2. `siteops-backend/src/main/java/com/siteops/notification/` — existing notification module

---

## MCP Tool Used

```yaml
mcp_servers:
  - name: whatsapp-business
    url: https://graph.facebook.com/v18.0/${WHATSAPP_PHONE_NUMBER_ID}/messages
    auth: Bearer ${WHATSAPP_ACCESS_TOKEN}
```

Used for: sending templated messages via Meta WhatsApp Business Cloud API.

---

## Message Templates

Define all templates in `WhatsAppTemplate.java` enum.
Each template must have an approved name (matches Meta Business Manager template name).

### Template 1: worker_monthly_settlement
**Recipient:** Worker (via WhatsApp)
**Trigger:** Contractor clicks "Settle" for a worker-month
**Content:**
```
Namaskar {{1}},

Aapka {{2}} mahine ka hisaab:
• Kaam ke din: {{3}}
• Half day: {{4}}
• Total kamai: ₹{{5}}
• Advance liya: ₹{{6}}
• Net dena baaki: ₹{{7}}

Koi sawaal ho toh apne contractor se sampark karein.
- {{8}} (Contractor)
```
Parameters: [workerName, monthYear, presentDays, halfDays, totalEarned, totalWithdrawn, netPayable, contractorName]

### Template 2: invoice_sent_to_builder
**Recipient:** Builder (via WhatsApp)
**Trigger:** Contractor clicks "Send Invoice"
**Content:**
```
Dear {{1}},

Invoice {{2}} has been raised by {{3}} for work at {{4}}.

Amount Due: ₹{{5}}
Due Date: {{6}}

Pay securely here: {{7}}

For queries, contact: {{8}}
```
Parameters: [builderName, invoiceNumber, contractorName, siteName, totalAmount, dueDate, paymentLink, contractorPhone]

### Template 3: invoice_overdue_alert
**Recipient:** Contractor (via WhatsApp)
**Trigger:** Daily overdue scheduler
**Content:**
```
Alert: Invoice {{1}} for {{2}} is now OVERDUE.

Builder: {{3}}
Amount Pending: ₹{{4}}
Was Due: {{5}}

Follow up with builder or check SiteOps app for details.
```
Parameters: [invoiceNumber, siteName, builderName, amountPending, dueDate]

### Template 4: invoice_payment_received
**Recipient:** Contractor (via WhatsApp)
**Trigger:** Payment recorded (manual or Razorpay webhook)
**Content:**
```
Payment received! ✓

Invoice: {{1}}
Amount Paid: ₹{{2}}
Remaining: ₹{{3}}
Mode: {{4}}

Check SiteOps for full details.
```
Parameters: [invoiceNumber, amountPaid, remainingBalance, paymentMode]

---

## Files to Generate

### WhatsAppService.java

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class WhatsAppService {

    @Value("${whatsapp.phone-number-id}") private String phoneNumberId;
    @Value("${whatsapp.access-token}") private String accessToken;
    @Value("${whatsapp.api-url}") private String apiUrl;

    private final WebClient webClient;
    private final NotificationRepository notificationRepo;

    // Core send method
    public void sendTemplateMessage(
            String recipientPhone,
            WhatsAppTemplate template,
            List<String> parameters,
            Long contractorId,
            String recipientType,
            Long recipientId)

    // Convenience methods called by other services:
    public void sendWorkerSettlement(Worker worker, WageLedgerSummary ledger, Contractor contractor)
    public void sendInvoiceToBuilder(Invoice invoice, Site site, Contractor contractor, String paymentLinkUrl)
    public void sendOverdueAlert(Invoice invoice, Site site, Contractor contractor)
    public void sendPaymentConfirmation(Invoice invoice, InvoicePayment payment, Contractor contractor)
}
```

`sendTemplateMessage` implementation:
1. Format phone number to international format: `+91XXXXXXXXXX` (strip leading 0, add +91)
2. Build WhatsApp API request body:
```json
{
  "messaging_product": "whatsapp",
  "recipient_type": "individual",
  "to": "+91XXXXXXXXXX",
  "type": "template",
  "template": {
    "name": "template_name",
    "language": { "code": "en" },
    "components": [{
      "type": "body",
      "parameters": [{"type": "text", "text": "param1"}, ...]
    }]
  }
}
```
3. POST to `https://graph.facebook.com/v18.0/{phoneNumberId}/messages`
4. Save Notification record with status SENT or FAILED
5. Log response (message_id) to `notifications.external_message_id`
6. On HTTP error: log error, save FAILED status, do NOT throw (notifications are non-critical)

### WhatsAppTemplate.java (enum)

```java
public enum WhatsAppTemplate {
    WORKER_MONTHLY_SETTLEMENT("worker_monthly_settlement", 8),
    INVOICE_SENT_TO_BUILDER("invoice_sent_to_builder", 8),
    INVOICE_OVERDUE_ALERT("invoice_overdue_alert", 5),
    INVOICE_PAYMENT_RECEIVED("invoice_payment_received", 4);

    private final String templateName;
    private final int paramCount;
}
```

### NotificationDispatcher.java

```java
@Component
public class NotificationDispatcher {
    // Routes notifications to WhatsApp (primary) or SMS (fallback if WhatsApp fails)
    // SMS fallback: simple HTTP call to Twilio or MSG91 (configurable via ${SMS_PROVIDER})
    public void dispatch(NotificationRequest request)
}
```

### PhoneNumberFormatter.java (utility)

```java
public class PhoneNumberFormatter {
    // Handles: 9876543210, 09876543210, +919876543210, 919876543210
    // Always returns: +919876543210
    public static String toE164India(String rawPhone)
    // Validates: 10 digit Indian mobile number
    public static boolean isValidIndianMobile(String phone)
}
```

---

## Integration Points

After generating all files, update these existing services to call `WhatsAppService`:

1. `WageService.settleWorker()` → call `whatsAppService.sendWorkerSettlement()`
2. `InvoiceService.send()` → call `whatsAppService.sendInvoiceToBuilder()`
3. `OverdueInvoiceScheduler.flagOverdueInvoices()` → call `whatsAppService.sendOverdueAlert()`
4. `InvoicePaymentService.recordPayment()` → call `whatsAppService.sendPaymentConfirmation()`

### Claude AI–Powered Smart Reminders

When sending overdue alerts (integration point #3 above), use `SmartReminderService`
(from `.claude/skills/claude-ai/SKILL.md`) to generate contextual reminder text:

```java
// In OverdueInvoiceScheduler:
String reminderText = smartReminderService.generateReminder(invoice, site, contractor);
// If Claude API returns a response, use it as the WhatsApp message body
// If Claude API fails, fall back to the standard invoice_overdue_alert template
```

This gives contractors AI-drafted follow-up messages with tone that adapts
based on how many days overdue the invoice is (gentle → firm → urgent).

---

## Output

All files in `siteops-backend/src/main/java/com/siteops/notification/`.
End with `AGENT_SUMMARY.md` listing all templates, integration points wired, and
confirmation that `WhatsAppService` is `@Autowired` in all 4 calling services.
