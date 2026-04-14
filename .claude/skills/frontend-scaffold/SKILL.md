---
name: frontend-scaffold
description: >
  React + Tailwind component patterns, routing structure, API hook conventions, and
  Zustand store patterns for SiteOps frontend. Load when generating React components,
  pages, hooks, or store. Contains attendance marking UX patterns and mobile-first
  design guidance.
user-invocable: false
---

# Frontend Scaffold — SiteOps React Patterns

## Route Structure (React Router v6)

```jsx
// src/App.jsx
<Routes>
  <Route path="/login" element={<LoginPage />} />
  <Route path="/sites/share/:token" element={<SiteSharePage />} />  {/* public */}
  <Route element={<ProtectedRoute />}>
    <Route element={<AppShell />}>
      <Route index element={<Navigate to="/dashboard" replace />} />
      <Route path="/dashboard" element={<DashboardPage />} />
      <Route path="/sites" element={<SitesListPage />} />
      <Route path="/sites/new" element={<SiteFormPage />} />
      <Route path="/sites/:siteId" element={<SiteDetailPage />} />
      <Route path="/sites/:siteId/edit" element={<SiteFormPage />} />
      <Route path="/workers" element={<WorkersListPage />} />
      <Route path="/workers/new" element={<WorkerFormPage />} />
      <Route path="/workers/:workerId" element={<WorkerDetailPage />} />
      <Route path="/attendance" element={<AttendancePage />} />
      <Route path="/attendance/history" element={<AttendanceHistoryPage />} />
      <Route path="/invoices" element={<InvoicesListPage />} />
      <Route path="/invoices/new" element={<InvoiceFormPage />} />
      <Route path="/invoices/:invoiceId" element={<InvoiceDetailPage />} />
      <Route path="/invoices/:invoiceId/edit" element={<InvoiceFormPage />} />
      <Route path="/reports" element={<ReportsPage />} />
      <Route path="/settings" element={<SettingsPage />} />
    </Route>
  </Route>
</Routes>
```

## Zustand Auth Store

```js
// src/store/authStore.js
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

export const useAuthStore = create(
  persist(
    (set, get) => ({
      user: null,
      accessToken: null,
      isAuthenticated: false,

      setAuth: (user, token) => set({
        user,
        accessToken: token,
        isAuthenticated: true,
      }),

      setToken: (token) => set({ accessToken: token }),

      logout: () => {
        set({ user: null, accessToken: null, isAuthenticated: false });
        localStorage.removeItem('siteops_lang');
      },

      // Helpers
      isContractorAdmin: () => get().user?.role === 'CONTRACTOR_ADMIN',
      isSupervisor: () => get().user?.role === 'SITE_SUPERVISOR',
      contractorId: () => get().user?.contractorId,
    }),
    {
      name: 'siteops-auth',
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({ user: state.user }),  // don't persist token
    }
  )
);
```

## Axios Client with JWT Interceptor

```js
// src/api/client.js
import axios from 'axios';
import { useAuthStore } from '../store/authStore';

const client = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || 'http://localhost:8080',
  withCredentials: true,  // for httpOnly refresh cookie
});

// Request: attach Bearer token
client.interceptors.request.use((config) => {
  const token = useAuthStore.getState().accessToken;
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Response: auto-unwrap .data.data, handle 401 refresh
let isRefreshing = false;
let failedQueue = [];

const processQueue = (error, token = null) => {
  failedQueue.forEach(p => error ? p.reject(error) : p.resolve(token));
  failedQueue = [];
};

client.interceptors.response.use(
  (response) => response.data?.data ?? response.data,
  async (error) => {
    const original = error.config;

    if (error.response?.status === 401 && !original._retry) {
      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        }).then((token) => {
          original.headers.Authorization = `Bearer ${token}`;
          return client(original);
        });
      }

      original._retry = true;
      isRefreshing = true;

      try {
        const res = await axios.post(
          `${import.meta.env.VITE_API_BASE_URL}/api/auth/refresh`,
          {},
          { withCredentials: true }
        );
        const newToken = res.data?.data?.accessToken;
        useAuthStore.getState().setToken(newToken);
        processQueue(null, newToken);
        original.headers.Authorization = `Bearer ${newToken}`;
        return client(original);
      } catch (refreshError) {
        processQueue(refreshError, null);
        useAuthStore.getState().logout();
        window.location.href = '/login';
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error.response?.data ?? error);
  }
);

export default client;
```

## React Query Hook Pattern

