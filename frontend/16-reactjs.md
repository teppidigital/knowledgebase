# React.js — Foundation, Best Practices & Patterns

## Category

Frontend Architecture — React.js

## Context

React is a declarative, component-based UI library for building interactive interfaces. This document covers its conceptual foundation, mental models, best practices, design patterns, and anti-patterns — structured as a reference for learning and for applying to production projects.

### React Mental Model

| Concept | What it Means in Practice |
|---------|--------------------------|
| **Declarative UI** | Describe *what* the UI looks like for a given state; React figures out *how* to update the DOM |
| **Component = function** | A component is a function that takes props and returns JSX; UI = f(state) |
| **Unidirectional data flow** | Data flows from parent to child via props; child communicates up via callbacks |
| **Reconciliation** | React diffs the virtual DOM tree on each render and applies the minimal set of real DOM mutations |
| **Concurrent Mode** | React 18+ can interrupt, pause, and resume renders — enabling `useTransition`, `Suspense`, streaming SSR |
| **Server Components (RSC)** | Components that run only on the server — no client JS cost, can query DBs directly |

### Core APIs Quick Reference

| API | Purpose | When to Use |
|-----|---------|------------|
| `useState` | Local mutable state | Simple values: counts, toggles, input text |
| `useReducer` | Complex state transitions | State with multiple sub-fields or actions |
| `useEffect` | Synchronise with external systems | DOM manipulation, subscriptions, non-React integrations |
| `useRef` | Mutable value without re-render / DOM access | Focus management, timer IDs, previous-value tracking |
| `useMemo` | Cache expensive computed values | Only when profiler confirms performance issue |
| `useCallback` | Stable function reference | Passed as prop to memoised children |
| `useContext` | Read context value | Auth, theme, locale — not a Redux replacement |
| `useTransition` | Mark an update as non-urgent | Search, tab switch — keep input responsive |
| `useDeferredValue` | Defer a derived value | Re-render heavy list after quick input stays snappy |
| `useId` | Stable unique ID across server/client | Accessible label–input pairing |

---

## Foundation

### 1. Component Model

Every React component follows the same contract:
- Receives **props** (immutable within the render)
- Manages **state** (mutable, triggers re-render on change)
- Returns **JSX** (description of the UI — compiled to `React.createElement` calls)

```tsx
// The simplest possible component
function Button({ label, onClick }: { label: string; onClick: () => void }) {
  return <button type="button" onClick={onClick}>{label}</button>;
}
```

**Rules of components:**
1. Component names must start with a capital letter (distinguishes from native DOM elements)
2. Return a single root element (or a Fragment: `<>...</>`)
3. Never mutate props — treat them as read-only
4. Side effects belong in `useEffect`, not during render

### 2. JSX Fundamentals

JSX is syntactic sugar for `React.createElement`. The compiler (Babel/SWC) transforms it at build time.

```tsx
// JSX
const el = <div className="card" aria-label="Payment">Hello</div>;

// Compiled to (React 17+ automatic runtime — no need to import React)
const el = _jsx("div", { className: "card", "aria-label": "Payment", children: "Hello" });
```

Key JSX rules:
- Use `className` instead of `class`; `htmlFor` instead of `for`
- Self-close empty elements: `<Input />` not `<Input></Input>`
- JavaScript expressions go in `{}`: `{user.name}`, `{items.map(...)}`
- `null`, `undefined`, `false` render nothing — useful for conditional rendering

### 3. State and Re-Rendering

React re-renders a component when:
1. Its **state** changes (`useState` / `useReducer` dispatch)
2. Its **parent re-renders** (unless wrapped in `React.memo`)
3. Its **context** value changes

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  // Functional update — always correct, never stale
  const increment = () => setCount(prev => prev + 1);

  return <button onClick={increment}>{count}</button>;
}
```

**State update batching (React 18+):** Multiple `setState` calls inside the same event handler are batched into a single re-render automatically — even across `await` boundaries (automatic batching).

### 4. The Effect Model

`useEffect` synchronises a component with something outside React (browser APIs, subscriptions, timers, non-React libraries). It is NOT a lifecycle hook — think of it as "synchronise when these deps change."

```tsx
useEffect(() => {
  // Setup: runs after render when [userId] changes
  const subscription = subscribe(userId, handleMessage);

  // Cleanup: runs before next effect execution and on unmount
  return () => subscription.unsubscribe();
}, [userId]);           // dependency array — re-run when userId changes
```

**The dependency array rules:**
- `[]` — run once after first render (mount)
- `[dep1, dep2]` — re-run when dep1 or dep2 changes
- *(no array)* — run after every render (almost never correct)
- Every reactive value used inside the effect (state, props, context) must be in the deps array

### 5. Rendering and Reconciliation

React's reconciler (`react-dom`) compares the previous and new virtual DOM trees (diffing):
- Nodes of a different type are unmounted and rebuilt from scratch
- List nodes are matched by `key` — always provide a stable, unique key for list items
- Components at the same position with the same type are updated in-place

```tsx
// Keys must be stable, unique, and from data — not array index
{payments.map(payment => (
  <PaymentRow key={payment.id} payment={payment} />  // ✅ stable ID
))}

// ❌ Never use array index as key if list can reorder or filter
{payments.map((p, i) => <PaymentRow key={i} payment={p} />)}
```

---

## Best Practices

### BP1. Keep Components Small and Focused

Each component should do one thing well. If a component renders a list, fetches data, handles form validation, and displays errors — split it.

```tsx
// ❌ One component doing everything
function PaymentDashboard() {
  const [payments, setPayments] = useState([]);
  const [loading, setLoading] = useState(false);
  const [newAmount, setNewAmount] = useState('');

  useEffect(() => { /* fetch */ }, []);
  const handleSubmit = () => { /* post */ };
  const handleDelete = (id) => { /* delete */ };

  return ( /* 200 lines of JSX */ );
}

// ✅ Separated concerns
function PaymentDashboard() {
  return (
    <>
      <CreatePaymentForm />
      <PaymentList />
    </>
  );
}
```

**Rule of thumb:** If you need to scroll to see all the JSX in a component, it's probably too large.

### BP2. Co-locate State with the Component That Owns It

Don't hoist state higher than necessary. State lives at the lowest common ancestor that needs it.

```tsx
// ❌ Global state for local concern
const usePaymentStore = create({ modalOpen: false });   // forces rerender everywhere

// ✅ Local state — only this component re-renders
function PaymentDetailModal({ paymentId }) {
  const [open, setOpen] = useState(false);
  return (
    <>
      <button onClick={() => setOpen(true)}>View</button>
      {open && <Modal paymentId={paymentId} onClose={() => setOpen(false)} />}
    </>
  );
}
```

### BP3. Derive State, Don't Sync State

If a value can be computed from existing state or props, compute it — don't copy it into separate state. State synchronisation creates bugs (stale values, missed updates).

```tsx
// ❌ Synchronising derived state (prone to bugs)
const [items, setItems] = useState([]);
const [total, setTotal] = useState(0);

