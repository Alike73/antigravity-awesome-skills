# React Development Best Practices

> **System rule for React/TypeScript/Tailwind/ShadCN applications**

---

## 1. SOLID Principles in React

### 1.1 Single Responsibility Principle (SRP)

**Each component has ONE clear purpose.**

- ✅ Small, focused components
- ✅ Extract data fetching into custom hooks
- ✅ API calls in service files
- ✅ Use composition for reusability

```typescript
// ❌ BAD: Component handles fetching, state, and rendering
const ListItems = () => {
  const [items, setItems] = useState([]);
  useEffect(() => {
    axios.get('/api/items').then(setItems);
  }, []);
  return <div>{items.map(...)}</div>;
}

// ✅ GOOD: Separated concerns
// hooks/useItems.ts
const useItems = () => {
  const { data, error, isLoading } = useQuery({
    queryKey: ['items'],
    queryFn: fetchItems,
  });
  return { items: data, error, isLoading };
}

// components/ItemsList.tsx
const ItemsList = () => {
  const { items, isLoading } = useItems();
  if (isLoading) return <Spinner />;
  return <div>{items.map(item => <Item key={item.id} {...item} />)}</div>;
}
```

**Exceptions:** Forms and tables can combine state + validation + submission.

---

### 1.2 Open/Closed Principle (OCP)

**Open for extension, closed for modification.**

- ✅ Use composition over configuration
- ✅ Slots pattern via `children` or named props
- ✅ Extend through new compositions, not code changes

```typescript
// ❌ BAD: Adding new card types = modify this file
const Card = ({ type }) => {
  if (type === 'user') return <UserCard />;
  if (type === 'product') return <ProductCard />;
  // New type? Modify here.
}

// ✅ GOOD: Extend via composition
const Card = ({ header, content, footer }) => (
  <div className="card">
    <div className="card-header">{header}</div>
    <div className="card-content">{content}</div>
    <div className="card-footer">{footer}</div>
  </div>
);

// Usage - extend without modifying Card
<Card 
  header={<UserAvatar />} 
  content={<UserBio />} 
  footer={<FollowButton />} 
/>

<Card 
  header={<ProductImage />} 
  content={<ProductDetails />} 
  footer={<AddToCart />} 
/>
```

---

### 1.3 Liskov Substitution Principle (LSP)

**Derived components can replace parent components.**

- ✅ Custom wrappers inherit all parent props via `{...props}`
- ✅ Don't break expected interfaces

```typescript
// ❌ BAD: Doesn't inherit button behavior
const SuccessButton = () => <span>Success</span>;

// ✅ GOOD: Inherits all button props
interface SuccessButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary';
}

const SuccessButton = ({ variant = 'primary', ...props }: SuccessButtonProps) => (
  <button 
    className={cn('btn', `btn-${variant}`)}
    {...props}
  >
    Success
  </button>
);
```

---

### 1.4 Interface Segregation Principle (ISP)

**Components shouldn't depend on props they don't use.**

- ✅ Pass objects when component needs multiple related fields
- ✅ Destructure inside component to show dependencies
- ❌ Don't pass entire objects to unrelated components

```typescript
// ✅ GOOD: Pass object, destructure inside
interface User {
  id: string;
  name: string;
  email: string;
  avatar: string;
}

const UserCard = ({ user }: { user: User }) => {
  const { name, avatar } = user; // Only use what you need
  return (
    <div>
      <img src={avatar} alt={name} />
      <h3>{name}</h3>
    </div>
  );
}

// ❌ BAD: Passing unrelated data
const OrderSummary = ({ user }: { user: User }) => {
  // OrderSummary doesn't need user data at all
  return <div>Total: $100</div>;
}

// ✅ GOOD: Pass only what's needed
const OrderSummary = ({ total }: { total: number }) => (
  <div>Total: ${total}</div>
);
```

**Rule of thumb:** Pass objects for cohesive data (user profile, product details). Extract primitives when passing to unrelated components.

---

### 1.5 Dependency Inversion Principle (DIP)

