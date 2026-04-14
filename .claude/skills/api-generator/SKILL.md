---
name: api-generator
description: >
  Provides REST API conventions, endpoint patterns, and Spring Boot code templates for
  SiteOps. Load when generating controllers, services, or API documentation. Includes
  standard CRUD patterns, pagination, error handling, and role-based security annotations.
user-invocable: false
---

# API Generator — SiteOps Spring Boot Patterns

## Standard Controller Template

```java
@RestController
@RequestMapping("/api/v1/{resource}s")
@RequiredArgsConstructor
@Tag(name = "{Resource} API")
public class {Resource}Controller {

    private final {Resource}Service service;

    @GetMapping
    @PreAuthorize("hasAnyRole('CONTRACTOR_ADMIN','SITE_SUPERVISOR','VIEWER')")
    public ResponseEntity<ApiResponse<PageResponse<{Resource}Response>>> list(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @AuthenticationPrincipal UserPrincipal principal) {
        return ResponseEntity.ok(ApiResponse.ok(
            service.findAll(page, size, principal.getContractorId())
        ));
    }

    @GetMapping("/{id}")
    @PreAuthorize("hasAnyRole('CONTRACTOR_ADMIN','SITE_SUPERVISOR','VIEWER')")
    public ResponseEntity<ApiResponse<{Resource}Response>> getById(
            @PathVariable Long id,
            @AuthenticationPrincipal UserPrincipal principal) {
        return ResponseEntity.ok(ApiResponse.ok(service.findById(id, principal.getContractorId())));
    }

    @PostMapping
    @PreAuthorize("hasAnyRole('CONTRACTOR_ADMIN','SITE_SUPERVISOR')")
    @Modifying
    public ResponseEntity<ApiResponse<{Resource}Response>> create(
            @Valid @RequestBody Create{Resource}Request request,
            @AuthenticationPrincipal UserPrincipal principal) {
        return ResponseEntity.status(HttpStatus.CREATED).body(
            ApiResponse.created(service.create(request, principal))
        );
    }

    @PutMapping("/{id}")
    @PreAuthorize("hasAnyRole('CONTRACTOR_ADMIN','SITE_SUPERVISOR')")
    @Modifying
    public ResponseEntity<ApiResponse<{Resource}Response>> update(
            @PathVariable Long id,
            @Valid @RequestBody Update{Resource}Request request,
            @AuthenticationPrincipal UserPrincipal principal) {
        return ResponseEntity.ok(ApiResponse.ok(service.update(id, request, principal)));
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('CONTRACTOR_ADMIN')")
    @Modifying
    public ResponseEntity<Void> delete(
            @PathVariable Long id,
            @AuthenticationPrincipal UserPrincipal principal) {
        service.softDelete(id, principal.getContractorId());
        return ResponseEntity.noContent().build();
    }
}
```

## Standard Response Types

```java
// Generic wrapper
public record ApiResponse<T>(T data, String message, int status) {
    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(data, "success", 200);
    }
    public static <T> ApiResponse<T> created(T data) {
        return new ApiResponse<>(data, "created", 201);
    }
}

// Paginated response
public record PageResponse<T>(
    List<T> content,
    int page,
    int size,
    long totalElements,
    int totalPages,
    boolean last
) {
    public static <T> PageResponse<T> from(Page<T> page) {
        return new PageResponse<>(
            page.getContent(), page.getNumber(), page.getSize(),
            page.getTotalElements(), page.getTotalPages(), page.isLast()
        );
    }
}

// Error response
public record ErrorResponse(String error, List<String> details, int status) {}
```

## Global Exception Handler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(404)
            .body(new ErrorResponse(ex.getMessage(), List.of(), 404));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<String> details = ex.getBindingResult().getFieldErrors()
            .stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .toList();
        return ResponseEntity.status(400)
            .body(new ErrorResponse("Validation failed", details, 400));
    }

    @ExceptionHandler(InsufficientBalanceException.class)
    public ResponseEntity<ErrorResponse> handleInsufficientBalance(InsufficientBalanceException ex) {
        return ResponseEntity.status(422)
            .body(new ErrorResponse(ex.getMessage(), List.of(), 422));
    }

    @ExceptionHandler(DuplicateAttendanceException.class)
    public ResponseEntity<ErrorResponse> handleDuplicate(DuplicateAttendanceException ex) {
        return ResponseEntity.status(409)
            .body(new ErrorResponse(ex.getMessage(), List.of(), 409));
    }

    @ExceptionHandler(BusinessRuleException.class)
    public ResponseEntity<ErrorResponse> handleBusinessRule(BusinessRuleException ex) {
        return ResponseEntity.status(400)
            .body(new ErrorResponse(ex.getMessage(), List.of(), 400));
    }

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleForbidden(AccessDeniedException ex) {
        return ResponseEntity.status(403)
            .body(new ErrorResponse("Access denied", List.of(), 403));
    }

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ErrorResponse> handleDataIntegrity(DataIntegrityViolationException ex) {
        return ResponseEntity.status(409)
            .body(new ErrorResponse("Data conflict — record already exists", List.of(), 409));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        log.error("Unhandled exception", ex);
        return ResponseEntity.status(500)
            .body(new ErrorResponse("Internal server error", List.of(), 500));
    }
}
```

## Custom Exceptions

```java
// Throw when entity not found: new ResourceNotFoundException("Worker", id)
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String entity, Long id) {
        super(entity + " not found with id: " + id);
    }
}

