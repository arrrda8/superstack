# State Management — Superstack Reference

## 1. Zustand

### Basic Store Creation

```tsx
import { create } from "zustand";

interface CounterStore {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
}

const useCounterStore = create<CounterStore>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}));

// Usage in component
function Counter() {
  const { count, increment } = useCounterStore();
  return <button onClick={increment}>{count}</button>;
}

// Selector pattern — only re-renders when count changes
function CountDisplay() {
  const count = useCounterStore((s) => s.count);
  return <span>{count}</span>;
}
```

### Slices Pattern

```tsx
import { create, type StateCreator } from "zustand";

// --- Auth Slice ---
interface AuthSlice {
  user: { id: string; name: string } | null;
  setUser: (user: AuthSlice["user"]) => void;
  logout: () => void;
}

const createAuthSlice: StateCreator<
  AuthSlice & UISlice,
  [],
  [],
  AuthSlice
> = (set) => ({
  user: null,
  setUser: (user) => set({ user }),
  logout: () => set({ user: null }),
});

// --- UI Slice ---
interface UISlice {
  sidebarOpen: boolean;
  toggleSidebar: () => void;
}

const createUISlice: StateCreator<
  AuthSlice & UISlice,
  [],
  [],
  UISlice
> = (set) => ({
  sidebarOpen: false,
  toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
});

// --- Combined Store ---
const useAppStore = create<AuthSlice & UISlice>()((...a) => ({
  ...createAuthSlice(...a),
  ...createUISlice(...a),
}));
```

### Persist Middleware

```tsx
import { create } from "zustand";
import { persist, createJSONStorage } from "zustand/middleware";

interface SettingsStore {
  theme: "light" | "dark";
  locale: string;
  setTheme: (theme: "light" | "dark") => void;
  setLocale: (locale: string) => void;
}

const useSettingsStore = create<SettingsStore>()(
  persist(
    (set) => ({
      theme: "light",
      locale: "de",
      setTheme: (theme) => set({ theme }),
      setLocale: (locale) => set({ locale }),
    }),
    {
      name: "settings-storage", // localStorage key
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({
        theme: state.theme,
        locale: state.locale,
      }), // only persist these fields
    }
  )
);
```

### Devtools Middleware

```tsx
import { create } from "zustand";
import { devtools } from "zustand/middleware";

const useStore = create<MyStore>()(
  devtools(
    (set) => ({
      count: 0,
      increment: () =>
        set(
          (state) => ({ count: state.count + 1 }),
          false, // replace = false
          "increment" // action name in devtools
        ),
    }),
    { name: "MyStore" } // store name in devtools
  )
);
```

### Immer Middleware

```tsx
import { create } from "zustand";
import { immer } from "zustand/middleware/immer";

interface TodoStore {
  todos: { id: string; text: string; done: boolean }[];
  addTodo: (text: string) => void;
  toggleTodo: (id: string) => void;
}

const useTodoStore = create<TodoStore>()(
  immer((set) => ({
    todos: [],
    addTodo: (text) =>
      set((state) => {
        state.todos.push({ id: crypto.randomUUID(), text, done: false });
      }),
    toggleTodo: (id) =>
      set((state) => {
        const todo = state.todos.find((t) => t.id === id);
        if (todo) todo.done = !todo.done;
      }),
  }))
);
```

---

## 2. TanStack Query

### QueryClient Setup (App Router)

```tsx
// lib/query-client.ts
import { QueryClient } from "@tanstack/react-query";

function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000, // 1 minute
        gcTime: 5 * 60 * 1000, // 5 minutes (garbage collection)
        refetchOnWindowFocus: false,
        retry: 1,
      },
    },
  });
}

let browserQueryClient: QueryClient | undefined;

export function getQueryClient() {
  if (typeof window === "undefined") {
    // Server: always make a new query client
    return makeQueryClient();
  }
  // Browser: reuse client
  if (!browserQueryClient) browserQueryClient = makeQueryClient();
  return browserQueryClient;
}
```

```tsx
// app/providers.tsx
"use client";

import { QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";
import { getQueryClient } from "@/lib/query-client";

export function Providers({ children }: { children: React.ReactNode }) {
  const queryClient = getQueryClient();

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

### useQuery / useMutation

```tsx
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";

// --- Fetch ---
function useProjects() {
  return useQuery({
    queryKey: ["projects"],
    queryFn: async () => {
      const res = await fetch("/api/projects");
      if (!res.ok) throw new Error("Failed to fetch projects");
      return res.json() as Promise<Project[]>;
    },
    staleTime: 30_000,
  });
}

