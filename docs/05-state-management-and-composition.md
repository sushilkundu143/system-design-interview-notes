# State Management and Composition Patterns in React

## 1. Mental model

State management answers **where data lives, who can change it, and how updates
reach the UI**. Composition answers **how small components and behaviors combine
into larger features without becoming tightly coupled**.

The two work together. A reusable modal should not need access to the entire
application store. A banking transfer workflow may need shared state across
several steps, but that state does not necessarily belong in a global store.

Start with these questions:

1. Who owns this value?
2. Which components need it?
3. How long must it survive?
4. Is it authoritative client data, remote data, or a derived value?
5. Does it need to appear in the URL or persist across sessions?

**Core principle:** Keep each piece of state in the narrowest appropriate scope,
and give it one clear source of truth.

## 2. Classify state before choosing a library

| State category | Example | Common owner |
| --- | --- | --- |
| Local UI state | Open menu, expanded row | Component |
| Shared feature state | Transfer draft across steps | Feature parent, reducer, or scoped provider |
| Global client state | Cross-feature workflow settings | Application store or narrowly scoped Context |
| Server state | Account list, transaction history | Server-state cache or framework data layer |
| URL state | Search, sort, page, selected tab | Router/search parameters |
| Form state | Input values, validation, dirty fields | Form component or form library |
| Derived state | Filtered list, total, validity | Computed from existing inputs |
| Persistent preferences | Preferred display mode | Storage-backed state with explicit lifecycle |

A value can cross categories. A transfer form selects an account from server
data, but the selected account ID is client-owned draft state.

Do not put everything in Redux simply because Redux exists in the application.
Do not copy server data into multiple stores without an explicit synchronization
strategy.

## 3. Local state and derived values

Use `useState` for simple component-owned values.

```jsx
function AccountSearch({ accounts }) {
  const [query, setQuery] = React.useState("");

  const visibleAccounts = accounts.filter((account) =>
    account.name.toLowerCase().includes(query.toLowerCase())
  );

  return (
    <section>
      <label htmlFor="account-search">Search accounts</label>
      <input
        id="account-search"
        value={query}
        onChange={(event) => setQuery(event.target.value)}
      />
      <ul>
        {visibleAccounts.map((account) => (
          <li key={account.id}>{account.name}</li>
        ))}
      </ul>
    </section>
  );
}
```

This is a standalone teaching example; an enterprise design system may provide
the actual input and list components.

`visibleAccounts` is derived. Storing it separately and synchronizing it through
an effect introduces another state value that can become inconsistent.

Use `useMemo` only when computation cost or referential stability justifies it;
it is an optimization, not a replacement for correct ownership.

### State is a snapshot

Handlers see the state from the render that created them. When the next value
depends on the previous value, use a functional update:

```jsx
setCount((previous) => previous + 1);
```

Do not mutate state objects or arrays directly. Create new values so React and
store selectors can detect changes correctly.

## 4. Lifting state and controlled components

When siblings need the same value, move it to their nearest appropriate common
owner.

```text
TransferPage owns selectedAccountId
  +-- AccountPicker receives value and onChange
  +-- TransferSummary receives selected account information
```

A controlled component receives its value and reports changes:

```jsx
function AccountPicker({ accounts, value, onChange }) {
  return (
    <select
      aria-label="Source account"
      value={value}
      onChange={(event) => onChange(event.target.value)}
    >
      <option value="">Select an account</option>
      {accounts.map((account) => (
        <option key={account.id} value={account.id}>
          {account.name}
        </option>
      ))}
    </select>
  );
}
```

The picker does not own another independent copy of the selection. Its parent
can coordinate selection, validation, and submission.

### Controlled versus uncontrolled

- **Controlled:** React props/state own the current value.
- **Uncontrolled:** The DOM owns the current value; code reads it through form
  submission or a ref, with an initial value such as `defaultValue`.

Controlled components support coordinated behavior. Uncontrolled inputs or
subscription-based form libraries can reduce update overhead for large forms.
Avoid unintentionally switching an input between controlled and uncontrolled.

## 5. Reducers for related transitions

Use `useReducer` when several values change together or transition logic deserves
a single explicit location.

