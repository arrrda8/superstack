# Form Patterns Reference — React / Next.js

## Table of Contents
- [1. React Hook Form + Zod — Foundation](#1-react-hook-form-zod-foundation)
  - [Schema Definition & Typed Form](#schema-definition-typed-form)
  - [Custom Zod Refinements](#custom-zod-refinements)
- [2. Multi-Step Wizard](#2-multi-step-wizard)
  - [Persist Wizard State to localStorage](#persist-wizard-state-to-localstorage)
- [3. Conditional Fields](#3-conditional-fields)
  - [Simple Show/Hide with `watch()`](#simple-showhide-with-watch)
- [4. Dynamic Field Arrays](#4-dynamic-field-arrays)
  - [Nested Field Arrays (e.g., Teams with Members)](#nested-field-arrays-eg-teams-with-members)
- [5. Server Actions Integration](#5-server-actions-integration)
  - [With `useActionState` (React 19 / Next.js 15+)](#with-useactionstate-react-19-nextjs-15)
  - [React Hook Form + Server Actions (Hybrid)](#react-hook-form-server-actions-hybrid)
- [6. File Upload in Forms](#6-file-upload-in-forms)
- [7. Autosave](#7-autosave)
- [8. Address Autocomplete (Google Places)](#8-address-autocomplete-google-places)
- [9. Accessible Form Errors](#9-accessible-form-errors)
  - [Key ARIA Patterns Summary](#key-aria-patterns-summary)
- [10. Complex Patterns](#10-complex-patterns)
  - [Dependent Dropdowns](#dependent-dropdowns)
  - [Inline Editing](#inline-editing)
  - [Search-as-you-type Filter](#search-as-you-type-filter)
  - [Currency & Phone Input Masks](#currency-phone-input-masks)
  - [Zod Schemas for Currency & Phone](#zod-schemas-for-currency-phone)
- [Quick Reference: Common Zod Patterns for Forms](#quick-reference-common-zod-patterns-for-forms)

Comprehensive patterns for form handling using **React Hook Form + Zod + shadcn/ui**.

---

## 1. React Hook Form + Zod — Foundation

### Schema Definition & Typed Form

```tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import {
  Form,
  FormControl,
  FormDescription,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form";

// 1. Define schema
const contactSchema = z.object({
  name: z.string().min(2, "Name must be at least 2 characters"),
  email: z.string().email("Invalid email address"),
  age: z.coerce.number().min(18, "Must be 18+").max(120),
  website: z.string().url("Invalid URL").optional().or(z.literal("")),
  message: z.string().min(10).max(500),
});

// 2. Infer TypeScript type from schema
type ContactFormValues = z.infer<typeof contactSchema>;

export function ContactForm() {
  // 3. Initialize form with resolver
  const form = useForm<ContactFormValues>({
    resolver: zodResolver(contactSchema),
    defaultValues: {
      name: "",
      email: "",
      age: undefined,
      website: "",
      message: "",
    },
  });

  // 4. Typed submit handler
  async function onSubmit(data: ContactFormValues) {
    console.log("Validated data:", data);
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        <FormField
          control={form.control}
          name="name"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Name</FormLabel>
              <FormControl>
                <Input placeholder="Max Mustermann" {...field} />
              </FormControl>
              <FormDescription>Your full name.</FormDescription>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl>
                <Input type="email" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="age"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Age</FormLabel>
              <FormControl>
                <Input type="number" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <Button type="submit" disabled={form.formState.isSubmitting}>
          {form.formState.isSubmitting ? "Submitting..." : "Submit"}
        </Button>
      </form>
    </Form>
  );
}
```

### Custom Zod Refinements

```tsx
const passwordSchema = z
  .object({
    password: z
      .string()
      .min(8)
      .regex(/[A-Z]/, "Must contain uppercase")
      .regex(/[0-9]/, "Must contain a number"),
    confirmPassword: z.string(),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: "Passwords don't match",
    path: ["confirmPassword"], // attach error to confirmPassword field
  });

// Async validation (e.g., check if username is taken)
const signupSchema = z.object({
  username: z
    .string()
    .min(3)
    .refine(
      async (val) => {
        const res = await fetch(`/api/check-username?u=${val}`);
        const { available } = await res.json();
        return available;
      },
      { message: "Username is already taken" }
    ),
});
```

---

## 2. Multi-Step Wizard

```tsx
"use client";

import { useState } from "react";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Progress } from "@/components/ui/progress";
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form";

// Step schemas
const personalSchema = z.object({
  firstName: z.string().min(1, "Required"),
  lastName: z.string().min(1, "Required"),
  email: z.string().email(),
});

const addressSchema = z.object({
  street: z.string().min(1, "Required"),
  city: z.string().min(1, "Required"),
  zip: z.string().min(5, "Invalid ZIP"),
});

const preferencesSchema = z.object({
  plan: z.enum(["free", "pro", "enterprise"]),
  newsletter: z.boolean().default(false),
});

// Combined schema for final submission
const fullSchema = personalSchema.merge(addressSchema).merge(preferencesSchema);
type WizardFormValues = z.infer<typeof fullSchema>;

// Step configuration
const steps = [
  { id: "personal", label: "Personal Info", schema: personalSchema },
  { id: "address", label: "Address", schema: addressSchema },
  { id: "preferences", label: "Preferences", schema: preferencesSchema },
] as const;

type StepFields = {
  personal: (keyof z.infer<typeof personalSchema>)[];
  address: (keyof z.infer<typeof addressSchema>)[];
  preferences: (keyof z.infer<typeof preferencesSchema>)[];
};

const stepFields: StepFields = {
  personal: ["firstName", "lastName", "email"],
  address: ["street", "city", "zip"],
  preferences: ["plan", "newsletter"],
};

export function MultiStepWizard() {
  const [currentStep, setCurrentStep] = useState(0);

  const form = useForm<WizardFormValues>({
    resolver: zodResolver(fullSchema),
    defaultValues: {
      firstName: "",
      lastName: "",
      email: "",
      street: "",
      city: "",
      zip: "",
      plan: "free",
      newsletter: false,
    },
    mode: "onTouched", // validate on blur for better UX
  });

  const progress = ((currentStep + 1) / steps.length) * 100;

  // Validate only current step fields before advancing
  async function handleNext() {
    const step = steps[currentStep];
    const fields = stepFields[step.id];
    const isValid = await form.trigger(fields);
    if (isValid) setCurrentStep((prev) => prev + 1);
  }

  function handleBack() {
    setCurrentStep((prev) => prev - 1);
  }

  async function onSubmit(data: WizardFormValues) {
    console.log("Final data:", data);
  }

  return (
    <div className="max-w-lg mx-auto space-y-6">
      {/* Progress indicator */}
      <div className="space-y-2">
        <div className="flex justify-between text-sm text-muted-foreground">
          {steps.map((step, i) => (
            <span
              key={step.id}
              className={i <= currentStep ? "text-primary font-medium" : ""}
            >
              {step.label}
            </span>
          ))}
        </div>
        <Progress value={progress} />
      </div>

      <Form {...form}>
        <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
          {/* Step 1: Personal */}
          {currentStep === 0 && (
            <>
              <FormField
                control={form.control}
                name="firstName"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel>First Name</FormLabel>
                    <FormControl>
                      <Input {...field} />
                    </FormControl>
                    <FormMessage />
                  </FormItem>
                )}
              />
              <FormField
                control={form.control}
                name="lastName"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel>Last Name</FormLabel>
                    <FormControl>
                      <Input {...field} />
                    </FormControl>
                    <FormMessage />
                  </FormItem>
                )}
              />
              <FormField
                control={form.control}
                name="email"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel>Email</FormLabel>
                    <FormControl>
                      <Input type="email" {...field} />
                    </FormControl>
                    <FormMessage />
                  </FormItem>
                )}
              />
            </>
          )}

          {/* Step 2: Address */}
          {currentStep === 1 && (
            <>
              <FormField
                control={form.control}
                name="street"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel>Street</FormLabel>
                    <FormControl>
                      <Input {...field} />
                    </FormControl>
                    <FormMessage />
                  </FormItem>
                )}
              />
              <FormField
                control={form.control}
                name="city"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel>City</FormLabel>
                    <FormControl>
                      <Input {...field} />
                    </FormControl>
                    <FormMessage />
                  </FormItem>
                )}
              />
              <FormField
                control={form.control}
                name="zip"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel>ZIP Code</FormLabel>
                    <FormControl>
                      <Input {...field} />
                    </FormControl>
                    <FormMessage />
                  </FormItem>
                )}
              />
            </>
          )}

          {/* Step 3: Preferences — render your fields here */}
          {currentStep === 2 && (
            <div className="text-center py-8 text-muted-foreground">
              {/* Plan selection, newsletter checkbox, etc. */}
            </div>
          )}

          {/* Navigation */}
          <div className="flex justify-between pt-4">
            <Button
              type="button"
              variant="outline"
              onClick={handleBack}
              disabled={currentStep === 0}
            >
              Back
            </Button>

            {currentStep < steps.length - 1 ? (
              <Button type="button" onClick={handleNext}>
                Next
              </Button>
            ) : (
              <Button type="submit">Submit</Button>
            )}
          </div>
        </form>
      </Form>
    </div>
  );
}
```

### Persist Wizard State to localStorage

```tsx
import { useEffect } from "react";

function useFormPersistence<T extends Record<string, any>>(
  form: ReturnType<typeof useForm<T>>,
  key: string
) {
  // Load saved data on mount
  useEffect(() => {
    const saved = localStorage.getItem(key);
    if (saved) {
      try {
        const parsed = JSON.parse(saved);
        form.reset(parsed);
      } catch {}
    }
  }, []);

  // Save on every change
  useEffect(() => {
    const subscription = form.watch((data) => {
      localStorage.setItem(key, JSON.stringify(data));
    });
    return () => subscription.unsubscribe();
  }, [form.watch]);

  function clearSaved() {
    localStorage.removeItem(key);
  }

  return { clearSaved };
}
```

---

## 3. Conditional Fields

```tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { Input } from "@/components/ui/input";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form";

// Dynamic schema with discriminated union
const baseSchema = z.object({
  employmentType: z.enum(["employed", "freelancer", "student"]),
});

const employedSchema = baseSchema.extend({
  employmentType: z.literal("employed"),
  companyName: z.string().min(1, "Company name required"),
  position: z.string().min(1, "Position required"),
});

const freelancerSchema = baseSchema.extend({
  employmentType: z.literal("freelancer"),
  taxId: z.string().min(1, "Tax ID required"),
  hourlyRate: z.coerce.number().min(1),
});

const studentSchema = baseSchema.extend({
  employmentType: z.literal("student"),
  university: z.string().min(1, "University required"),
  graduationYear: z.coerce.number().min(2024).max(2030),
});

const formSchema = z.discriminatedUnion("employmentType", [
  employedSchema,
  freelancerSchema,
  studentSchema,
]);

type FormValues = z.infer<typeof formSchema>;

export function ConditionalForm() {
  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: {
      employmentType: "employed",
      companyName: "",
      position: "",
    },
  });

  // watch() returns reactive value
  const employmentType = form.watch("employmentType");

  // Reset dependent fields when type changes
  function handleTypeChange(value: string) {
    form.setValue("employmentType", value as FormValues["employmentType"]);
    // Clear all conditional fields
    form.resetField("companyName" as any);
    form.resetField("position" as any);
    form.resetField("taxId" as any);
    form.resetField("hourlyRate" as any);
    form.resetField("university" as any);
    form.resetField("graduationYear" as any);
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(console.log)} className="space-y-4">
        <FormField
          control={form.control}
          name="employmentType"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Employment Type</FormLabel>
              <Select
                onValueChange={handleTypeChange}
                defaultValue={field.value}
              >
                <FormControl>
                  <SelectTrigger>
                    <SelectValue />
                  </SelectTrigger>
                </FormControl>
                <SelectContent>
                  <SelectItem value="employed">Employed</SelectItem>
                  <SelectItem value="freelancer">Freelancer</SelectItem>
                  <SelectItem value="student">Student</SelectItem>
                </SelectContent>
              </Select>
              <FormMessage />
            </FormItem>
          )}
        />

        {/* Conditional fields based on employmentType */}
        {employmentType === "employed" && (
          <>
            <FormField
              control={form.control}
              name="companyName"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>Company Name</FormLabel>
                  <FormControl>
                    <Input {...field} />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />
            <FormField
              control={form.control}
              name="position"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>Position</FormLabel>
                  <FormControl>
                    <Input {...field} />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />
          </>
        )}

        {employmentType === "freelancer" && (
          <>
            <FormField
              control={form.control}
              name="taxId"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>Tax ID</FormLabel>
                  <FormControl>
                    <Input {...field} />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />
            <FormField
              control={form.control}
              name="hourlyRate"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>Hourly Rate (EUR)</FormLabel>
                  <FormControl>
                    <Input type="number" {...field} />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />
          </>
        )}

        {employmentType === "student" && (
          <>
            <FormField
              control={form.control}
              name="university"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>University</FormLabel>
                  <FormControl>
                    <Input {...field} />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />
          </>
        )}
      </form>
    </Form>
  );
}
```

### Simple Show/Hide with `watch()`

```tsx
// Quick pattern: toggle a field based on a checkbox
const showBilling = form.watch("differentBillingAddress");

return (
  <>
    <FormField
      control={form.control}
      name="differentBillingAddress"
      render={({ field }) => (
        <FormItem className="flex items-center gap-2">
          <FormControl>
            <Checkbox checked={field.value} onCheckedChange={field.onChange} />
          </FormControl>
          <FormLabel>Different billing address</FormLabel>
        </FormItem>
      )}
    />

    {showBilling && (
      <div className="space-y-4 pl-6 border-l-2">
        {/* billing address fields */}
      </div>
    )}
  </>
);
```

---

## 4. Dynamic Field Arrays

```tsx
"use client";

import { useForm, useFieldArray } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Trash2, Plus, GripVertical } from "lucide-react";
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form";

const lineItemSchema = z.object({
  description: z.string().min(1, "Description required"),
  quantity: z.coerce.number().min(1, "Min 1"),
  unitPrice: z.coerce.number().min(0.01, "Price required"),
});

const invoiceSchema = z.object({
  invoiceNumber: z.string().min(1),
  clientName: z.string().min(1),
  items: z
    .array(lineItemSchema)
    .min(1, "At least one line item required")
    .max(20, "Maximum 20 items"),
});

type InvoiceFormValues = z.infer<typeof invoiceSchema>;

export function InvoiceForm() {
  const form = useForm<InvoiceFormValues>({
    resolver: zodResolver(invoiceSchema),
    defaultValues: {
      invoiceNumber: "",
      clientName: "",
      items: [{ description: "", quantity: 1, unitPrice: 0 }],
    },
  });

  const { fields, append, remove, move } = useFieldArray({
    control: form.control,
    name: "items",
  });

  // Watch items for total calculation
  const watchedItems = form.watch("items");
  const total = watchedItems.reduce(
    (sum, item) => sum + (item.quantity || 0) * (item.unitPrice || 0),
    0
  );

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(console.log)} className="space-y-6">
        {/* Header fields */}
        <div className="grid grid-cols-2 gap-4">
          <FormField
            control={form.control}
            name="invoiceNumber"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Invoice #</FormLabel>
                <FormControl>
                  <Input {...field} />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />
          <FormField
            control={form.control}
            name="clientName"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Client</FormLabel>
                <FormControl>
                  <Input {...field} />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />
        </div>

        {/* Dynamic line items */}
        <div className="space-y-3">
          <div className="grid grid-cols-[auto_1fr_80px_100px_40px] gap-2 text-sm font-medium text-muted-foreground">
            <span />
            <span>Description</span>
            <span>Qty</span>
            <span>Price</span>
            <span />
          </div>

          {fields.map((field, index) => (
            <div
              key={field.id}
              className="grid grid-cols-[auto_1fr_80px_100px_40px] gap-2 items-start"
            >
              {/* Drag handle */}
              <button
                type="button"
                className="mt-2 cursor-grab text-muted-foreground"
                onPointerDown={() => {
                  // For reordering: move(fromIndex, toIndex)
                }}
              >
                <GripVertical className="h-4 w-4" />
              </button>

              <FormField
                control={form.control}
                name={`items.${index}.description`}
                render={({ field }) => (
                  <FormItem>
                    <FormControl>
                      <Input placeholder="Item description" {...field} />
                    </FormControl>
                    <FormMessage />
                  </FormItem>
                )}
              />

              <FormField
                control={form.control}
                name={`items.${index}.quantity`}
                render={({ field }) => (
                  <FormItem>
                    <FormControl>
                      <Input type="number" min={1} {...field} />
                    </FormControl>
                    <FormMessage />
                  </FormItem>
                )}
              />

              <FormField
                control={form.control}
                name={`items.${index}.unitPrice`}
                render={({ field }) => (
                  <FormItem>
                    <FormControl>
                      <Input type="number" step="0.01" {...field} />
                    </FormControl>
                    <FormMessage />
                  </FormItem>
                )}
              />

              <Button
                type="button"
                variant="ghost"
                size="icon"
                onClick={() => remove(index)}
                disabled={fields.length === 1}
              >
                <Trash2 className="h-4 w-4" />
              </Button>
            </div>
          ))}

          {/* Root-level array error */}
          {form.formState.errors.items?.root && (
            <p className="text-sm text-destructive">
              {form.formState.errors.items.root.message}
            </p>
          )}

          <Button
            type="button"
            variant="outline"
            size="sm"
            onClick={() =>
              append({ description: "", quantity: 1, unitPrice: 0 })
            }
          >
            <Plus className="h-4 w-4 mr-1" /> Add Item
          </Button>
        </div>

        {/* Total */}
        <div className="text-right text-lg font-semibold">
          Total: {new Intl.NumberFormat("de-DE", {
            style: "currency",
            currency: "EUR",
          }).format(total)}
        </div>

        <Button type="submit">Create Invoice</Button>
      </form>
    </Form>
  );
}
```

### Nested Field Arrays (e.g., Teams with Members)

```tsx
const schema = z.object({
  teams: z.array(
    z.object({
      teamName: z.string().min(1),
      members: z.array(
        z.object({
          name: z.string().min(1),
          role: z.string().min(1),
        })
      ).min(1),
    })
  ),
});

// In component: nest useFieldArray calls
function TeamFields({ teamIndex }: { teamIndex: number }) {
  const { fields, append, remove } = useFieldArray({
    name: `teams.${teamIndex}.members`,
  });

  return (
    <>
      {fields.map((field, memberIndex) => (
        <FormField
          key={field.id}
          name={`teams.${teamIndex}.members.${memberIndex}.name`}
          render={({ field }) => (
            <FormItem>
              <FormControl>
                <Input {...field} />
              </FormControl>
            </FormItem>
          )}
        />
      ))}
      <Button type="button" onClick={() => append({ name: "", role: "" })}>
        Add Member
      </Button>
    </>
  );
}
```

---

## 5. Server Actions Integration

### With `useActionState` (React 19 / Next.js 15+)

```tsx
// app/actions/contact.ts
"use server";

import { z } from "zod";

const contactSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
  message: z.string().min(10),
});

export type ContactState = {
  success: boolean;
  message: string;
  errors?: Record<string, string[]>;
};

export async function submitContact(
  prevState: ContactState,
  formData: FormData
): Promise<ContactState> {
  const raw = Object.fromEntries(formData);
  const parsed = contactSchema.safeParse(raw);

  if (!parsed.success) {
    return {
      success: false,
      message: "Validation failed",
      errors: parsed.error.flatten().fieldErrors,
    };
  }

  // Do server-side work (DB insert, email, etc.)
  try {
    // await db.insert(contacts).values(parsed.data);
    return { success: true, message: "Message sent successfully!" };
  } catch {
    return { success: false, message: "Server error. Please try again." };
  }
}
```

```tsx
// app/contact/page.tsx
"use client";

import { useActionState } from "react";
import { submitContact, type ContactState } from "@/app/actions/contact";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { Label } from "@/components/ui/label";
import { Alert, AlertDescription } from "@/components/ui/alert";

const initialState: ContactState = { success: false, message: "" };

export default function ContactPage() {
  const [state, formAction, isPending] = useActionState(
    submitContact,
    initialState
  );

  return (
    <form action={formAction} className="space-y-4 max-w-md">
      {state.message && (
        <Alert variant={state.success ? "default" : "destructive"}>
          <AlertDescription>{state.message}</AlertDescription>
        </Alert>
      )}

      <div className="space-y-2">
        <Label htmlFor="name">Name</Label>
        <Input id="name" name="name" required />
        {state.errors?.name && (
          <p className="text-sm text-destructive">{state.errors.name[0]}</p>
        )}
      </div>

      <div className="space-y-2">
        <Label htmlFor="email">Email</Label>
        <Input id="email" name="email" type="email" required />
        {state.errors?.email && (
          <p className="text-sm text-destructive">{state.errors.email[0]}</p>
        )}
      </div>

      <div className="space-y-2">
        <Label htmlFor="message">Message</Label>
        <Textarea id="message" name="message" required />
        {state.errors?.message && (
          <p className="text-sm text-destructive">{state.errors.message[0]}</p>
        )}
      </div>

      {/* Progressive enhancement: works without JS */}
      <Button type="submit" disabled={isPending}>
        {isPending ? "Sending..." : "Send Message"}
      </Button>
    </form>
  );
}
```

### React Hook Form + Server Actions (Hybrid)

```tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { useTransition } from "react";
import { submitContact } from "@/app/actions/contact";
import { toast } from "sonner";

const schema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
  message: z.string().min(10),
});

export function HybridForm() {
  const [isPending, startTransition] = useTransition();

  const form = useForm<z.infer<typeof schema>>({
    resolver: zodResolver(schema),
    defaultValues: { name: "", email: "", message: "" },
  });

  function onSubmit(data: z.infer<typeof schema>) {
    startTransition(async () => {
      const formData = new FormData();
      Object.entries(data).forEach(([k, v]) => formData.append(k, v));

      const result = await submitContact(
        { success: false, message: "" },
        formData
      );

      if (result.success) {
        toast.success(result.message);
        form.reset();
      } else {
        toast.error(result.message);
        // Map server errors back to form
        if (result.errors) {
          Object.entries(result.errors).forEach(([field, messages]) => {
            form.setError(field as any, { message: messages[0] });
          });
        }
      }
    });
  }

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {/* ... RHF fields with client-side validation + server action submit */}
    </form>
  );
}
```

---

## 6. File Upload in Forms

```tsx
"use client";

import { useCallback, useState } from "react";
import { useDropzone, type FileRejection } from "react-dropzone";
import { useForm, useController } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { Button } from "@/components/ui/button";
import { Progress } from "@/components/ui/progress";
import { X, Upload, FileIcon } from "lucide-react";
import { cn } from "@/lib/utils";

// File validation schema
const MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB
const ACCEPTED_TYPES = ["image/jpeg", "image/png", "image/webp", "application/pdf"];

const uploadSchema = z.object({
  title: z.string().min(1),
  files: z
    .array(
      z.object({
        file: z.instanceof(File),
        preview: z.string().optional(),
      })
    )
    .min(1, "Upload at least one file")
    .max(5, "Maximum 5 files"),
});

type UploadFormValues = z.infer<typeof uploadSchema>;

export function FileUploadForm() {
  const [uploadProgress, setUploadProgress] = useState<Record<string, number>>({});

  const form = useForm<UploadFormValues>({
    resolver: zodResolver(uploadSchema),
    defaultValues: { title: "", files: [] },
  });

  const { field: filesField } = useController({
    control: form.control,
    name: "files",
  });

  const onDrop = useCallback(
    (accepted: File[], rejected: FileRejection[]) => {
      const newFiles = accepted.map((file) => ({
        file,
        preview: file.type.startsWith("image/")
          ? URL.createObjectURL(file)
          : undefined,
      }));
      filesField.onChange([...filesField.value, ...newFiles]);

      rejected.forEach((r) => {
        const errors = r.errors.map((e) => e.message).join(", ");
        console.warn(`Rejected ${r.file.name}: ${errors}`);
      });
    },
    [filesField]
  );

  const { getRootProps, getInputProps, isDragActive } = useDropzone({
    onDrop,
    accept: {
      "image/jpeg": [".jpg", ".jpeg"],
      "image/png": [".png"],
      "image/webp": [".webp"],
      "application/pdf": [".pdf"],
    },
    maxSize: MAX_FILE_SIZE,
    maxFiles: 5 - filesField.value.length,
  });

  function removeFile(index: number) {
    const updated = [...filesField.value];
    const removed = updated.splice(index, 1)[0];
    if (removed.preview) URL.revokeObjectURL(removed.preview);
    filesField.onChange(updated);
  }

  async function onSubmit(data: UploadFormValues) {
    // Upload files with progress tracking
    for (const { file } of data.files) {
      const formData = new FormData();
      formData.append("file", file);

      await new Promise<void>((resolve) => {
        const xhr = new XMLHttpRequest();
        xhr.open("POST", "/api/upload");

        xhr.upload.onprogress = (e) => {
          if (e.lengthComputable) {
            setUploadProgress((prev) => ({
              ...prev,
              [file.name]: Math.round((e.loaded / e.total) * 100),
            }));
          }
        };

        xhr.onload = () => resolve();
        xhr.send(formData);
      });
    }
  }

  return (
    <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
      {/* Dropzone */}
      <div
        {...getRootProps()}
        className={cn(
          "border-2 border-dashed rounded-lg p-8 text-center cursor-pointer transition-colors",
          isDragActive
            ? "border-primary bg-primary/5"
            : "border-muted-foreground/25 hover:border-primary/50"
        )}
      >
        <input {...getInputProps()} />
        <Upload className="h-8 w-8 mx-auto mb-2 text-muted-foreground" />
        {isDragActive ? (
          <p>Drop files here...</p>
        ) : (
          <div>
            <p className="font-medium">Click or drag files here</p>
            <p className="text-sm text-muted-foreground mt-1">
              JPG, PNG, WebP, PDF up to 5MB
            </p>
          </div>
        )}
      </div>

      {/* Error */}
      {form.formState.errors.files && (
        <p className="text-sm text-destructive">
          {form.formState.errors.files.message}
        </p>
      )}

      {/* File previews */}
      {filesField.value.length > 0 && (
        <div className="grid grid-cols-2 gap-3">
          {filesField.value.map((item, index) => (
            <div
              key={index}
              className="relative border rounded-lg p-2 flex items-center gap-2"
            >
              {item.preview ? (
                <img
                  src={item.preview}
                  alt=""
                  className="h-12 w-12 object-cover rounded"
                />
              ) : (
                <FileIcon className="h-12 w-12 text-muted-foreground" />
              )}
              <div className="flex-1 min-w-0">
                <p className="text-sm truncate">{item.file.name}</p>
                <p className="text-xs text-muted-foreground">
                  {(item.file.size / 1024).toFixed(0)} KB
                </p>
                {uploadProgress[item.file.name] !== undefined && (
                  <Progress
                    value={uploadProgress[item.file.name]}
                    className="h-1 mt-1"
                  />
                )}
              </div>
              <Button
                type="button"
                variant="ghost"
                size="icon"
                className="h-6 w-6"
                onClick={() => removeFile(index)}
              >
                <X className="h-3 w-3" />
              </Button>
            </div>
          ))}
        </div>
      )}

      <Button type="submit" disabled={form.formState.isSubmitting}>
        Upload
      </Button>
    </form>
  );
}
```

---

## 7. Autosave

```tsx
"use client";

import { useEffect, useRef, useCallback, useState } from "react";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { Check, Loader2, AlertCircle } from "lucide-react";
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form";

const noteSchema = z.object({
  title: z.string().min(1),
  content: z.string(),
  tags: z.string(),
});

type NoteFormValues = z.infer<typeof noteSchema>;

type SaveStatus = "idle" | "saving" | "saved" | "error" | "conflict";

// Debounce hook
function useDebouncedCallback<T extends (...args: any[]) => any>(
  callback: T,
  delay: number
) {
  const timeoutRef = useRef<ReturnType<typeof setTimeout>>();

  const cancel = useCallback(() => {
    if (timeoutRef.current) clearTimeout(timeoutRef.current);
  }, []);

  const debouncedFn = useCallback(
    (...args: Parameters<T>) => {
      cancel();
      timeoutRef.current = setTimeout(() => callback(...args), delay);
    },
    [callback, delay, cancel]
  );

  useEffect(() => cancel, [cancel]);

  return { debouncedFn, cancel };
}

export function AutosaveForm({
  noteId,
  initialData,
}: {
  noteId: string;
  initialData: NoteFormValues & { updatedAt: string };
}) {
  const [saveStatus, setSaveStatus] = useState<SaveStatus>("idle");
  const lastSavedRef = useRef(initialData.updatedAt);

  const form = useForm<NoteFormValues>({
    resolver: zodResolver(noteSchema),
    defaultValues: initialData,
  });

  // Save function with conflict detection
  const save = useCallback(
    async (data: NoteFormValues) => {
      setSaveStatus("saving");

      try {
        const res = await fetch(`/api/notes/${noteId}`, {
          method: "PATCH",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({
            ...data,
            expectedVersion: lastSavedRef.current,
          }),
        });

        if (res.status === 409) {
          setSaveStatus("conflict");
          return;
        }

        if (!res.ok) throw new Error("Save failed");

        const result = await res.json();
        lastSavedRef.current = result.updatedAt;
        setSaveStatus("saved");

        // Reset back to "idle" after 2s
        setTimeout(() => setSaveStatus("idle"), 2000);
      } catch {
        setSaveStatus("error");
      }
    },
    [noteId]
  );

  // Debounced save — triggers 1s after last keystroke
  const { debouncedFn: debouncedSave } = useDebouncedCallback(save, 1000);

  // Watch all fields and trigger autosave on changes
  useEffect(() => {
    const subscription = form.watch((data) => {
      if (form.formState.isDirty) {
        debouncedSave(data as NoteFormValues);
      }
    });
    return () => subscription.unsubscribe();
  }, [form, debouncedSave]);

  return (
    <div className="space-y-4">
      {/* Save status indicator */}
      <div className="flex items-center gap-2 text-sm text-muted-foreground">
        {saveStatus === "saving" && (
          <>
            <Loader2 className="h-3 w-3 animate-spin" />
            <span>Saving...</span>
          </>
        )}
        {saveStatus === "saved" && (
          <>
            <Check className="h-3 w-3 text-green-500" />
            <span>Saved</span>
          </>
        )}
        {saveStatus === "error" && (
          <>
            <AlertCircle className="h-3 w-3 text-destructive" />
            <span className="text-destructive">
              Save failed.{" "}
              <button
                className="underline"
                onClick={() => save(form.getValues())}
              >
                Retry
              </button>
            </span>
          </>
        )}
        {saveStatus === "conflict" && (
          <div className="bg-yellow-50 border border-yellow-200 p-3 rounded text-yellow-800">
            This note was edited elsewhere.{" "}
            <button className="underline font-medium" onClick={() => window.location.reload()}>
              Reload latest version
            </button>
          </div>
        )}
      </div>

      <Form {...form}>
        <form className="space-y-4">
          <FormField
            control={form.control}
            name="title"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Title</FormLabel>
                <FormControl>
                  <Input {...field} />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />
          <FormField
            control={form.control}
            name="content"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Content</FormLabel>
                <FormControl>
                  <Textarea rows={10} {...field} />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />
        </form>
      </Form>
    </div>
  );
}
```

---

## 8. Address Autocomplete (Google Places)

```tsx
"use client";

import { useEffect, useRef, useState } from "react";
import { useFormContext } from "react-hook-form";
import { Input } from "@/components/ui/input";
import {
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form";
import { Switch } from "@/components/ui/switch";

// Load Google Maps script
function useGoogleMapsScript(apiKey: string) {
  const [loaded, setLoaded] = useState(false);

  useEffect(() => {
    if (window.google?.maps?.places) {
      setLoaded(true);
      return;
    }

    const script = document.createElement("script");
    script.src = `https://maps.googleapis.com/maps/api/js?key=${apiKey}&libraries=places`;
    script.async = true;
    script.onload = () => setLoaded(true);
    document.head.appendChild(script);

    return () => {
      document.head.removeChild(script);
    };
  }, [apiKey]);

  return loaded;
}

interface AddressFields {
  street: string;
  city: string;
  state: string;
  zip: string;
  country: string;
}

interface AddressAutocompleteProps {
  prefix: string; // e.g., "shippingAddress" or "billingAddress"
  apiKey: string;
}

export function AddressAutocomplete({
  prefix,
  apiKey,
}: AddressAutocompleteProps) {
  const form = useFormContext();
  const inputRef = useRef<HTMLInputElement>(null);
  const autocompleteRef = useRef<google.maps.places.Autocomplete | null>(null);
  const [manualEntry, setManualEntry] = useState(false);
  const isLoaded = useGoogleMapsScript(apiKey);

  useEffect(() => {
    if (!isLoaded || !inputRef.current || manualEntry) return;

    autocompleteRef.current = new google.maps.places.Autocomplete(
      inputRef.current,
      {
        types: ["address"],
        componentRestrictions: { country: ["de", "at", "ch"] }, // DACH region
        fields: ["address_components", "formatted_address"],
      }
    );

    autocompleteRef.current.addListener("place_changed", () => {
      const place = autocompleteRef.current?.getPlace();
      if (!place?.address_components) return;

      const components = place.address_components;
      const get = (type: string) =>
        components.find((c) => c.types.includes(type))?.long_name || "";

      const streetNumber = get("street_number");
      const route = get("route");

      form.setValue(`${prefix}.street`, `${route} ${streetNumber}`.trim(), {
        shouldValidate: true,
      });
      form.setValue(`${prefix}.city`, get("locality"), {
        shouldValidate: true,
      });
      form.setValue(
        `${prefix}.state`,
        get("administrative_area_level_1"),
        { shouldValidate: true }
      );
      form.setValue(`${prefix}.zip`, get("postal_code"), {
        shouldValidate: true,
      });
      form.setValue(`${prefix}.country`, get("country"), {
        shouldValidate: true,
      });
    });

    return () => {
      google.maps.event.clearInstanceListeners(autocompleteRef.current!);
    };
  }, [isLoaded, manualEntry, prefix, form]);

  return (
    <div className="space-y-4">
      {/* Toggle for manual entry fallback */}
      <div className="flex items-center gap-2 text-sm">
        <Switch checked={manualEntry} onCheckedChange={setManualEntry} />
        <span>Enter address manually</span>
      </div>

      {!manualEntry ? (
        /* Autocomplete input */
        <FormField
          control={form.control}
          name={`${prefix}.street`}
          render={({ field }) => (
            <FormItem>
              <FormLabel>Address</FormLabel>
              <FormControl>
                <Input
                  {...field}
                  ref={(el) => {
                    field.ref(el);
                    (inputRef as any).current = el;
                  }}
                  placeholder="Start typing an address..."
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
      ) : (
        /* Manual entry fields */
        <>
          <FormField
            control={form.control}
            name={`${prefix}.street`}
            render={({ field }) => (
              <FormItem>
                <FormLabel>Street</FormLabel>
                <FormControl>
                  <Input {...field} />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />
        </>
      )}

      {/* Always visible detail fields (populated by autocomplete or manual) */}
      <div className="grid grid-cols-2 gap-4">
        <FormField
          control={form.control}
          name={`${prefix}.zip`}
          render={({ field }) => (
            <FormItem>
              <FormLabel>ZIP</FormLabel>
              <FormControl>
                <Input {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <FormField
          control={form.control}
          name={`${prefix}.city`}
          render={({ field }) => (
            <FormItem>
              <FormLabel>City</FormLabel>
              <FormControl>
                <Input {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
      </div>

      <FormField
        control={form.control}
        name={`${prefix}.country`}
        render={({ field }) => (
          <FormItem>
            <FormLabel>Country</FormLabel>
            <FormControl>
              <Input {...field} />
            </FormControl>
            <FormMessage />
          </FormItem>
        )}
      />
    </div>
  );
}
```

---

## 9. Accessible Form Errors

```tsx
"use client";

import { useEffect, useRef } from "react";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";

const schema = z.object({
  email: z.string().email("Please enter a valid email address"),
  password: z.string().min(8, "Password must be at least 8 characters"),
});

type FormValues = z.infer<typeof schema>;

// Live region for screen reader announcements
function ErrorAnnouncer({ errors }: { errors: Record<string, any> }) {
  const errorMessages = Object.values(errors)
    .map((e: any) => e?.message)
    .filter(Boolean)
    .join(". ");

  return (
    <div
      role="alert"
      aria-live="assertive"
      aria-atomic="true"
      className="sr-only"
    >
      {errorMessages && `Form errors: ${errorMessages}`}
    </div>
  );
}

export function AccessibleForm() {
  const errorSummaryRef = useRef<HTMLDivElement>(null);

  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitted },
  } = useForm<FormValues>({
    resolver: zodResolver(schema),
  });

  // Focus the error summary on failed submission
  useEffect(() => {
    if (isSubmitted && Object.keys(errors).length > 0) {
      errorSummaryRef.current?.focus();
    }
  }, [isSubmitted, errors]);

  function onSubmit(data: FormValues) {
    console.log(data);
  }

  const errorEntries = Object.entries(errors);

  return (
    <form
      onSubmit={handleSubmit(onSubmit)}
      className="space-y-4 max-w-md"
      noValidate
      aria-label="Login form"
    >
      {/* Error summary — linked to individual fields */}
      {errorEntries.length > 0 && (
        <div
          ref={errorSummaryRef}
          tabIndex={-1}
          role="alert"
          className="rounded-md border border-destructive bg-destructive/5 p-4 focus:outline-none focus:ring-2 focus:ring-destructive"
        >
          <h2 className="text-sm font-semibold text-destructive mb-2">
            Please fix the following errors:
          </h2>
          <ul className="list-disc pl-4 text-sm text-destructive space-y-1">
            {errorEntries.map(([field, error]) => (
              <li key={field}>
                <a
                  href={`#field-${field}`}
                  className="underline hover:no-underline"
                  onClick={(e) => {
                    e.preventDefault();
                    document.getElementById(`field-${field}`)?.focus();
                  }}
                >
                  {(error as any)?.message}
                </a>
              </li>
            ))}
          </ul>
        </div>
      )}

      {/* Screen reader live announcements */}
      <ErrorAnnouncer errors={errors} />

      {/* Email field with full ARIA attributes */}
      <div className="space-y-2">
        <Label htmlFor="field-email">
          Email <span aria-hidden="true">*</span>
          <span className="sr-only">(required)</span>
        </Label>
        <Input
          id="field-email"
          type="email"
          aria-required="true"
          aria-invalid={!!errors.email}
          aria-describedby={
            errors.email ? "email-error" : "email-description"
          }
          {...register("email")}
        />
        {errors.email ? (
          <p id="email-error" role="alert" className="text-sm text-destructive">
            {errors.email.message}
          </p>
        ) : (
          <p id="email-description" className="text-sm text-muted-foreground">
            We'll never share your email.
          </p>
        )}
      </div>

      {/* Password field */}
      <div className="space-y-2">
        <Label htmlFor="field-password">
          Password <span aria-hidden="true">*</span>
          <span className="sr-only">(required)</span>
        </Label>
        <Input
          id="field-password"
          type="password"
          aria-required="true"
          aria-invalid={!!errors.password}
          aria-describedby={
            errors.password ? "password-error" : "password-description"
          }
          {...register("password")}
        />
        {errors.password ? (
          <p
            id="password-error"
            role="alert"
            className="text-sm text-destructive"
          >
            {errors.password.message}
          </p>
        ) : (
          <p
            id="password-description"
            className="text-sm text-muted-foreground"
          >
            At least 8 characters.
          </p>
        )}
      </div>

      <Button type="submit">Sign In</Button>
    </form>
  );
}
```

### Key ARIA Patterns Summary

| Pattern | Attribute | Purpose |
|---------|-----------|---------|
| Mark invalid fields | `aria-invalid="true"` | Screen readers announce "invalid entry" |
| Link error to field | `aria-describedby="field-error-id"` | Error read when field is focused |
| Required indicator | `aria-required="true"` | Announces required status |
| Error summary | `role="alert"` + `tabIndex={-1}` | Focus on submit failure, read aloud |
| Live announcements | `aria-live="assertive"` | Announce errors without focus change |

---

## 10. Complex Patterns

### Dependent Dropdowns

```tsx
"use client";

import { useEffect, useState } from "react";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form";

const schema = z.object({
  country: z.string().min(1, "Select a country"),
  state: z.string().min(1, "Select a state"),
  city: z.string().min(1, "Select a city"),
});

// Simulated data — normally fetched from API
const regionData: Record<string, Record<string, string[]>> = {
  DE: {
    Hessen: ["Frankfurt", "Wiesbaden", "Darmstadt"],
    Bayern: ["Munich", "Nuremberg", "Augsburg"],
    Berlin: ["Berlin"],
  },
  AT: {
    Wien: ["Wien"],
    Tirol: ["Innsbruck", "Kufstein"],
  },
};

export function DependentDropdowns() {
  const form = useForm<z.infer<typeof schema>>({
    resolver: zodResolver(schema),
    defaultValues: { country: "", state: "", city: "" },
  });

  const selectedCountry = form.watch("country");
  const selectedState = form.watch("state");

  const states = selectedCountry ? Object.keys(regionData[selectedCountry] || {}) : [];
  const cities =
    selectedCountry && selectedState
      ? regionData[selectedCountry]?.[selectedState] || []
      : [];

  // Reset dependent fields when parent changes
  useEffect(() => {
    form.setValue("state", "");
    form.setValue("city", "");
  }, [selectedCountry]);

  useEffect(() => {
    form.setValue("city", "");
  }, [selectedState]);

  return (
    <Form {...form}>
      <form className="space-y-4">
        <FormField
          control={form.control}
          name="country"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Country</FormLabel>
              <Select onValueChange={field.onChange} value={field.value}>
                <FormControl>
                  <SelectTrigger>
                    <SelectValue placeholder="Select country" />
                  </SelectTrigger>
                </FormControl>
                <SelectContent>
                  <SelectItem value="DE">Germany</SelectItem>
                  <SelectItem value="AT">Austria</SelectItem>
                </SelectContent>
              </Select>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="state"
          render={({ field }) => (
            <FormItem>
              <FormLabel>State</FormLabel>
              <Select
                onValueChange={field.onChange}
                value={field.value}
                disabled={!selectedCountry}
              >
                <FormControl>
                  <SelectTrigger>
                    <SelectValue placeholder="Select state" />
                  </SelectTrigger>
                </FormControl>
                <SelectContent>
                  {states.map((s) => (
                    <SelectItem key={s} value={s}>
                      {s}
                    </SelectItem>
                  ))}
                </SelectContent>
              </Select>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="city"
          render={({ field }) => (
            <FormItem>
              <FormLabel>City</FormLabel>
              <Select
                onValueChange={field.onChange}
                value={field.value}
                disabled={!selectedState}
              >
                <FormControl>
                  <SelectTrigger>
                    <SelectValue placeholder="Select city" />
                  </SelectTrigger>
                </FormControl>
                <SelectContent>
                  {cities.map((c) => (
                    <SelectItem key={c} value={c}>
                      {c}
                    </SelectItem>
                  ))}
                </SelectContent>
              </Select>
              <FormMessage />
            </FormItem>
          )}
        />
      </form>
    </Form>
  );
}
```

### Inline Editing

```tsx
"use client";

import { useState, useRef, useEffect } from "react";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { Input } from "@/components/ui/input";
import { Check, X, Pencil } from "lucide-react";
import { Button } from "@/components/ui/button";

const cellSchema = z.object({
  value: z.string().min(1, "Cannot be empty"),
});

interface InlineEditCellProps {
  initialValue: string;
  onSave: (value: string) => Promise<void>;
  label: string;
}

export function InlineEditCell({
  initialValue,
  onSave,
  label,
}: InlineEditCellProps) {
  const [isEditing, setIsEditing] = useState(false);
  const inputRef = useRef<HTMLInputElement>(null);

  const form = useForm<z.infer<typeof cellSchema>>({
    resolver: zodResolver(cellSchema),
    defaultValues: { value: initialValue },
  });

  useEffect(() => {
    if (isEditing) inputRef.current?.focus();
  }, [isEditing]);

  async function handleSave() {
    const valid = await form.trigger();
    if (!valid) return;

    const newValue = form.getValues("value");
    await onSave(newValue);
    setIsEditing(false);
  }

  function handleCancel() {
    form.reset({ value: initialValue });
    setIsEditing(false);
  }

  // Handle keyboard shortcuts
  function handleKeyDown(e: React.KeyboardEvent) {
    if (e.key === "Enter") handleSave();
    if (e.key === "Escape") handleCancel();
  }

  if (!isEditing) {
    return (
      <div
        className="group flex items-center gap-2 px-2 py-1 rounded hover:bg-muted cursor-pointer"
        onClick={() => setIsEditing(true)}
        role="button"
        aria-label={`Edit ${label}`}
        tabIndex={0}
        onKeyDown={(e) => e.key === "Enter" && setIsEditing(true)}
      >
        <span>{initialValue}</span>
        <Pencil className="h-3 w-3 opacity-0 group-hover:opacity-100 text-muted-foreground" />
      </div>
    );
  }

  return (
    <div className="flex items-center gap-1">
      <Input
        {...form.register("value")}
        ref={(el) => {
          form.register("value").ref(el);
          (inputRef as any).current = el;
        }}
        onKeyDown={handleKeyDown}
        className="h-8"
        aria-label={label}
      />
      <Button type="button" variant="ghost" size="icon" className="h-8 w-8" onClick={handleSave}>
        <Check className="h-3 w-3" />
      </Button>
      <Button type="button" variant="ghost" size="icon" className="h-8 w-8" onClick={handleCancel}>
        <X className="h-3 w-3" />
      </Button>
    </div>
  );
}
```

### Search-as-you-type Filter

```tsx
"use client";

import { useState, useDeferredValue, useMemo } from "react";
import { Input } from "@/components/ui/input";
import { Search } from "lucide-react";

interface Item {
  id: string;
  name: string;
  category: string;
}

export function SearchFilter({ items }: { items: Item[] }) {
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  const filtered = useMemo(() => {
    if (!deferredQuery) return items;
    const q = deferredQuery.toLowerCase();
    return items.filter(
      (item) =>
        item.name.toLowerCase().includes(q) ||
        item.category.toLowerCase().includes(q)
    );
  }, [items, deferredQuery]);

  return (
    <div className="space-y-4">
      <div className="relative">
        <Search className="absolute left-3 top-1/2 -translate-y-1/2 h-4 w-4 text-muted-foreground" />
        <Input
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Search items..."
          className="pl-9"
          role="searchbox"
          aria-label="Search items"
        />
      </div>

      <div className={isStale ? "opacity-60 transition-opacity" : ""}>
        <p className="text-sm text-muted-foreground mb-2">
          {filtered.length} results
        </p>
        <ul className="space-y-2">
          {filtered.map((item) => (
            <li
              key={item.id}
              className="p-3 border rounded-lg flex justify-between"
            >
              <span>{item.name}</span>
              <span className="text-sm text-muted-foreground">
                {item.category}
              </span>
            </li>
          ))}
        </ul>
      </div>
    </div>
  );
}
```

### Currency & Phone Input Masks

```tsx
"use client";

import { forwardRef, type ChangeEvent } from "react";
import { Input } from "@/components/ui/input";
import { useController, type Control } from "react-hook-form";

// Currency input — stores cents, displays formatted EUR
interface CurrencyInputProps {
  name: string;
  control: Control<any>;
  label?: string;
}

export function CurrencyInput({ name, control, label }: CurrencyInputProps) {
  const {
    field,
    fieldState: { error },
  } = useController({ name, control });

  // Format cents to EUR display
  function formatCurrency(cents: number): string {
    return new Intl.NumberFormat("de-DE", {
      style: "currency",
      currency: "EUR",
    }).format(cents / 100);
  }

  // Parse user input to cents
  function handleChange(e: ChangeEvent<HTMLInputElement>) {
    // Remove everything except digits and comma/period
    const raw = e.target.value.replace(/[^\d.,]/g, "");
    // Normalize to use period as decimal separator
    const normalized = raw.replace(",", ".");
    const asFloat = parseFloat(normalized);
    if (!isNaN(asFloat)) {
      field.onChange(Math.round(asFloat * 100));
    }
  }

  return (
    <div className="space-y-2">
      {label && (
        <label className="text-sm font-medium" htmlFor={name}>
          {label}
        </label>
      )}
      <Input
        id={name}
        value={field.value ? formatCurrency(field.value) : ""}
        onChange={handleChange}
        onBlur={field.onBlur}
        placeholder="0,00 EUR"
        inputMode="decimal"
      />
      {error && <p className="text-sm text-destructive">{error.message}</p>}
    </div>
  );
}

// Phone input — German format with auto-formatting
export function PhoneInput({ name, control }: { name: string; control: Control<any> }) {
  const { field, fieldState: { error } } = useController({ name, control });

  function formatPhone(value: string): string {
    // Strip everything except digits and leading +
    const digits = value.replace(/[^\d+]/g, "");

    // Format as German phone: +49 XXX XXXXXXX
    if (digits.startsWith("+49")) {
      const rest = digits.slice(3);
      if (rest.length <= 3) return `+49 ${rest}`;
      if (rest.length <= 7) return `+49 ${rest.slice(0, 3)} ${rest.slice(3)}`;
      return `+49 ${rest.slice(0, 3)} ${rest.slice(3, 10)}`;
    }

    // Format as local: 0XXX XXXXXXX
    if (digits.startsWith("0")) {
      if (digits.length <= 4) return digits;
      if (digits.length <= 8) return `${digits.slice(0, 4)} ${digits.slice(4)}`;
      return `${digits.slice(0, 4)} ${digits.slice(4, 11)}`;
    }

    return digits;
  }

  function handleChange(e: ChangeEvent<HTMLInputElement>) {
    const formatted = formatPhone(e.target.value);
    field.onChange(formatted);
  }

  return (
    <div className="space-y-2">
      <Input
        value={field.value || ""}
        onChange={handleChange}
        onBlur={field.onBlur}
        placeholder="+49 XXX XXXXXXX"
        type="tel"
        inputMode="tel"
      />
      {error && <p className="text-sm text-destructive">{error.message}</p>}
    </div>
  );
}
```

### Zod Schemas for Currency & Phone

```tsx
const formSchema = z.object({
  amount: z
    .number()
    .min(100, "Minimum 1,00 EUR")
    .max(10000000, "Maximum 100.000 EUR"),

  phone: z
    .string()
    .regex(
      /^(\+49\s?\d{3}\s?\d{4,7}|0\d{3,4}\s?\d{4,8})$/,
      "Invalid German phone number"
    ),
});
```

---

## Quick Reference: Common Zod Patterns for Forms

```tsx
import { z } from "zod";

// Optional string that can be empty
z.string().optional().or(z.literal(""))

// Coerce string from form input to number
z.coerce.number().min(0)

// Date from date picker
z.coerce.date().min(new Date(), "Must be in the future")

// Enum from select
z.enum(["option1", "option2", "option3"])

// File validation (client-side)
z.instanceof(File)
  .refine((f) => f.size < 5_000_000, "Max 5MB")
  .refine((f) => ["image/png", "image/jpeg"].includes(f.type), "PNG or JPG only")

// Transform on parse
z.string().transform((val) => val.trim().toLowerCase())

// Conditional/dependent validation
z.object({
  hasCompany: z.boolean(),
  companyName: z.string().optional(),
}).refine(
  (data) => !data.hasCompany || (data.companyName && data.companyName.length > 0),
  { message: "Company name required", path: ["companyName"] }
)

// German-specific patterns
const germanZip = z.string().regex(/^\d{5}$/, "PLZ muss 5 Ziffern haben");
const germanIBAN = z.string().regex(/^DE\d{20}$/, "Invalid German IBAN");
const ustId = z.string().regex(/^DE\d{9}$/, "Invalid USt-IdNr.");
```