// --- Mutation ---
function useCreateProject() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (data: CreateProjectInput) => {
      const res = await fetch("/api/projects", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(data),
      });
      if (!res.ok) throw new Error("Failed to create project");
      return res.json() as Promise<Project>;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["projects"] });
    },
  });
}

// Usage
function ProjectForm() {
  const { mutate, isPending } = useCreateProject();
  return (
    <button onClick={() => mutate({ name: "New" })} disabled={isPending}>
      {isPending ? "Creating..." : "Create Project"}
    </button>
  );
}
```

### Prefetching in RSC

```tsx
// app/projects/page.tsx (Server Component)
import {
  dehydrate,
  HydrationBoundary,
  QueryClient,
} from "@tanstack/react-query";
import { ProjectList } from "./project-list";

export default async function ProjectsPage() {
  const queryClient = new QueryClient();

  await queryClient.prefetchQuery({
    queryKey: ["projects"],
    queryFn: () => fetchProjects(), // server-side fetch
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <ProjectList />
    </HydrationBoundary>
  );
}
```

### staleTime vs gcTime

| Setting      | Meaning                                                        | Default   |
| ------------ | -------------------------------------------------------------- | --------- |
| `staleTime`  | How long data is considered fresh (no refetch)                 | `0`       |
| `gcTime`     | How long unused/inactive data stays in cache before GC         | `5 min`   |

```
staleTime: 0        -> refetch on every mount
staleTime: Infinity  -> never refetch automatically
gcTime: 0           -> remove from cache immediately when unused
```

---

## 3. Optimistic Updates

```tsx
function useToggleTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({ id, done }: { id: string; done: boolean }) => {
      const res = await fetch(`/api/todos/${id}`, {
        method: "PATCH",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ done }),
      });
      return res.json();
    },

    // Optimistic update
    onMutate: async ({ id, done }) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ["todos"] });

      // Snapshot previous value
      const previousTodos = queryClient.getQueryData<Todo[]>(["todos"]);

      // Optimistically update
      queryClient.setQueryData<Todo[]>(["todos"], (old) =>
        old?.map((t) => (t.id === id ? { ...t, done } : t))
      );

      return { previousTodos }; // context for rollback
    },

    // Rollback on error
    onError: (_err, _vars, context) => {
      if (context?.previousTodos) {
        queryClient.setQueryData(["todos"], context.previousTodos);
      }
    },

    // Always refetch after settle
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ["todos"] });
    },
  });
}
```

---

## 4. Infinite Scroll

### useInfiniteQuery with Cursor Pagination

```tsx
import { useInfiniteQuery } from "@tanstack/react-query";
import { useEffect, useRef } from "react";

interface PageResponse {
  items: Item[];
  nextCursor: string | null;
}

function useInfiniteItems() {
  return useInfiniteQuery({
    queryKey: ["items"],
    queryFn: async ({ pageParam }): Promise<PageResponse> => {
      const params = new URLSearchParams();
      if (pageParam) params.set("cursor", pageParam);
      params.set("limit", "20");
      const res = await fetch(`/api/items?${params}`);
      return res.json();
    },
    initialPageParam: null as string | null,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });
}
```

### Intersection Observer Trigger

```tsx
function InfiniteList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteItems();

  const observerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const el = observerRef.current;
    if (!el) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting && hasNextPage && !isFetchingNextPage) {
          fetchNextPage();
        }
      },
      { threshold: 0.1 }
    );

    observer.observe(el);
    return () => observer.disconnect();
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  const allItems = data?.pages.flatMap((page) => page.items) ?? [];

  return (
    <div>
      {allItems.map((item) => (
        <ItemCard key={item.id} item={item} />
      ))}

      {/* Sentinel element */}
      <div ref={observerRef} className="h-10">
        {isFetchingNextPage && <Spinner />}
      </div>
    </div>
  );
}
```

---

## 5. nuqs — URL State Management

### Setup (App Router)

```tsx
// app/layout.tsx
import { NuqsAdapter } from "nuqs/adapters/next/app";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <NuqsAdapter>{children}</NuqsAdapter>
      </body>
    </html>
  );
}
```

### useQueryState

```tsx
"use client";

import { useQueryState, parseAsInteger, parseAsStringEnum } from "nuqs";

type SortOrder = "asc" | "desc";