```tsx
type Draft = {
  sourceAccountId: string;
  destinationAccountId: string;
  amountText: string;
};

type Action =
  | { type: "sourceSelected"; accountId: string }
  | { type: "destinationSelected"; accountId: string }
  | { type: "amountChanged"; value: string }
  | { type: "reset" };

const initialDraft: Draft = {
  sourceAccountId: "",
  destinationAccountId: "",
  amountText: "",
};

function draftReducer(state: Draft, action: Action): Draft {
  switch (action.type) {
    case "sourceSelected":
      return { ...state, sourceAccountId: action.accountId };
    case "destinationSelected":
      return { ...state, destinationAccountId: action.accountId };
    case "amountChanged":
      return { ...state, amountText: action.value };
    case "reset":
      return initialDraft;
    default: {
      const unreachable: never = action;
      throw new Error(`Unsupported draft action: ${String(unreachable)}`);
    }
  }
}
```

The reducer is pure: no network calls, timers, storage writes, or mutations.
Keeping an amount as text during editing permits intermediate input such as an
empty field. Convert and validate it with currency-aware rules before submission.

Reducers help organize transitions, but they do not automatically enforce a
business workflow. For complex processes, model valid states explicitly.

## 6. State machines and avoiding impossible states

Several independent booleans can permit contradictory combinations:

```text
isSubmitting = true
isSuccess = true
hasError = true
```

A discriminated union makes allowed states clearer:

```tsx
type SubmissionState =
  | { status: "idle" }
  | { status: "submitting"; requestId: string }
  | { status: "succeeded"; transferId: string }
  | { status: "failed"; message: string };
```

Define valid transitions, such as:

```text
idle -> submitting -> succeeded
                  -> failed -> submitting
```

For asynchronous financial operations, also distinguish accepted/pending from
completed. A network timeout may leave the result unknown rather than prove
failure. Resolve status with the backend instead of blindly submitting again.

A state-machine library can help with complex guards, parallel states, and
effects, but an explicit reducer/union is often enough for a modest workflow.

## 7. Context: distribution, not automatic state management

Context lets descendants read a provided value without passing it through every
intermediate component. State still comes from a hook, store, or another owner.

Good uses include:

- Theme and localization.
- A feature-scoped transfer draft.
- Dependencies such as a configured API client.
- Low-frequency shared settings.

### Provider design

```tsx
import {
  createContext,
  useContext,
  useState,
  type Dispatch,
  type ReactNode,
  type SetStateAction,
} from "react";

const StepContext = createContext<number | undefined>(undefined);
const SetStepContext =
  createContext<Dispatch<SetStateAction<number>> | undefined>(undefined);

export function TransferProvider({ children }: { children: ReactNode }) {
  const [step, setStep] = useState(0);

  return (
    <StepContext.Provider value={step}>
      <SetStepContext.Provider value={setStep}>
        {children}
      </SetStepContext.Provider>
    </StepContext.Provider>
  );
}

export function useTransferStep() {
  const step = useContext(StepContext);
  if (step === undefined) {
    throw new Error("useTransferStep requires TransferProvider");
  }
  return step;
}

export function useSetTransferStep() {
  const setStep = useContext(SetStepContext);
  if (setStep === undefined) {
    throw new Error("useSetTransferStep requires TransferProvider");
  }
  return setStep;
}
```

Separate state and dispatch contexts can avoid context-triggered updates for
consumers that only need the stable setter. They do not prevent every possible
rerender caused by parent rendering or other inputs.

### Performance caveats

Consumers update when their context value changes. A newly created object such
as `value={{ state, actions }}` has a new identity each render.

Memoizing a provider value can prevent unnecessary identity changes, but every
consumer of that context can still update when any included value changes.
Split unrelated contexts or use selector-based subscriptions for high-frequency,
large shared state. `React.memo` does not block a component's own context updates.

## 8. Redux, Redux Toolkit, and Flux

Flux describes unidirectional data flow:

```text
User interaction -> Action -> State transition -> UI -> Next interaction
```

Redux follows this general model with a store and reducers. Modern Redux
applications normally use Redux Toolkit rather than handwritten legacy
boilerplate.

Useful reasons to choose Redux:

- Many distant features coordinate shared client state.
- Transitions need consistent conventions and debugging.
- Middleware, selectors, and tooling are valuable.
- The application already uses Redux successfully.

### Redux Toolkit example