**Depend on abstractions (props), not concrete implementations.**

- ✅ Pass behavior through props (callbacks)
- ✅ Components shouldn't know about API endpoints
- ✅ Inject dependencies

```typescript
// ❌ BAD: Form coupled to specific API endpoint
const CreateUserForm = () => {
  const handleSubmit = async (data) => {
    await axios.post('/api/users', data); // Concrete dependency
  };
  return <form onSubmit={handleSubmit}>...</form>;
};

// ✅ GOOD: Form accepts generic onSubmit
interface FormProps {
  onSubmit: (data: FormData) => Promise<void>;
}

const UserForm = ({ onSubmit }: FormProps) => (
  <form onSubmit={onSubmit}>...</form>
);

// Usage - same form, different behaviors
<UserForm onSubmit={handleCreate} />
<UserForm onSubmit={handleEdit} />
```

---

## 2. React Hooks Rules

**Critical patterns for hook usage:**

- ✅ Only call at **top level** (not in loops/conditions/nested functions)
- ✅ Custom hooks **must start with `use`**
- ✅ Dependencies array must include **all values used inside**
- ✅ Use ESLint `react-hooks/exhaustive-deps` to enforce

```typescript
// ❌ BAD: Hook in condition
if (isLoggedIn) {
  useEffect(() => { ... });
}

// ✅ GOOD: Condition inside hook
useEffect(() => {
  if (isLoggedIn) { ... }
}, [isLoggedIn]);

// ❌ BAD: Missing dependency
useEffect(() => {
  fetchData(userId); // userId not in deps
}, []);

// ✅ GOOD: Complete dependencies
useEffect(() => {
  fetchData(userId);
}, [userId]);
```

---

## 3. Error Handling

### 3.1 Error Boundaries

**Catch React render errors with class component boundaries.**

```typescript
interface ErrorBoundaryProps {
  fallback?: (error: Error, reset: () => void) => ReactNode;
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
  children: ReactNode;
}

interface ErrorBoundaryState {
  error: Error | null;
}

class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  state: ErrorBoundaryState = { error: null };

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    // Log to service (Sentry, LogRocket, etc.)
    console.error('ErrorBoundary caught:', error, errorInfo);
    this.props.onError?.(error, errorInfo);
  }

  reset = () => {
    this.setState({ error: null });
  };

  render() {
    const { error } = this.state;
    const { fallback, children } = this.props;

    if (error) {
      return fallback?.(error, this.reset) ?? (
        <DefaultErrorScreen error={error} onReset={this.reset} />
      );
    }

    return children;
  }
}

// Usage
<ErrorBoundary 
  fallback={(error, reset) => (
    <div>
      <h1>Something went wrong</h1>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  )}
  onError={(error) => {
    // Sentry.captureException(error);
  }}
>
  <App />
</ErrorBoundary>
```

---

### 3.2 Async Error Handling

**Use React Query, SWR, or throw in render for async errors.**

```typescript
// ✅ GOOD: React Query (recommended)
const { data, error, isLoading } = useQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId),
  useErrorBoundary: true, // Throws to ErrorBoundary
});

// ✅ GOOD: Custom hook with error state
const useFetch = <T,>(url: string) => {
  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    const fetchData = async () => {
      try {
        const response = await fetch(url);
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        const json = await response.json();
        setData(json);
      } catch (err) {
        setError(err instanceof Error ? err : new Error('Unknown error'));
      } finally {
        setIsLoading(false);
      }
    };
    fetchData();
  }, [url]);

  if (error) throw error; // ErrorBoundary catches

  return { data, isLoading };
};
```

---

### 3.3 Event Handler Errors

**Wrap async handlers in try/catch.**

```typescript
const handleSubmit = async (data: FormData) => {
  try {
    await createUser(data);
    toast.success('User created!');
  } catch (error) {
    toast.error(error instanceof Error ? error.message : 'Failed to create user');
    console.error('Submit failed:', error);
  }
};
```

---

## 4. Clean Code Practices

### 4.1 Extract List Rendering