useEffect(() => {
  setTotal(items.reduce((sum, item) => sum + item.amount, 0));
}, [items]);

// ✅ Derive during render — always in sync, no extra state
const [items, setItems] = useState([]);
const total = items.reduce((sum, item) => sum + item.amount, 0);
```

### BP4. Use TypeScript Strictly

All components, hooks, and API types should be fully typed. Avoid `any`.

```tsx
// ✅ Typed component props
interface PaymentCardProps {
  payment: Payment;
  onApprove: (paymentId: string) => Promise<void>;
  isReadOnly?: boolean;
}

function PaymentCard({ payment, onApprove, isReadOnly = false }: PaymentCardProps) {
  // ...
}

// ✅ Typed event handlers
const handleAmountChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  setAmount(e.target.value);
};

// ✅ Typed refs
const inputRef = useRef<HTMLInputElement>(null);
```

### BP5. Handle All Async States Explicitly

Every data-fetching operation has four states: idle, loading, error, success. Handle all of them.

```tsx
function PaymentList() {
  const { data, isPending, isError, error } = useQuery({
    queryKey: ['payments'],
    queryFn: fetchPayments,
  });

  if (isPending) return <LoadingSkeleton />;
  if (isError)   return <ErrorMessage message={error.message} />;
  if (!data?.length) return <EmptyState message="No payments found" />;

  return <ul>{data.map(p => <PaymentRow key={p.id} payment={p} />)}</ul>;
}
```

### BP6. Never Forget Cleanup in Effects

Any effect that registers a listener, opens a connection, or starts a timer must clean it up.

```tsx
useEffect(() => {
  const controller = new AbortController();

  fetch('/api/payments', { signal: controller.signal })
    .then(res => res.json())
    .then(setPayments)
    .catch(err => { if (err.name !== 'AbortError') setError(err); });

  return () => controller.abort();   // cancel fetch on unmount or re-run
}, []);
```

### BP7. Accessibility First

React makes it easy to create inaccessible UIs by omitting ARIA roles, labels, and keyboard handlers. Treat accessibility as architecture.

```tsx
// ❌ div soup — invisible to screen readers
<div onClick={handleApprove}>Approve</div>

// ✅ Semantic, keyboard-accessible, labelled
<button
  type="button"
  onClick={handleApprove}
  aria-label={`Approve payment ${payment.id}`}
  disabled={isProcessing}
>
  {isProcessing ? <Spinner /> : 'Approve'}
</button>
```

### BP8. Keep Effects Out of User Event Handlers

If something happens in response to a user action — use the event handler, not an effect. Effects are for *synchronisation*, event handlers are for *responding to user input*.

```tsx
// ❌ Using an effect to respond to user action
const [submitted, setSubmitted] = useState(false);

useEffect(() => {
  if (submitted) {
    submitPayment(data);
    setSubmitted(false);
  }
}, [submitted]);

// ✅ Run directly in the event handler
const handleSubmit = async (e: React.FormEvent) => {
  e.preventDefault();
  await submitPayment(data);
};
```

---

## Design Patterns

### DP1. Custom Hook (Logic Extraction Pattern)

Extract reusable stateful logic into a custom hook. A custom hook is any function that starts with `use` and may call other hooks.

**When to use:** Logic shared across multiple components; complex effect + state combinations; abstracting over external libraries.

```tsx
// Custom hook: encapsulates all payment fetching logic
function usePayments(status?: PaymentStatus) {
  return useQuery({
    queryKey: ['payments', status],
    queryFn: () => paymentApi.list({ status }),
    select: (data) => data.sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime()),
  });
}

// Custom hook: encapsulates payment creation mutation
function useCreatePayment() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: paymentApi.create,
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['payments'] }),
  });
}

// Component is now clean — no data-fetching logic
function PaymentList() {
  const { data: payments, isPending } = usePayments('pending');
  const { mutate: createPayment } = useCreatePayment();
  // ...
}
```

**Rules for custom hooks:**
- Name starts with `use`
- May call React hooks
- May not be called conditionally or inside loops
- Returns values or functions that components need

---

### DP2. Compound Components Pattern

A group of related components that share implicit state through context. The parent manages state; child components consume it without explicit prop-drilling.

**When to use:** Tabs, accordions, dropdowns, multi-step forms — any component with tight internal coordination.

```tsx
// Context (private — not exported)
interface TabsContextValue {
  activeTab: string;
  setActiveTab: (tab: string) => void;
}
const TabsContext = createContext<TabsContextValue | null>(null);

function useTabs() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('useTabs must be used within <Tabs>');
  return ctx;
}

// Parent component
function Tabs({ defaultTab, children }: { defaultTab: string; children: ReactNode }) {
  const [activeTab, setActiveTab] = useState(defaultTab);
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div role="tablist">{children}</div>
    </TabsContext.Provider>
  );
}

// Sub-components — consume context implicitly
function Tab({ id, children }: { id: string; children: ReactNode }) {
  const { activeTab, setActiveTab } = useTabs();
  return (
    <button
      role="tab"
      aria-selected={activeTab === id}
      onClick={() => setActiveTab(id)}
    >
      {children}
    </button>
  );
}

function TabPanel({ id, children }: { id: string; children: ReactNode }) {
  const { activeTab } = useTabs();
  if (activeTab !== id) return null;
  return <div role="tabpanel">{children}</div>;
}

// Attach as static properties for ergonomic API
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// Usage — reads like natural HTML, state is invisible to caller
<Tabs defaultTab="details">
  <Tabs.Tab id="details">Details</Tabs.Tab>
  <Tabs.Tab id="history">History</Tabs.Tab>
  <Tabs.Panel id="details"><PaymentDetails /></Tabs.Panel>
  <Tabs.Panel id="history"><PaymentHistory /></Tabs.Panel>
</Tabs>
```

---

### DP3. Render Props / Children as Function

Pass a function as `children` (or a named prop) so the parent component controls *what* to render while the child controls *how* and *when* to call the function.

**When to use:** Sharing stateful logic with rendering flexibility; legacy codebases before custom hooks; libraries like `react-hook-form` `Controller`.

```tsx
// Data provider using children-as-function
interface DataProviderProps<T> {
  queryKey: string[];
  queryFn: () => Promise<T[]>;
  children: (data: T[], isPending: boolean) => ReactNode;
}

function DataProvider<T>({ queryKey, queryFn, children }: DataProviderProps<T>) {
  const { data = [], isPending } = useQuery({ queryKey, queryFn });
  return <>{children(data, isPending)}</>;
}

// Usage — caller decides how to render the data
<DataProvider queryKey={['payments']} queryFn={fetchPayments}>
  {(payments, isPending) =>
    isPending
      ? <Skeleton />
      : <PaymentTable payments={payments} />
  }
