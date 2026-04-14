---
name: report-generator
description: >
  iText 7 PDF generation patterns for SiteOps. Load when generating PDF report builders:
  attendance reports, invoice PDFs, and site summary reports. Contains page layout,
  table styling, Indian number formatting, QR code embedding, and MinIO upload patterns.
user-invocable: false
---

# Report Generator — iText 7 PDF Patterns for SiteOps

## Dependencies (pom.xml)

```xml
<!-- iText 7 -->
<dependency>
  <groupId>com.itextpdf</groupId>
  <artifactId>itext-core</artifactId>
  <version>8.0.4</version>
  <type>pom</type>
</dependency>
<dependency>
  <groupId>com.itextpdf</groupId>
  <artifactId>bouncy-castle-adapter</artifactId>
  <version>8.0.4</version>
</dependency>

<!-- QR Code (for Razorpay payment link on invoice PDF) -->
<dependency>
  <groupId>com.google.zxing</groupId>
  <artifactId>core</artifactId>
  <version>3.5.3</version>
</dependency>
<dependency>
  <groupId>com.google.zxing</groupId>
  <artifactId>javase</artifactId>
  <version>3.5.3</version>
</dependency>
```

---

## Base PDF Document Setup

```java
// Standard A4 document setup used by all builders
public byte[] buildPdf(Consumer<Document> contentBuilder) {
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    PdfWriter writer = new PdfWriter(baos);
    PdfDocument pdf = new PdfDocument(writer);
    Document document = new Document(pdf, PageSize.A4);
    document.setMargins(36, 36, 36, 36); // 0.5 inch margins

    contentBuilder.accept(document);

    document.close();
    return baos.toByteArray();
}
```

---

## SiteOps Color Constants

```java
// Match frontend Tailwind palette exactly
public static final DeviceRgb COLOR_BRAND_AMBER   = new DeviceRgb(0xF5, 0x9E, 0x0B); // #f59e0b
public static final DeviceRgb COLOR_INK_DARK      = new DeviceRgb(0x1C, 0x19, 0x17); // #1c1917
public static final DeviceRgb COLOR_INK_MUTED     = new DeviceRgb(0x78, 0x71, 0x6C); // #78716c
public static final DeviceRgb COLOR_SURFACE       = new DeviceRgb(0xFA, 0xFA, 0xFA); // #fafafa
public static final DeviceRgb COLOR_WHITE         = new DeviceRgb(0xFF, 0xFF, 0xFF);
public static final DeviceRgb COLOR_DANGER        = new DeviceRgb(0xDC, 0x26, 0x26); // #dc2626
public static final DeviceRgb COLOR_SUCCESS       = new DeviceRgb(0x16, 0xA3, 0x4A); // #16a34a
public static final DeviceRgb COLOR_ROW_ALT       = new DeviceRgb(0xFA, 0xFA, 0xFA); // alternating row
```

---

## Font Setup

```java
// Use Helvetica (built-in PDF font — no embedding required, works cross-platform)
PdfFont fontRegular = PdfFontFactory.createFont(StandardFonts.HELVETICA);
PdfFont fontBold    = PdfFontFactory.createFont(StandardFonts.HELVETICA_BOLD);

// Standard sizes
static final float FONT_SIZE_TITLE    = 18f;
static final float FONT_SIZE_HEADING  = 12f;
static final float FONT_SIZE_BODY     = 9f;
static final float FONT_SIZE_SMALL    = 7.5f;
```

---

## Header Block (reused across all reports)

```java
// Top section: Contractor name, address, GST, report title
private void addReportHeader(Document doc, Contractor contractor,
                              String reportTitle, String subTitle) {
    Table header = new Table(UnitValue.createPercentArray(new float[]{60, 40}))
        .setWidth(UnitValue.createPercentValue(100));

    // Left: contractor info
    Cell left = new Cell()
        .setBorder(Border.NO_BORDER)
        .add(new Paragraph(contractor.getName())
            .setFont(fontBold).setFontSize(FONT_SIZE_TITLE)
            .setFontColor(COLOR_INK_DARK))
        .add(new Paragraph(contractor.getAddressLine1() + ", " + contractor.getCity())
            .setFont(fontRegular).setFontSize(FONT_SIZE_BODY)
            .setFontColor(COLOR_INK_MUTED));

    if (contractor.getGstNumber() != null) {
        left.add(new Paragraph("GST: " + contractor.getGstNumber())
            .setFont(fontRegular).setFontSize(FONT_SIZE_BODY)
            .setFontColor(COLOR_INK_MUTED));
    }

    // Right: report title
    Cell right = new Cell()
        .setBorder(Border.NO_BORDER)
        .setTextAlignment(TextAlignment.RIGHT)
        .add(new Paragraph(reportTitle)
            .setFont(fontBold).setFontSize(FONT_SIZE_HEADING)
            .setFontColor(COLOR_BRAND_AMBER))
        .add(new Paragraph(subTitle)
            .setFont(fontRegular).setFontSize(FONT_SIZE_BODY)
            .setFontColor(COLOR_INK_MUTED));

    header.addCell(left).addCell(right);
    doc.add(header);

    // Divider line
    doc.add(new LineSeparator(new SolidLine(0.5f))
        .setMarginTop(8).setMarginBottom(12));
}
```