```js
// src/api/hooks/useSites.js
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import client from '../client';

export const useSites = (params = {}) => useQuery({
  queryKey: ['sites', params],
  queryFn: () => client.get('/api/v1/sites', { params }),
  staleTime: 30_000,
});

export const useSite = (id) => useQuery({
  queryKey: ['sites', id],
  queryFn: () => client.get(`/api/v1/sites/${id}`),
  enabled: !!id,
});

export const useCreateSite = () => {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (data) => client.post('/api/v1/sites', data),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['sites'] }),
  });
};

export const useUpdateSite = (id) => {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (data) => client.put(`/api/v1/sites/${id}`, data),
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: ['sites'] });
      qc.invalidateQueries({ queryKey: ['sites', id] });
    },
  });
};
```

## Attendance Marking Component Pattern

```jsx
// The MOST IMPORTANT UX in SiteOps — must be fast on mobile
// State machine per worker: PRESENT → ABSENT → HALF_DAY → PRESENT (cycle)

const STATUS_CYCLE = ['PRESENT', 'ABSENT', 'HALF_DAY'];
const STATUS_LABELS = { PRESENT: 'P', ABSENT: 'A', HALF_DAY: 'H' };
const STATUS_COLORS = {
  PRESENT: 'bg-success text-white',
  ABSENT: 'bg-danger text-white',
  HALF_DAY: 'bg-warning text-white',
};

function WorkerAttendanceRow({ worker, status, onToggle, dailyRate }) {
  const earned = status === 'PRESENT' ? dailyRate
    : status === 'HALF_DAY' ? dailyRate / 2
    : 0;

  return (
    <div className="flex items-center py-3 border-b border-ink-faint last:border-0">
      <div className="flex-1 min-w-0">
        <div className="font-medium text-ink truncate">{worker.name}</div>
        <div className="text-sm text-ink-muted">{worker.skillType} · ₹{dailyRate}/day</div>
      </div>
      <div className="text-sm text-ink-muted mr-4">
        {earned > 0 ? `₹${earned}` : '—'}
      </div>
      <button
        onClick={() => onToggle(worker.id)}
        className={clsx(
          'w-10 h-10 rounded-full font-bold text-sm transition-all',
          STATUS_COLORS[status]
        )}
      >
        {STATUS_LABELS[status]}
      </button>
    </div>
  );
}
```

## Offline Support Pattern

```js
// src/utils/offlineQueue.js
// IndexedDB queue for offline attendance marking

const DB_NAME = 'siteops-offline';
const STORE_NAME = 'pending-requests';

export const queueRequest = async (endpoint, data) => {
  // Store {endpoint, data, timestamp} in IndexedDB
};

export const syncPendingRequests = async (clientInstance) => {
  // On navigator.onLine → true, process queue FIFO
  // On success: remove from queue
  // On failure: keep in queue, show error toast
};

// In AttendancePage — wrap bulk mark:
const handleSaveAttendance = async (data) => {
  if (!navigator.onLine) {
    await queueRequest('/api/v1/attendance/bulk', data);
    toast.info(t('attendance.queuedForSync'));
    return;
  }
  await bulkMark.mutateAsync(data);
};
```

## Component Library Patterns

```jsx
// Button
<Button variant="primary|secondary|danger|ghost" size="sm|md|lg" loading={bool}>
  Label
</Button>

// Badge for status
<StatusBadge status="DRAFT|SENT|PAID|OVERDUE|ACTIVE|COMPLETED" />
// Maps to appropriate color + label

// Currency display
<Amount value={123456} /> → ₹1,23,456
<Amount value={123456} compact /> → ₹1.23L

// Mobile bottom navigation (shown on < lg screens)
<BottomNav items={[
  { to: '/dashboard', icon: HomeIcon, label: t('nav.dashboard') },
  { to: '/attendance', icon: CheckSquareIcon, label: t('nav.attendance') },
  { to: '/invoices', icon: FileTextIcon, label: t('nav.invoices') },
  { to: '/workers', icon: UsersIcon, label: t('nav.workers') },
]} />
```

## Form Validation with Zod + React Hook Form

```jsx
// Example: Invoice form schema
const invoiceSchema = z.object({
  siteId: z.number({ required_error: 'Site is required' }),
  builderName: z.string().min(1, 'Builder name is required'),
  builderPhone: z.string().regex(/^[6-9]\d{9}$/, 'Invalid Indian mobile number'),
  lineItems: z.array(z.object({
    description: z.string().min(1),
    quantity: z.number().positive(),
    unitPrice: z.number().positive(),
  })).min(1, 'At least one line item required'),
  dueDate: z.string().min(1, 'Due date is required'),
});

const { register, handleSubmit, formState: { errors }, watch } = useForm({
  resolver: zodResolver(invoiceSchema),
  defaultValues: { lineItems: [{ description: '', quantity: 1, unit: 'Sqft', unitPrice: 0 }] },
});
```