// Throw when wage withdrawal > net payable
public class InsufficientBalanceException extends RuntimeException {
    public InsufficientBalanceException(BigDecimal requested, BigDecimal available) {
        super("Cannot withdraw ₹" + requested + ". Available balance: ₹" + available);
    }
}

// Throw when attendance already exists for that day
public class DuplicateAttendanceException extends RuntimeException {
    public DuplicateAttendanceException(LocalDate date) {
        super("Attendance already marked for " + date);
    }
}

// General business rule violation
public class BusinessRuleException extends RuntimeException {
    public BusinessRuleException(String message) { super(message); }
}
```

## UserPrincipal

```java
// Stored in SecurityContextHolder after JWT auth
public class UserPrincipal implements UserDetails {
    private Long id;             // users.id
    private Long contractorId;   // contractor this user belongs to
    private String phone;
    private String role;         // CONTRACTOR_ADMIN, SITE_SUPERVISOR, VIEWER
    // ... implements UserDetails methods
}
```

## Pagination Defaults

All list endpoints:
- `?page=0&size=20` (default)
- `?sort=createdAt,desc` (default sort)
- Max page size: 100 (throw BusinessRuleException if > 100)

## Contractor Isolation

**Every service method that reads or writes data MUST filter by `contractorId`.**
A contractor must never see another contractor's data.

```java
// Always add contractor filter in repositories:
Page<Worker> findByContractorIdAndDeletedAtIsNull(Long contractorId, Pageable pageable);

// Always validate ownership in service:
Site site = siteRepo.findByIdAndContractorId(siteId, contractorId)
    .orElseThrow(() -> new ResourceNotFoundException("Site", siteId));
```

## Complete API Endpoint Map

### Auth
```
POST   /api/auth/login               → {accessToken, user}
POST   /api/auth/refresh             → {accessToken}  (reads refreshToken cookie)
POST   /api/auth/logout              → 204
```

### Sites
```
GET    /api/v1/sites                 → PageResponse<SiteResponse>
POST   /api/v1/sites                 → SiteResponse
GET    /api/v1/sites/{id}            → SiteResponse
PUT    /api/v1/sites/{id}            → SiteResponse
DELETE /api/v1/sites/{id}            → 204
GET    /api/v1/sites/{id}/dashboard  → SiteDashboardResponse (workers, attendance%, invoice status)
GET    /api/v1/sites/share/{token}   → PublicSiteProgressResponse  (NO AUTH)
```

### Workers
```
GET    /api/v1/workers               → PageResponse<WorkerResponse>
POST   /api/v1/workers               → WorkerResponse
GET    /api/v1/workers/{id}          → WorkerResponse
PUT    /api/v1/workers/{id}          → WorkerResponse
DELETE /api/v1/workers/{id}          → 204
GET    /api/v1/workers/{id}/ledger   → WageLedgerResponse (computed)
```

### Attendance
```
POST   /api/v1/attendance/bulk       → {saved: N}
GET    /api/v1/attendance/site/{siteId}/date/{date}  → List<AttendanceResponse>
GET    /api/v1/attendance/site/{siteId}/month/{year}/{month}  → AttendanceSummaryResponse
```

### Wages
```
GET    /api/v1/wages/worker/{workerId}/summary     → WageLedgerResponse
POST   /api/v1/wages/withdrawals                   → WithdrawalResponse
POST   /api/v1/wages/settle/{workerId}/{siteId}    → SettlementResponse
GET    /api/v1/wages/worker/{workerId}/withdrawals → PageResponse<WithdrawalResponse>
```

### Invoices
```
GET    /api/v1/invoices              → PageResponse<InvoiceResponse>
POST   /api/v1/invoices              → InvoiceResponse
GET    /api/v1/invoices/{id}         → InvoiceDetailResponse
PUT    /api/v1/invoices/{id}         → InvoiceResponse
DELETE /api/v1/invoices/{id}         → 204
POST   /api/v1/invoices/{id}/send    → InvoiceResponse
POST   /api/v1/invoices/{id}/payment → InvoiceResponse
GET    /api/v1/invoices/{id}/payment-link  → RazorpayPaymentLinkDto
```

### Progress
```
GET    /api/v1/sites/{siteId}/progress       → PageResponse<ProgressUpdateResponse>
POST   /api/v1/sites/{siteId}/progress       → ProgressUpdateResponse
POST   /api/v1/progress/{id}/photos          → ProgressUpdateResponse (multipart)
```

### Reports
```
POST   /api/v1/reports/attendance         → ReportJobResponse
POST   /api/v1/reports/invoice/{id}       → ReportJobResponse
POST   /api/v1/reports/site-summary/{id}  → ReportJobResponse
GET    /api/v1/reports/{jobId}            → ReportJobResponse
GET    /api/v1/reports/{jobId}/download   → PDF stream
```

### Notifications
```
GET    /api/v1/notifications              → PageResponse<NotificationResponse>
```

### Files
```
POST   /api/v1/files/upload               → FileResponse (multipart)
GET    /api/v1/files/{id}/download-url    → {url: presignedUrl}
```

### Webhooks (NO AUTH)
```
POST   /api/webhooks/razorpay             → 200 OK (always)
```