```tsx
import { createSlice, type PayloadAction } from "@reduxjs/toolkit";

const preferencesSlice = createSlice({
  name: "preferences",
  initialState: { compactView: false },
  reducers: {
    compactViewChanged(state, action: PayloadAction<boolean>) {
      state.compactView = action.payload;
    },
  },
});

export const { compactViewChanged } = preferencesSlice.actions;
export default preferencesSlice.reducer;
```

Redux Toolkit uses Immer for these reducers. The mutation-like syntax updates a
draft and produces immutable results; it does not justify mutating ordinary
React state outside that mechanism.

### Selectors and normalization

Subscribe to the smallest needed value:

```tsx
const compactView = useSelector(
  (state: RootState) => state.preferences.compactView
);
```

`RootState` should come from your configured store. React-Redux's default selector
comparison uses reference equality, so returning a new object each time can
cause unnecessary updates. Use stable/memoized selectors or an appropriate
comparison when needed.

For related entities, normalized data can avoid repeated copies:

```text
accountsById: { "a1": account, "a2": account }
accountIds: ["a1", "a2"]
selectedAccountId: "a1"
```

Do not persist nonserializable objects such as DOM nodes or live promises in an
ordinary Redux state model. Keep network side effects in suitable middleware,
thunks, or a query layer, not reducers.

## 9. Server state is different from client state

Server state is remotely owned, asynchronous, shared, and potentially stale.
It needs fetching, freshness rules, deduplication, retry policies, and mutation
coordination.

Tools such as TanStack Query and RTK Query manage these concerns. They do not
replace all local or global client state.

```tsx
const accountsQuery = useQuery({
  queryKey: ["accounts", institutionId, userId],
  queryFn: ({ signal }) =>
    fetchAccounts({ institutionId, userId, signal }),
  staleTime: 30_000,
});
```

This TanStack Query-style example assumes an implemented API function that
checks HTTP errors, validates data, and supports cancellation.

### Key concerns

- Include dimensions that change the result in the query key.
- Treat stale time as a freshness policy, not necessarily a polling interval.
- Invalidate or update relevant queries after successful mutations.
- Show initial loading separately from background refreshing.
- Prevent older responses from overwriting newer selections in manual fetching.
- Clear/scope user-specific caches during logout or identity changes.

Query keys isolate cached data; they do not enforce authorization. The backend
must verify access independently.

### Optimistic updates

An optimistic update changes the UI before the server confirms success:

1. Cancel or coordinate relevant in-flight fetches.
2. Save the previous cached state.
3. Apply the optimistic result.
4. Send the mutation.
5. Roll back on failure where safe.
6. Reconcile with authoritative server state.

Concurrent optimistic mutations can make naive snapshot rollback incorrect.
Use a suitable mutation strategy and server versioning where needed.

For a bank transfer, show a pending operation rather than pretending money
movement is complete before confirmation. Use backend idempotency to protect
against duplicate submissions.

## 10. URL, form, and persisted state

### URL state

Put shareable/bookmarkable state such as filters, sort, and pagination in the
URL. Parse and validate it, define defaults, and handle browser back/forward
navigation.

Avoid keeping an independent URL value and local value that drift apart.
Choose an owner and an explicit synchronization policy.

Do not place passwords, tokens, or sensitive account data in URLs.

### Form state

Keep field values, touched state, and validation in the form or a form library
unless other features genuinely need them. Not every keystroke belongs in a
global store.

Client validation improves UX; server validation enforces correctness.

### Persistence

Persist selected preferences rather than the entire store automatically.
Plan schema migrations, expiry, logout cleanup, storage failure handling, and
cross-tab behavior.

Do not persist sensitive banking data or credentials in browser storage without
an approved security design. Browser persistence can also create hydration
mismatches if the initial server and client views disagree.

## 11. Composition over large configuration-driven components

Composition builds features by combining focused pieces instead of creating a
single component with dozens of flags.

Less flexible:

```jsx
<Card
  showHeader
  showFooter
  showTransferButton
  useWarningStyle
  showAccountBalance
/>
```

More composable:

```jsx
<Card>
  <CardHeader title="Transfer money" />
  <AccountBalance />
  <TransferForm />
  <CardFooter>
    <TransferHelp />
  </CardFooter>
</Card>
```

