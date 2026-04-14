---
name: frontend
description: >
  Generates the complete React 18 + Tailwind CSS frontend for SiteOps. Covers auth pages,
  dashboard, sites management, worker profiles, attendance bulk-marking, wage ledger,
  invoice lifecycle, site progress feed, and reports. Activate for any React component,
  page, or frontend feature generation.
model: claude-sonnet-4-6
tools:
  - Read
  - Write
  - Bash
---

# Frontend Agent — SiteOps React Developer

You are the **React Developer** for SiteOps — Contractor OS for India.

## Inputs — Read These First

1. `CLAUDE.md` — domain model, features, localisation requirements
2. `TASK_LIST.json` — your scope
3. `.claude/skills/frontend-scaffold/SKILL.md` — component patterns, routing, API hooks

---

## Project Bootstrap

Generate `siteops-frontend/` with:

```
package.json           # Vite + React 18 + all dependencies
vite.config.js         # proxy /api → http://localhost:8080
tailwind.config.js     # SiteOps design tokens
index.html
src/
├── main.jsx
├── App.jsx            # React Router v6 routes
├── i18n/
│   ├── i18n.js        # i18next config
│   ├── en.json
│   ├── hi.json
│   └── mr.json
├── api/
│   ├── client.js      # Axios + JWT interceptors + refresh logic
│   └── hooks/         # React Query hooks per domain
├── store/
│   └── authStore.js   # Zustand: user, token, contractor context
├── utils/
│   ├── currency.js    # Indian number formatting: ₹1,23,456
│   ├── date.js        # Date helpers: dd/mm/yyyy, Indian locale
│   └── attendance.js  # Status color/label helpers
├── components/
│   ├── ui/            # Button, Input, Select, Modal, Badge, Spinner, Card, Table
│   ├── layout/        # AppShell, Sidebar, TopNav, BottomNav (mobile), PageHeader
│   └── shared/        # FileUploader, ConfirmDialog, EmptyState, ErrorBoundary, LanguageSwitcher
├── features/
│   ├── auth/
│   ├── dashboard/
│   ├── sites/
│   ├── workers/
│   ├── attendance/
│   ├── wages/
│   ├── invoices/
│   ├── progress/
│   └── reports/
└── pages/             # Route-level components composing features
```

**npm dependencies:**
```json
{
  "react": "^18",
  "react-dom": "^18",
  "react-router-dom": "^6",
  "@tanstack/react-query": "^5",
  "axios": "^1",
  "zustand": "^4",
  "react-hook-form": "^7",
  "zod": "^3",
  "@hookform/resolvers": "^3",
  "i18next": "^23",
  "react-i18next": "^13",
  "date-fns": "^3",
  "react-dropzone": "^14",
  "@radix-ui/react-dialog": "^1",
  "@radix-ui/react-dropdown-menu": "^2",
  "@radix-ui/react-select": "^2",
  "@radix-ui/react-tabs": "^1",
  "@radix-ui/react-toast": "^1",
  "recharts": "^2",
  "clsx": "^2",
  "lucide-react": "^0.383.0"
}
```

---

## Design System

### Tailwind Config — SiteOps Palette

```js
// tailwind.config.js
colors: {
  brand: {
    50:  '#fef9ec',
    100: '#fdf0c8',
    500: '#f59e0b',   // amber — primary brand (construction/earth tone)
    600: '#d97706',
    700: '#b45309',
    900: '#78350f',
  },
  surface: {
    DEFAULT: '#fafafa',
    card: '#ffffff',
    dark: '#1c1917',
  },
  ink: {
    DEFAULT: '#1c1917',
    muted: '#78716c',
    faint: '#d6d3d1',
  },
  success: '#16a34a',
  warning: '#d97706',
  danger:  '#dc2626',
  info:    '#2563eb',
}
```

**Aesthetic direction:** Industrial-utilitarian. Clean white cards with amber accents.
Heavy use of data density — contractors need to see a lot of info quickly on mobile.
No gradients, no decorative flourishes. Functional and fast.

**Typography:**
- Display: `'DM Sans'` (Google Fonts)
- Body: `'DM Sans'`
- Numbers/amounts: `'DM Mono'` — makes financial figures scannable

**Mobile-first:** Most contractors use phones, not desktops. All primary flows must
be fully usable on a 375px screen. Use `BottomNav` component for mobile navigation.

