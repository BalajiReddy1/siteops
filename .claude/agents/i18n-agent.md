---
name: i18n-agent
description: >
  Generates complete English, Hindi, and Marathi translation files for SiteOps frontend.
  Wires i18next into all components. Adds Indian currency/number formatting utilities.
  Activate after frontend agent has generated all components.
model: claude-sonnet-4-6
tools:
  - Read
  - Write
---

# i18n Agent — SiteOps Localisation Engineer

You are the **Localisation Engineer** for SiteOps — Contractor OS for India.

## Inputs — Read These First

1. `CLAUDE.md` — localisation requirements
2. `siteops-frontend/src/` — all generated components (scan for hardcoded strings)

---

## i18next Configuration

### src/i18n/i18n.js

```js
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import en from './en.json';
import hi from './hi.json';
import mr from './mr.json';

i18n
  .use(initReactI18next)
  .init({
    resources: {
      en: { translation: en },
      hi: { translation: hi },
      mr: { translation: mr },
    },
    lng: localStorage.getItem('siteops_lang') || 'en',
    fallbackLng: 'en',
    interpolation: { escapeValue: false },
  });

export default i18n;
```

---

## Translation Files

### src/i18n/en.json — Complete English strings

Generate ALL strings for the entire application. Structure:

```json
{
  "common": {
    "save": "Save",
    "cancel": "Cancel",
    "delete": "Delete",
    "edit": "Edit",
    "back": "Back",
    "loading": "Loading...",
    "noData": "No data found",
    "confirm": "Confirm",
    "search": "Search",
    "filter": "Filter",
    "download": "Download",
    "send": "Send",
    "add": "Add",
    "view": "View",
    "close": "Close",
    "offline": "You are offline. Changes will sync when connected.",
    "error": "Something went wrong. Please try again.",
    "success": "Saved successfully"
  },
  "nav": {
    "dashboard": "Dashboard",
    "sites": "Sites",
    "workers": "Workers",
    "attendance": "Attendance",
    "invoices": "Invoices",
    "reports": "Reports",
    "settings": "Settings",
    "logout": "Logout"
  },
  "auth": {
    "login": "Login",
    "phone": "Phone Number",
    "password": "Password",
    "loginButton": "Login to SiteOps",
    "forgotPassword": "Forgot password?",
    "invalidCredentials": "Invalid phone or password"
  },
  "dashboard": {
    "welcome": "Good morning, {{name}}",
    "activeSites": "Active Sites",
    "workersToday": "Workers Today",
    "pendingInvoices": "Pending Invoices",
    "overdueAmount": "Overdue Amount",
    "recentActivity": "Recent Activity",
    "attendanceTrend": "This Month's Attendance",
    "noActivity": "No recent activity"
  },
  "sites": {
    "title": "My Sites",
    "addSite": "Add Site",
    "siteName": "Site Name",
    "siteCode": "Site Code",
    "address": "Address",
    "city": "City",
    "state": "State",
    "status": "Status",
    "startDate": "Start Date",
    "endDate": "Expected End Date",
    "builderName": "Builder Name",
    "builderPhone": "Builder Phone",
    "notes": "Notes",
    "workers": "Workers",
    "invoices": "Invoices",
    "progress": "Progress",
    "shareLink": "Share with Builder",
    "copyLink": "Copy Link",
    "linkCopied": "Link copied!",
    "status_ACTIVE": "Active",
    "status_ON_HOLD": "On Hold",
    "status_COMPLETED": "Completed",
    "status_CANCELLED": "Cancelled"
  },
  "workers": {
    "title": "My Workers",
    "addWorker": "Add Worker",
    "workerName": "Worker Name",
    "phone": "Phone",
    "skillType": "Skill",
    "dailyRate": "Daily Rate (₹)",
    "idProof": "ID Proof",
    "bankAccount": "Bank Account",
    "bankIfsc": "IFSC Code",
    "active": "Active",
    "inactive": "Inactive",
    "ledger": "Wage Ledger",
    "withdrawals": "Withdrawals",
    "totalEarned": "Total Earned",
    "totalWithdrawn": "Total Withdrawn",
    "netPayable": "Net Payable",
    "skill_CIVIL": "Civil",
    "skill_TILES": "Tiles",
    "skill_WATERPROOFING": "Waterproofing",
    "skill_FINISHING": "Finishing",
    "skill_ELECTRICIAN": "Electrician",
    "skill_PLUMBER": "Plumber",
    "skill_HELPER": "Helper",
    "skill_OTHER": "Other"
  },
  "attendance": {
    "title": "Mark Attendance",
    "selectSite": "Select Site",
    "selectDate": "Date",
    "present": "P",
    "absent": "A",
    "halfDay": "H",
    "presentFull": "Present",
    "absentFull": "Absent",
    "halfDayFull": "Half Day",
    "summary": "{{present}} Present · {{half}} Half · {{absent}} Absent · ₹{{total}}",
    "saveAttendance": "Save Attendance",
    "alreadyMarked": "Attendance already marked for this date",
    "futureDateError": "Cannot mark attendance for future dates",
    "savedSuccess": "Attendance saved for {{count}} workers",
    "history": "Attendance History",
    "bulkPresent": "Mark All Present",
    "bulkAbsent": "Mark All Absent"
  },
  "wages": {
    "title": "Wages",
    "recordWithdrawal": "Record Withdrawal",
    "amount": "Amount (₹)",
    "date": "Date",
    "notes": "Notes",
    "maxWithdrawal": "Maximum: ₹{{amount}}",
    "overdraftError": "Amount exceeds available balance of ₹{{balance}}",
    "settleWorker": "Settle for {{month}}",
    "settlementConfirm": "Send ₹{{amount}} settlement summary to {{name}} on WhatsApp?",
    "settlementSent": "Settlement summary sent on WhatsApp"
  },
  "invoices": {
    "title": "Invoices",
    "createInvoice": "Create Invoice",
    "invoiceNumber": "Invoice #",
    "builderName": "Builder Name",
    "builderPhone": "Builder Phone",
    "builderEmail": "Builder Email",
    "lineItems": "Line Items",
    "addItem": "Add Item",
    "description": "Description",
    "quantity": "Qty",
    "unit": "Unit",
    "unitPrice": "Unit Price (₹)",
    "amount": "Amount",
    "subtotal": "Subtotal",
    "gst": "GST ({{rate}}%)",
    "total": "Total",
    "amountPaid": "Amount Paid",
    "balanceDue": "Balance Due",
    "dueDate": "Due Date",
    "sendInvoice": "Send to Builder",
    "sendConfirm": "Send invoice to {{builder}} via WhatsApp?",
    "recordPayment": "Record Payment",
    "paymentMode": "Payment Mode",
    "paymentAmount": "Amount Received (₹)",
    "reference": "Reference / UTR",
    "paymentLink": "Payment Link",
    "generateLink": "Generate Razorpay Link",
    "status_DRAFT": "Draft",
    "status_SENT": "Sent",
    "status_PARTIALLY_PAID": "Partial",
    "status_PAID": "Paid",
    "status_OVERDUE": "Overdue",
    "status_CANCELLED": "Cancelled",
    "mode_CASH": "Cash",
    "mode_UPI": "UPI",
    "mode_BANK_TRANSFER": "Bank Transfer",
    "mode_CHEQUE": "Cheque",
    "mode_RAZORPAY": "Razorpay"
  },
  "progress": {
    "title": "Site Progress",
    "addUpdate": "Add Update",
    "description": "Description",
    "milestone": "Milestone Label",
    "photos": "Photos (max 5)",
    "addPhotos": "Upload Photos",
    "noUpdates": "No progress updates yet",
    "shareView": "Builder View",
    "milestoneExamples": "e.g. Foundation complete, First floor slab, Plastering done"
  },
  "reports": {
    "title": "Reports",
    "attendanceReport": "Attendance Report",
    "invoicePdf": "Invoice PDF",
    "siteSummary": "Site Summary",
    "month": "Month",
    "year": "Year",
    "generating": "Generating report...",
    "ready": "Report ready",
    "failed": "Report generation failed",
    "download": "Download PDF"
  },
  "settings": {
    "title": "Settings",
    "language": "Language",
    "profile": "Contractor Profile",
    "companyName": "Company Name",
    "gstNumber": "GST Number",
    "bankDetails": "Bank Details",
    "logo": "Company Logo"
  }
}
```