```typescript
// ✅ GOOD: Dedicated list component
const BrandButtonsList = ({ brands, activeBrandCode, onBrandClick }) => (
  <>
    {brands.map(({ id, name, code }) => (
      <Button 
        key={id} 
        variant={activeBrandCode === code ? 'default' : 'outline'}
        onClick={() => onBrandClick(code)}
      >
        {name}
      </Button>
    ))}
  </>
);
```

---

### 4.2 Extract Helper Functions

**Move non-hook utilities outside components.**

```typescript
// utils/date.ts
const RU_DATE_FORMATTER = new Intl.DateTimeFormat('ru-RU', {
  year: 'numeric',
  month: 'long',
  day: 'numeric',
});

export const formatDate = (date: string | Date): string => {
  return RU_DATE_FORMATTER.format(new Date(date));
};

// Component
import { formatDate } from '@/utils/date';

const OrderDate = ({ date }: { date: string }) => (
  <time dateTime={date}>{formatDate(date)}</time>
);
```

---

### 4.3 Extract Complex Conditions

```typescript
// ❌ BAD
if (isInitialLoad && !messages.length && isNewMessage) {
  return;
}

// ✅ GOOD
const shouldSkipScroll = 
  isInitialLoad && 
  messages.length === 0 && 
  isNewMessage;

if (shouldSkipScroll) return;
```

---

### 4.4 No Magic Numbers

```typescript
// ❌ BAD
if (price > 1000) {
  return price * 0.85;
}

// ✅ GOOD
const DISCOUNT_THRESHOLD = 1000;
const DISCOUNT_RATE = 0.15;

if (price > DISCOUNT_THRESHOLD) {
  return price * (1 - DISCOUNT_RATE);
}
```

---

### 4.5 Safe Property Access

```typescript
// ✅ GOOD
const vehicle = order?.vehicle;
const model = vehicle?.model?.name;
const displayName = model ?? 'No data';

return <div>{displayName}</div>;
```

---

## 5. TypeScript Best Practices

### 5.1 Component Props

```typescript
// ✅ GOOD: Interface for props
interface UserCardProps {
  user: User;
  onEdit?: (id: string) => void;
}

// Don't use React.FC (deprecated pattern)
const UserCard = ({ user, onEdit }: UserCardProps) => {
  return (
    <div>
      <h3>{user.name}</h3>
      {onEdit && <button onClick={() => onEdit(user.id)}>Edit</button>}
    </div>
  );
}
```

---

### 5.2 Avoid `any`

```typescript
// ❌ BAD
const handleData = (data: any) => { ... }

// ✅ GOOD
const handleData = (data: unknown) => {
  if (isUser(data)) {
    // Type narrowed to User
  }
}

// Type guard
const isUser = (data: unknown): data is User => {
  return (
    typeof data === 'object' &&
    data !== null &&
    'id' in data &&
    'name' in data
  );
}
```

---

### 5.3 Explicit Return Types for Complex Logic

```typescript
// ✅ GOOD
const calculateDiscount = (price: number): number => {
  if (price > DISCOUNT_THRESHOLD) {
    return price * (1 - DISCOUNT_RATE);
  }
  return price;
};
```

---

## 6. Performance Optimization

### 6.1 When to Memoize

```typescript
// ✅ Use useMemo for expensive calculations
const sortedItems = useMemo(
  () => items.sort((a, b) => a.name.localeCompare(b.name)),
  [items]
);

// ✅ Use useCallback for callbacks passed to memoized children
const handleClick = useCallback((id: string) => {
  setSelectedId(id);
}, []);

// ❌ Don't memo everything - measure first
// Most components don't need it
```

---

### 6.2 Virtualize Long Lists

```typescript
// ✅ Use react-window or @tanstack/virtual for 100+ items
import { useVirtualizer } from '@tanstack/react-virtual';

const VirtualList = ({ items }) => {
  const parentRef = useRef<HTMLDivElement>(null);
  
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
  });

  return (
    <div ref={parentRef} style={{ height: '400px', overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px` }}>
        {virtualizer.getVirtualItems().map(virtualItem => (
          <div key={virtualItem.key} style={{ height: '50px' }}>
            {items[virtualItem.index].name}
          </div>
        ))}
      </div>
    </div>
  );
};
```

---

### 6.3 Code Splitting

```typescript
// ✅ Lazy load heavy components
const HeavyChart = lazy(() => import('./HeavyChart'));