</DataProvider>
```

---

### DP4. Provider Pattern (Context + Custom Hook)

Encapsulate global or subtree-scoped state in a Context Provider, exposed through a custom hook that validates usage.

**When to use:** Auth state, theme, locale, feature flags — any value needed by many components at different nesting levels.

```tsx
// auth-context.tsx
interface AuthContextValue {
  user: User | null;
  isAuthenticated: boolean;
  login: (credentials: Credentials) => Promise<void>;
  logout: () => void;
}

const AuthContext = createContext<AuthContextValue | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  const login = async (credentials: Credentials) => {
    const user = await authApi.login(credentials);
    setUser(user);
  };

  const logout = () => {
    authApi.logout();
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, isAuthenticated: !!user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// Custom hook with usage guard
export function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within <AuthProvider>');
  return ctx;
}

// Usage
function ProfileMenu() {
  const { user, logout } = useAuth();
  return <button onClick={logout}>{user?.name}</button>;
}
```

---

### DP5. Higher-Order Component (HOC)

A function that takes a component and returns a new component with additional behaviour. Used for cross-cutting concerns (auth guard, analytics tracking, feature flag gating).

**When to use:** Wrapping third-party components; class component ecosystems; code that can't be refactored to hooks. Prefer custom hooks for new code.

```tsx
// withAuthGuard HOC — redirects unauthenticated users
function withAuthGuard<P extends object>(Component: ComponentType<P>) {
  return function AuthGuardedComponent(props: P) {
    const { isAuthenticated } = useAuth();
    const navigate = useNavigate();

    useEffect(() => {
      if (!isAuthenticated) navigate('/login', { replace: true });
    }, [isAuthenticated, navigate]);

    if (!isAuthenticated) return null;
    return <Component {...props} />;
  };
}

// Usage
const ProtectedPaymentDashboard = withAuthGuard(PaymentDashboard);
```

---

### DP6. Controlled vs. Uncontrolled Components

**Controlled:** React state drives the input value — full control, enables validation and transformation.

**Uncontrolled:** DOM manages the value — the ref reads it when needed. Simpler for large forms; used by React Hook Form internally for performance.

```tsx
// Controlled — React is the source of truth
function ControlledInput() {
  const [value, setValue] = useState('');
  return (
    <input
      value={value}
      onChange={e => setValue(e.target.value)}
    />
  );
}

// Uncontrolled — DOM is the source of truth (accessed via ref)
function UncontrolledInput() {
  const inputRef = useRef<HTMLInputElement>(null);
  const handleSubmit = () => console.log(inputRef.current?.value);
  return <input ref={inputRef} defaultValue="" />;
}

// Best practice: React Hook Form (uncontrolled internally, controlled API externally)
const { register, handleSubmit } = useForm<PaymentFormData>();
return (
  <form onSubmit={handleSubmit(onSubmit)}>
    <input {...register('amount', { required: true, min: 0.01 })} />
  </form>
);
```

---

### DP7. Error Boundary Pattern

React Error Boundaries are class components that catch render-time errors in their subtree and display a fallback UI instead of crashing the page.

**When to use:** Wrap every major UI region (each route, each critical widget) with an error boundary.

```tsx
// Generic error boundary (class component — required for componentDidCatch)
class ErrorBoundary extends Component<
  { fallback: ReactNode; children: ReactNode },
  { hasError: boolean; error: Error | null }
> {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    // Report to Sentry, Datadog, etc.
    logger.error('UI Error', { error, componentStack: info.componentStack });
  }

  render() {
    if (this.state.hasError) return this.props.fallback;
    return this.props.children;
  }
}

// Functional wrapper using react-error-boundary library
import { ErrorBoundary } from 'react-error-boundary';

function PaymentApp() {
  return (
    <ErrorBoundary
      fallbackRender={({ error, resetErrorBoundary }) => (
        <ErrorCard message={error.message} onRetry={resetErrorBoundary} />
      )}
      onError={(error, info) => reportError(error, info)}
    >
      <PaymentDashboard />
    </ErrorBoundary>
  );
}
```

---

### DP8. Suspense + Lazy Loading (Code Splitting)

`React.lazy` + `Suspense` enables component-level code splitting — the component's JS bundle is only loaded when the component is first rendered.

```tsx
// Lazy load heavy pages — each gets its own JS chunk
const PaymentDashboard = lazy(() => import('./pages/PaymentDashboard'));
const AnalyticsDashboard = lazy(() => import('./pages/AnalyticsDashboard'));
const AdminPanel = lazy(() => import('./pages/AdminPanel'));

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/payments"  element={<PaymentDashboard />} />
        <Route path="/analytics" element={<AnalyticsDashboard />} />
        <Route path="/admin"     element={<AdminPanel />} />
      </Routes>
    </Suspense>
  );
}
```

---

### DP9. State Machine Pattern (XState / useReducer)

Model complex UI states as a finite state machine — a set of well-defined states and legal transitions between them.

**When to use:** Multi-step forms, checkout flows, onboarding wizards, payment state machines (idle → pending → success / failed / cancelled).

```tsx
// Payment state machine using useReducer
type PaymentState =
  | { status: 'idle' }
  | { status: 'submitting' }
  | { status: 'success'; transactionId: string }
  | { status: 'error'; message: string };

type PaymentAction =
  | { type: 'SUBMIT' }
  | { type: 'SUCCESS'; transactionId: string }
  | { type: 'FAILURE'; message: string }
  | { type: 'RESET' };

function paymentReducer(state: PaymentState, action: PaymentAction): PaymentState {
  switch (action.type) {
    case 'SUBMIT':  return { status: 'submitting' };
    case 'SUCCESS': return { status: 'success', transactionId: action.transactionId };
    case 'FAILURE': return { status: 'error', message: action.message };
    case 'RESET':   return { status: 'idle' };
    default:        return state;
  }
}

function PaymentForm() {
  const [state, dispatch] = useReducer(paymentReducer, { status: 'idle' });

  const handleSubmit = async (data: PaymentFormData) => {
    dispatch({ type: 'SUBMIT' });
    try {
      const { transactionId } = await paymentApi.submit(data);
      dispatch({ type: 'SUCCESS', transactionId });
    } catch (err) {
      dispatch({ type: 'FAILURE', message: (err as Error).message });
    }
  };

  if (state.status === 'success')
    return <ConfirmationScreen transactionId={state.transactionId} />;

  return (
    <form onSubmit={handleSubmit}>
      {state.status === 'error' && <ErrorBanner message={state.message} />}
      <SubmitButton isLoading={state.status === 'submitting'} />
    </form>
  );
}
```

---

### DP10. Smart / Dumb (Container / Presentational) Pattern

Separate components that know about data from components that only render UI. "Smart" components fetch data, manage state, and handle business logic. "Dumb" (presentational) components receive everything via props and emit events.

```tsx
// Dumb — pure UI, no data dependencies, fully testable
interface PaymentCardProps {
  amount: number;
  currency: string;
  status: PaymentStatus;
  onApprove: () => void;
  onReject: () => void;
}
function PaymentCard({ amount, currency, status, onApprove, onReject }: PaymentCardProps) {
  return (
    <div className="payment-card">
      <span>{formatCurrency(amount, currency)}</span>
      <StatusBadge status={status} />
      <button onClick={onApprove}>Approve</button>
      <button onClick={onReject}>Reject</button>
    </div>
  );
}