Boolean props are not inherently wrong. Use them for genuine options, not to
encode unrelated business workflows in a shared UI component.

## 12. Common composition patterns

### Children and named slots

Use `children` for a main content area and named props for clear layout regions:

```jsx
function PageLayout({ navigation, heading, children, actions }) {
  return (
    <div className="page-layout">
      <nav aria-label="Profile">{navigation}</nav>
      <main>
        <header>
          {heading}
          {actions}
        </header>
        {children}
      </main>
    </div>
  );
}
```

The layout owns structure, not feature-specific API calls or global workflow
state. This is useful for a profile shell hosting settings and alert pages.

### Container and presentation separation

A feature container handles fetching, routing, and mutation orchestration.
Presentation components receive explicit props and callbacks.

```text
AccountsPage: fetches and coordinates
  +-- AccountsList: renders accounts
        +-- AccountRow: renders one account
```

Do not create a container for every trivial component. Separate responsibilities
where it makes testing and reuse clearer.

### Custom hooks

Hooks share behavior, not JSX:

```text
useAccounts       -> Server-data access
useTransferDraft  -> Workflow behavior
AccountPicker     -> UI
```

Two calls to a stateful custom hook normally create independent state. A hook
only shares state between callers when it uses a shared provider, external store,
or another shared resource.

### Compound components

Related components can cooperate through a scoped provider:

```jsx
<Tabs value={activeTab} onValueChange={setActiveTab}>
  <Tabs.List>
    <Tabs.Trigger value="accounts">Accounts</Tabs.Trigger>
    <Tabs.Trigger value="alerts">Alerts</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Panel value="accounts">
    <Accounts />
  </Tabs.Panel>
  <Tabs.Panel value="alerts">
    <Alerts />
  </Tabs.Panel>
</Tabs>
```

The parent coordinates active state, while consumers choose content and layout.
A production tabs component also needs correct roles, labels, focus management,
arrow-key behavior, and selected/tabindex relationships.

Do not assume Context alone supplies accessibility.

### Render props

A component can expose behavior/state through a function prop:

```jsx
<DataLoader>
  {({ data, status }) => (
    <Results data={data} status={status} />
  )}
</DataLoader>
```

Useful when consumers need flexible rendering. Hooks often simplify logic reuse,
but render props remain useful for APIs that own rendering lifecycle.

### Higher-order components

An HOC wraps a component to add behavior, for example legacy authorization or
instrumentation wrappers.

Trade-offs include wrapper nesting, prop collisions, and harder type/ref
handling. Prefer hooks or explicit composition for many new designs, but do not
rewrite working legacy HOCs without a reason.

## 13. Component identity and state preservation

React associates state with a component's position, type, and key in the tree.

- A stable identity preserves state across ordinary rerenders.
- Changing the key can intentionally reset state.
- Index keys in reorderable lists can associate state with the wrong item.
- Defining a component function inside another component can create a new type
  on each render and unexpectedly reset child state.

For an account-specific draft:

```jsx
<TransferForm key={accountId} accountId={accountId} />
```

This intentionally resets the form when the account changes. Use it only if
discarding the previous draft is the intended behavior.

## 14. Performance without premature optimization

First profile the interaction. A rerender is not the same as replacing the DOM,
and not every rerender is expensive.

Useful techniques:

- Keep rapidly changing state close to its consumers.
- Split unrelated Context values.
- Subscribe to narrow store selectors.
- Virtualize large lists where appropriate.
- Debounce expensive search/network requests when UX permits.
- Use stable keys and avoid duplicated derived state.
- Apply `memo`, `useMemo`, and `useCallback` where measured costs or dependency
  identity justify them.

Memoization is not a correctness guarantee. Modern React compiler configurations
may automate some optimization, but ownership and subscription design still
matter.

## 15. Enterprise example: banking application

```text
Application
  +-- Institution/theme providers
  +-- Server-state query provider
  +-- Global client store, if required
  +-- Profile layout
  |     +-- Navigation
  |     +-- Alert preferences feature
  |           +-- Query-backed saved preferences
  |           +-- Local/scoped editable draft
  |           +-- Reusable preference controls
  +-- Transfer feature
        +-- Feature reducer/provider
        +-- Account query data
        +-- Controlled form components
        +-- Submission/status workflow
```