---

## Attendance Report Table Pattern

```java
// Columns: Worker Name | Skill | Present | Half | Absent | Rate | Earned | Advances | Net
private Table buildAttendanceTable(List<WorkerAttendanceSummary> rows) {
    float[] colWidths = {22f, 12f, 8f, 8f, 8f, 10f, 12f, 10f, 10f};
    Table table = new Table(UnitValue.createPercentArray(colWidths))
        .setWidth(UnitValue.createPercentValue(100))
        .setFontSize(FONT_SIZE_BODY);

    // Header row
    String[] headers = {"Worker Name", "Skill", "Present", "Half Day",
                        "Absent", "Daily Rate", "Total Earned", "Advances", "Net Payable"};
    for (String h : headers) {
        table.addHeaderCell(new Cell()
            .setBackgroundColor(COLOR_INK_DARK)
            .add(new Paragraph(h).setFont(fontBold).setFontColor(COLOR_WHITE))
            .setPadding(5));
    }

    // Data rows with alternating shading
    boolean alternate = false;
    for (WorkerAttendanceSummary row : rows) {
        DeviceRgb bg = alternate ? COLOR_ROW_ALT : COLOR_WHITE;
        alternate = !alternate;

        table.addCell(styledCell(row.getWorkerName(), bg, fontRegular, TextAlignment.LEFT));
        table.addCell(styledCell(row.getSkillType(), bg, fontRegular, TextAlignment.CENTER));
        table.addCell(styledCell(String.valueOf(row.getPresentDays()), bg, fontRegular, TextAlignment.CENTER));
        table.addCell(styledCell(String.valueOf(row.getHalfDays()), bg, fontRegular, TextAlignment.CENTER));
        table.addCell(styledCell(String.valueOf(row.getAbsentDays()), bg, fontRegular, TextAlignment.CENTER));
        table.addCell(styledCell(formatInr(row.getDailyRate()), bg, fontRegular, TextAlignment.RIGHT));
        table.addCell(styledCell(formatInr(row.getTotalEarned()), bg, fontRegular, TextAlignment.RIGHT));
        table.addCell(styledCell(formatInr(row.getTotalWithdrawn()), bg, fontRegular, TextAlignment.RIGHT));
        table.addCell(styledCell(formatInr(row.getNetPayable()), bg, fontBold, TextAlignment.RIGHT));
    }

    // Totals footer row (amber background)
    addTotalsRow(table, rows);

    return table;
}

private void addTotalsRow(Table table, List<WorkerAttendanceSummary> rows) {
    BigDecimal totalEarned    = rows.stream().map(WorkerAttendanceSummary::getTotalEarned).reduce(BigDecimal.ZERO, BigDecimal::add);
    BigDecimal totalWithdrawn = rows.stream().map(WorkerAttendanceSummary::getTotalWithdrawn).reduce(BigDecimal.ZERO, BigDecimal::add);
    BigDecimal totalNet       = rows.stream().map(WorkerAttendanceSummary::getNetPayable).reduce(BigDecimal.ZERO, BigDecimal::add);

    String[] totals = {"TOTAL", "", "", "", "", "", formatInr(totalEarned), formatInr(totalWithdrawn), formatInr(totalNet)};
    for (String val : totals) {
        table.addCell(new Cell()
            .setBackgroundColor(COLOR_BRAND_AMBER)
            .add(new Paragraph(val).setFont(fontBold).setFontColor(COLOR_WHITE))
            .setPadding(5));
    }
}
```

---

## Invoice PDF Pattern