// Smart — knows about data, delegates rendering
function PaymentCardContainer({ paymentId }: { paymentId: string }) {
  const { data: payment } = usePayment(paymentId);
  const { mutate: approve } = useApprovePayment();
  const { mutate: reject }  = useRejectPayment();

  if (!payment) return <Skeleton />;
  return (
    <PaymentCard
      {...payment}
      onApprove={() => approve(paymentId)}
      onReject={() => reject(paymentId)}
    />
  );
}
```

---

## Anti-Patterns

### AP1. Over-Fetching with useEffect for Data Loading

Using `useEffect` + `useState` for data fetching creates boilerplate, race conditions, and missing states. It's the most commonly seen React anti-pattern.

```tsx
// ❌ Manual data fetching — race condition on fast navigation, no caching, no error handling
function PaymentList() {
  const [payments, setPayments] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    setLoading(true);
    fetch('/api/payments')
      .then(res => res.json())
      .then(data => {
        setPayments(data);     // stale closure + race condition if component unmounts
        setLoading(false);
      });
  }, []);
}

// ✅ Use TanStack Query — caching, deduplication, background refresh, race-condition-free
function PaymentList() {
  const { data: payments = [], isPending } = useQuery({
    queryKey: ['payments'],
    queryFn: () => fetch('/api/payments').then(r => r.json()),
  });
}
```

---

### AP2. Stale Closure in useEffect

Forgetting to include values used inside an effect in the dependency array causes the effect to read stale state from the closure.

```tsx
// ❌ Stale closure — selectedId inside the effect is always the initial value
const [selectedId, setSelectedId] = useState('');

useEffect(() => {
  const subscription = eventBus.subscribe('payment:updated', (id) => {
    if (id === selectedId) {    // selectedId is stale!
      refetch();
    }
  });
  return () => subscription.unsubscribe();
}, []);   // ← missing selectedId

// ✅ Include all dependencies
useEffect(() => {
  const subscription = eventBus.subscribe('payment:updated', (id) => {
    if (id === selectedId) refetch();
  });
  return () => subscription.unsubscribe();
}, [selectedId, refetch]);  // re-subscribe when selectedId changes
```

---

### AP3. Prop Drilling Through Many Layers

Passing props through intermediary components that don't use them creates tight coupling and makes refactoring painful.

```tsx
// ❌ Prop drilling — theme passed through 4 layers; middle components don't use it
<App theme={theme}>
  <Layout theme={theme}>
    <Sidebar theme={theme}>
      <NavItem theme={theme} />    {/* only this component actually uses theme */}
    </Sidebar>
  </Layout>
</App>

// ✅ Context — only the component that needs it reads from context
const ThemeContext = createContext<Theme>(defaultTheme);

function App() {
  return (
    <ThemeContext.Provider value={theme}>
      <Layout>          {/* no theme prop */}
        <Sidebar>       {/* no theme prop */}
          <NavItem />   {/* reads from context directly */}
        </Sidebar>
      </Layout>
    </ThemeContext.Provider>
  );
}
```

---

### AP4. Using Array Index as List Key

Key instability on filtered, sorted, or reordered lists causes React to re-render and lose state incorrectly.

```tsx
// ❌ Index keys break when the list is filtered, sorted, or reordered
{filteredPayments.map((p, i) => <PaymentRow key={i} payment={p} />)}

// ✅ Use the entity's stable unique ID
{filteredPayments.map(p => <PaymentRow key={p.id} payment={p} />)}

// If data has no IDs (e.g. a static options list that never reorders), index is acceptable:
{STATIC_CURRENCIES.map((c, i) => <option key={i} value={c.code}>{c.name}</option>)}
```

---

### AP5. Overusing useMemo / useCallback

Premature optimisation with `useMemo` and `useCallback` adds complexity and maintenance cost with minimal benefit. Most components re-render fast enough without memoisation.

```tsx
// ❌ Memoising cheap computations — overhead of useMemo > cost of computation
const label = useMemo(() => `${user.firstName} ${user.lastName}`, [user]);

// ✅ Just compute it inline — string concatenation is nanoseconds
const label = `${user.firstName} ${user.lastName}`;

// ✅ useMemo is justified when the computation is genuinely expensive
const reportData = useMemo(
  () => processTransactions(rawData),   // transforms 100K rows
  [rawData]
);

// ✅ useCallback is justified when the function is passed to a memoised child
// that would re-render unnecessarily without a stable reference
const handleApprove = useCallback(
  (paymentId: string) => approvePayment(paymentId),
  [approvePayment]
);
```

---

### AP6. Mutating State Directly

React state must be treated as immutable. Direct mutation does not trigger re-render.

```tsx
// ❌ Direct mutation — React does not detect the change, no re-render
const [payments, setPayments] = useState<Payment[]>([]);
payments.push(newPayment);       // mutates array — React doesn't know
setPayments(payments);           // same reference — React bails out

// ✅ Return a new reference
setPayments(prev => [...prev, newPayment]);

// ❌ Mutating nested object
const [user, setUser] = useState<User>({ name: 'Alice', address: { city: 'NYC' } });
user.address.city = 'London';  // mutation
setUser(user);                 // same reference — no re-render

// ✅ Return new nested reference
setUser(prev => ({
  ...prev,
  address: { ...prev.address, city: 'London' },
}));
```

---

### AP7. Deriving State with an Effect (setState in useEffect)

Setting state inside a `useEffect` in response to another state change creates an extra render cycle — and is often a sign that the derived value should be computed during render instead.

```tsx
// ❌ Extra render cycle for no benefit
const [items, setItems] = useState<Item[]>([]);
const [total, setTotal] = useState(0);

useEffect(() => {
  setTotal(items.reduce((s, i) => s + i.price, 0));  // causes a second render
}, [items]);

// ✅ Derive during render — zero extra renders, always correct
const total = items.reduce((s, i) => s + i.price, 0);
```

---

### AP8. Huge Context Value Causing Unnecessary Re-Renders

Every consumer of a context re-renders when *any* part of the context value changes — even if the individual consumer only uses one field.

```tsx
// ❌ One context for everything — a locale change re-renders components using auth state
const AppContext = createContext({ user, locale, theme, setUser, setLocale, setTheme });

// ✅ Split contexts by change frequency
const AuthContext   = createContext({ user, setUser });      // changes on login/logout
const ThemeContext  = createContext({ theme, setTheme });    // changes on theme toggle
const LocaleContext = createContext({ locale, setLocale });  // changes on language switch
```

---

### AP9. Not Cleaning Up Effects (Memory Leaks)

Effects that subscribe without unsubscribing, or set state after the component unmounts, cause memory leaks and spurious state updates.

```tsx
// ❌ No cleanup — subscription leaks; state update after unmount = warning
useEffect(() => {
  const ws = new WebSocket('wss://api.example.com/payments');
  ws.onmessage = (e) => setPayments(JSON.parse(e.data));   // may fire after unmount
  // no cleanup!
}, []);

