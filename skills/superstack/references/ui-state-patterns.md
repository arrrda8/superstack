# UI State Patterns — React / Next.js Reference

## Table of Contents
- [1. Error Boundaries](#1-error-boundaries)
  - [Base Error Boundary Component](#base-error-boundary-component)
  - [Per-Route Error Boundary (App Router)](#per-route-error-boundary-app-router)
  - [Global Error Boundary (App Router)](#global-error-boundary-app-router)
  - [Usage](#usage)
- [2. Skeleton Screens](#2-skeleton-screens)
  - [Base Skeleton Component](#base-skeleton-component)
  - [Shimmer Animation (alternative to pulse)](#shimmer-animation-alternative-to-pulse)
  - [Content-Aware Skeletons](#content-aware-skeletons)
  - [Suspense Integration](#suspense-integration)
- [3. Empty States](#3-empty-states)
  - [Generic Empty State Component](#generic-empty-state-component)
  - [First-Use vs. No-Results Patterns](#first-use-vs-no-results-patterns)
  - [Usage in a List Component](#usage-in-a-list-component)
- [4. Toast Notifications](#4-toast-notifications)
  - [Sonner Setup](#sonner-setup)
  - [Toast Patterns](#toast-patterns)
  - [Usage](#usage)
- [5. Offline Detection](#5-offline-detection)
  - [useOnlineStatus Hook](#useonlinestatus-hook)
  - [Offline Banner Component](#offline-banner-component)
  - [Offline Action Queue](#offline-action-queue)
  - [Usage](#usage)
- [6. Optimistic Updates](#6-optimistic-updates)
  - [TanStack Query Optimistic Mutation](#tanstack-query-optimistic-mutation)
  - [Usage](#usage)
  - [Optimistic Add (new item)](#optimistic-add-new-item)
- [7. Loading States](#7-loading-states)
  - [Loading Button with Spinner](#loading-button-with-spinner)
  - [Page-Level Loading (App Router)](#page-level-loading-app-router)
  - [Streaming with Suspense (granular loading)](#streaming-with-suspense-granular-loading)
  - [Full-Page Spinner (for route transitions)](#full-page-spinner-for-route-transitions)
- [8. 404 / 403 / 500 Pages](#8-404-403-500-pages)
  - [not-found.tsx (404)](#not-foundtsx-404)
  - [Triggering not-found programmatically](#triggering-not-found-programmatically)
  - [error.tsx (500 — per-route)](#errortsx-500-per-route)
  - [403 Forbidden Page](#403-forbidden-page)
- [9. Confirmation Dialogs](#9-confirmation-dialogs)
  - [AlertDialog Pattern (shadcn/ui)](#alertdialog-pattern-shadcnui)
  - [Async Confirmation Hook](#async-confirmation-hook)
  - [Usage](#usage)
- [10. Form Validation Feedback](#10-form-validation-feedback)
  - [Field-Level Inline Errors (React Hook Form + Zod)](#field-level-inline-errors-react-hook-form-zod)
  - [Reusable Form Field Component](#reusable-form-field-component)
  - [Server-Side Validation Feedback (Server Actions)](#server-side-validation-feedback-server-actions)

Comprehensive patterns for handling UI states in React/Next.js (App Router) projects. All examples use TypeScript, Tailwind CSS, and shadcn/ui where applicable.

---

## 1. Error Boundaries

### Base Error Boundary Component

```tsx
// components/error-boundary.tsx
"use client";

import { Component, type ErrorInfo, type ReactNode } from "react";
import * as Sentry from "@sentry/nextjs";
import { Button } from "@/components/ui/button";
import { AlertTriangle } from "lucide-react";

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    // Log to Sentry
    Sentry.captureException(error, {
      extra: { componentStack: errorInfo.componentStack },
    });

    this.props.onError?.(error, errorInfo);
  }

  handleReset = () => {
    this.setState({ hasError: false, error: null });
  };

  render() {
    if (this.state.hasError) {
      if (this.props.fallback) return this.props.fallback;

      return (
        <div className="flex flex-col items-center justify-center gap-4 rounded-lg border border-destructive/20 bg-destructive/5 p-8">
          <AlertTriangle className="h-10 w-10 text-destructive" />
          <div className="text-center">
            <h3 className="text-lg font-semibold">Something went wrong</h3>
            <p className="mt-1 text-sm text-muted-foreground">
              {this.state.error?.message || "An unexpected error occurred."}
            </p>
          </div>
          <Button onClick={this.handleReset} variant="outline">
            Try again
          </Button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

### Per-Route Error Boundary (App Router)

```tsx
// app/dashboard/error.tsx
"use client";

import { useEffect } from "react";
import * as Sentry from "@sentry/nextjs";
import { Button } from "@/components/ui/button";

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    Sentry.captureException(error);
  }, [error]);

  return (
    <div className="flex min-h-[400px] flex-col items-center justify-center gap-4">
      <h2 className="text-xl font-semibold">Dashboard failed to load</h2>
      <p className="text-sm text-muted-foreground">
        Error: {error.message}
      </p>
      <Button onClick={reset}>Retry</Button>
    </div>
  );
}
```

### Global Error Boundary (App Router)

```tsx
// app/global-error.tsx
"use client";

import * as Sentry from "@sentry/nextjs";
import { useEffect } from "react";

export default function GlobalError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    Sentry.captureException(error);
  }, [error]);

  return (
    <html>
      <body className="flex min-h-screen items-center justify-center bg-background">
        <div className="text-center">
          <h1 className="text-2xl font-bold">Something went wrong</h1>
          <p className="mt-2 text-muted-foreground">
            A critical error occurred. Please try again.
          </p>
          <button
            onClick={reset}
            className="mt-4 rounded-md bg-primary px-4 py-2 text-primary-foreground"
          >
            Try again
          </button>
        </div>
      </body>
    </html>
  );
}
```

### Usage

```tsx
// Wrap specific sections
<ErrorBoundary>
  <DashboardWidget />
</ErrorBoundary>

// Custom fallback
<ErrorBoundary fallback={<p>Widget unavailable</p>}>
  <UnstableComponent />
</ErrorBoundary>

// With error callback
<ErrorBoundary onError={(err) => console.error("Caught:", err)}>
  <FeatureSection />
</ErrorBoundary>
```

---

## 2. Skeleton Screens

### Base Skeleton Component

```tsx
// components/ui/skeleton.tsx
import { cn } from "@/lib/utils";

function Skeleton({
  className,
  ...props
}: React.HTMLAttributes<HTMLDivElement>) {
  return (
    <div
      className={cn("animate-pulse rounded-md bg-muted", className)}
      {...props}
    />
  );
}

export { Skeleton };
```

### Shimmer Animation (alternative to pulse)

```css
/* globals.css */
@keyframes shimmer {
  0% {
    background-position: -200% 0;
  }
  100% {
    background-position: 200% 0;
  }
}

.skeleton-shimmer {
  background: linear-gradient(
    90deg,
    hsl(var(--muted)) 25%,
    hsl(var(--muted-foreground) / 0.08) 50%,
    hsl(var(--muted)) 75%
  );
  background-size: 200% 100%;
  animation: shimmer 1.5s ease-in-out infinite;
}
```

```tsx
// components/ui/skeleton-shimmer.tsx
import { cn } from "@/lib/utils";

export function SkeletonShimmer({
  className,
  ...props
}: React.HTMLAttributes<HTMLDivElement>) {
  return (
    <div
      className={cn("skeleton-shimmer rounded-md", className)}
      {...props}
    />
  );
}
```

### Content-Aware Skeletons

```tsx
// components/skeletons/card-skeleton.tsx
import { Skeleton } from "@/components/ui/skeleton";
import { Card, CardContent, CardHeader } from "@/components/ui/card";

export function CardSkeleton() {
  return (
    <Card>
      <CardHeader className="gap-2">
        <Skeleton className="h-5 w-1/3" />
        <Skeleton className="h-4 w-2/3" />
      </CardHeader>
      <CardContent className="space-y-3">
        <Skeleton className="h-4 w-full" />
        <Skeleton className="h-4 w-4/5" />
        <Skeleton className="h-4 w-3/5" />
      </CardContent>
    </Card>
  );
}

// components/skeletons/table-skeleton.tsx
export function TableSkeleton({ rows = 5, cols = 4 }: { rows?: number; cols?: number }) {
  return (
    <div className="rounded-md border">
      {/* Header */}
      <div className="flex gap-4 border-b p-4">
        {Array.from({ length: cols }).map((_, i) => (
          <Skeleton key={i} className="h-4 flex-1" />
        ))}
      </div>
      {/* Rows */}
      {Array.from({ length: rows }).map((_, rowIdx) => (
        <div key={rowIdx} className="flex gap-4 border-b p-4 last:border-0">
          {Array.from({ length: cols }).map((_, colIdx) => (
            <Skeleton key={colIdx} className="h-4 flex-1" />
          ))}
        </div>
      ))}
    </div>
  );
}

// components/skeletons/dashboard-skeleton.tsx
export function DashboardSkeleton() {
  return (
    <div className="space-y-6">
      {/* KPI cards */}
      <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
        {Array.from({ length: 4 }).map((_, i) => (
          <Card key={i}>
            <CardContent className="p-6">
              <Skeleton className="h-4 w-24" />
              <Skeleton className="mt-2 h-8 w-16" />
            </CardContent>
          </Card>
        ))}
      </div>
      {/* Chart area */}
      <Skeleton className="h-[300px] w-full rounded-lg" />
      {/* Table */}
      <TableSkeleton rows={5} cols={5} />
    </div>
  );
}
```

### Suspense Integration

```tsx
// app/dashboard/page.tsx
import { Suspense } from "react";
import { DashboardSkeleton } from "@/components/skeletons/dashboard-skeleton";
import { Dashboard } from "@/components/dashboard";

export default function DashboardPage() {
  return (
    <Suspense fallback={<DashboardSkeleton />}>
      <Dashboard />
    </Suspense>
  );
}

// app/dashboard/loading.tsx — automatic Suspense wrapper for the route
import { DashboardSkeleton } from "@/components/skeletons/dashboard-skeleton";

export default function Loading() {
  return <DashboardSkeleton />;
}
```

---

## 3. Empty States

### Generic Empty State Component

```tsx
// components/empty-state.tsx
import { type ReactNode } from "react";
import { Button } from "@/components/ui/button";
import { type LucideIcon } from "lucide-react";

interface EmptyStateProps {
  icon: LucideIcon;
  title: string;
  description: string;
  action?: {
    label: string;
    onClick: () => void;
  };
  secondaryAction?: {
    label: string;
    onClick: () => void;
  };
  children?: ReactNode;
}

export function EmptyState({
  icon: Icon,
  title,
  description,
  action,
  secondaryAction,
  children,
}: EmptyStateProps) {
  return (
    <div className="flex min-h-[300px] flex-col items-center justify-center rounded-lg border border-dashed p-8 text-center">
      <div className="flex h-16 w-16 items-center justify-center rounded-full bg-muted">
        <Icon className="h-8 w-8 text-muted-foreground" />
      </div>
      <h3 className="mt-4 text-lg font-semibold">{title}</h3>
      <p className="mt-1 max-w-sm text-sm text-muted-foreground">
        {description}
      </p>
      {(action || secondaryAction) && (
        <div className="mt-6 flex gap-3">
          {action && (
            <Button onClick={action.onClick}>{action.label}</Button>
          )}
          {secondaryAction && (
            <Button variant="outline" onClick={secondaryAction.onClick}>
              {secondaryAction.label}
            </Button>
          )}
        </div>
      )}
      {children}
    </div>
  );
}
```

### First-Use vs. No-Results Patterns

```tsx
// components/empty-states/first-use.tsx
import { EmptyState } from "@/components/empty-state";
import { Rocket, Search } from "lucide-react";

// First-use: user has never created items
export function FirstUseCampaigns({ onCreate }: { onCreate: () => void }) {
  return (
    <EmptyState
      icon={Rocket}
      title="Create your first campaign"
      description="Campaigns help you organize and track your marketing efforts. Get started by creating one."
      action={{ label: "Create Campaign", onClick: onCreate }}
    />
  );
}

// No results: user searched/filtered but nothing matched
export function NoSearchResults({ query, onClear }: { query: string; onClear: () => void }) {
  return (
    <EmptyState
      icon={Search}
      title="No results found"
      description={`No items match "${query}". Try adjusting your search or filters.`}
      action={{ label: "Clear filters", onClick: onClear }}
    />
  );
}
```

### Usage in a List Component

```tsx
// components/campaign-list.tsx
"use client";

import { useCampaigns } from "@/hooks/use-campaigns";
import { FirstUseCampaigns, NoSearchResults } from "@/components/empty-states/first-use";

export function CampaignList({ searchQuery }: { searchQuery: string }) {
  const { data: campaigns, isLoading } = useCampaigns();

  if (isLoading) return <TableSkeleton rows={5} cols={4} />;

  // Distinguish between first-use and no-results
  if (!campaigns?.length) {
    if (searchQuery) {
      return <NoSearchResults query={searchQuery} onClear={() => {}} />;
    }
    return <FirstUseCampaigns onCreate={() => {}} />;
  }

  return (
    <ul>
      {campaigns.map((c) => (
        <li key={c.id}>{c.name}</li>
      ))}
    </ul>
  );
}
```

---

## 4. Toast Notifications

### Sonner Setup

```tsx
// app/layout.tsx
import { Toaster } from "sonner";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="de">
      <body>
        {children}
        <Toaster
          position="bottom-right"
          toastOptions={{
            duration: 4000,
            classNames: {
              toast: "bg-background border-border text-foreground shadow-lg",
              title: "font-semibold",
              description: "text-muted-foreground text-sm",
              success: "border-green-500/30 bg-green-50 dark:bg-green-950",
              error: "border-destructive/30 bg-red-50 dark:bg-red-950",
            },
          }}
          richColors
        />
      </body>
    </html>
  );
}
```

### Toast Patterns

```tsx
// lib/toast-patterns.ts
import { toast } from "sonner";

// Basic variants
export function showSuccess(message: string) {
  toast.success(message);
}

export function showError(message: string, description?: string) {
  toast.error(message, { description, duration: 6000 });
}

// Loading → success/error (promise pattern)
export async function toastPromise<T>(
  promise: Promise<T>,
  messages: { loading: string; success: string; error: string }
) {
  return toast.promise(promise, {
    loading: messages.loading,
    success: messages.success,
    error: messages.error,
  });
}

// Action toast (e.g., undo)
export function showUndoToast(message: string, onUndo: () => void) {
  toast(message, {
    action: {
      label: "Undo",
      onClick: onUndo,
    },
    duration: 5000,
  });
}

// Persistent toast (e.g., for long operations)
export function showPersistentLoading(message: string) {
  return toast.loading(message, { duration: Infinity });
}
```

### Usage

```tsx
"use client";

import { toast } from "sonner";
import { toastPromise, showUndoToast } from "@/lib/toast-patterns";

function SaveButton() {
  const handleSave = async () => {
    await toastPromise(
      fetch("/api/save", { method: "POST" }),
      {
        loading: "Saving changes...",
        success: "Changes saved",
        error: "Failed to save changes",
      }
    );
  };

  return <button onClick={handleSave}>Save</button>;
}

function DeleteButton({ onDelete, onUndo }: { onDelete: () => void; onUndo: () => void }) {
  const handleDelete = () => {
    onDelete();
    showUndoToast("Item deleted", onUndo);
  };

  return <button onClick={handleDelete}>Delete</button>;
}
```

---

## 5. Offline Detection

### useOnlineStatus Hook

```tsx
// hooks/use-online-status.ts
"use client";

import { useSyncExternalStore, useCallback } from "react";

function subscribe(callback: () => void) {
  window.addEventListener("online", callback);
  window.addEventListener("offline", callback);
  return () => {
    window.removeEventListener("online", callback);
    window.removeEventListener("offline", callback);
  };
}

function getSnapshot() {
  return navigator.onLine;
}

function getServerSnapshot() {
  return true; // Assume online during SSR
}

export function useOnlineStatus() {
  return useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
}
```

### Offline Banner Component

```tsx
// components/offline-banner.tsx
"use client";

import { useOnlineStatus } from "@/hooks/use-online-status";
import { WifiOff } from "lucide-react";

export function OfflineBanner() {
  const isOnline = useOnlineStatus();

  if (isOnline) return null;

  return (
    <div
      role="alert"
      className="fixed inset-x-0 top-0 z-50 flex items-center justify-center gap-2 bg-yellow-500 px-4 py-2 text-sm font-medium text-yellow-950"
    >
      <WifiOff className="h-4 w-4" />
      You are offline. Changes will be synced when you reconnect.
    </div>
  );
}
```

### Offline Action Queue

```tsx
// lib/offline-queue.ts
type QueuedAction = {
  id: string;
  action: () => Promise<void>;
  createdAt: number;
};

class OfflineQueue {
  private queue: QueuedAction[] = [];
  private processing = false;

  enqueue(action: () => Promise<void>) {
    const id = crypto.randomUUID();
    this.queue.push({ id, action, createdAt: Date.now() });

    if (navigator.onLine) {
      this.flush();
    }
    return id;
  }

  async flush() {
    if (this.processing || this.queue.length === 0) return;
    this.processing = true;

    while (this.queue.length > 0) {
      const item = this.queue[0];
      try {
        await item.action();
        this.queue.shift(); // Remove on success
      } catch {
        break; // Stop on failure, retry later
      }
    }

    this.processing = false;
  }

  get pending() {
    return this.queue.length;
  }
}

export const offlineQueue = new OfflineQueue();

// Auto-flush when coming back online
if (typeof window !== "undefined") {
  window.addEventListener("online", () => offlineQueue.flush());
}
```

### Usage

```tsx
"use client";

import { offlineQueue } from "@/lib/offline-queue";
import { useOnlineStatus } from "@/hooks/use-online-status";
import { toast } from "sonner";

function LikeButton({ postId }: { postId: string }) {
  const isOnline = useOnlineStatus();

  const handleLike = () => {
    // Optimistic UI update here...

    if (!isOnline) {
      offlineQueue.enqueue(() =>
        fetch(`/api/posts/${postId}/like`, { method: "POST" }).then(() => {})
      );
      toast("Queued for sync when online");
    } else {
      fetch(`/api/posts/${postId}/like`, { method: "POST" });
    }
  };

  return <button onClick={handleLike}>Like</button>;
}
```

---

## 6. Optimistic Updates

### TanStack Query Optimistic Mutation

```tsx
// hooks/use-update-task.ts
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { toast } from "sonner";

interface Task {
  id: string;
  title: string;
  completed: boolean;
}

async function updateTask(task: Task): Promise<Task> {
  const res = await fetch(`/api/tasks/${task.id}`, {
    method: "PATCH",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(task),
  });
  if (!res.ok) throw new Error("Failed to update task");
  return res.json();
}

export function useUpdateTask() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: updateTask,

    onMutate: async (updatedTask) => {
      // Cancel outgoing refetches so they don't overwrite our optimistic update
      await queryClient.cancelQueries({ queryKey: ["tasks"] });

      // Snapshot previous value for rollback
      const previousTasks = queryClient.getQueryData<Task[]>(["tasks"]);

      // Optimistically update the cache
      queryClient.setQueryData<Task[]>(["tasks"], (old) =>
        old?.map((t) => (t.id === updatedTask.id ? { ...t, ...updatedTask } : t))
      );

      return { previousTasks };
    },

    onError: (_err, _updatedTask, context) => {
      // Rollback on error
      if (context?.previousTasks) {
        queryClient.setQueryData(["tasks"], context.previousTasks);
      }
      toast.error("Failed to update task. Changes reverted.");
    },

    onSettled: () => {
      // Refetch to ensure server state
      queryClient.invalidateQueries({ queryKey: ["tasks"] });
    },
  });
}
```

### Usage

```tsx
"use client";

import { useUpdateTask } from "@/hooks/use-update-task";

function TaskItem({ task }: { task: Task }) {
  const { mutate, isPending } = useUpdateTask();

  return (
    <div className="flex items-center gap-3">
      <input
        type="checkbox"
        checked={task.completed}
        onChange={() => mutate({ ...task, completed: !task.completed })}
        disabled={isPending}
      />
      <span className={task.completed ? "line-through text-muted-foreground" : ""}>
        {task.title}
      </span>
      {isPending && (
        <span className="text-xs text-muted-foreground">Saving...</span>
      )}
    </div>
  );
}
```

### Optimistic Add (new item)

```tsx
// hooks/use-add-task.ts
export function useAddTask() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (title: string) => {
      const res = await fetch("/api/tasks", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ title }),
      });
      if (!res.ok) throw new Error("Failed to create task");
      return res.json();
    },

    onMutate: async (title) => {
      await queryClient.cancelQueries({ queryKey: ["tasks"] });
      const previousTasks = queryClient.getQueryData<Task[]>(["tasks"]);

      // Add optimistic task with temporary ID
      const optimisticTask: Task = {
        id: `temp-${Date.now()}`,
        title,
        completed: false,
      };

      queryClient.setQueryData<Task[]>(["tasks"], (old) => [
        ...(old || []),
        optimisticTask,
      ]);

      return { previousTasks };
    },

    onError: (_err, _title, context) => {
      if (context?.previousTasks) {
        queryClient.setQueryData(["tasks"], context.previousTasks);
      }
      toast.error("Failed to create task");
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ["tasks"] });
    },
  });
}
```

---

## 7. Loading States

### Loading Button with Spinner

```tsx
// components/ui/loading-button.tsx
import { Button, type ButtonProps } from "@/components/ui/button";
import { Loader2 } from "lucide-react";

interface LoadingButtonProps extends ButtonProps {
  loading?: boolean;
  loadingText?: string;
}

export function LoadingButton({
  loading,
  loadingText,
  children,
  disabled,
  ...props
}: LoadingButtonProps) {
  return (
    <Button disabled={disabled || loading} {...props}>
      {loading && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
      {loading && loadingText ? loadingText : children}
    </Button>
  );
}
```

### Page-Level Loading (App Router)

```tsx
// app/dashboard/loading.tsx
import { Skeleton } from "@/components/ui/skeleton";

export default function Loading() {
  return (
    <div className="flex flex-col gap-6 p-6">
      <Skeleton className="h-8 w-48" />
      <div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-4">
        {Array.from({ length: 4 }).map((_, i) => (
          <Skeleton key={i} className="h-32 rounded-xl" />
        ))}
      </div>
      <Skeleton className="h-[400px] rounded-xl" />
    </div>
  );
}
```

### Streaming with Suspense (granular loading)

```tsx
// app/dashboard/page.tsx
import { Suspense } from "react";
import { Skeleton } from "@/components/ui/skeleton";

// Each section loads independently — first available renders first
export default function DashboardPage() {
  return (
    <div className="space-y-6 p-6">
      <h1 className="text-2xl font-bold">Dashboard</h1>

      {/* KPIs load fast */}
      <Suspense fallback={<KpiSkeleton />}>
        <KpiCards />
      </Suspense>

      {/* Chart may take longer */}
      <Suspense fallback={<Skeleton className="h-[400px] rounded-xl" />}>
        <RevenueChart />
      </Suspense>

      {/* Table loads independently */}
      <Suspense fallback={<TableSkeleton rows={10} cols={5} />}>
        <RecentOrders />
      </Suspense>
    </div>
  );
}

function KpiSkeleton() {
  return (
    <div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-4">
      {Array.from({ length: 4 }).map((_, i) => (
        <Skeleton key={i} className="h-28 rounded-xl" />
      ))}
    </div>
  );
}
```

### Full-Page Spinner (for route transitions)

```tsx
// components/page-spinner.tsx
import { Loader2 } from "lucide-react";

export function PageSpinner() {
  return (
    <div className="flex min-h-[60vh] items-center justify-center">
      <Loader2 className="h-8 w-8 animate-spin text-muted-foreground" />
    </div>
  );
}
```

---

## 8. 404 / 403 / 500 Pages

### not-found.tsx (404)

```tsx
// app/not-found.tsx
import Link from "next/link";
import { Button } from "@/components/ui/button";
import { FileQuestion } from "lucide-react";

export default function NotFound() {
  return (
    <div className="flex min-h-[80vh] flex-col items-center justify-center gap-4 text-center">
      <FileQuestion className="h-16 w-16 text-muted-foreground" />
      <h1 className="text-4xl font-bold">404</h1>
      <p className="max-w-md text-muted-foreground">
        The page you are looking for does not exist or has been moved.
      </p>
      <Button asChild>
        <Link href="/">Back to Home</Link>
      </Button>
    </div>
  );
}
```

### Triggering not-found programmatically

```tsx
// app/projects/[id]/page.tsx
import { notFound } from "next/navigation";
import { getProject } from "@/lib/data";

export default async function ProjectPage({ params }: { params: { id: string } }) {
  const project = await getProject(params.id);

  if (!project) {
    notFound(); // Renders the nearest not-found.tsx
  }

  return <div>{project.name}</div>;
}
```

### error.tsx (500 — per-route)

```tsx
// app/error.tsx
"use client";

import { useEffect } from "react";
import { Button } from "@/components/ui/button";
import { AlertOctagon } from "lucide-react";

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    console.error(error);
  }, [error]);

  return (
    <div className="flex min-h-[80vh] flex-col items-center justify-center gap-4 text-center">
      <AlertOctagon className="h-16 w-16 text-destructive" />
      <h1 className="text-4xl font-bold">500</h1>
      <p className="max-w-md text-muted-foreground">
        An internal error occurred. Our team has been notified.
      </p>
      <Button onClick={reset}>Try again</Button>
    </div>
  );
}
```

### 403 Forbidden Page

```tsx
// components/forbidden.tsx
import Link from "next/link";
import { Button } from "@/components/ui/button";
import { ShieldX } from "lucide-react";

export function Forbidden() {
  return (
    <div className="flex min-h-[80vh] flex-col items-center justify-center gap-4 text-center">
      <ShieldX className="h-16 w-16 text-destructive" />
      <h1 className="text-4xl font-bold">403</h1>
      <p className="max-w-md text-muted-foreground">
        You do not have permission to access this page.
      </p>
      <div className="flex gap-3">
        <Button asChild variant="outline">
          <Link href="/">Go Home</Link>
        </Button>
        <Button asChild>
          <Link href="/login">Sign In</Link>
        </Button>
      </div>
    </div>
  );
}

// Usage in a server component or middleware:
// if (!hasAccess) return <Forbidden />;
```

---

## 9. Confirmation Dialogs

### AlertDialog Pattern (shadcn/ui)

```tsx
// components/confirm-dialog.tsx
"use client";

import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
  AlertDialogTrigger,
} from "@/components/ui/alert-dialog";
import { buttonVariants } from "@/components/ui/button";

interface ConfirmDialogProps {
  trigger: React.ReactNode;
  title: string;
  description: string;
  confirmLabel?: string;
  cancelLabel?: string;
  variant?: "default" | "destructive";
  onConfirm: () => void | Promise<void>;
}

export function ConfirmDialog({
  trigger,
  title,
  description,
  confirmLabel = "Confirm",
  cancelLabel = "Cancel",
  variant = "default",
  onConfirm,
}: ConfirmDialogProps) {
  return (
    <AlertDialog>
      <AlertDialogTrigger asChild>{trigger}</AlertDialogTrigger>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>{title}</AlertDialogTitle>
          <AlertDialogDescription>{description}</AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel>{cancelLabel}</AlertDialogCancel>
          <AlertDialogAction
            onClick={onConfirm}
            className={variant === "destructive" ? buttonVariants({ variant: "destructive" }) : ""}
          >
            {confirmLabel}
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}
```

### Async Confirmation Hook

```tsx
// hooks/use-confirm.tsx
"use client";

import { useState, useCallback, useRef } from "react";
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";
import { buttonVariants } from "@/components/ui/button";

interface ConfirmOptions {
  title: string;
  description: string;
  confirmLabel?: string;
  cancelLabel?: string;
  variant?: "default" | "destructive";
}

export function useConfirm() {
  const [open, setOpen] = useState(false);
  const [options, setOptions] = useState<ConfirmOptions>({
    title: "",
    description: "",
  });

  const resolveRef = useRef<(value: boolean) => void>();

  const confirm = useCallback((opts: ConfirmOptions): Promise<boolean> => {
    setOptions(opts);
    setOpen(true);
    return new Promise<boolean>((resolve) => {
      resolveRef.current = resolve;
    });
  }, []);

  const handleConfirm = () => {
    resolveRef.current?.(true);
    setOpen(false);
  };

  const handleCancel = () => {
    resolveRef.current?.(false);
    setOpen(false);
  };

  const ConfirmationDialog = (
    <AlertDialog open={open} onOpenChange={setOpen}>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>{options.title}</AlertDialogTitle>
          <AlertDialogDescription>{options.description}</AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel onClick={handleCancel}>
            {options.cancelLabel || "Cancel"}
          </AlertDialogCancel>
          <AlertDialogAction
            onClick={handleConfirm}
            className={
              options.variant === "destructive"
                ? buttonVariants({ variant: "destructive" })
                : ""
            }
          >
            {options.confirmLabel || "Confirm"}
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );

  return { confirm, ConfirmationDialog };
}
```

### Usage

```tsx
"use client";

import { useConfirm } from "@/hooks/use-confirm";
import { Button } from "@/components/ui/button";
import { Trash2 } from "lucide-react";

function ProjectSettings() {
  const { confirm, ConfirmationDialog } = useConfirm();

  const handleDelete = async () => {
    const confirmed = await confirm({
      title: "Delete project?",
      description:
        "This action cannot be undone. All data associated with this project will be permanently deleted.",
      confirmLabel: "Delete",
      variant: "destructive",
    });

    if (confirmed) {
      await fetch("/api/projects/123", { method: "DELETE" });
    }
  };

  return (
    <>
      <Button variant="destructive" onClick={handleDelete}>
        <Trash2 className="mr-2 h-4 w-4" />
        Delete Project
      </Button>
      {ConfirmationDialog}
    </>
  );
}
```

---

## 10. Form Validation Feedback

### Field-Level Inline Errors (React Hook Form + Zod)

```tsx
// components/forms/create-campaign-form.tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { LoadingButton } from "@/components/ui/loading-button";
import { toast } from "sonner";
import { CheckCircle2 } from "lucide-react";

const campaignSchema = z.object({
  name: z
    .string()
    .min(3, "Name must be at least 3 characters")
    .max(100, "Name must be under 100 characters"),
  budget: z
    .number({ invalid_type_error: "Budget must be a number" })
    .min(1, "Budget must be at least 1")
    .max(1_000_000, "Budget cannot exceed 1,000,000"),
  email: z.string().email("Please enter a valid email address"),
  startDate: z.string().min(1, "Start date is required"),
});

type CampaignFormData = z.infer<typeof campaignSchema>;

export function CreateCampaignForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting, isSubmitSuccessful },
    reset,
  } = useForm<CampaignFormData>({
    resolver: zodResolver(campaignSchema),
  });

  const onSubmit = async (data: CampaignFormData) => {
    const res = await fetch("/api/campaigns", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(data),
    });

    if (!res.ok) {
      toast.error("Failed to create campaign");
      return;
    }

    toast.success("Campaign created");
    reset();
  };

  // Success state
  if (isSubmitSuccessful) {
    return (
      <div className="flex flex-col items-center gap-3 rounded-lg border border-green-500/20 bg-green-50 p-8 text-center dark:bg-green-950">
        <CheckCircle2 className="h-10 w-10 text-green-600" />
        <h3 className="font-semibold">Campaign Created</h3>
        <p className="text-sm text-muted-foreground">
          Your campaign is ready to go.
        </p>
        <Button variant="outline" onClick={() => reset()}>
          Create another
        </Button>
      </div>
    );
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4" noValidate>
      {/* Error summary (for screen readers and complex forms) */}
      {Object.keys(errors).length > 0 && (
        <div
          role="alert"
          className="rounded-md border border-destructive/30 bg-destructive/5 p-3"
        >
          <p className="text-sm font-medium text-destructive">
            Please fix {Object.keys(errors).length} error(s) below:
          </p>
          <ul className="mt-1 list-inside list-disc text-sm text-destructive/80">
            {Object.entries(errors).map(([field, error]) => (
              <li key={field}>{error?.message}</li>
            ))}
          </ul>
        </div>
      )}

      {/* Field with inline error */}
      <div className="space-y-2">
        <Label htmlFor="name">Campaign Name</Label>
        <Input
          id="name"
          {...register("name")}
          aria-invalid={!!errors.name}
          aria-describedby={errors.name ? "name-error" : undefined}
          className={errors.name ? "border-destructive focus-visible:ring-destructive" : ""}
        />
        {errors.name && (
          <p id="name-error" className="text-sm text-destructive" role="alert">
            {errors.name.message}
          </p>
        )}
      </div>

      <div className="space-y-2">
        <Label htmlFor="budget">Budget (EUR)</Label>
        <Input
          id="budget"
          type="number"
          {...register("budget", { valueAsNumber: true })}
          aria-invalid={!!errors.budget}
          aria-describedby={errors.budget ? "budget-error" : undefined}
          className={errors.budget ? "border-destructive focus-visible:ring-destructive" : ""}
        />
        {errors.budget && (
          <p id="budget-error" className="text-sm text-destructive" role="alert">
            {errors.budget.message}
          </p>
        )}
      </div>

      <div className="space-y-2">
        <Label htmlFor="email">Contact Email</Label>
        <Input
          id="email"
          type="email"
          {...register("email")}
          aria-invalid={!!errors.email}
          aria-describedby={errors.email ? "email-error" : undefined}
          className={errors.email ? "border-destructive focus-visible:ring-destructive" : ""}
        />
        {errors.email && (
          <p id="email-error" className="text-sm text-destructive" role="alert">
            {errors.email.message}
          </p>
        )}
      </div>

      <div className="space-y-2">
        <Label htmlFor="startDate">Start Date</Label>
        <Input
          id="startDate"
          type="date"
          {...register("startDate")}
          aria-invalid={!!errors.startDate}
          aria-describedby={errors.startDate ? "startDate-error" : undefined}
          className={errors.startDate ? "border-destructive focus-visible:ring-destructive" : ""}
        />
        {errors.startDate && (
          <p id="startDate-error" className="text-sm text-destructive" role="alert">
            {errors.startDate.message}
          </p>
        )}
      </div>

      <LoadingButton type="submit" loading={isSubmitting} loadingText="Creating...">
        Create Campaign
      </LoadingButton>
    </form>
  );
}
```

### Reusable Form Field Component

```tsx
// components/forms/form-field.tsx
import { type FieldError } from "react-hook-form";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { cn } from "@/lib/utils";

interface FormFieldProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: FieldError;
  description?: string;
}

export function FormField({
  label,
  error,
  description,
  id,
  className,
  ...props
}: FormFieldProps) {
  const fieldId = id || props.name;
  const errorId = `${fieldId}-error`;
  const descId = `${fieldId}-description`;

  return (
    <div className="space-y-2">
      <Label htmlFor={fieldId}>{label}</Label>
      {description && (
        <p id={descId} className="text-sm text-muted-foreground">
          {description}
        </p>
      )}
      <Input
        id={fieldId}
        aria-invalid={!!error}
        aria-describedby={
          [error ? errorId : null, description ? descId : null]
            .filter(Boolean)
            .join(" ") || undefined
        }
        className={cn(
          error && "border-destructive focus-visible:ring-destructive",
          className
        )}
        {...props}
      />
      {error && (
        <p id={errorId} className="text-sm text-destructive" role="alert">
          {error.message}
        </p>
      )}
    </div>
  );
}
```

### Server-Side Validation Feedback (Server Actions)

```tsx
// app/actions/create-campaign.ts
"use server";

import { z } from "zod";

const schema = z.object({
  name: z.string().min(3),
  budget: z.number().min(1),
});

type FormState = {
  success: boolean;
  errors?: Record<string, string[]>;
  message?: string;
};

export async function createCampaign(
  _prevState: FormState,
  formData: FormData
): Promise<FormState> {
  const parsed = schema.safeParse({
    name: formData.get("name"),
    budget: Number(formData.get("budget")),
  });

  if (!parsed.success) {
    return {
      success: false,
      errors: parsed.error.flatten().fieldErrors,
    };
  }

  // Check for server-side uniqueness
  const exists = await checkNameExists(parsed.data.name);
  if (exists) {
    return {
      success: false,
      errors: { name: ["A campaign with this name already exists"] },
    };
  }

  await saveCampaign(parsed.data);
  return { success: true, message: "Campaign created" };
}
```

```tsx
// components/forms/server-form.tsx
"use client";

import { useActionState } from "react";
import { createCampaign } from "@/app/actions/create-campaign";
import { LoadingButton } from "@/components/ui/loading-button";

export function ServerForm() {
  const [state, formAction, isPending] = useActionState(createCampaign, {
    success: false,
  });

  return (
    <form action={formAction} className="space-y-4">
      <div className="space-y-2">
        <label htmlFor="name">Name</label>
        <input id="name" name="name" className="..." />
        {state.errors?.name?.map((err) => (
          <p key={err} className="text-sm text-destructive">{err}</p>
        ))}
      </div>

      <LoadingButton type="submit" loading={isPending}>
        Create
      </LoadingButton>

      {state.success && (
        <p className="text-sm text-green-600">{state.message}</p>
      )}
    </form>
  );
}
```