Ownership choices:

- Institution configuration comes from its existing application boundary.
- Account balances remain server-owned.
- An unsaved transfer draft belongs to the transfer feature.
- Search/filter state goes in the URL when navigation should preserve it.
- A dropdown's open state stays local.
- Saved preferences are reconciled after a successful API mutation.
- Feature flags control composition, not server authorization.

Avoid mounting duplicate global providers inside reusable children. A separate
feature-scoped provider is appropriate only when it intentionally defines a new
state scope.

## 16. Testing strategy

Test behavior at the appropriate boundary:

- **Reducers:** transitions, reset, and invalid business-state prevention.
- **Components:** accessible controls and user interactions with React Testing
  Library.
- **Providers/hooks:** correct scope, missing-provider errors, and identity changes.
- **API integration:** loading, failures, invalidation, cancellation, and races.
- **End-to-end:** full workflows, navigation, reload, and authorization boundaries.

Use Jest or the project's existing runner and controlled network mocks. Prefer
user-visible assertions over checking internal implementation details.

High-value scenarios:

1. Changing the selected account updates every dependent summary.
2. A failed save preserves the draft and exposes a retryable error.
3. Logout does not expose the previous user's cached accounts.
4. Browser back restores URL-driven filters.
5. Duplicate submission does not create duplicate operations.
6. A delayed response for an old selection does not replace newer data.
7. Reordering a list preserves each row's correct local state.

## 17. Decision guide

| Need | Starting choice |
| --- | --- |
| One simple component-owned value | `useState` |
| Related local transitions | `useReducer` |
| Siblings share a value | Lift to common owner |
| Scoped descendants share feature state | Reducer/state plus Context |
| Complex cross-feature client coordination | Redux Toolkit or an appropriate existing store |
| Remote data caching and mutations | TanStack Query, RTK Query, or framework data layer |
| Bookmarkable navigation state | URL/router |
| Large form lifecycle | Suitable form library or scoped form model |
| Rich reusable UI structure | Children, slots, or compound components |
| Reusable behavioral logic | Custom hook |

These choices can coexist. Select the smallest set that meets the application's
requirements rather than introducing a tool for every row.

## 18. Interview questions

**Context or Redux?**

Context distributes values. Redux provides structured state transitions,
subscriptions, middleware, and tooling. Use Context for suitable scoped or
low-frequency values; use a store when shared-state complexity justifies it.

**Why not keep API data in ordinary Redux slices?**

You can, but you must implement cache lifecycles yourself. RTK Query or another
query layer usually provides a more purpose-built model for remote state.

**Does a custom hook automatically share state?**

No. Each invocation normally has its own hook state unless it accesses a shared
resource.

**How do you avoid prop drilling?**

First use composition to avoid unnecessary intermediaries. Use scoped Context
or a store when many descendants genuinely need the same value. Explicit props
remain useful and are not inherently a problem.

**What makes a component reusable?**

A focused responsibility, explicit inputs, meaningful callbacks, accessible
behavior, and limited coupling to routing, APIs, and global business state.

**How do you prevent too many rerenders?**

Measure first, colocate state, narrow subscriptions, split contexts, stabilize
expensive derived outputs where useful, and avoid unnecessary global updates.

## 19. Interview summary

> I classify state by ownership and lifetime, keep UI state local, use reducers
> for related transitions, Context for scoped sharing, and Redux when
> cross-feature coordination warrants it. I manage remote data through a query
> layer and build reusable UI with children, slots, compound components, and
> hooks, while testing accessibility, asynchronous failures, and state boundaries.

## References

- [React: choosing state structure](https://react.dev/learn/choosing-the-state-structure)
- [React: sharing state between components](https://react.dev/learn/sharing-state-between-components)
- [React: scaling with reducer and Context](https://react.dev/learn/scaling-up-with-reducer-and-context)
- [React: preserving and resetting state](https://react.dev/learn/preserving-and-resetting-state)
- [Redux Toolkit](https://redux-toolkit.js.org/)
- [RTK Query](https://redux-toolkit.js.org/rtk-query/overview)
- [TanStack Query](https://tanstack.com/query/latest/docs/framework/react/overview)
- [Testing Library guiding principles](https://testing-library.com/docs/guiding-principles/)