// ✅ Always close/unsubscribe in the cleanup function
useEffect(() => {
  const ws = new WebSocket('wss://api.example.com/payments');
  ws.onmessage = (e) => setPayments(JSON.parse(e.data));
  return () => ws.close();   // cleanup on unmount
}, []);
```

---

### AP10. Not Memoising Heavy Computed Lists Shared With Many Consumers

The opposite of AP5: when a genuinely expensive computation or a large derived array is consumed by many components, not memoising it forces recomputation on every parent re-render.

```tsx
// ❌ Re-processes 10,000 transactions on every parent render
function AnalyticsDashboard({ transactions }: { transactions: Transaction[] }) {
  // This runs on every render — even when unrelated state changes
  const chartData = buildChartSeries(transactions);

  return <BarChart data={chartData} />;
}

// ✅ Memoise expensive transformation
function AnalyticsDashboard({ transactions }: { transactions: Transaction[] }) {
  const chartData = useMemo(() => buildChartSeries(transactions), [transactions]);
  return <BarChart data={chartData} />;
}
```

---

## Quick Reference Card

### Hooks Decision Tree

```
I need to store UI state that triggers re-render
  └── useState (simple value) or useReducer (complex object / state machine)

I need a mutable value without triggering re-render
  └── useRef

I need to synchronise with something outside React
  └── useEffect (with cleanup)

I need to share state across many components
  └── Context + useContext — or Zustand for global app state

I need remote data from an API
  └── TanStack Query (useQuery / useMutation)

I need an expensive computation to not re-run on every render
  └── useMemo — only after profiling confirms a problem

I need a stable function reference for a memoised child
  └── useCallback

The user should see stale content while new content loads
  └── useTransition + isPending (mark the update as non-urgent)
```

### Category vs. Pattern

| # | Category | Pattern | Use Case |
|---|----------|---------|---------|
| 1 | Logic reuse | Custom Hook | Shared stateful logic, data fetching |
| 2 | Component composition | Compound Components | Tabs, menus, accordions |
| 3 | Rendering flexibility | Render Props | Dynamic rendering controlled by caller |
| 4 | State sharing | Provider + Context | Auth, theme, locale |
| 5 | Cross-cutting concerns | HOC | Auth guard, analytics, feature gating |
| 6 | Input handling | Controlled component | Real-time validation + React Hook Form |
| 7 | Resilience | Error Boundary | Per-route / per-widget crash isolation |
| 8 | Bundle size | Suspense + lazy | Route-level code splitting |
| 9 | Complex state flow | useReducer state machine | Checkout, multi-step forms, payment flow |
| 10 | Separation of concerns | Smart / Dumb split | Testable presentational components |

#### 1. Custom Hook — Shared stateful logic, data fetching

Extract logic shared across components (debouncing, WebSocket connection, polling) into a reusable hook.

```tsx
// useDebounce — reused in search, autocomplete, filter inputs
function useDebounce<T>(value: T, delayMs: number): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delayMs);
    return () => clearTimeout(timer);
  }, [value, delayMs]);

  return debounced;
}

// One-liner usage anywhere
function SearchBar() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);   // only fires API after 300 ms idle

  const { data } = useQuery({
    queryKey: ['search', debouncedQuery],
    queryFn: () => searchApi.query(debouncedQuery),
    enabled: debouncedQuery.length > 1,
  });
  // ...
}
```

#### 2. Compound Components — Tabs, menus, accordions

A parent component owns state; child sub-components consume it via context — no prop-drilling, reads like natural HTML.

```tsx
// Accordion built with Compound Components
const AccordionContext = createContext<{
  openId: string | null;
  toggle: (id: string) => void;
} | null>(null);

function Accordion({ children }: { children: ReactNode }) {
  const [openId, setOpenId] = useState<string | null>(null);
  const toggle = (id: string) => setOpenId(prev => (prev === id ? null : id));
  return (
    <AccordionContext.Provider value={{ openId, toggle }}>
      <div>{children}</div>
    </AccordionContext.Provider>
  );
}

function AccordionItem({ id, title, children }: { id: string; title: string; children: ReactNode }) {
  const ctx = useContext(AccordionContext)!;
  const isOpen = ctx.openId === id;
  return (
    <div>
      <button aria-expanded={isOpen} onClick={() => ctx.toggle(id)}>{title}</button>
      {isOpen && <div role="region">{children}</div>}
    </div>
  );
}

Accordion.Item = AccordionItem;

// Usage — zero prop drilling
<Accordion>
  <Accordion.Item id="fees" title="Fee Schedule"><FeeTable /></Accordion.Item>
  <Accordion.Item id="limits" title="Transfer Limits"><LimitsTable /></Accordion.Item>
</Accordion>
```

#### 3. Render Props — Dynamic rendering controlled by caller

The child component handles *when* to call the function; the caller decides *what* to render. Best for headless behaviour components.

```tsx
// Headless mouse-tracker — behaviour only, no UI opinions
function MouseTracker({ children }: { children: (pos: { x: number; y: number }) => ReactNode }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });

  useEffect(() => {
    const handler = (e: MouseEvent) => setPos({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);

  return <>{children(pos)}</>;
}

// Caller decides what to render
<MouseTracker>
  {({ x, y }) => <Tooltip style={{ left: x + 12, top: y + 12 }}>x={x}, y={y}</Tooltip>}
</MouseTracker>
```

#### 4. Provider + Context — Auth, theme, locale

Centralise subtree-scoped state (auth, theme, locale) behind a Provider and expose it via a validated custom hook.

```tsx
// Theme context — avoids prop-drilling colour mode through every component
type Theme = 'light' | 'dark';
const ThemeContext = createContext<{ theme: Theme; toggle: () => void } | null>(null);

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light');
  const toggle = () => setTheme(t => (t === 'light' ? 'dark' : 'light'));

  useEffect(() => {
    document.documentElement.dataset.theme = theme;   // CSS [data-theme] selector
  }, [theme]);

  return <ThemeContext.Provider value={{ theme, toggle }}>{children}</ThemeContext.Provider>;
}

export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be used within <ThemeProvider>');
  return ctx;
}

// Any component, at any depth
function Header() {
  const { theme, toggle } = useTheme();
  return <button onClick={toggle}>{theme === 'light' ? '🌙' : '☀️'}</button>;
}
```

#### 5. HOC — Auth guard, analytics, feature gating

Wraps a component with additional cross-cutting behaviour without modifying its source. Use sparingly — prefer custom hooks for new code.

```tsx
// withFeatureFlag HOC — gates a component behind a feature flag
function withFeatureFlag<P extends object>(flag: string, Component: ComponentType<P>) {
  return function FlaggedComponent(props: P) {
    const flags = useFeatureFlags();                 // custom hook reading flag config
    if (!flags[flag]) return null;                  // or render a fallback
    return <Component {...props} />;
  };
}