<Suspense fallback={<ChartSkeleton />}>
  <HeavyChart data={data} />
</Suspense>
```

---

### 6.4 Avoid Inline Objects/Functions

```typescript
// ❌ BAD: New object every render
<Component style={{ margin: 10 }} />

// ✅ GOOD: Extract to constant
const styles = { margin: 10 };
<Component style={styles} />

// ❌ BAD: New function every render
<button onClick={() => handleClick(id)}>Click</button>

// ✅ GOOD: useCallback if Component is memoized
const onClick = useCallback(() => handleClick(id), [id]);
<MemoizedButton onClick={onClick} />
```

---

### 6.5 Debounce Expensive Operations

```typescript
import { useDebouncedCallback } from 'use-debounce';

const SearchInput = () => {
  const search = useDebouncedCallback(
    (value: string) => {
      fetchResults(value);
    },
    300 // 300ms delay
  );

  return <input onChange={(e) => search(e.target.value)} />;
};
```

---

## 7. State Management

### 7.1 Choose the Right Tool

- **Local UI state:** `useState`, `useReducer`
- **Server state:** React Query, SWR, TanStack Query
- **Global UI state:** Zustand, Jotai (avoid Redux for new projects)
- **URL state:** Next.js router, React Router
- **Form state:** react-hook-form, Formik

```typescript
// ✅ Server state with React Query
const { data: user } = useQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId),
});

// ✅ Global UI state with Zustand
import { create } from 'zustand';

const useStore = create<Store>((set) => ({
  theme: 'light',
  toggleTheme: () => set((state) => ({ 
    theme: state.theme === 'light' ? 'dark' : 'light' 
  })),
}));
```

---

### 7.2 Context Performance

**❌ Don't use Context for frequently changing values** (causes re-renders).

```typescript
// ❌ BAD: Theme changes = all consumers re-render
const ThemeContext = createContext({ theme, setTheme });

// ✅ GOOD: Split contexts or use Zustand/Jotai
const ThemeContext = createContext(theme);
const ThemeUpdaterContext = createContext(setTheme);
```

---

## 8. Forms

### 8.1 Use a Form Library

```typescript
// ✅ react-hook-form + Zod
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

type FormData = z.infer<typeof schema>;

const LoginForm = () => {
  const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(schema),
  });

  const onSubmit = async (data: FormData) => {
    await login(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email')} />
      {errors.email && <span>{errors.email.message}</span>}
      
      <input type="password" {...register('password')} />
      {errors.password && <span>{errors.password.message}</span>}
      
      <button type="submit">Login</button>
    </form>
  );
};
```

---

## 9. Accessibility

- ✅ Semantic HTML (`<button>`, `<nav>`, `<main>`)
- ✅ ARIA labels on interactive elements
- ✅ Form inputs have associated `<label>`
- ✅ Images have `alt` text (empty `alt=""` for decorative)
- ✅ WCAG AA contrast (4.5:1 for text)
- ✅ Keyboard navigation (Tab, Enter, Escape)

```typescript
// ✅ GOOD
<button aria-label="Close dialog" onClick={onClose}>
  <X />
</button>

<label htmlFor="email">Email</label>
<input id="email" type="email" />

<img src={avatar} alt={`${name}'s profile picture`} />

// Test with:
// - axe DevTools
// - Lighthouse accessibility audit
// - Keyboard only (no mouse)
```

---

## 10. Tailwind CSS

### 10.1 Class Organization

```typescript
// ✅ Organize by concern
<div className="
  flex items-center justify-between
  p-4 rounded-lg
  bg-white hover:bg-gray-50
  border border-gray-200
  transition-colors duration-200
">
```

---

### 10.2 Use `cn()` Utility

```typescript
import { cn } from '@/lib/utils';