function ProductFilters() {
  // String param: ?search=foo
  const [search, setSearch] = useQueryState("search", {
    defaultValue: "",
    shallow: true, // no server roundtrip
  });

  // Integer param: ?page=2
  const [page, setPage] = useQueryState(
    "page",
    parseAsInteger.withDefault(1)
  );

  // Enum param: ?sort=desc
  const [sort, setSort] = useQueryState(
    "sort",
    parseAsStringEnum<SortOrder>(["asc", "desc"]).withDefault("asc")
  );

  return (
    <div>
      <input
        value={search}
        onChange={(e) => setSearch(e.target.value || null)} // null removes param
        placeholder="Search..."
      />
      <select
        value={sort}
        onChange={(e) => setSort(e.target.value as SortOrder)}
      >
        <option value="asc">A-Z</option>
        <option value="desc">Z-A</option>
      </select>
      <button onClick={() => setPage((p) => p + 1)}>
        Page {page} — Next
      </button>
    </div>
  );
}
```

### Multiple Params with useQueryStates

```tsx
import { useQueryStates, parseAsInteger, parseAsString } from "nuqs";

const [filters, setFilters] = useQueryStates({
  search: parseAsString.withDefault(""),
  page: parseAsInteger.withDefault(1),
  category: parseAsString,
});

// Update multiple at once (single URL update)
setFilters({ search: "shoes", page: 1 });

// Access
console.log(filters.search, filters.page, filters.category);
```

---

## 6. React Context

### When to Use

- Theme (light/dark)
- Locale / i18n
- Auth state (current user, rarely changes)
- Feature flags
- DI (providing services/adapters)

### When NOT to Use

- Frequently changing state (e.g., mouse position, form inputs, real-time data)
- State consumed by many distant components with different update frequencies
- Large state objects where any change triggers re-renders in all consumers

### Correct Pattern

```tsx
"use client";

import { createContext, useContext, useState, useCallback } from "react";

interface ThemeContextValue {
  theme: "light" | "dark";
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextValue | null>(null);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<"light" | "dark">("light");

  const toggleTheme = useCallback(() => {
    setTheme((t) => (t === "light" ? "dark" : "light"));
  }, []);

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("useTheme must be used within ThemeProvider");
  return ctx;
}
```

---

## 7. Server State vs Client State — Decision Matrix

| Question                                              | Server State (TanStack Query) | Client State (Zustand / useState) |
| ----------------------------------------------------- | :---------------------------: | :-------------------------------: |
| Data comes from API/DB?                               |              YES              |                                   |
| Shared across users?                                  |              YES              |                                   |
| Needs cache invalidation?                             |              YES              |                                   |
| UI-only (sidebar open, modal visible)?                |                               |               YES                 |
| User input before submission?                         |                               |               YES                 |
| Needs to survive page refresh (URL state)?            |          nuqs / URL           |                                   |
| Needs to survive page refresh (localStorage)?         |                               |          Zustand + persist         |
| Multiple components read same API data?               |              YES              |                                   |
| Optimistic updates needed?                            |              YES              |                                   |

**Rule of thumb:** If it came from the server, use TanStack Query. If it's local to the UI, use Zustand or useState.

---

## 8. Form State

### React Hook Form (Recommended for complex forms)

```tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const schema = z.object({
  name: z.string().min(2, "Name must be at least 2 characters"),
  email: z.string().email("Invalid email"),
  role: z.enum(["admin", "user"]),
});

type FormData = z.infer<typeof schema>;

function UserForm({ onSubmit }: { onSubmit: (data: FormData) => void }) {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    reset,
  } = useForm<FormData>({
    resolver: zodResolver(schema),
    defaultValues: { name: "", email: "", role: "user" },
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <input {...register("name")} placeholder="Name" />
        {errors.name && <span className="text-red-500">{errors.name.message}</span>}
      </div>

      <div>
        <input {...register("email")} placeholder="Email" />
        {errors.email && <span className="text-red-500">{errors.email.message}</span>}
      </div>

      <select {...register("role")}>
        <option value="user">User</option>
        <option value="admin">Admin</option>
      </select>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Saving..." : "Save"}
      </button>
    </form>
  );
}
```

### When to Lift Form State

- Use `useState` for simple 1-2 field forms
- Use React Hook Form for 3+ fields, validation, or dynamic fields
- Lift state when parent needs to coordinate multiple form sections or show a preview

---

## 9. Global UI State — Zustand Patterns

```tsx
import { create } from "zustand";

