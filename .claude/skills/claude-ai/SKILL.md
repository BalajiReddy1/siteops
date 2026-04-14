---
name: claude-ai
description: >
  Claude API integration patterns for SiteOps AI-powered features. Load when generating
  the AI service module: smart payment reminders, AI cost estimator, and invoice auto-fill.
  Contains API request structure, prompt templates, and response parsing patterns.
user-invocable: false
---

# Claude AI — Anthropic API Patterns for SiteOps

## Dependencies (pom.xml)

```xml
<!-- HTTP client for Claude API (reuses existing WebClient from spring-boot-starter-webflux) -->
<!-- No additional dependency needed — uses Spring WebClient -->
```

---

## Configuration (application.yml)

```yaml
claude:
  api-key: ${CLAUDE_API_KEY}
  api-url: https://api.anthropic.com/v1/messages
  model: claude-sonnet-4-20250514
  max-tokens: 1024
```

---

## ClaudeAIService.java — Core Service

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class ClaudeAIService {

    @Value("${claude.api-key}") private String apiKey;
    @Value("${claude.api-url}") private String apiUrl;
    @Value("${claude.model}") private String model;
    @Value("${claude.max-tokens}") private int maxTokens;

    private final WebClient.Builder webClientBuilder;

    /**
     * Send a prompt to Claude and return the text response.
     * Used by all AI features in SiteOps.
     */
    public String complete(String systemPrompt, String userMessage) {
        Map<String, Object> requestBody = Map.of(
            "model", model,
            "max_tokens", maxTokens,
            "system", systemPrompt,
            "messages", List.of(
                Map.of("role", "user", "content", userMessage)
            )
        );

        try {
            Map<String, Object> response = webClientBuilder.build()
                .post()
                .uri(apiUrl)
                .header("x-api-key", apiKey)
                .header("anthropic-version", "2023-06-01")
                .header("content-type", "application/json")
                .bodyValue(requestBody)
                .retrieve()
                .bodyToMono(Map.class)
                .block();

            // Extract text from response.content[0].text
            List<Map<String, Object>> content = (List<Map<String, Object>>) response.get("content");
            return (String) content.get(0).get("text");

        } catch (Exception e) {
            log.error("Claude API call failed: {}", e.getMessage());
            return null;  // AI features are non-critical — return null, caller handles gracefully
        }
    }
}
```

---

## Feature 1: Smart Payment Reminders

```java
@Service
@RequiredArgsConstructor
public class SmartReminderService {

    private final ClaudeAIService claudeAI;

    private static final String SYSTEM_PROMPT = """
        You are a professional payment reminder writer for Indian construction contractors.
        Write a short WhatsApp message (max 200 words) reminding a builder about an overdue payment.
        The tone should be:
        - Gentle and polite if 1-7 days overdue
        - Firm but professional if 8-30 days overdue
        - Direct and urgent if 30+ days overdue
        Reply with ONLY the message text. No subject line, no formatting, no explanation.
        Use a mix of English and Hindi (Hinglish) that Indian builders commonly speak.
        """;

    /**
     * Generate a contextual payment reminder message using Claude.
     * Falls back to a static template if Claude API is unavailable.
     */
    public String generateReminder(Invoice invoice, Site site, Contractor contractor) {
        long daysOverdue = ChronoUnit.DAYS.between(invoice.getDueDate(), LocalDate.now());

        String userMessage = String.format(
            "Invoice: %s\nBuilder: %s\nSite: %s\nAmount Due: ₹%s\nDue Date: %s\nDays Overdue: %d\nContractor: %s\nContractor Phone: %s",
            invoice.getInvoiceNumber(),
            invoice.getBuilderName(),
            site.getName(),
            invoice.getTotalAmount().subtract(invoice.getAmountPaid()),
            invoice.getDueDate(),
            daysOverdue,
            contractor.getName(),
            contractor.getPhone()
        );

        String aiResponse = claudeAI.complete(SYSTEM_PROMPT, userMessage);

        if (aiResponse != null && !aiResponse.isBlank()) {
            return aiResponse;
        }

        // Fallback: static template
        return String.format(
            "Reminder: Invoice %s for ₹%s is overdue by %d days. Please clear the payment at your earliest convenience. - %s",
            invoice.getInvoiceNumber(),
            invoice.getTotalAmount().subtract(invoice.getAmountPaid()),
            daysOverdue,
            contractor.getName()
        );
    }
}
```

---

## Feature 2: AI Cost Estimator

```java
@Service
@RequiredArgsConstructor
public class CostEstimatorService {

    private final ClaudeAIService claudeAI;