const Button = ({ variant, className, ...props }) => (
  <button
    className={cn(
      'px-4 py-2 rounded font-medium',
      variant === 'primary' && 'bg-blue-600 text-white',
      variant === 'secondary' && 'bg-gray-200 text-gray-900',
      className // User overrides
    )}
    {...props}
  />
);
```

---

### 10.3 Extend Theme, Avoid Arbitrary Values

```typescript
// tailwind.config.ts
export default {
  theme: {
    extend: {
      colors: {
        brand: {
          primary: '#3B82F6',
          secondary: '#10B981',
        },
      },
    },
  },
};

// ✅ Use theme
<div className="bg-brand-primary text-white">

// ❌ Avoid arbitrary (unless truly one-off)
<div className="bg-[#3B82F6]">
```

---

## 11. ShadCN UI

### 11.1 Treat as Owned Code

**ShadCN is a code generator, not a package.**

- ✅ Run `npx shadcn@latest add <component>` to add components
- ✅ Customize generated code directly (it's yours)
- ✅ Use `cn()` for class merging

```typescript
// ✅ Extend ShadCN Button
import { Button } from '@/components/ui/button';
import { cn } from '@/lib/utils';

interface CustomButtonProps extends React.ComponentProps<typeof Button> {
  isLoading?: boolean;
}

const CustomButton = ({ isLoading, children, className, ...props }: CustomButtonProps) => (
  <Button 
    className={cn(isLoading && 'opacity-50 cursor-not-allowed', className)}
    disabled={isLoading || props.disabled}
    {...props}
  >
    {isLoading ? <Spinner /> : children}
  </Button>
);
```

---

## 12. Testing

### 12.1 Testing Library

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

// ✅ Query by role/label (not testID)
test('submits form', async () => {
  const handleSubmit = jest.fn();
  render(<LoginForm onSubmit={handleSubmit} />);
  
  await userEvent.type(screen.getByLabelText('Email'), 'test@example.com');
  await userEvent.type(screen.getByLabelText('Password'), 'password123');
  await userEvent.click(screen.getByRole('button', { name: /login/i }));
  
  expect(handleSubmit).toHaveBeenCalledWith({
    email: 'test@example.com',
    password: 'password123',
  });
});
```

---

### 12.2 Mock at Network Boundary

```typescript
// ✅ Use MSW (Mock Service Worker)
import { rest } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  rest.get('/api/user', (req, res, ctx) => {
    return res(ctx.json({ id: '1', name: 'Test User' }));
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

---

## 13. File Structure

```text
src/
├── components/
│   ├── ui/              # ShadCN components
│   ├── features/        # Feature-specific components
│   └── layout/          # Layout components
├── hooks/               # Custom hooks
├── lib/                 # Third-party setup (axios, react-query)
├── utils/               # Pure utility functions
├── api/                 # API client and endpoints
├── types/               # TypeScript type definitions
├── constants/           # App constants
└── app/                 # Pages (Next.js) or routes
```

**Naming:**
- Components: `PascalCase.tsx`
- Hooks: `useCamelCase.ts`
- Utils: `camelCase.ts`
- Constants: `UPPER_SNAKE_CASE.ts`

---

## 14. Final Checklist

Before code review:

- [ ] All components follow SRP
- [ ] No prop drilling (use composition/context/state management)
- [ ] Magic numbers → constants
- [ ] Complex conditions → named variables
- [ ] Utility functions outside components
- [ ] ErrorBoundary wraps app
- [ ] Async operations have error handling
- [ ] No `any` types
- [ ] No `React.FC`
- [ ] Dependencies arrays complete
- [ ] Performance optimizations where measured
- [ ] Accessibility requirements met
- [ ] Tests cover critical paths

---

**Priority Levels:**

- 🔴 **Critical:** Missing error handling, a11y violations, security issues
- 🟡 **Important:** TypeScript types, performance, state management
- 🟢 **Nice-to-have:** Formatting, naming, comments

---

*Apply during code reviews and before merging to ensure maintainable, production-ready React applications.*