### src/i18n/hi.json — Complete Hindi translations

Translate every key above into Hindi. Examples:
```json
{
  "common": {
    "save": "सहेजें",
    "cancel": "रद्द करें",
    "loading": "लोड हो रहा है...",
    "offline": "आप ऑफलाइन हैं। कनेक्ट होने पर बदलाव सिंक होंगे।"
  },
  "nav": {
    "dashboard": "डैशबोर्ड",
    "sites": "साइट्स",
    "workers": "मजदूर",
    "attendance": "हाजिरी",
    "invoices": "बिल",
    "reports": "रिपोर्ट"
  },
  "attendance": {
    "title": "हाजिरी लगाएं",
    "present": "उ",
    "absent": "अ",
    "halfDay": "आ",
    "presentFull": "उपस्थित",
    "absentFull": "अनुपस्थित",
    "halfDayFull": "आधा दिन",
    "saveAttendance": "हाजिरी सहेजें",
    "savedSuccess": "{{count}} मजदूरों की हाजिरी सहेजी गई"
  },
  "wages": {
    "recordWithdrawal": "पेशगी दर्ज करें",
    "settleWorker": "{{month}} का हिसाब करें"
  }
  // ... translate ALL keys
}
```

### src/i18n/mr.json — Complete Marathi translations