    private static final String SYSTEM_PROMPT = """
        You are an expert Indian construction cost estimator.
        Given the work type, area in sq.ft., and city, estimate:
        1. Total material cost (in INR)
        2. Total labour cost (in INR)
        3. Number of labour days needed
        4. Key material quantities
        
        Use current Indian market rates for 2024-2025.
        Return ONLY a valid JSON object with keys:
        materialCost, labourCost, labourDays, totalEstimate, materials (array of {name, quantity, unit, rate})
        No explanation outside the JSON.
        """;

    public CostEstimate estimate(String workType, double areaSqFt, String city) {
        String userMessage = String.format(
            "Work Type: %s\nArea: %.0f sq.ft.\nCity: %s",
            workType, areaSqFt, city
        );

        String aiResponse = claudeAI.complete(SYSTEM_PROMPT, userMessage);

        if (aiResponse != null) {
            try {
                ObjectMapper mapper = new ObjectMapper();
                return mapper.readValue(aiResponse, CostEstimate.class);
            } catch (Exception e) {
                log.warn("Failed to parse cost estimate response: {}", e.getMessage());
            }
        }
        return null;
    }
}

// CostEstimate.java
public record CostEstimate(
    BigDecimal materialCost,
    BigDecimal labourCost,
    int labourDays,
    BigDecimal totalEstimate,
    List<MaterialItem> materials
) {}

public record MaterialItem(
    String name,
    double quantity,
    String unit,
    BigDecimal rate
) {}
```

---

## Feature 3: AI Invoice Auto-Fill

```java
@Service
@RequiredArgsConstructor
public class InvoiceAutoFillService {

    private final ClaudeAIService claudeAI;

    private static final String SYSTEM_PROMPT = """
        You are an Indian construction invoice line-item generator.
        Given work details (type, area, site), generate professional invoice line items.
        Return ONLY a valid JSON array of objects with keys:
        description, quantity, unit, unitPrice
        Use realistic Indian market prices. Include GST-ready descriptions.
        """;

    public List<InvoiceLineItemSuggestion> suggestLineItems(
            String workType, double areaSqFt, String siteName) {
        String userMessage = String.format(
            "Work: %s\nArea: %.0f sq.ft.\nSite: %s\nGenerate 3-6 line items.",
            workType, areaSqFt, siteName
        );

        String aiResponse = claudeAI.complete(SYSTEM_PROMPT, userMessage);

        if (aiResponse != null) {
            try {
                ObjectMapper mapper = new ObjectMapper();
                return mapper.readValue(aiResponse,
                    new TypeReference<List<InvoiceLineItemSuggestion>>() {});
            } catch (Exception e) {
                log.warn("Failed to parse invoice suggestions: {}", e.getMessage());
            }
        }
        return List.of();
    }
}
```

---

## REST API Endpoints

```java
@RestController
@RequestMapping("/api/v1/ai")
@RequiredArgsConstructor
@PreAuthorize("hasRole('CONTRACTOR_ADMIN')")
public class AIController {

    private final CostEstimatorService costEstimator;
    private final InvoiceAutoFillService invoiceAutoFill;
    private final SmartReminderService smartReminder;

    @PostMapping("/estimate")
    public ResponseEntity<ApiResponse<CostEstimate>> estimateCost(
            @Valid @RequestBody CostEstimateRequest request) {
        CostEstimate estimate = costEstimator.estimate(
            request.workType(), request.areaSqFt(), request.city());
        return ResponseEntity.ok(ApiResponse.ok(estimate));
    }

    @PostMapping("/invoice-suggest")
    public ResponseEntity<ApiResponse<List<InvoiceLineItemSuggestion>>> suggestLineItems(
            @Valid @RequestBody InvoiceSuggestRequest request) {
        var suggestions = invoiceAutoFill.suggestLineItems(
            request.workType(), request.areaSqFt(), request.siteName());
        return ResponseEntity.ok(ApiResponse.ok(suggestions));
    }

    @GetMapping("/reminder-preview/{invoiceId}")
    public ResponseEntity<ApiResponse<String>> previewReminder(@PathVariable Long invoiceId) {
        // Load invoice, site, contractor from DB, then generate
        String reminder = smartReminder.generateReminder(invoice, site, contractor);
        return ResponseEntity.ok(ApiResponse.ok(reminder));
    }
}
```

---

## Non-Critical Principle

All AI features follow the same principle as WhatsApp notifications:
1. **Never block the primary operation** — if Claude API is down, use fallback.
2. **Log failures, don't throw** — AI is an enhancement, not a requirement.
3. **Always have a static fallback** — every AI-generated output has a template alternative.
4. **Budget-safe** — all calls use `max_tokens: 1024` to control costs.