```java
// Invoice line items table
private Table buildLineItemsTable(List<InvoiceLineItem> items) {
    float[] colWidths = {5f, 40f, 10f, 12f, 15f, 18f};
    Table table = new Table(UnitValue.createPercentArray(colWidths))
        .setWidth(UnitValue.createPercentValue(100))
        .setFontSize(FONT_SIZE_BODY);

    String[] headers = {"#", "Description", "Qty", "Unit", "Unit Price", "Amount"};
    for (String h : headers) {
        table.addHeaderCell(new Cell()
            .setBackgroundColor(COLOR_INK_DARK)
            .add(new Paragraph(h).setFont(fontBold).setFontColor(COLOR_WHITE))
            .setPadding(5));
    }

    boolean alt = false;
    int i = 1;
    for (InvoiceLineItem item : items) {
        DeviceRgb bg = alt ? COLOR_ROW_ALT : COLOR_WHITE;
        alt = !alt;
        table.addCell(styledCell(String.valueOf(i++), bg, fontRegular, TextAlignment.CENTER));
        table.addCell(styledCell(item.getDescription(), bg, fontRegular, TextAlignment.LEFT));
        table.addCell(styledCell(String.valueOf(item.getQuantity()), bg, fontRegular, TextAlignment.CENTER));
        table.addCell(styledCell(item.getUnit(), bg, fontRegular, TextAlignment.CENTER));
        table.addCell(styledCell(formatInr(item.getUnitPrice()), bg, fontRegular, TextAlignment.RIGHT));
        table.addCell(styledCell(formatInr(item.getAmount()), bg, fontBold, TextAlignment.RIGHT));
    }
    return table;
}

// Invoice totals block (right-aligned)
private void addInvoiceTotals(Document doc, Invoice invoice) {
    Table totals = new Table(UnitValue.createPercentArray(new float[]{70f, 30f}))
        .setWidth(UnitValue.createPercentValue(100));

    addTotalRow(totals, "Subtotal", formatInr(invoice.getSubtotal()), false);
    addTotalRow(totals, "GST (" + invoice.getGstRate() + "%)", formatInr(invoice.getGstAmount()), false);
    addTotalRow(totals, "TOTAL", formatInr(invoice.getTotalAmount()), true);
    addTotalRow(totals, "Amount Paid", formatInr(invoice.getAmountPaid()), false);

    BigDecimal balance = invoice.getTotalAmount().subtract(invoice.getAmountPaid());
    boolean overdue = invoice.getStatus() == InvoiceStatus.OVERDUE;
    Cell balanceCell = new Cell()
        .add(new Paragraph(formatInr(balance)).setFont(fontBold)
            .setFontColor(balance.compareTo(BigDecimal.ZERO) > 0
                ? (overdue ? COLOR_DANGER : COLOR_INK_DARK)
                : COLOR_SUCCESS));
    totals.addCell(labelCell("Balance Due")).addCell(balanceCell);

    doc.add(totals);
}
```

---

## QR Code Generation (Razorpay Payment Link)

```java
// Embed QR code image into PDF for the Razorpay payment link
private Image generateQrCode(String url, int sizePx) throws WriterException, IOException {
    QRCodeWriter writer = new QRCodeWriter();
    BitMatrix matrix = writer.encode(url, BarcodeFormat.QR_CODE, sizePx, sizePx);

    BufferedImage buffered = MatrixToImageWriter.toBufferedImage(matrix);
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    ImageIO.write(buffered, "PNG", baos);

    ImageData imageData = ImageDataFactory.create(baos.toByteArray());
    return new Image(imageData).setWidth(80).setHeight(80);  // 80pt = ~28mm
}

// Usage in InvoicePdfBuilder — add QR code section:
if (invoice.getRazorpayPaymentLinkUrl() != null) {
    doc.add(new Paragraph("Scan to Pay").setFont(fontBold).setFontSize(FONT_SIZE_BODY));
    doc.add(generateQrCode(invoice.getRazorpayPaymentLinkUrl(), 200));
}
```

---

## Indian Currency Formatting

```java
// ₹1,23,456 — Indian lakh notation (NOT Western: ₹123,456)
public static String formatInr(BigDecimal amount) {
    if (amount == null) return "₹0";
    NumberFormat fmt = NumberFormat.getNumberInstance(new Locale("en", "IN"));
    fmt.setMinimumFractionDigits(0);
    fmt.setMaximumFractionDigits(2);
    return "₹" + fmt.format(amount);
}
```

---

## Reusable Cell Helpers