// --- Modal Store ---
interface ModalStore {
  modals: Record<string, boolean>;
  modalData: Record<string, unknown>;
  openModal: (id: string, data?: unknown) => void;
  closeModal: (id: string) => void;
  isOpen: (id: string) => boolean;
}

const useModalStore = create<ModalStore>((set, get) => ({
  modals: {},
  modalData: {},
  openModal: (id, data) =>
    set((s) => ({
      modals: { ...s.modals, [id]: true },
      modalData: data ? { ...s.modalData, [id]: data } : s.modalData,
    })),
  closeModal: (id) =>
    set((s) => ({
      modals: { ...s.modals, [id]: false },
    })),
  isOpen: (id) => !!get().modals[id],
}));

// Usage
function DeleteButton({ itemId }: { itemId: string }) {
  const openModal = useModalStore((s) => s.openModal);
  return (
    <button onClick={() => openModal("confirm-delete", { itemId })}>
      Delete
    </button>
  );
}

function ConfirmDeleteModal() {
  const isOpen = useModalStore((s) => s.isOpen("confirm-delete"));
  const data = useModalStore((s) => s.modalData["confirm-delete"]) as
    | { itemId: string }
    | undefined;
  const closeModal = useModalStore((s) => s.closeModal);

  if (!isOpen) return null;

  return (
    <Dialog open onOpenChange={() => closeModal("confirm-delete")}>
      <p>Delete item {data?.itemId}?</p>
    </Dialog>
  );
}

// --- Toast Store ---
interface Toast {
  id: string;
  message: string;
  type: "success" | "error" | "info";
}

interface ToastStore {
  toasts: Toast[];
  addToast: (message: string, type: Toast["type"]) => void;
  removeToast: (id: string) => void;
}

const useToastStore = create<ToastStore>((set) => ({
  toasts: [],
  addToast: (message, type) => {
    const id = crypto.randomUUID();
    set((s) => ({ toasts: [...s.toasts, { id, message, type }] }));
    setTimeout(() => {
      set((s) => ({ toasts: s.toasts.filter((t) => t.id !== id) }));
    }, 5000);
  },
  removeToast: (id) =>
    set((s) => ({ toasts: s.toasts.filter((t) => t.id !== id) })),
}));

// Usage
const { addToast } = useToastStore.getState();
addToast("Project created!", "success");
```

---

## 10. Hydration — Zustand + SSR

### The Problem

Zustand persist reads from localStorage on mount, which causes a mismatch between server-rendered HTML and client state.

### Solution 1: useEffect Guard

```tsx
"use client";

import { useEffect, useState } from "react";

function ThemeToggle() {
  const theme = useSettingsStore((s) => s.theme);
  const setTheme = useSettingsStore((s) => s.setTheme);
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setMounted(true);
  }, []);

  if (!mounted) {
    // Render a skeleton or default state
    return <button aria-label="Toggle theme">Theme</button>;
  }

  return (
    <button onClick={() => setTheme(theme === "light" ? "dark" : "light")}>
      {theme === "light" ? "Dark" : "Light"}
    </button>
  );
}
```

### Solution 2: onRehydrateStorage + skipHydration

```tsx
import { create } from "zustand";
import { persist } from "zustand/middleware";

const useStore = create(
  persist(
    (set) => ({
      count: 0,
      increment: () => set((s) => ({ count: s.count + 1 })),
    }),
    {
      name: "counter",
      skipHydration: true, // don't hydrate automatically
    }
  )
);

// Manually hydrate in a client component
"use client";
import { useEffect } from "react";

function HydrationGate({ children }: { children: React.ReactNode }) {
  useEffect(() => {
    useStore.persist.rehydrate();
  }, []);

  return <>{children}</>;
}
```

### Solution 3: suppressHydrationWarning

Only for cosmetic mismatches (e.g., timestamps, random IDs). Do NOT use this to hide real state mismatches.

```tsx
<time suppressHydrationWarning>{new Date().toLocaleString()}</time>
```

### Best Practice Checklist

1. Always use `skipHydration: true` in persist config if SSR is involved.
2. Use a `mounted` state guard for any component that reads persisted store values.
3. Never use `suppressHydrationWarning` for actual state differences.
4. Consider using `useEffect` to apply the persisted value after mount.
5. For critical persisted state (theme, locale), set the initial value via cookie/header on the server to avoid flash.
