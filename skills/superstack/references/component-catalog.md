# Superstack Component Catalog

## Table of Contents
- [1. Button](#1-button)
  - [Props Interface](#props-interface)
  - [Variants & Sizes](#variants--sizes)
  - [Accessibility](#accessibility)
  - [Implementation Notes](#implementation-notes)
- [2. Input](#2-input)
  - [Props Interface](#props-interface-1)
  - [Accessibility](#accessibility-1)
  - [Variants & Sizes](#variants--sizes-1)
  - [Implementation Notes](#implementation-notes-1)
- [3. Select](#3-select)
  - [Props Interface](#props-interface-2)
  - [Accessibility](#accessibility-2)
  - [Implementation Notes](#implementation-notes-2)
- [4. Modal / Dialog](#4-modal--dialog)
  - [Props Interface](#props-interface-3)
  - [Accessibility](#accessibility-3)
  - [Variants & Sizes](#variants--sizes-2)
  - [Implementation Notes](#implementation-notes-3)
- [5. DataTable](#5-datatable)
  - [Props Interface](#props-interface-4)
  - [Accessibility](#accessibility-4)
  - [Implementation Notes](#implementation-notes-4)
- [6. Command Palette](#6-command-palette)
  - [Props Interface](#props-interface-5)
  - [Accessibility](#accessibility-5)
  - [Implementation Notes](#implementation-notes-5)
- [7. Toast](#7-toast)
  - [Props Interface](#props-interface-6)
  - [Accessibility](#accessibility-6)
  - [Variants](#variants)
  - [Implementation Notes](#implementation-notes-6)
- [8. Dropdown Menu](#8-dropdown-menu)
  - [Props Interface](#props-interface-7)
  - [Accessibility](#accessibility-7)
  - [Implementation Notes](#implementation-notes-7)
- [9. Tabs](#9-tabs)
  - [Props Interface](#props-interface-8)
  - [Accessibility](#accessibility-8)
  - [Implementation Notes](#implementation-notes-8)
- [10. Avatar](#10-avatar)
  - [Props Interface](#props-interface-9)
  - [Sizes](#sizes)
  - [Accessibility](#accessibility-9)
  - [Implementation Notes](#implementation-notes-9)
- [11. Badge](#11-badge)
  - [Props Interface](#props-interface-10)
  - [Accessibility](#accessibility-10)
  - [Implementation Notes](#implementation-notes-10)
- [12. Card](#12-card)
  - [Props Interface](#props-interface-11)
  - [Accessibility](#accessibility-11)
  - [Implementation Notes](#implementation-notes-11)
- [13. Sidebar / Navigation](#13-sidebar--navigation)
  - [Props Interface](#props-interface-12)
  - [Accessibility](#accessibility-12)
  - [Implementation Notes](#implementation-notes-12)
- [14. FileUpload](#14-fileupload)
  - [Props Interface](#props-interface-13)
  - [Accessibility](#accessibility-13)
  - [Implementation Notes](#implementation-notes-13)
- [15. DatePicker](#15-datepicker)
  - [Props Interface](#props-interface-14)
  - [Presets (Common)](#presets-common)
  - [Accessibility](#accessibility-14)
  - [Implementation Notes](#implementation-notes-14)

15 mandatory components for every Superstack project. Built with React, TypeScript, Tailwind CSS, and shadcn/ui conventions (Radix UI primitives + `class-variance-authority`).

---

## 1. Button

### Props Interface

```typescript
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "secondary" | "destructive" | "ghost" | "link";
  size?: "sm" | "md" | "lg" | "icon";
  loading?: boolean;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
  asChild?: boolean; // Radix Slot pattern — render as child element (e.g. <Link>)
}
```

### Variants & Sizes

| Variant | Background | Text | Border | Hover |
|---|---|---|---|---|
| primary | `bg-primary` | `text-primary-foreground` | none | `bg-primary/90` |
| secondary | `bg-secondary` | `text-secondary-foreground` | none | `bg-secondary/80` |
| destructive | `bg-destructive` | `text-destructive-foreground` | none | `bg-destructive/90` |
| ghost | transparent | `text-foreground` | none | `bg-accent` |
| link | transparent | `text-primary` | none | underline |

| Size | Padding | Height | Font |
|---|---|---|---|
| sm | `px-3` | `h-8` | `text-xs` |
| md | `px-4` | `h-10` | `text-sm` |
| lg | `px-6` | `h-12` | `text-base` |
| icon | `p-0` | `h-10 w-10` | — |

### Accessibility

- Native `<button>` element (or `asChild` slot).
- `aria-disabled="true"` when `loading` or `disabled` — keep button focusable but inert.
- `aria-busy="true"` when `loading`.
- Spinner replaces `leftIcon` during loading; keep text visible for screen readers.
- Minimum touch target 44x44 px on mobile.

### Implementation Notes

- Use `cva` for variant/size matrix.
- `asChild` uses Radix `<Slot>` to merge props onto child element.
- Loading state: disable click handler, show spinner, preserve button width with `min-w-[original]`.

---

## 2. Input

### Props Interface

```typescript
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label?: string;
  helperText?: string;
  error?: string; // replaces helperText when present
  prefix?: React.ReactNode; // e.g. "$" or icon
  suffix?: React.ReactNode; // e.g. unit label or icon button
}
```

### Accessibility

- `<label>` linked via `htmlFor` / `id` (auto-generated if not provided).
- `aria-describedby` pointing to helper text or error message element.
- `aria-invalid="true"` when `error` is set.
- `aria-required="true"` when `required`.
- Error messages use `role="alert"` for live announcement.

### Variants & Sizes

- Default, error (red ring), disabled (muted background).
- Sizes: `sm` (h-8), `md` (h-10), `lg` (h-12).

### Implementation Notes

- Wrap in a `<div>` containing label, input row (prefix + input + suffix), and message slot.
- Use `React.useId()` for stable, SSR-safe IDs.
- Prefix/suffix are decorative containers inside the input border, not separate elements.

---

## 3. Select

### Props Interface

```typescript
interface SelectProps<T = string> {
  options: SelectOption<T>[];
  value?: T | T[];
  onChange: (value: T | T[]) => void;
  placeholder?: string;
  searchable?: boolean;
  multi?: boolean;
  creatable?: boolean; // allow user to add new options
  onCreateOption?: (inputValue: string) => SelectOption<T> | Promise<SelectOption<T>>;
  async?: boolean;
  onSearch?: (query: string) => Promise<SelectOption<T>[]>;
  clearable?: boolean;
  disabled?: boolean;
  error?: string;
  label?: string;
}

interface SelectOption<T = string> {
  value: T;
  label: string;
  icon?: React.ReactNode;
  disabled?: boolean;
  group?: string;
}
```

### Accessibility

- `role="combobox"` on trigger, `role="listbox"` on dropdown.
- `aria-expanded`, `aria-activedescendant` for keyboard tracking.
- Arrow keys navigate, Enter selects, Escape closes.
- Type-ahead: typing characters jumps to matching option.
- Multi-select: selected items as chips with `aria-label="Remove {label}"` on remove button.

### Implementation Notes

- Build on top of Radix `<Popover>` + custom listbox, or use cmdk for searchable variant.
- Async: debounce `onSearch` (300ms), show loading spinner in dropdown.
- Creatable: show "Create {input}" option at bottom when no match.
- Virtualize long lists (>100 items) with `@tanstack/react-virtual`.

---

## 4. Modal / Dialog

### Props Interface

```typescript
interface DialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  title: string;
  description?: string;
  children: React.ReactNode;
  size?: "sm" | "md" | "lg" | "xl" | "full";
  preventClose?: boolean; // disable escape and click-outside
  initialFocusRef?: React.RefObject<HTMLElement>;
}
```

### Accessibility

- Built on Radix `<Dialog>` — provides focus trap, return focus, `role="dialog"`, `aria-modal="true"`.
- `aria-labelledby` points to title, `aria-describedby` points to description.
- Escape closes (unless `preventClose`).
- Click outside overlay closes (unless `preventClose`).
- Scroll lock on `<body>` when open.

### Variants & Sizes

| Size | Max Width |
|---|---|
| sm | `max-w-sm` (384px) |
| md | `max-w-lg` (512px) |
| lg | `max-w-2xl` (672px) |
| xl | `max-w-4xl` (896px) |
| full | `max-w-[calc(100vw-2rem)]` |

### Implementation Notes

- Stacked modals: each new dialog gets higher `z-index`; closing removes top-most.
- Animate with Framer Motion: overlay fade, content scale+fade.
- Long content: dialog body scrolls (`overflow-y-auto max-h-[80vh]`), header and footer stay fixed.
- Use `<DialogHeader>`, `<DialogBody>`, `<DialogFooter>` composition components.

---

## 5. DataTable

### Props Interface

```typescript
interface DataTableProps<TData> {
  columns: ColumnDef<TData>[]; // TanStack Table column definitions
  data: TData[];
  pagination?: {
    pageSize?: number;
    pageSizeOptions?: number[];
    serverSide?: boolean;
    totalRows?: number;
    onPageChange?: (page: number, pageSize: number) => void;
  };
  sorting?: {
    serverSide?: boolean;
    onSortChange?: (sorting: SortingState) => void;
  };
  filtering?: {
    globalFilter?: boolean;
    columnFilters?: boolean;
    serverSide?: boolean;
    onFilterChange?: (filters: ColumnFiltersState) => void;
  };
  selection?: {
    enabled?: boolean;
    onSelectionChange?: (rows: TData[]) => void;
  };
  bulkActions?: BulkAction<TData>[];
  emptyState?: React.ReactNode;
  loading?: boolean;
  columnVisibility?: boolean; // show column toggle dropdown
  stickyHeader?: boolean;
}

interface BulkAction<TData> {
  label: string;
  icon?: React.ReactNode;
  variant?: "default" | "destructive";
  onAction: (rows: TData[]) => void | Promise<void>;
}
```

### Accessibility

- Use `<table>`, `<thead>`, `<tbody>`, `<th>`, `<td>` — real table semantics.
- Sortable columns: `aria-sort="ascending" | "descending" | "none"`, clickable `<th>` as `<button>`.
- Row selection: checkboxes with `aria-label="Select row"`.
- Pagination: `nav[aria-label="Pagination"]` with labeled buttons.

### Implementation Notes

- Built on `@tanstack/react-table` v8.
- Server-side: parent manages state, table just renders.
- Loading: skeleton rows matching column count.
- Empty state: centered illustration + message + optional CTA.
- Column visibility: dropdown with checkboxes per column.
- Bulk actions toolbar appears above table when rows selected.

---

## 6. Command Palette

### Props Interface

```typescript
interface CommandPaletteProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  placeholder?: string;
  sections: CommandSection[];
  recentItems?: CommandItem[];
  onSelect: (item: CommandItem) => void;
  footer?: React.ReactNode;
}

interface CommandSection {
  heading: string;
  items: CommandItem[];
}

interface CommandItem {
  id: string;
  label: string;
  description?: string;
  icon?: React.ReactNode;
  shortcut?: string[]; // e.g. ["Cmd", "S"]
  keywords?: string[]; // hidden search terms
  onSelect?: () => void;
}
```

### Accessibility

- Uses cmdk library (Cmd+K pattern).
- `role="dialog"` with `aria-label="Command palette"`.
- `role="listbox"` for results, items are `role="option"`.
- `aria-activedescendant` tracks highlighted item.
- Arrow up/down navigate, Enter selects, Escape closes.

### Implementation Notes

- Global `Cmd+K` / `Ctrl+K` listener registered at app root.
- Fuzzy search over `label`, `description`, `keywords`.
- Sections rendered as groups with headings.
- Recent items shown when input is empty.
- Animate with scale + fade via Framer Motion.

---

## 7. Toast

### Props Interface

```typescript
// Typically used via a hook/function, not JSX
interface ToastOptions {
  variant?: "success" | "error" | "warning" | "info";
  title: string;
  description?: string;
  action?: {
    label: string;
    onClick: () => void;
  };
  duration?: number; // ms, default 5000, 0 = persistent
  dismissible?: boolean; // default true
}

// Usage: toast({ variant: "success", title: "Saved!" })
// Usage: toast.success("Saved!")
```

### Accessibility

- Container: `role="region"`, `aria-label="Notifications"`, `aria-live="polite"`.
- Error toasts: `aria-live="assertive"`.
- Action button and dismiss button focusable.
- Pause auto-dismiss on hover/focus.

### Variants

| Variant | Icon | Border Color |
|---|---|---|
| success | CheckCircle | `border-green-500` |
| error | XCircle | `border-red-500` |
| warning | AlertTriangle | `border-yellow-500` |
| info | Info | `border-blue-500` |

### Implementation Notes

- Use `sonner` or custom Radix-based toast.
- Stack management: max 3 visible, older ones collapse into a count.
- Position: bottom-right default, configurable via `<Toaster position="top-center" />`.
- Swipe to dismiss on mobile.

---

## 8. Dropdown Menu

### Props Interface

```typescript
interface DropdownMenuProps {
  trigger: React.ReactNode;
  items: DropdownMenuItem[];
  align?: "start" | "center" | "end";
  side?: "top" | "right" | "bottom" | "left";
}

type DropdownMenuItem =
  | { type: "item"; label: string; icon?: React.ReactNode; shortcut?: string; disabled?: boolean; onSelect: () => void }
  | { type: "checkbox"; label: string; checked: boolean; onCheckedChange: (checked: boolean) => void }
  | { type: "radio"; label: string; value: string }
  | { type: "sub"; label: string; icon?: React.ReactNode; items: DropdownMenuItem[] }
  | { type: "separator" }
  | { type: "label"; text: string };
```

### Accessibility

- Built on Radix `<DropdownMenu>`.
- `role="menu"`, items are `role="menuitem"`, `role="menuitemcheckbox"`, `role="menuitemradio"`.
- Arrow keys navigate, Enter/Space selects, Escape closes, right arrow opens submenu, left arrow closes submenu.
- `aria-expanded` on trigger.

### Implementation Notes

- Submenus open on hover (desktop) or tap (mobile).
- Shortcut text right-aligned in item row.
- Icons left-aligned, same width column for visual alignment.
- Animate with Radix built-in or Framer Motion.

---

## 9. Tabs

### Props Interface

```typescript
interface TabsProps {
  defaultValue?: string;
  value?: string; // controlled
  onValueChange?: (value: string) => void;
  orientation?: "horizontal" | "vertical";
  children: React.ReactNode; // <TabsList> + <TabsContent>
}

interface TabsTriggerProps {
  value: string;
  icon?: React.ReactNode;
  badge?: string | number;
  disabled?: boolean;
}

interface TabsContentProps {
  value: string;
  lazy?: boolean; // only mount when active
  keepMounted?: boolean; // keep in DOM after first mount
}
```

### Accessibility

- Built on Radix `<Tabs>`.
- `role="tablist"`, `role="tab"`, `role="tabpanel"`.
- `aria-selected` on active tab, `aria-controls` / `aria-labelledby` linking.
- Arrow keys move between tabs (horizontal: left/right, vertical: up/down).
- Home/End jump to first/last tab.

### Implementation Notes

- Controlled or uncontrolled via `value` / `defaultValue`.
- Lazy loading: wrap content in `{active && <Component />}` or use `forceMount` with CSS visibility.
- Active indicator: animated underline via `layoutId` (Framer Motion) or CSS transform.
- Badge: small counter next to tab label.

---

## 10. Avatar

### Props Interface

```typescript
interface AvatarProps {
  src?: string | null;
  alt: string;
  fallback: string; // initials, e.g. "EA"
  size?: "xs" | "sm" | "md" | "lg" | "xl";
  status?: "online" | "offline" | "busy" | "away";
}

interface AvatarGroupProps {
  max?: number; // show "+N" overflow
  size?: AvatarProps["size"];
  children: React.ReactNode; // Avatar elements
}
```

### Sizes

| Size | Dimensions | Font |
|---|---|---|
| xs | `h-6 w-6` | `text-[10px]` |
| sm | `h-8 w-8` | `text-xs` |
| md | `h-10 w-10` | `text-sm` |
| lg | `h-12 w-12` | `text-base` |
| xl | `h-16 w-16` | `text-lg` |

### Accessibility

- `<img>` with `alt` text.
- Fallback initials in `<span aria-label="{alt}">`.
- Status indicator: `aria-label="Status: online"` on the dot.

### Implementation Notes

- Use Radix `<Avatar>` — handles image load states (loading, loaded, error).
- Fallback: show initials on colored background (deterministic color from name hash).
- Status dot: absolute positioned, bottom-right, with ring matching parent background.
- AvatarGroup: negative margin overlap, last avatar on top (z-index), "+N" as final circle.

---

## 11. Badge

### Props Interface

```typescript
interface BadgeProps {
  variant?: "default" | "secondary" | "destructive" | "outline" | "success" | "warning";
  size?: "sm" | "md";
  dot?: boolean; // colored dot indicator before text
  removable?: boolean;
  onRemove?: () => void;
  icon?: React.ReactNode;
  children: React.ReactNode;
}
```

### Accessibility

- Rendered as `<span>` (non-interactive) or `<div>` with remove button.
- Remove button: `aria-label="Remove {children}"`.
- Dot indicator is decorative (`aria-hidden="true"`).

### Implementation Notes

- Use `cva` for variant styles.
- Small: `text-xs px-2 py-0.5`. Medium: `text-sm px-2.5 py-1`.
- Dot: `h-1.5 w-1.5 rounded-full` before text, color matches variant.
- Removable: add `X` button on right side, `hover:bg-muted` on the X area.

---

## 12. Card

### Props Interface

```typescript
interface CardProps extends React.HTMLAttributes<HTMLDivElement> {
  asLink?: boolean; // makes entire card clickable
  href?: string;
  loading?: boolean; // show skeleton
  hover?: boolean; // enable hover effect
}

// Composition components
const CardHeader: React.FC<{ children: React.ReactNode }>;
const CardTitle: React.FC<{ children: React.ReactNode }>;
const CardDescription: React.FC<{ children: React.ReactNode }>;
const CardContent: React.FC<{ children: React.ReactNode }>;
const CardFooter: React.FC<{ children: React.ReactNode }>;
```

### Accessibility

- If clickable: wrap in `<a>` or use `role="link"` with `tabIndex={0}` and Enter/Space handler.
- Card title should be a heading (`<h3>` typically).
- Interactive elements inside clickable cards need `stopPropagation`.

### Implementation Notes

- Base: `rounded-lg border bg-card text-card-foreground shadow-sm`.
- Hover effect: `hover:shadow-md hover:border-primary/20 transition-all`.
- Loading skeleton: use `<Skeleton>` components matching card layout.
- Composition pattern like shadcn/ui — each sub-component is a styled div.

---

## 13. Sidebar / Navigation

### Props Interface

```typescript
interface SidebarProps {
  items: NavItem[];
  collapsed?: boolean;
  onCollapsedChange?: (collapsed: boolean) => void;
  header?: React.ReactNode; // logo area
  footer?: React.ReactNode; // user menu area
}

interface NavItem {
  label: string;
  href: string;
  icon: React.ReactNode;
  badge?: string | number;
  active?: boolean;
  children?: NavItem[]; // nested items
  section?: string; // group heading
}
```

### Accessibility

- `<nav aria-label="Main navigation">`.
- Current page: `aria-current="page"`.
- Collapsible groups: `aria-expanded` on toggle button.
- Mobile: rendered as `<Sheet>` (slide-in drawer) with focus trap.
- Keyboard: Tab through items, Enter activates, arrow keys optional.

### Implementation Notes

- Collapsed mode: icons only, tooltips on hover (via Radix `<Tooltip>`).
- Nested items: indented, parent toggles with chevron icon rotating.
- Active state: `bg-accent text-accent-foreground` with left border indicator.
- Mobile: use Radix `<Sheet>` or Vaul drawer, triggered by hamburger button.
- Persist collapsed state in `localStorage` or cookie.
- Transition width with `transition-[width]` for smooth collapse animation.

---

## 14. FileUpload

### Props Interface

```typescript
interface FileUploadProps {
  accept?: string; // MIME types, e.g. "image/*,.pdf"
  maxSize?: number; // bytes
  maxFiles?: number;
  multiple?: boolean;
  value?: FileWithPreview[];
  onChange: (files: FileWithPreview[]) => void;
  onError?: (errors: FileError[]) => void;
  disabled?: boolean;
  preview?: boolean; // show image thumbnails
}

interface FileWithPreview {
  file: File;
  preview?: string; // object URL for images
  progress?: number; // 0-100 upload progress
  status?: "pending" | "uploading" | "complete" | "error";
  error?: string;
}

interface FileError {
  file: File;
  code: "file-too-large" | "file-invalid-type" | "too-many-files";
  message: string;
}
```

### Accessibility

- Drop zone: `role="button"`, `tabIndex={0}`, Enter/Space opens file dialog.
- `aria-label="Upload files"` on drop zone.
- Drag state: visual indicator (dashed border change), `aria-live` region announcing "Drop files here".
- File list: each file with name, size, status, and remove button.
- Progress: `role="progressbar"`, `aria-valuenow`, `aria-valuemin`, `aria-valuemax`.

### Implementation Notes

- Use `react-dropzone` for drag-and-drop handling.
- Validate file type and size client-side before upload.
- Image preview: create `URL.createObjectURL()`, revoke on unmount.
- Progress: integrate with upload function (e.g. `XMLHttpRequest` or `tus` for resumable uploads).
- Upload to Supabase Storage: `supabase.storage.from('bucket').upload(path, file)`.

---

## 15. DatePicker

### Props Interface

```typescript
interface DatePickerProps {
  mode?: "single" | "range";
  value?: Date | DateRange;
  onChange: (value: Date | DateRange | undefined) => void;
  presets?: DatePreset[];
  locale?: string; // e.g. "de-DE"
  minDate?: Date;
  maxDate?: Date;
  disabled?: boolean;
  placeholder?: string;
}

interface DateRange {
  from: Date;
  to: Date;
}

interface DatePreset {
  label: string; // e.g. "Last 7 days"
  getValue: () => Date | DateRange;
}
```

### Presets (Common)

```typescript
const defaultPresets: DatePreset[] = [
  { label: "Today", getValue: () => new Date() },
  { label: "Last 7 days", getValue: () => ({ from: subDays(new Date(), 7), to: new Date() }) },
  { label: "Last 30 days", getValue: () => ({ from: subDays(new Date(), 30), to: new Date() }) },
  { label: "This month", getValue: () => ({ from: startOfMonth(new Date()), to: new Date() }) },
  { label: "Last month", getValue: () => ({ from: startOfMonth(subMonths(new Date(), 1)), to: endOfMonth(subMonths(new Date(), 1)) }) },
];
```

### Accessibility

- Trigger button shows formatted date, `aria-label="Choose date"`.
- Calendar: `role="grid"`, cells are `role="gridcell"`.
- Arrow keys navigate days, Page Up/Down navigate months.
- Selected date: `aria-selected="true"`.
- Range: both endpoints highlighted, range visually indicated.

### Implementation Notes

- Use `react-day-picker` (used by shadcn/ui date picker) + Radix `<Popover>`.
- Locale-aware formatting with `date-fns` `format()` and locale imports.
- Range mode: click start date, then end date. Hover shows preview range.
- Presets sidebar: list of buttons on the left side of the popover.
- Format display: `dd.MM.yyyy` for German locale, `MM/dd/yyyy` for US.