// Analytics HOC — tracks mount/unmount events automatically
function withAnalytics<P extends object>(eventName: string, Component: ComponentType<P>) {
  return function TrackedComponent(props: P) {
    useEffect(() => {
      analytics.track(`${eventName}.viewed`);
      return () => analytics.track(`${eventName}.hidden`);
    }, []);
    return <Component {...props} />;
  };
}

// Compose multiple HOCs
const ProtectedAnalyticsPayments = withFeatureFlag(
  'payments-v2',
  withAnalytics('payments-dashboard', PaymentDashboard)
);
```

#### 6. Controlled Component — Real-time validation + React Hook Form

React state is the single source of truth for the input value, enabling real-time validation and transformation.

```tsx
// Real-time IBAN format + validation
function IbanInput({ onValidIban }: { onValidIban: (iban: string) => void }) {
  const [raw, setRaw] = useState('');
  const [error, setError] = useState<string | null>(null);

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    // Format: insert space every 4 chars
    const formatted = e.target.value.replace(/\s/g, '').match(/.{1,4}/g)?.join(' ') ?? '';
    setRaw(formatted);

    const stripped = formatted.replace(/\s/g, '');
    if (stripped.length === 22 && isValidIban(stripped)) {
      setError(null);
      onValidIban(stripped);
    } else if (stripped.length > 0) {
      setError('Invalid IBAN');
    }
  };

  return (
    <div>
      <input
        value={raw}
        onChange={handleChange}
        aria-invalid={!!error}
        aria-describedby="iban-error"
        placeholder="GB29 NWBK 6016 1331 9268 19"
      />
      {error && <span id="iban-error" role="alert">{error}</span>}
    </div>
  );
}
```

#### 7. Error Boundary — Per-route / per-widget crash isolation

Wrap each independent UI region so a crash in one widget does not bring down the whole page.

```tsx
import { ErrorBoundary } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }: FallbackProps) {
  return (
    <div role="alert" className="error-card">
      <p>Something went wrong: {error.message}</p>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

// Wrap each major region independently
function DashboardLayout() {
  return (
    <div className="dashboard-grid">
      <ErrorBoundary FallbackComponent={ErrorFallback}>
        <PaymentSummaryWidget />          {/* crash here doesn't affect siblings */}
      </ErrorBoundary>
      <ErrorBoundary FallbackComponent={ErrorFallback}>
        <RecentTransactionsWidget />
      </ErrorBoundary>
      <ErrorBoundary FallbackComponent={ErrorFallback}>
        <AnalyticsChartWidget />
      </ErrorBoundary>
    </div>
  );
}
```

#### 8. Suspense + Lazy — Route-level code splitting

`React.lazy` delays downloading a component's JS until it is first rendered, reducing the initial bundle.

```tsx
const PaymentsPage  = lazy(() => import('./pages/Payments'));
const TransfersPage = lazy(() => import('./pages/Transfers'));
const SettingsPage  = lazy(() => import('./pages/Settings'));

// Preload on hover — feels instant when user clicks
const preloadPayments = () => import('./pages/Payments');

function App() {
  return (
    // Single Suspense boundary for all routes — shows skeleton during navigation
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/payments"  element={<PaymentsPage />} />
        <Route path="/transfers" element={<TransfersPage />} />
        <Route path="/settings"  element={<SettingsPage />} />
      </Routes>
    </Suspense>
  );
}

// Nav link preloads chunk on hover — zero-latency perceived navigation
<Link to="/payments" onMouseEnter={preloadPayments}>Payments</Link>
```

#### 9. useReducer State Machine — Checkout, multi-step forms, payment flow

Model each UI stage as a named state with explicit legal transitions — impossible states become impossible values.

```tsx
// Multi-step checkout: step machine + data accumulation
type CheckoutStep = 'cart' | 'shipping' | 'payment' | 'review' | 'confirmed';

interface CheckoutState {
  step: CheckoutStep;
  shipping: ShippingDetails | null;
  paymentMethod: PaymentMethod | null;
}

type CheckoutAction =
  | { type: 'GO_TO_SHIPPING' }
  | { type: 'SAVE_SHIPPING'; details: ShippingDetails }
  | { type: 'SAVE_PAYMENT'; method: PaymentMethod }
  | { type: 'CONFIRM' }
  | { type: 'RESET' };

const INITIAL: CheckoutState = { step: 'cart', shipping: null, paymentMethod: null };

function checkoutReducer(state: CheckoutState, action: CheckoutAction): CheckoutState {
  switch (action.type) {
    case 'GO_TO_SHIPPING': return { ...state, step: 'shipping' };
    case 'SAVE_SHIPPING':  return { ...state, step: 'payment', shipping: action.details };
    case 'SAVE_PAYMENT':   return { ...state, step: 'review',  paymentMethod: action.method };
    case 'CONFIRM':        return { ...state, step: 'confirmed' };
    case 'RESET':          return INITIAL;
    default:               return state;
  }
}

function CheckoutFlow() {
  const [state, dispatch] = useReducer(checkoutReducer, INITIAL);

  const stepMap: Record<CheckoutStep, ReactNode> = {
    cart:      <CartStep      onNext={() => dispatch({ type: 'GO_TO_SHIPPING' })} />,
    shipping:  <ShippingStep  onNext={d  => dispatch({ type: 'SAVE_SHIPPING', details: d })} />,
    payment:   <PaymentStep   onNext={m  => dispatch({ type: 'SAVE_PAYMENT',  method: m  })} />,
    review:    <ReviewStep    state={state} onConfirm={() => dispatch({ type: 'CONFIRM' })} />,
    confirmed: <ConfirmedStep onReset={() => dispatch({ type: 'RESET' })} />,
  };

  return <>{stepMap[state.step]}</>;
}
```

#### 10. Smart / Dumb Split — Testable presentational components

Push all data concerns into Container (Smart) components; keep Presentational (Dumb) components prop-only so they can be snapshot-tested and used in Storybook without any mocking.

```tsx
// Dumb — zero side effects, fully testable in isolation
interface TransactionRowProps {
  id: string;
  amount: number;
  currency: string;
  counterparty: string;
  status: 'pending' | 'completed' | 'failed';
  onRetry?: () => void;
}

function TransactionRow({ id, amount, currency, counterparty, status, onRetry }: TransactionRowProps) {
  return (
    <tr>
      <td>{id}</td>
      <td>{formatCurrency(amount, currency)}</td>
      <td>{counterparty}</td>
      <td><StatusBadge status={status} /></td>
      <td>{status === 'failed' && <button onClick={onRetry}>Retry</button>}</td>
    </tr>
  );
}

// Smart — handles data, delegates rendering to Dumb
function TransactionRowContainer({ transactionId }: { transactionId: string }) {
  const { data }         = useTransaction(transactionId);
  const { mutate: retry } = useRetryTransaction();

  if (!data) return <tr><td colSpan={5}><Skeleton /></td></tr>;

  return (
    <TransactionRow
      {...data}
      onRetry={data.status === 'failed' ? () => retry(transactionId) : undefined}
    />
  );
}

// In Storybook — no mocking needed for the Dumb component
export const Failed: Story = {
  args: { id: 'txn_001', amount: 150, currency: 'GBP', counterparty: 'ACME', status: 'failed' },
};
```

### Common Anti-Patterns at a Glance

| # | Anti-Pattern | Tell-tale Sign | Fix |
|---|-------------|---------------|-----|
| 1 | useEffect for data fetching | `useEffect(() => { fetch... setLoading... })` | TanStack Query |
| 2 | Stale closure | Effect reads state that's always the initial value | Add missing deps to dep array |
| 3 | Prop drilling | 3+ levels of unused props | Context or Zustand |
| 4 | Index as key | `key={index}` in a filterable list | `key={item.id}` |
| 5 | Premature useMemo | `useMemo` on string concat or simple math | Remove it |
| 6 | Direct state mutation | `array.push` then `setArray(array)` | `setArray(prev => [...prev, item])` |
| 7 | setState in effect for derived value | `useEffect(() => setSomething(compute(state)))` | Compute during render |
| 8 | God context | Single context for all app state | Split contexts by domain |
| 9 | Missing cleanup | `useEffect` with WebSocket / timer but no `return () => ...` | Add cleanup |
| 10 | Skipping memoisation for expensive ops | `buildChartSeries(10kRows)` on every render | `useMemo` |

---

#### AP1. useEffect for Data Fetching

```tsx
// ❌ Manual fetch — race condition, no caching, missing error/loading states
function PaymentList() {
  const [payments, setPayments] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    setLoading(true);
    fetch('/api/payments')
      .then(res => res.json())
      .then(data => {
        setPayments(data);   // 🐛 component may have unmounted; stale if user navigates fast
        setLoading(false);
      });
  }, []);                    // 🐛 no cleanup, no error handling, no deduplication

  if (loading) return <Spinner />;
  return <Table rows={payments} />;
}