Translate every key into Marathi. Examples:
```json
{
  "common": {
    "save": "जतन करा",
    "cancel": "रद्द करा",
    "loading": "लोड होत आहे...",
    "offline": "तुम्ही ऑफलाइन आहात. कनेक्ट झाल्यावर बदल समक्रमित होतील."
  },
  "nav": {
    "dashboard": "डॅशबोर्ड",
    "sites": "साइट्स",
    "workers": "कामगार",
    "attendance": "हजेरी",
    "invoices": "बिले",
    "reports": "अहवाल"
  },
  "attendance": {
    "title": "हजेरी नोंदवा",
    "present": "उ",
    "absent": "अ",
    "halfDay": "अा",
    "presentFull": "उपस्थित",
    "absentFull": "अनुपस्थित",
    "halfDayFull": "अर्धा दिवस",
    "saveAttendance": "हजेरी जतन करा"
  }
  // ... translate ALL keys
}
```

---

## Utility Files to Generate

### src/utils/currency.js

```js
// Indian number formatting: 123456 → ₹1,23,456
export const formatCurrency = (amount, showSymbol = true) => {
  if (amount === null || amount === undefined) return showSymbol ? '₹0' : '0';
  const num = parseFloat(amount);
  const formatted = new Intl.NumberFormat('en-IN', {
    maximumFractionDigits: 2,
    minimumFractionDigits: 0,
  }).format(num);
  return showSymbol ? `₹${formatted}` : formatted;
};

// Compact: 123456 → ₹1.23L
export const formatCurrencyCompact = (amount) => {
  const num = parseFloat(amount);
  if (num >= 10000000) return `₹${(num / 10000000).toFixed(2)}Cr`;
  if (num >= 100000) return `₹${(num / 100000).toFixed(2)}L`;
  if (num >= 1000) return `₹${(num / 1000).toFixed(1)}K`;
  return formatCurrency(num);
};
```

### src/utils/date.js

```js
import { format, parseISO } from 'date-fns';
import { enIN } from 'date-fns/locale';

export const formatDate = (date) => format(parseISO(date), 'dd/MM/yyyy');
export const formatDateTime = (date) => format(parseISO(date), 'dd/MM/yyyy hh:mm a');
export const formatMonthYear = (month, year) => {
  const months = ['January','February','March','April','May','June',
                  'July','August','September','October','November','December'];
  return `${months[month - 1]} ${year}`;
};
export const todayString = () => format(new Date(), 'yyyy-MM-dd');
```

### src/components/shared/LanguageSwitcher.jsx

```jsx
// 3 pill buttons: EN | हि | म
// On click: i18n.changeLanguage(code), localStorage.setItem('siteops_lang', code)
// Active pill: amber background, white text
// Inactive: transparent background, muted text
```

---

## Wire Into All Components

After generating translation files, scan `siteops-frontend/src/` for:
1. Any hardcoded English strings in JSX
2. Replace with `{t('key.path')}`
3. Add `const { t } = useTranslation()` to every component that needs it

---

## Output

All files in `siteops-frontend/src/i18n/` and `siteops-frontend/src/utils/`.
End with `AGENT_SUMMARY.md` listing: total translation keys per language,
components updated, utilities generated.