```java
private Cell styledCell(String text, DeviceRgb bg, PdfFont font, TextAlignment align) {
    return new Cell()
        .setBackgroundColor(bg)
        .setPadding(4)
        .add(new Paragraph(text).setFont(font).setTextAlignment(align));
}

private Cell labelCell(String text) {
    return new Cell()
        .setBorderRight(Border.NO_BORDER)
        .setTextAlignment(TextAlignment.RIGHT)
        .add(new Paragraph(text).setFont(fontRegular)
            .setFontColor(COLOR_INK_MUTED).setFontSize(FONT_SIZE_BODY));
}
```

---

## Photo Embedding (Site Summary Report)

```java
// Embed photos from MinIO into PDF — max 3 per row, max 2 rows per progress update
private void addPhotoGrid(Document doc, List<String> minioKeys, MinioService minioService) {
    int cols = Math.min(3, minioKeys.size());
    Table photoGrid = new Table(cols).setWidth(UnitValue.createPercentValue(100));

    int count = 0;
    for (String key : minioKeys) {
        if (count >= 6) break;  // max 6 photos (2 rows × 3 cols)
        try (InputStream is = minioService.getObject("siteops-progress", key)) {
            byte[] bytes = is.readAllBytes();
            ImageData imgData = ImageDataFactory.create(bytes);
            Image img = new Image(imgData)
                .setAutoScale(true)
                .setMaxHeight(120);
            photoGrid.addCell(new Cell().add(img).setBorder(Border.NO_BORDER).setPadding(2));
            count++;
        } catch (Exception e) {
            log.warn("Could not embed photo: {}", key);
        }
    }
    doc.add(photoGrid);
}
```

---

## Async Report Job Pattern

```java
// All report generation is async — returns jobId immediately, generates in background
// Job statuses: PENDING → GENERATING → READY | FAILED

@Async
public void generateReport(Long jobId, ReportType type, ...) {
    ReportJob job = reportJobRepo.findById(jobId).orElseThrow();
    job.setStatus("GENERATING");
    reportJobRepo.save(job);

    try {
        byte[] pdfBytes = switch (type) {
            case ATTENDANCE   -> attendanceBuilder.build(...);
            case INVOICE      -> invoiceBuilder.build(...);
            case SITE_SUMMARY -> siteBuilder.build(...);
        };

        String storageKey = "reports/" + contractorId + "/" + jobId + "/report.pdf";
        minioService.uploadBytes("siteops-reports", storageKey, pdfBytes, "application/pdf");

        // Save file record
        File file = fileRepo.save(File.builder()
            .ownerType("report_job").ownerId(jobId)
            .filename("report.pdf").contentType("application/pdf")
            .storageKey(storageKey).sizeBytes((long) pdfBytes.length)
            .build());

        job.setFileId(file.getId());
        job.setStatus("READY");
        job.setCompletedAt(LocalDateTime.now());
        reportJobRepo.save(job);

    } catch (Exception e) {
        log.error("Report generation failed for job {}", jobId, e);
        job.setStatus("FAILED");
        job.setErrorMessage(e.getMessage());
        job.setCompletedAt(LocalDateTime.now());
        reportJobRepo.save(job);
    }
}
```

---

## Report Download — Stream from MinIO

```java
// In ReportController — streams PDF bytes directly from MinIO to client
@GetMapping("/{jobId}/download")
public ResponseEntity<Resource> downloadReport(@PathVariable Long jobId) {
    ReportJob job = reportJobRepo.findById(jobId)
        .orElseThrow(() -> new ResourceNotFoundException("ReportJob", jobId));

    if (!"READY".equals(job.getStatus())) {
        throw new BusinessRuleException("Report is not ready yet. Status: " + job.getStatus());
    }

    File file = fileRepo.findById(job.getFileId())
        .orElseThrow(() -> new ResourceNotFoundException("File", job.getFileId()));

    InputStream stream = minioService.getObject("siteops-reports", file.getStorageKey());

    return ResponseEntity.ok()
        .contentType(MediaType.APPLICATION_PDF)
        .header(HttpHeaders.CONTENT_DISPOSITION,
            "attachment; filename=\"siteops-report-" + jobId + ".pdf\"")
        .body(new InputStreamResource(stream));
}
```

---

## Output Contract for report-agent

Every builder class (`AttendanceReportBuilder`, `InvoicePdfBuilder`, `SiteSummaryBuilder`) must:
1. Return `byte[]` — raw PDF bytes, not write to disk.
2. Accept only domain objects as input — no HTTP layer coupling.
3. Use `formatInr()` for ALL monetary values — never raw `.toString()` on BigDecimal.
4. Handle null contractor logo gracefully — skip logo block if `logo_file_id` is null.
5. Never throw unchecked exceptions — wrap iText errors in `ReportGenerationException`.