// ✅ TanStack Query — caching, deduplication, background refetch, race-condition-safe
function PaymentList() {
  const { data: payments = [], isPending, isError } = useQuery({
    queryKey: ['payments'],
    queryFn: () => fetch('/api/payments').then(res => res.json()),
  });

  if (isPending) return <Spinner />;
  if (isError)   return <ErrorMessage />;
  return <Table rows={payments} />;
}
```

---

#### AP2. Stale Closure

```tsx
// ❌ Stale closure — count inside the effect is captured at mount (always 0)
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(count);      // 🐛 always prints 0 — stale closure
      setCount(count + 1);     // 🐛 always sets to 1
    }, 1000);
    return () => clearInterval(id);
  }, []);                      // 🐛 missing count in deps

  return <p>{count}</p>;
}

// ✅ Option A — add missing dep (re-registers interval each tick)
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(id);
}, [count]);

// ✅ Option B (preferred) — functional update reads latest value, no dep needed
useEffect(() => {
  const id = setInterval(() => setCount(prev => prev + 1), 1000);
  return () => clearInterval(id);
}, []);   // safe — functional update never closes over stale state
```

---

#### AP3. Prop Drilling

```tsx
// ❌ Prop drilling — userId passed through layers that don't use it
function App() {
  const userId = useCurrentUserId();
  return <Layout userId={userId} />;
}
function Layout({ userId }: { userId: string }) {
  return <Sidebar userId={userId} />;          // Layout doesn't use userId
}
function Sidebar({ userId }: { userId: string }) {
  return <UserAvatar userId={userId} />;       // Sidebar doesn't use userId either
}
function UserAvatar({ userId }: { userId: string }) {
  const user = useUser(userId);                // Only this component actually needs it
  return <img src={user.avatarUrl} alt={user.name} />;
}

// ✅ Context — components read what they need directly
const UserContext = createContext<string | null>(null);

function App() {
  const userId = useCurrentUserId();
  return (
    <UserContext.Provider value={userId}>
      <Layout />                               // no prop needed
    </UserContext.Provider>
  );
}
function Layout()  { return <Sidebar />; }     // no prop needed
function Sidebar() { return <UserAvatar />; }  // no prop needed
function UserAvatar() {
  const userId = useContext(UserContext)!;      // reads directly from context
  const user = useUser(userId);
  return <img src={user.avatarUrl} alt={user.name} />;
}
```

---

#### AP4. Index as Key

```tsx
const payments = [
  { id: 'p1', label: 'Rent' },
  { id: 'p2', label: 'Utilities' },
  { id: 'p3', label: 'Groceries' },
];

// ❌ index as key — removing or reordering items confuses React's reconciler;
//    input values get mapped to the wrong rows after a delete
payments.map((p, index) => (
  <PaymentRow key={index} payment={p} />   // 🐛 after deleting p1: p2 gets key=0, p3 gets key=1
));

// ✅ stable, unique ID as key — React correctly identifies each row across re-renders
payments.map((p) => (
  <PaymentRow key={p.id} payment={p} />    // ✅ p2 always has key="p2"
));

// ✅ When no natural ID exists, generate one at creation time — never at render time
const [items, setItems] = useState(() =>
  initialItems.map(item => ({ ...item, id: crypto.randomUUID() }))
);
```

---

#### AP5. Premature useMemo

```tsx
// ❌ Wrapping trivial work in useMemo — the memoisation overhead exceeds the computation
function GreetingBanner({ firstName, lastName }: { firstName: string; lastName: string }) {
  // Concatenating two strings takes ~0 ms. useMemo adds memory allocation + comparison cost.
  const fullName = useMemo(() => `${firstName} ${lastName}`, [firstName, lastName]); // ❌

  const greeting = useMemo(() => greeting.toUpperCase(), [greeting]); // ❌ toUpperCase is O(n), not expensive

  return <h1>Welcome, {fullName}</h1>;
}

// ✅ Compute during render — no overhead, no indirection
function GreetingBanner({ firstName, lastName }: { firstName: string; lastName: string }) {
  const fullName = `${firstName} ${lastName}`;   // plain variable — fine
  return <h1>Welcome, {fullName}</h1>;
}

// ✅ useMemo IS justified here — filtering + sorting a large dataset
function TransactionTable({ transactions }: { transactions: Transaction[] }) {
  const sorted = useMemo(
    () => transactions
      .filter(t => t.status === 'completed')
      .sort((a, b) => b.amount - a.amount),
    [transactions]   // only re-runs when the array reference changes
  );
  return <Table rows={sorted} />;
}
```

---

#### AP6. Direct State Mutation

```tsx
// ❌ Mutating state directly — React doesn't detect the change because the reference is the same
function TagEditor() {
  const [tags, setTags] = useState(['urgent', 'review']);

  const addTag = (tag: string) => {
    tags.push(tag);     // 🐛 mutates the existing array — same reference
    setTags(tags);      // 🐛 React bails out — prevState === nextState
  };

  const removeTag = (index: number) => {
    tags.splice(index, 1);  // 🐛 same problem
    setTags(tags);
  };
  // ...
}