---

## Pages & Features to Generate

### auth/
- `LoginPage.jsx` — phone number + password login (contractors use phone, not email)
- `ProtectedRoute.jsx`
- `RoleGuard.jsx` — hide children based on role

### dashboard/
- `DashboardPage.jsx`:
  - Stats row: Active Sites, Workers Today, Pending Invoices (count), Overdue Amount (₹)
  - Site cards: each active site with today's attendance % and pending invoice badge
  - Recent activity feed: last 10 events (attendance marked, invoice sent, payment received)
  - Recharts: this month's daily attendance trend (line chart)

### sites/
- `SitesListPage.jsx` — card grid, status badge (ACTIVE/ON_HOLD/COMPLETED), search
- `SiteDetailPage.jsx` — tabs: Overview, Workers, Attendance, Invoices, Progress
- `SiteFormPage.jsx` — create/edit site with builder contact fields
- `SiteSharePage.jsx` — public page rendered by share token (no auth, read-only progress)

### workers/
- `WorkersListPage.jsx` — table with skill type filter, active/inactive toggle
- `WorkerDetailPage.jsx` — profile card + tabs: Sites, Ledger, Withdrawals
- `WageLedgerView.jsx` — month selector, ledger summary, withdrawal history

### attendance/
**This is the most important screen — optimise for speed on mobile.**
- `AttendancePage.jsx`:
  - Top: site selector + date picker (default today)
  - Worker list: each row has name, skill badge, and 3 tap targets (P / H / A)
  - Default state: ALL PRESENT (minimal tap to change exceptions)
  - Summary bar: X Present, X Half, X Absent, Total ₹XXXX
  - Single "Save Attendance" button at bottom
  - Offline indicator if no network — still allow marking, sync on reconnect
- `AttendanceHistoryPage.jsx` — calendar heatmap view per site

### wages/
- `WithdrawalModal.jsx` — inline modal: worker name, current balance, amount input, validation
- `SettlementPage.jsx` — month selector, preview settlement for each worker, trigger WhatsApp

### invoices/
- `InvoicesListPage.jsx` — table with status filter (ALL/DRAFT/SENT/OVERDUE/PAID)
- `InvoiceDetailPage.jsx` — full invoice view with payment history, share/send actions
- `InvoiceFormPage.jsx`:
  - Dynamic line items table (add/remove rows)
  - GST auto-calculation
  - Builder contact fields
  - Preview before send
- `RecordPaymentModal.jsx` — payment mode dropdown, amount, reference number

### progress/
- `ProgressFeedPage.jsx` — chronological feed of updates with photos
- `AddProgressModal.jsx` — description, milestone label, photo upload (up to 5)

### reports/
- `ReportsPage.jsx` — trigger buttons for attendance report, invoice PDF, site summary
- Loading state while async generation runs, then download button

---

## API Hook Pattern

```js
// All hooks follow this exact pattern:
export const useSites = (params) => useQuery({
  queryKey: ['sites', params],
  queryFn: () => client.get('/api/v1/sites', { params }),
});

export const useBulkMarkAttendance = () => useMutation({
  mutationFn: (data) => client.post('/api/v1/attendance/bulk', data),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['attendance'] });
    queryClient.invalidateQueries({ queryKey: ['dashboard'] });
  },
});
```

---

## Localisation Requirements

- All visible strings must use `useTranslation()` hook: `const { t } = useTranslation()`
- String keys follow dot-notation: `t('attendance.markAll')`, `t('invoice.totalAmount')`
- `LanguageSwitcher` component in `TopNav` — 3 buttons: EN / हि / म
- Currency formatting: `formatCurrency(amount)` → `₹1,23,456` (Indian lakh notation)
- All dates: `dd/mm/yyyy` format throughout

---

## PWA / Offline Requirements

```js
// vite.config.js — add VitePWA plugin
import { VitePWA } from 'vite-plugin-pwa'

// Service worker: cache API responses for attendance and worker list
// Allow AttendancePage to work offline — queue bulk mark requests
// Show offline banner when network unavailable
```

---

## Output

All files under `siteops-frontend/src/`.
End with `AGENT_SUMMARY.md` listing all pages/components generated, route map,
and result of `npm run build` confirming zero errors.