// ✅ Always produce a new reference
function TagEditor() {
  const [tags, setTags] = useState(['urgent', 'review']);

  const addTag = (tag: string) =>
    setTags(prev => [...prev, tag]);                         // new array

  const removeTag = (index: number) =>
    setTags(prev => prev.filter((_, i) => i !== index));    // new array

  // For objects — spread to copy, then override the changed field
  const [user, setUser] = useState({ name: 'Alice', role: 'admin' });
  const updateRole = (role: string) =>
    setUser(prev => ({ ...prev, role }));                    // new object
}
```

---

#### AP7. setState in Effect for Derived Value

```tsx
// ❌ Syncing derived state via useEffect — causes a double render per change
function PaymentSummary({ payments }: { payments: Payment[] }) {
  const [total, setTotal] = useState(0);

  useEffect(() => {
    setTotal(payments.reduce((sum, p) => sum + p.amount, 0));
    //       ^ triggers a second render after the first render with stale total
  }, [payments]);

  return <p>Total: {formatCurrency(total)}</p>;
}

// ✅ Compute during render — single render, no intermediate stale frame
function PaymentSummary({ payments }: { payments: Payment[] }) {
  const total = payments.reduce((sum, p) => sum + p.amount, 0);  // derived inline
  return <p>Total: {formatCurrency(total)}</p>;
}

// ✅ Expensive derivation — useMemo, not useEffect + setState
function PaymentSummary({ payments }: { payments: Payment[] }) {
  const { total, average, max } = useMemo(() => ({
    total:   payments.reduce((s, p) => s + p.amount, 0),
    average: payments.length ? payments.reduce((s, p) => s + p.amount, 0) / payments.length : 0,
    max:     Math.max(...payments.map(p => p.amount), 0),
  }), [payments]);

  return <dl><dt>Total</dt><dd>{total}</dd><dt>Avg</dt><dd>{average}</dd></dl>;
}
```

---

#### AP8. God Context

```tsx
// ❌ One giant context — any change to auth, theme, OR cart re-renders every consumer
interface AppState {
  user: User | null;
  theme: 'light' | 'dark';
  cart: CartItem[];
  notifications: Notification[];
  featureFlags: Record<string, boolean>;
}
const AppContext = createContext<AppState>({} as AppState);
// Every component that reads ANY of these re-renders when ANY field changes

// ✅ Split by domain — each context only re-renders its own consumers
const AuthContext    = createContext<AuthValue | null>(null);
const ThemeContext   = createContext<ThemeValue | null>(null);
const CartContext    = createContext<CartValue | null>(null);
const NotifContext   = createContext<NotifValue | null>(null);

// Compose providers at the root
function AppProviders({ children }: { children: ReactNode }) {
  return (
    <AuthProvider>
      <ThemeProvider>
        <CartProvider>
          <NotifProvider>{children}</NotifProvider>
        </CartProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}

// CartButton only re-renders when cart changes — auth/theme changes are invisible to it
function CartButton() {
  const { items } = useCart();   // only subscribed to CartContext
  return <button>Cart ({items.length})</button>;
}
```

---

#### AP9. Missing Cleanup

```tsx
// ❌ WebSocket opened but never closed — leak on every mount/re-mount
function LivePriceFeed({ symbol }: { symbol: string }) {
  const [price, setPrice] = useState<number | null>(null);

  useEffect(() => {
    const ws = new WebSocket(`wss://prices.example.com/${symbol}`);
    ws.onmessage = (e) => setPrice(JSON.parse(e.data).price);
    // 🐛 no cleanup — old connection lingers when symbol changes or component unmounts
  }, [symbol]);

  return <span>{price ?? '—'}</span>;
}

// ✅ Close the connection in the cleanup function
function LivePriceFeed({ symbol }: { symbol: string }) {
  const [price, setPrice] = useState<number | null>(null);

  useEffect(() => {
    const ws = new WebSocket(`wss://prices.example.com/${symbol}`);
    ws.onmessage = (e) => setPrice(JSON.parse(e.data).price);

    return () => {
      ws.close();   // ✅ runs when symbol changes (old ws closed) or component unmounts
    };
  }, [symbol]);

  return <span>{price ?? '—'}</span>;
}

// ✅ Same pattern for timers, event listeners, IntersectionObserver, etc.
useEffect(() => {
  const observer = new IntersectionObserver(callback, { threshold: 0.1 });
  observer.observe(targetRef.current!);
  return () => observer.disconnect();   // ✅ cleanup
}, []);
```

---

#### AP10. Skipping Memoisation for Expensive Operations

```tsx
// ❌ Heavy computation runs on every render — any unrelated state change (hover, tooltip)
//    triggers a full recalculation of 10k rows
function RevenueChart({ transactions }: { transactions: Transaction[] }) {
  // This runs on EVERY render of RevenueChart — including parent re-renders
  const series = buildChartSeries(transactions);     // 🐛 O(n log n), ~20ms on 10k rows
  const bands  = computeMovingAverage(transactions, 30);

  return <LineChart series={series} bands={bands} />;
}

// ✅ useMemo — computation only re-runs when transactions reference changes
function RevenueChart({ transactions }: { transactions: Transaction[] }) {
  const series = useMemo(
    () => buildChartSeries(transactions),
    [transactions]
  );
  const bands = useMemo(
    () => computeMovingAverage(transactions, 30),
    [transactions]
  );

  return <LineChart series={series} bands={bands} />;
}

// ✅ Also memo the component itself if its parent re-renders frequently
const RevenueChart = memo(function RevenueChart({ transactions }: { transactions: Transaction[] }) {
  const series = useMemo(() => buildChartSeries(transactions), [transactions]);
  const bands  = useMemo(() => computeMovingAverage(transactions, 30), [transactions]);
  return <LineChart series={series} bands={bands} />;
});
// React.memo skips re-rendering entirely when transactions prop reference is stable
```

---

## References

- [React Documentation (react.dev)](https://react.dev/)
- [React Patterns — Alex Kondov (Tao of React)](https://alexkondov.com/tao-of-react/)
- [React — Thinking in React](https://react.dev/learn/thinking-in-react)
- [React — You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- [React — Managing State](https://react.dev/learn/managing-state)
- [TanStack Query](https://tanstack.com/query/latest)
- [XState — State Machines for React](https://xstate.js.org/)
- [Kent C. Dodds — Compound Components](https://kentcdodds.com/blog/compound-components-with-react-hooks)
- [Kent C. Dodds — Application State Management](https://kentcdodds.com/blog/application-state-management-with-react)
- [Dan Abramov — Writing Resilient Components](https://overreacted.io/writing-resilient-components/)
