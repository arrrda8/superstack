# Onboarding Patterns

Reference for user onboarding: product tours, checklists, activation tracking, welcome wizards, sample data, empty states, progressive disclosure, email drips, help systems, and success measurement.

---

## 1. Product Tours

Step-by-step tooltips highlighting UI elements. Can use react-joyride or a custom implementation.

```tsx
// components/onboarding/product-tour.tsx
"use client";

import { useState, useEffect, useCallback } from "react";
import { createPortal } from "react-dom";
import { motion, AnimatePresence } from "framer-motion";

interface TourStep {
  target: string;       // CSS selector
  title: string;
  content: string;
  placement?: "top" | "bottom" | "left" | "right";
  action?: "click" | "input" | "none";
  onEnter?: () => void; // Run when step activates
}

interface ProductTourProps {
  steps: TourStep[];
  tourId: string;
  onComplete: () => void;
  onSkip: () => void;
}

export function ProductTour({ steps, tourId, onComplete, onSkip }: ProductTourProps) {
  const [currentStep, setCurrentStep] = useState(0);
  const [targetRect, setTargetRect] = useState<DOMRect | null>(null);

  const step = steps[currentStep];

  const updatePosition = useCallback(() => {
    if (!step) return;
    const element = document.querySelector(step.target);
    if (element) {
      element.scrollIntoView({ behavior: "smooth", block: "center" });
      setTargetRect(element.getBoundingClientRect());
    }
  }, [step]);

  useEffect(() => {
    updatePosition();
    window.addEventListener("resize", updatePosition);
    step?.onEnter?.();
    return () => window.removeEventListener("resize", updatePosition);
  }, [currentStep, updatePosition, step]);

  const next = () => {
    if (currentStep < steps.length - 1) {
      setCurrentStep((prev) => prev + 1);
    } else {
      onComplete();
    }
  };

  const prev = () => {
    if (currentStep > 0) setCurrentStep((prev) => prev - 1);
  };

  if (!targetRect) return null;

  return createPortal(
    <>
      {/* Overlay with spotlight cutout */}
      <div className="fixed inset-0 z-[9998]">
        <svg className="absolute inset-0 h-full w-full">
          <defs>
            <mask id="spotlight-mask">
              <rect width="100%" height="100%" fill="white" />
              <rect
                x={targetRect.left - 8}
                y={targetRect.top - 8}
                width={targetRect.width + 16}
                height={targetRect.height + 16}
                rx="8"
                fill="black"
              />
            </mask>
          </defs>
          <rect
            width="100%"
            height="100%"
            fill="rgba(0,0,0,0.5)"
            mask="url(#spotlight-mask)"
          />
        </svg>
      </div>

      {/* Tooltip */}
      <AnimatePresence mode="wait">
        <motion.div
          key={currentStep}
          initial={{ opacity: 0, y: 8 }}
          animate={{ opacity: 1, y: 0 }}
          exit={{ opacity: 0, y: -8 }}
          className="fixed z-[9999] w-80 rounded-xl bg-white p-5 shadow-2xl dark:bg-zinc-900"
          style={{
            top: targetRect.bottom + 16,
            left: targetRect.left + targetRect.width / 2 - 160,
          }}
        >
          <div className="mb-1 text-xs font-medium text-zinc-400">
            Step {currentStep + 1} of {steps.length}
          </div>
          <h3 className="mb-2 text-base font-semibold text-zinc-900 dark:text-white">
            {step.title}
          </h3>
          <p className="mb-4 text-sm text-zinc-600 dark:text-zinc-300">
            {step.content}
          </p>

          <div className="flex items-center justify-between">
            <button
              onClick={onSkip}
              className="text-xs text-zinc-400 hover:text-zinc-600"
            >
              Skip tour
            </button>
            <div className="flex gap-2">
              {currentStep > 0 && (
                <button
                  onClick={prev}
                  className="rounded-lg px-3 py-1.5 text-sm text-zinc-600 hover:bg-zinc-100"
                >
                  Back
                </button>
              )}
              <button
                onClick={next}
                className="rounded-lg bg-zinc-900 px-3 py-1.5 text-sm text-white hover:bg-zinc-800 dark:bg-white dark:text-zinc-900"
              >
                {currentStep === steps.length - 1 ? "Done" : "Next"}
              </button>
            </div>
          </div>

          {/* Progress dots */}
          <div className="mt-3 flex justify-center gap-1">
            {steps.map((_, i) => (
              <div
                key={i}
                className={`h-1.5 w-1.5 rounded-full transition-colors ${
                  i === currentStep ? "bg-zinc-900 dark:bg-white" : "bg-zinc-200 dark:bg-zinc-700"
                }`}
              />
            ))}
          </div>
        </motion.div>
      </AnimatePresence>
    </>,
    document.body
  );
}

// --- Usage ---
const DASHBOARD_TOUR_STEPS: TourStep[] = [
  {
    target: '[data-tour="sidebar-nav"]',
    title: "Navigation",
    content: "Use the sidebar to navigate between sections of your dashboard.",
  },
  {
    target: '[data-tour="create-project"]',
    title: "Create your first project",
    content: "Click here to create a new project and start tracking your work.",
  },
  {
    target: '[data-tour="settings"]',
    title: "Settings",
    content: "Customize your workspace, invite team members, and manage billing.",
  },
];

// In your page component:
// <ProductTour
//   steps={DASHBOARD_TOUR_STEPS}
//   tourId="dashboard-v1"
//   onComplete={() => markTourComplete("dashboard-v1")}
//   onSkip={() => markTourSkipped("dashboard-v1")}
// />
```

---

## 2. Onboarding Checklist

Progress checklist component with persistent state and reward on completion.

```tsx
// components/onboarding/checklist.tsx
"use client";

import { useState, useEffect } from "react";
import { motion, AnimatePresence } from "framer-motion";
import { CheckCircle2, Circle, ChevronDown, Gift } from "lucide-react";

interface ChecklistItem {
  id: string;
  title: string;
  description: string;
  href?: string;
  action?: () => void;
  isComplete: boolean;
}

interface OnboardingChecklistProps {
  items: ChecklistItem[];
  onItemComplete: (itemId: string) => void;
  onAllComplete: () => void;
}

export function OnboardingChecklist({
  items,
  onItemComplete,
  onAllComplete,
}: OnboardingChecklistProps) {
  const [isExpanded, setIsExpanded] = useState(true);
  const [showReward, setShowReward] = useState(false);

  const completedCount = items.filter((i) => i.isComplete).length;
  const progress = (completedCount / items.length) * 100;
  const allComplete = completedCount === items.length;

  useEffect(() => {
    if (allComplete) {
      setShowReward(true);
      onAllComplete();
    }
  }, [allComplete, onAllComplete]);

  return (
    <div className="rounded-xl border border-zinc-200 bg-white shadow-sm dark:border-zinc-800 dark:bg-zinc-900">
      {/* Header */}
      <button
        onClick={() => setIsExpanded(!isExpanded)}
        className="flex w-full items-center justify-between p-4"
      >
        <div className="flex items-center gap-3">
          <div className="text-sm font-semibold text-zinc-900 dark:text-white">
            Getting started
          </div>
          <span className="text-xs text-zinc-400">
            {completedCount}/{items.length} complete
          </span>
        </div>
        <ChevronDown
          className={`h-4 w-4 text-zinc-400 transition-transform ${
            isExpanded ? "rotate-180" : ""
          }`}
        />
      </button>

      {/* Progress bar */}
      <div className="mx-4 h-1.5 overflow-hidden rounded-full bg-zinc-100 dark:bg-zinc-800">
        <motion.div
          className="h-full rounded-full bg-emerald-500"
          initial={{ width: 0 }}
          animate={{ width: `${progress}%` }}
          transition={{ duration: 0.5, ease: "easeOut" }}
        />
      </div>

      {/* Items */}
      <AnimatePresence>
        {isExpanded && (
          <motion.div
            initial={{ height: 0 }}
            animate={{ height: "auto" }}
            exit={{ height: 0 }}
            className="overflow-hidden"
          >
            <div className="space-y-1 p-2 pt-3">
              {items.map((item) => (
                <button
                  key={item.id}
                  onClick={() => {
                    if (!item.isComplete) {
                      item.action?.();
                      onItemComplete(item.id);
                    }
                  }}
                  className={`flex w-full items-start gap-3 rounded-lg p-3 text-left transition-colors hover:bg-zinc-50 dark:hover:bg-zinc-800 ${
                    item.isComplete ? "opacity-60" : ""
                  }`}
                >
                  {item.isComplete ? (
                    <CheckCircle2 className="mt-0.5 h-5 w-5 shrink-0 text-emerald-500" />
                  ) : (
                    <Circle className="mt-0.5 h-5 w-5 shrink-0 text-zinc-300" />
                  )}
                  <div>
                    <div
                      className={`text-sm font-medium ${
                        item.isComplete
                          ? "text-zinc-400 line-through"
                          : "text-zinc-900 dark:text-white"
                      }`}
                    >
                      {item.title}
                    </div>
                    <div className="text-xs text-zinc-400">{item.description}</div>
                  </div>
                </button>
              ))}
            </div>
          </motion.div>
        )}
      </AnimatePresence>

      {/* Reward modal */}
      <AnimatePresence>
        {showReward && (
          <motion.div
            initial={{ opacity: 0, scale: 0.9 }}
            animate={{ opacity: 1, scale: 1 }}
            exit={{ opacity: 0, scale: 0.9 }}
            className="m-4 rounded-lg bg-gradient-to-r from-emerald-50 to-teal-50 p-4 dark:from-emerald-950/30 dark:to-teal-950/30"
          >
            <div className="flex items-center gap-3">
              <Gift className="h-6 w-6 text-emerald-500" />
              <div>
                <div className="text-sm font-semibold text-emerald-800 dark:text-emerald-300">
                  Setup complete!
                </div>
                <div className="text-xs text-emerald-600 dark:text-emerald-400">
                  You have unlocked all features. Happy building!
                </div>
              </div>
            </div>
          </motion.div>
        )}
      </AnimatePresence>
    </div>
  );
}

// --- Persistent state hook ---
// hooks/use-onboarding-checklist.ts
import { useEffect, useState, useCallback } from "react";

export function useOnboardingChecklist(userId: string) {
  const [completedItems, setCompletedItems] = useState<Set<string>>(new Set());

  useEffect(() => {
    // Load from DB or localStorage
    async function load() {
      const res = await fetch(`/api/onboarding/checklist?userId=${userId}`);
      const data = await res.json();
      setCompletedItems(new Set(data.completed));
    }
    load();
  }, [userId]);

  const markComplete = useCallback(
    async (itemId: string) => {
      setCompletedItems((prev) => new Set([...prev, itemId]));
      await fetch("/api/onboarding/checklist", {
        method: "POST",
        body: JSON.stringify({ userId, itemId }),
      });
    },
    [userId]
  );

  return { completedItems, markComplete };
}
```

---

## 3. Activation Metrics

Define activation events, track time-to-value, and identify drop-off points.

```tsx
// lib/activation.ts

// --- Define activation events ---
interface ActivationMilestone {
  id: string;
  name: string;
  description: string;
  isRequired: boolean;
  weight: number; // 0-1, importance for activation score
}

const ACTIVATION_MILESTONES: ActivationMilestone[] = [
  { id: "signup_complete", name: "Signed up", description: "Account created", isRequired: true, weight: 0.1 },
  { id: "profile_setup", name: "Profile set up", description: "Name and avatar added", isRequired: false, weight: 0.1 },
  { id: "first_project", name: "First project created", description: "Created their first project", isRequired: true, weight: 0.3 },
  { id: "invite_team", name: "Team member invited", description: "Invited at least one team member", isRequired: false, weight: 0.2 },
  { id: "first_integration", name: "Integration connected", description: "Connected a third-party service", isRequired: true, weight: 0.3 },
];

// --- Track activation events ---
// lib/activation-tracker.ts
interface ActivationEvent {
  userId: string;
  milestoneId: string;
  timestamp: Date;
  metadata?: Record<string, unknown>;
}

async function trackActivation(event: ActivationEvent) {
  // Store event
  await db.activationEvent.upsert({
    where: {
      userId_milestoneId: { userId: event.userId, milestoneId: event.milestoneId },
    },
    create: {
      userId: event.userId,
      milestoneId: event.milestoneId,
      completedAt: event.timestamp,
      metadata: event.metadata ?? {},
    },
    update: {}, // Don't overwrite if already completed
  });

  // Check if user is now "activated"
  const completed = await db.activationEvent.findMany({
    where: { userId: event.userId },
  });

  const completedIds = new Set(completed.map((e) => e.milestoneId));
  const requiredMilestones = ACTIVATION_MILESTONES.filter((m) => m.isRequired);
  const isActivated = requiredMilestones.every((m) => completedIds.has(m.id));

  if (isActivated) {
    await db.user.update({
      where: { id: event.userId },
      data: { activatedAt: new Date() },
    });

    // Track time-to-value
    const user = await db.user.findUnique({ where: { id: event.userId } });
    const timeToValue = event.timestamp.getTime() - user!.createdAt.getTime();
    console.log(`User ${event.userId} activated in ${timeToValue / 1000 / 60} minutes`);
  }
}

// --- Activation score calculation ---
function calculateActivationScore(
  completedMilestoneIds: string[],
  milestones: ActivationMilestone[] = ACTIVATION_MILESTONES
): number {
  const completedSet = new Set(completedMilestoneIds);
  return milestones.reduce((score, milestone) => {
    return score + (completedSet.has(milestone.id) ? milestone.weight : 0);
  }, 0);
}

// --- Drop-off analysis query ---
// app/api/admin/activation-funnel/route.ts
export async function GET() {
  const totalUsers = await db.user.count();

  const funnel = await Promise.all(
    ACTIVATION_MILESTONES.map(async (milestone) => {
      const count = await db.activationEvent.count({
        where: { milestoneId: milestone.id },
      });
      return {
        milestone: milestone.name,
        count,
        percentage: ((count / totalUsers) * 100).toFixed(1),
      };
    })
  );

  return Response.json({ totalUsers, funnel });
}
```

---

## 4. Welcome Wizard

Multi-step setup flow with personalization questions, skip option, and progress indicator.

```tsx
// components/onboarding/welcome-wizard.tsx
"use client";

import { useState } from "react";
import { motion, AnimatePresence } from "framer-motion";
import { useRouter } from "next/navigation";

interface WizardStep {
  id: string;
  title: string;
  subtitle: string;
  component: React.ComponentType<{
    data: Record<string, unknown>;
    onUpdate: (data: Record<string, unknown>) => void;
  }>;
  isSkippable: boolean;
}

const WIZARD_STEPS: WizardStep[] = [
  {
    id: "role",
    title: "What is your role?",
    subtitle: "This helps us customize your experience.",
    component: RoleStep,
    isSkippable: false,
  },
  {
    id: "team_size",
    title: "How big is your team?",
    subtitle: "We will set up your workspace accordingly.",
    component: TeamSizeStep,
    isSkippable: true,
  },
  {
    id: "use_case",
    title: "What will you use this for?",
    subtitle: "Select all that apply.",
    component: UseCaseStep,
    isSkippable: true,
  },
  {
    id: "integrations",
    title: "Connect your tools",
    subtitle: "Import your data from existing services.",
    component: IntegrationsStep,
    isSkippable: true,
  },
];

export function WelcomeWizard() {
  const router = useRouter();
  const [currentStep, setCurrentStep] = useState(0);
  const [wizardData, setWizardData] = useState<Record<string, unknown>>({});

  const step = WIZARD_STEPS[currentStep];
  const StepComponent = step.component;
  const isLastStep = currentStep === WIZARD_STEPS.length - 1;

  const handleNext = async () => {
    if (isLastStep) {
      // Save all wizard data
      await fetch("/api/onboarding/wizard", {
        method: "POST",
        body: JSON.stringify(wizardData),
      });
      router.push("/dashboard");
    } else {
      setCurrentStep((prev) => prev + 1);
    }
  };

  const handleSkip = () => {
    if (isLastStep) {
      handleNext();
    } else {
      setCurrentStep((prev) => prev + 1);
    }
  };

  const handleSkipAll = async () => {
    await fetch("/api/onboarding/wizard", {
      method: "POST",
      body: JSON.stringify({ ...wizardData, skipped: true }),
    });
    router.push("/dashboard");
  };

  return (
    <div className="flex min-h-screen items-center justify-center bg-zinc-50 dark:bg-zinc-950">
      <div className="w-full max-w-lg">
        {/* Progress bar */}
        <div className="mb-8 flex items-center gap-2">
          {WIZARD_STEPS.map((_, i) => (
            <div
              key={i}
              className={`h-1 flex-1 rounded-full transition-colors ${
                i <= currentStep
                  ? "bg-zinc-900 dark:bg-white"
                  : "bg-zinc-200 dark:bg-zinc-800"
              }`}
            />
          ))}
        </div>

        {/* Step content */}
        <AnimatePresence mode="wait">
          <motion.div
            key={step.id}
            initial={{ opacity: 0, x: 20 }}
            animate={{ opacity: 1, x: 0 }}
            exit={{ opacity: 0, x: -20 }}
            transition={{ duration: 0.2 }}
          >
            <h1 className="mb-2 text-2xl font-bold text-zinc-900 dark:text-white">
              {step.title}
            </h1>
            <p className="mb-8 text-zinc-500">{step.subtitle}</p>

            <StepComponent
              data={wizardData}
              onUpdate={(update) =>
                setWizardData((prev) => ({ ...prev, ...update }))
              }
            />
          </motion.div>
        </AnimatePresence>

        {/* Actions */}
        <div className="mt-8 flex items-center justify-between">
          <div className="flex gap-3">
            {step.isSkippable && (
              <button
                onClick={handleSkip}
                className="text-sm text-zinc-400 hover:text-zinc-600"
              >
                Skip this step
              </button>
            )}
          </div>
          <div className="flex gap-3">
            <button
              onClick={handleSkipAll}
              className="rounded-lg px-4 py-2 text-sm text-zinc-500 hover:bg-zinc-100 dark:hover:bg-zinc-800"
            >
              Skip setup
            </button>
            <button
              onClick={handleNext}
              className="rounded-lg bg-zinc-900 px-6 py-2 text-sm font-medium text-white hover:bg-zinc-800 dark:bg-white dark:text-zinc-900 dark:hover:bg-zinc-200"
            >
              {isLastStep ? "Get started" : "Continue"}
            </button>
          </div>
        </div>
      </div>
    </div>
  );
}

// --- Example step component ---
function RoleStep({
  data,
  onUpdate,
}: {
  data: Record<string, unknown>;
  onUpdate: (data: Record<string, unknown>) => void;
}) {
  const roles = [
    { id: "founder", label: "Founder / CEO", icon: "rocket" },
    { id: "developer", label: "Developer", icon: "code" },
    { id: "designer", label: "Designer", icon: "palette" },
    { id: "marketer", label: "Marketer", icon: "megaphone" },
    { id: "pm", label: "Product Manager", icon: "clipboard" },
    { id: "other", label: "Other", icon: "user" },
  ];

  return (
    <div className="grid grid-cols-2 gap-3">
      {roles.map((role) => (
        <button
          key={role.id}
          onClick={() => onUpdate({ role: role.id })}
          className={`rounded-xl border-2 p-4 text-left transition-colors ${
            data.role === role.id
              ? "border-zinc-900 bg-zinc-50 dark:border-white dark:bg-zinc-800"
              : "border-zinc-200 hover:border-zinc-300 dark:border-zinc-700"
          }`}
        >
          <div className="text-sm font-medium text-zinc-900 dark:text-white">
            {role.label}
          </div>
        </button>
      ))}
    </div>
  );
}

function TeamSizeStep({ data, onUpdate }: { data: Record<string, unknown>; onUpdate: (d: Record<string, unknown>) => void }) {
  const sizes = ["Just me", "2-5", "6-20", "21-50", "50+"];
  return (
    <div className="flex flex-col gap-2">
      {sizes.map((size) => (
        <button
          key={size}
          onClick={() => onUpdate({ teamSize: size })}
          className={`rounded-lg border-2 p-3 text-left text-sm transition-colors ${
            data.teamSize === size
              ? "border-zinc-900 dark:border-white"
              : "border-zinc-200 dark:border-zinc-700"
          }`}
        >
          {size}
        </button>
      ))}
    </div>
  );
}

function UseCaseStep({ data, onUpdate }: { data: Record<string, unknown>; onUpdate: (d: Record<string, unknown>) => void }) {
  return <div>Use case selection UI</div>;
}

function IntegrationsStep({ data, onUpdate }: { data: Record<string, unknown>; onUpdate: (d: Record<string, unknown>) => void }) {
  return <div>Integration connection UI</div>;
}
```

---

## 5. Sample Data

Seed demo data for new users with an "explore with sample data" option and cleanup.

```tsx
// lib/onboarding/sample-data.ts
import "server-only";

interface SampleDataConfig {
  userId: string;
  templates: SampleTemplate[];
}

interface SampleTemplate {
  type: string;
  data: Record<string, unknown>[];
}

const SAMPLE_DATA_TEMPLATES: SampleTemplate[] = [
  {
    type: "project",
    data: [
      {
        name: "Marketing Website Redesign",
        description: "A sample project to explore project management features.",
        status: "in_progress",
        tasks: [
          { title: "Design homepage mockup", status: "done" },
          { title: "Build component library", status: "in_progress" },
          { title: "Write copy for About page", status: "todo" },
          { title: "Set up analytics", status: "todo" },
        ],
      },
    ],
  },
  {
    type: "contact",
    data: [
      { name: "Alex Johnson", email: "alex@example.com", company: "Acme Corp" },
      { name: "Sarah Chen", email: "sarah@example.com", company: "TechStart Inc" },
    ],
  },
];

async function seedSampleData(userId: string): Promise<string[]> {
  const createdIds: string[] = [];

  for (const template of SAMPLE_DATA_TEMPLATES) {
    for (const item of template.data) {
      const record = await db[template.type].create({
        data: {
          ...item,
          userId,
          isSampleData: true, // Flag for cleanup
          createdAt: new Date(),
        },
      });
      createdIds.push(record.id);
    }
  }

  // Mark user as having sample data
  await db.user.update({
    where: { id: userId },
    data: { hasSampleData: true },
  });

  return createdIds;
}

async function cleanupSampleData(userId: string): Promise<number> {
  // Delete all sample data for user across all tables
  const tables = ["project", "task", "contact"];
  let totalDeleted = 0;

  for (const table of tables) {
    const result = await db[table].deleteMany({
      where: { userId, isSampleData: true },
    });
    totalDeleted += result.count;
  }

  await db.user.update({
    where: { id: userId },
    data: { hasSampleData: false },
  });

  return totalDeleted;
}

// --- API routes ---
// app/api/onboarding/sample-data/route.ts
export async function POST(request: Request) {
  const { userId } = await request.json();
  const ids = await seedSampleData(userId);
  return Response.json({ created: ids.length, ids });
}

export async function DELETE(request: Request) {
  const { userId } = await request.json();
  const deleted = await cleanupSampleData(userId);
  return Response.json({ deleted });
}

// --- Component ---
// components/onboarding/sample-data-banner.tsx
"use client";

export function SampleDataBanner({ onCleanup }: { onCleanup: () => void }) {
  return (
    <div className="flex items-center justify-between rounded-lg border border-amber-200 bg-amber-50 px-4 py-3 dark:border-amber-800 dark:bg-amber-950/30">
      <div className="flex items-center gap-2">
        <span className="text-sm text-amber-800 dark:text-amber-200">
          You are viewing sample data.
        </span>
      </div>
      <div className="flex gap-2">
        <button
          onClick={onCleanup}
          className="rounded-md bg-amber-200 px-3 py-1 text-xs font-medium text-amber-900 hover:bg-amber-300"
        >
          Remove sample data
        </button>
      </div>
    </div>
  );
}
```

---

## 6. Empty State CTAs

Guide users from empty dashboards to their first action with inline tutorials.

```tsx
// components/empty-states/empty-state.tsx
import { LucideIcon } from "lucide-react";

interface EmptyStateProps {
  icon: LucideIcon;
  title: string;
  description: string;
  action: {
    label: string;
    onClick: () => void;
  };
  secondaryAction?: {
    label: string;
    onClick: () => void;
  };
  illustration?: React.ReactNode;
}

export function EmptyState({
  icon: Icon,
  title,
  description,
  action,
  secondaryAction,
  illustration,
}: EmptyStateProps) {
  return (
    <div className="flex flex-col items-center justify-center px-4 py-16 text-center">
      {illustration ?? (
        <div className="mb-4 flex h-16 w-16 items-center justify-center rounded-2xl bg-zinc-100 dark:bg-zinc-800">
          <Icon className="h-8 w-8 text-zinc-400" />
        </div>
      )}

      <h3 className="mb-2 text-lg font-semibold text-zinc-900 dark:text-white">
        {title}
      </h3>
      <p className="mb-6 max-w-sm text-sm text-zinc-500">
        {description}
      </p>

      <div className="flex items-center gap-3">
        <button
          onClick={action.onClick}
          className="rounded-lg bg-zinc-900 px-5 py-2.5 text-sm font-medium text-white hover:bg-zinc-800 dark:bg-white dark:text-zinc-900"
        >
          {action.label}
        </button>
        {secondaryAction && (
          <button
            onClick={secondaryAction.onClick}
            className="rounded-lg border border-zinc-200 px-5 py-2.5 text-sm text-zinc-600 hover:bg-zinc-50 dark:border-zinc-700 dark:text-zinc-400"
          >
            {secondaryAction.label}
          </button>
        )}
      </div>
    </div>
  );
}

// --- Contextual empty states per section ---
// components/empty-states/projects-empty.tsx
import { FolderPlus, PlayCircle } from "lucide-react";

export function ProjectsEmpty({
  onCreateProject,
  onLoadSampleData,
}: {
  onCreateProject: () => void;
  onLoadSampleData: () => void;
}) {
  return (
    <EmptyState
      icon={FolderPlus}
      title="No projects yet"
      description="Create your first project to start organizing your work. Or explore with sample data to see what is possible."
      action={{ label: "Create project", onClick: onCreateProject }}
      secondaryAction={{ label: "Try with sample data", onClick: onLoadSampleData }}
    />
  );
}

// --- Inline tutorial empty state ---
export function InlineTutorialEmpty({
  onStart,
}: {
  onStart: () => void;
}) {
  const steps = [
    "Create a project",
    "Add your first task",
    "Invite a team member",
  ];

  return (
    <div className="flex flex-col items-center py-16">
      <PlayCircle className="mb-4 h-12 w-12 text-zinc-300" />
      <h3 className="mb-6 text-lg font-semibold">Get started in 3 steps</h3>

      <ol className="mb-8 space-y-3 text-left">
        {steps.map((step, i) => (
          <li key={i} className="flex items-center gap-3 text-sm text-zinc-600">
            <span className="flex h-6 w-6 items-center justify-center rounded-full bg-zinc-100 text-xs font-medium">
              {i + 1}
            </span>
            {step}
          </li>
        ))}
      </ol>

      <button
        onClick={onStart}
        className="rounded-lg bg-zinc-900 px-6 py-2.5 text-sm font-medium text-white hover:bg-zinc-800 dark:bg-white dark:text-zinc-900"
      >
        Start now
      </button>
    </div>
  );
}
```

---

## 7. Progressive Disclosure

Show features gradually, unlock advanced features after basics, and provide feature hints.

```tsx
// lib/progressive-disclosure.ts

// --- Feature levels ---
type FeatureLevel = "beginner" | "intermediate" | "advanced";

interface Feature {
  id: string;
  name: string;
  level: FeatureLevel;
  unlockedBy?: string[]; // Feature IDs that must be used first
  description: string;
}

const FEATURES: Feature[] = [
  { id: "basic_editor", name: "Text Editor", level: "beginner", description: "Create and edit documents" },
  { id: "templates", name: "Templates", level: "beginner", description: "Start from pre-built templates" },
  { id: "collaboration", name: "Real-time Collaboration", level: "intermediate", unlockedBy: ["basic_editor"], description: "Work together in real-time" },
  { id: "automations", name: "Automations", level: "intermediate", unlockedBy: ["basic_editor", "templates"], description: "Automate repetitive tasks" },
  { id: "api_access", name: "API Access", level: "advanced", unlockedBy: ["automations"], description: "Programmatic access to your data" },
  { id: "custom_webhooks", name: "Custom Webhooks", level: "advanced", unlockedBy: ["automations", "api_access"], description: "Send events to external services" },
];

async function getUnlockedFeatures(userId: string): Promise<Feature[]> {
  const usedFeatures = await db.featureUsage.findMany({
    where: { userId },
    select: { featureId: true },
  });

  const usedSet = new Set(usedFeatures.map((f) => f.featureId));

  return FEATURES.filter((feature) => {
    if (!feature.unlockedBy || feature.unlockedBy.length === 0) return true;
    return feature.unlockedBy.every((dep) => usedSet.has(dep));
  });
}

async function trackFeatureUsage(userId: string, featureId: string) {
  await db.featureUsage.upsert({
    where: { userId_featureId: { userId, featureId } },
    create: { userId, featureId, firstUsedAt: new Date(), useCount: 1 },
    update: { useCount: { increment: 1 }, lastUsedAt: new Date() },
  });
}

// --- Component: Feature gate ---
// components/onboarding/feature-gate.tsx
"use client";

import { Lock } from "lucide-react";

interface FeatureGateProps {
  featureId: string;
  isUnlocked: boolean;
  requiredFeatures: string[];
  children: React.ReactNode;
}

export function FeatureGate({
  featureId,
  isUnlocked,
  requiredFeatures,
  children,
}: FeatureGateProps) {
  if (isUnlocked) return <>{children}</>;

  return (
    <div className="relative">
      <div className="pointer-events-none opacity-40 blur-[2px]">{children}</div>
      <div className="absolute inset-0 flex items-center justify-center">
        <div className="rounded-xl bg-white/90 p-6 text-center shadow-lg backdrop-blur dark:bg-zinc-900/90">
          <Lock className="mx-auto mb-3 h-8 w-8 text-zinc-400" />
          <p className="mb-1 text-sm font-medium text-zinc-900 dark:text-white">
            Feature locked
          </p>
          <p className="text-xs text-zinc-500">
            Complete {requiredFeatures.join(" and ")} to unlock this feature.
          </p>
        </div>
      </div>
    </div>
  );
}

// --- Feature hint tooltip ---
// components/onboarding/feature-hint.tsx
"use client";

import { useState } from "react";
import { Sparkles, X } from "lucide-react";

interface FeatureHintProps {
  featureId: string;
  title: string;
  description: string;
  children: React.ReactNode;
}

export function FeatureHint({ featureId, title, description, children }: FeatureHintProps) {
  const [isDismissed, setIsDismissed] = useState(false);

  return (
    <div className="relative">
      {children}
      {!isDismissed && (
        <div className="absolute -top-2 right-0 z-10 translate-y-[-100%]">
          <div className="flex items-start gap-2 rounded-lg bg-indigo-600 p-3 text-white shadow-lg">
            <Sparkles className="mt-0.5 h-4 w-4 shrink-0" />
            <div>
              <div className="text-xs font-semibold">{title}</div>
              <div className="text-xs opacity-80">{description}</div>
            </div>
            <button onClick={() => {
              setIsDismissed(true);
              localStorage.setItem(`hint-dismissed-${featureId}`, "true");
            }}>
              <X className="h-3 w-3" />
            </button>
          </div>
          <div className="ml-4 h-2 w-2 rotate-45 bg-indigo-600" />
        </div>
      )}
    </div>
  );
}
```

---

## 8. Email Onboarding Drip

Day 0/1/3/7/14 email sequence with tips, feature highlights, and re-engagement.

```tsx
// lib/onboarding/email-drip.ts
import "server-only";

interface DripEmail {
  id: string;
  day: number;          // Days after signup
  subject: string;
  template: string;     // React Email template name
  condition?: (user: UserProfile) => boolean; // Only send if true
}

const ONBOARDING_DRIP: DripEmail[] = [
  {
    id: "welcome",
    day: 0,
    subject: "Welcome to {appName} - here is how to get started",
    template: "welcome",
  },
  {
    id: "day1_quickstart",
    day: 1,
    subject: "Create your first project in 2 minutes",
    template: "quickstart",
    condition: (user) => !user.hasCreatedProject,
  },
  {
    id: "day3_features",
    day: 3,
    subject: "3 features you should try this week",
    template: "feature-highlights",
  },
  {
    id: "day7_checkin",
    day: 7,
    subject: "How is it going? Here are some tips",
    template: "checkin",
    condition: (user) => !user.isActivated,
  },
  {
    id: "day14_reengage",
    day: 14,
    subject: "We miss you - here is what is new",
    template: "re-engagement",
    condition: (user) => !user.isActivated && !user.wasActiveLastWeek,
  },
];

// --- Cron job to send drip emails ---
// app/api/cron/onboarding-drip/route.ts
import { Resend } from "resend";
import { WelcomeEmail } from "@/emails/welcome";
import { QuickstartEmail } from "@/emails/quickstart";

const resend = new Resend(process.env.RESEND_API_KEY);

const EMAIL_TEMPLATES: Record<string, React.ComponentType<{ user: UserProfile }>> = {
  welcome: WelcomeEmail,
  quickstart: QuickstartEmail,
  // ... more templates
};

export async function GET(request: Request) {
  const authHeader = request.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return Response.json({ error: "Unauthorized" }, { status: 401 });
  }

  const results = { sent: 0, skipped: 0, errors: 0 };

  for (const dripEmail of ONBOARDING_DRIP) {
    // Find users who signed up exactly N days ago
    const targetDate = new Date();
    targetDate.setDate(targetDate.getDate() - dripEmail.day);
    const startOfDay = new Date(targetDate.setHours(0, 0, 0, 0));
    const endOfDay = new Date(targetDate.setHours(23, 59, 59, 999));

    const users = await db.user.findMany({
      where: {
        createdAt: { gte: startOfDay, lte: endOfDay },
        emailOptOut: false,
      },
    });

    for (const user of users) {
      // Check if already sent
      const alreadySent = await db.dripEmailLog.findUnique({
        where: { userId_emailId: { userId: user.id, emailId: dripEmail.id } },
      });
      if (alreadySent) {
        results.skipped++;
        continue;
      }

      // Check condition
      if (dripEmail.condition && !dripEmail.condition(user as UserProfile)) {
        results.skipped++;
        continue;
      }

      try {
        const Template = EMAIL_TEMPLATES[dripEmail.template];
        await resend.emails.send({
          from: "App <hello@yourdomain.com>",
          to: user.email,
          subject: dripEmail.subject.replace("{appName}", "YourApp"),
          react: Template({ user: user as UserProfile }),
        });

        await db.dripEmailLog.create({
          data: { userId: user.id, emailId: dripEmail.id, sentAt: new Date() },
        });

        results.sent++;
      } catch (error) {
        console.error(`Failed to send ${dripEmail.id} to ${user.email}:`, error);
        results.errors++;
      }
    }
  }

  return Response.json(results);
}

// --- React Email template example ---
// emails/welcome.tsx
import { Html, Head, Body, Container, Heading, Text, Button, Hr } from "@react-email/components";

interface WelcomeEmailProps {
  user: { name: string; email: string };
}

export function WelcomeEmail({ user }: WelcomeEmailProps) {
  return (
    <Html>
      <Head />
      <Body style={{ fontFamily: "system-ui, sans-serif", backgroundColor: "#fafafa" }}>
        <Container style={{ maxWidth: "480px", margin: "0 auto", padding: "40px 20px" }}>
          <Heading style={{ fontSize: "24px", fontWeight: 600, color: "#18181b" }}>
            Welcome, {user.name}!
          </Heading>
          <Text style={{ color: "#52525b", lineHeight: 1.6 }}>
            Thank you for signing up. Here is how to get the most out of your account:
          </Text>

          <div style={{ margin: "24px 0", padding: "16px", backgroundColor: "#fff", borderRadius: "8px", border: "1px solid #e4e4e7" }}>
            <Text style={{ fontWeight: 600, marginBottom: "8px" }}>Quick start:</Text>
            <Text>1. Create your first project</Text>
            <Text>2. Invite your team</Text>
            <Text>3. Connect your tools</Text>
          </div>

          <Button
            href={`${process.env.NEXT_PUBLIC_APP_URL}/dashboard`}
            style={{
              backgroundColor: "#18181b",
              color: "#fff",
              padding: "12px 24px",
              borderRadius: "8px",
              textDecoration: "none",
              fontWeight: 500,
            }}
          >
            Go to Dashboard
          </Button>

          <Hr style={{ margin: "32px 0", borderColor: "#e4e4e7" }} />
          <Text style={{ fontSize: "12px", color: "#a1a1aa" }}>
            Questions? Reply to this email or visit our help center.
          </Text>
        </Container>
      </Body>
    </Html>
  );
}
```

---

## 9. Help System

Contextual help tooltips, help center link, and chat widget integration.

```tsx
// components/help/contextual-help.tsx
"use client";

import { useState, useRef, useEffect } from "react";
import { HelpCircle, ExternalLink, MessageCircle, X } from "lucide-react";
import { AnimatePresence, motion } from "framer-motion";

interface HelpTooltipProps {
  title: string;
  content: string;
  learnMoreUrl?: string;
}

export function HelpTooltip({ title, content, learnMoreUrl }: HelpTooltipProps) {
  const [isOpen, setIsOpen] = useState(false);
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    function handleClickOutside(e: MouseEvent) {
      if (ref.current && !ref.current.contains(e.target as Node)) {
        setIsOpen(false);
      }
    }
    document.addEventListener("mousedown", handleClickOutside);
    return () => document.removeEventListener("mousedown", handleClickOutside);
  }, []);

  return (
    <div className="relative inline-block" ref={ref}>
      <button
        onClick={() => setIsOpen(!isOpen)}
        className="text-zinc-400 hover:text-zinc-600 dark:hover:text-zinc-300"
        aria-label="Help"
      >
        <HelpCircle className="h-4 w-4" />
      </button>

      <AnimatePresence>
        {isOpen && (
          <motion.div
            initial={{ opacity: 0, y: 4 }}
            animate={{ opacity: 1, y: 0 }}
            exit={{ opacity: 0, y: 4 }}
            className="absolute left-1/2 top-full z-50 mt-2 w-64 -translate-x-1/2 rounded-lg bg-white p-4 shadow-xl ring-1 ring-zinc-200 dark:bg-zinc-900 dark:ring-zinc-700"
          >
            <h4 className="mb-1 text-sm font-semibold text-zinc-900 dark:text-white">{title}</h4>
            <p className="text-xs leading-relaxed text-zinc-500">{content}</p>
            {learnMoreUrl && (
              <a
                href={learnMoreUrl}
                target="_blank"
                rel="noopener noreferrer"
                className="mt-2 flex items-center gap-1 text-xs font-medium text-blue-600 hover:text-blue-500"
              >
                Learn more <ExternalLink className="h-3 w-3" />
              </a>
            )}
          </motion.div>
        )}
      </AnimatePresence>
    </div>
  );
}

// --- Help center launcher ---
// components/help/help-launcher.tsx
"use client";

export function HelpLauncher() {
  const [isOpen, setIsOpen] = useState(false);

  const helpLinks = [
    { label: "Documentation", href: "/docs", icon: "book" },
    { label: "Video tutorials", href: "/tutorials", icon: "play" },
    { label: "API reference", href: "/api-docs", icon: "code" },
    { label: "Community", href: "/community", icon: "users" },
    { label: "Contact support", action: () => openChat(), icon: "message" },
  ];

  return (
    <>
      <button
        onClick={() => setIsOpen(true)}
        className="fixed bottom-6 right-6 z-50 flex h-12 w-12 items-center justify-center rounded-full bg-zinc-900 text-white shadow-lg hover:bg-zinc-800 dark:bg-white dark:text-zinc-900"
        aria-label="Help"
      >
        <MessageCircle className="h-5 w-5" />
      </button>

      <AnimatePresence>
        {isOpen && (
          <motion.div
            initial={{ opacity: 0, y: 16, scale: 0.95 }}
            animate={{ opacity: 1, y: 0, scale: 1 }}
            exit={{ opacity: 0, y: 16, scale: 0.95 }}
            className="fixed bottom-20 right-6 z-50 w-72 rounded-xl bg-white shadow-2xl ring-1 ring-zinc-200 dark:bg-zinc-900 dark:ring-zinc-700"
          >
            <div className="flex items-center justify-between border-b border-zinc-100 px-4 py-3 dark:border-zinc-800">
              <span className="text-sm font-semibold">Help & Resources</span>
              <button onClick={() => setIsOpen(false)}>
                <X className="h-4 w-4 text-zinc-400" />
              </button>
            </div>
            <div className="p-2">
              {helpLinks.map((link) => (
                <a
                  key={link.label}
                  href={link.href}
                  onClick={(e) => {
                    if (link.action) {
                      e.preventDefault();
                      link.action();
                    }
                  }}
                  className="flex items-center gap-3 rounded-lg px-3 py-2.5 text-sm text-zinc-600 hover:bg-zinc-50 dark:text-zinc-300 dark:hover:bg-zinc-800"
                >
                  {link.label}
                </a>
              ))}
            </div>
          </motion.div>
        )}
      </AnimatePresence>
    </>
  );
}

// --- Chat widget integration (Intercom example) ---
// components/help/intercom-provider.tsx
"use client";

import { useEffect } from "react";

interface IntercomUser {
  id: string;
  name: string;
  email: string;
  createdAt: number;
}

export function IntercomProvider({ user }: { user: IntercomUser }) {
  useEffect(() => {
    // Load Intercom script
    const w = window as any;
    w.intercomSettings = {
      api_base: "https://api-iam.intercom.io",
      app_id: process.env.NEXT_PUBLIC_INTERCOM_APP_ID,
      user_id: user.id,
      name: user.name,
      email: user.email,
      created_at: user.createdAt,
    };

    const script = document.createElement("script");
    script.async = true;
    script.src = `https://widget.intercom.io/widget/${process.env.NEXT_PUBLIC_INTERCOM_APP_ID}`;
    document.body.appendChild(script);

    return () => {
      document.body.removeChild(script);
      if (w.Intercom) w.Intercom("shutdown");
    };
  }, [user]);

  return null;
}

function openChat() {
  const w = window as any;
  if (w.Intercom) {
    w.Intercom("show");
  }
}
```

---

## 10. Measuring Onboarding Success

Track onboarding completion rate, time to first value, drop-off analysis, and A/B testing flows.

```tsx
// lib/onboarding/analytics.ts

// --- Core metrics ---
interface OnboardingMetrics {
  totalSignups: number;
  completedOnboarding: number;
  completionRate: number;
  medianTimeToFirstValue: number;      // minutes
  activationRate: number;
  dropOffByStep: Record<string, number>;
}

async function getOnboardingMetrics(
  startDate: Date,
  endDate: Date
): Promise<OnboardingMetrics> {
  const totalSignups = await db.user.count({
    where: { createdAt: { gte: startDate, lte: endDate } },
  });

  const completedOnboarding = await db.user.count({
    where: {
      createdAt: { gte: startDate, lte: endDate },
      onboardingCompletedAt: { not: null },
    },
  });

  const activatedUsers = await db.user.count({
    where: {
      createdAt: { gte: startDate, lte: endDate },
      activatedAt: { not: null },
    },
  });

  // Time to first value (median)
  const timeToValues = await db.$queryRaw<{ ttv: number }[]>`
    SELECT EXTRACT(EPOCH FROM (activated_at - created_at)) / 60 as ttv
    FROM users
    WHERE created_at >= ${startDate}
      AND created_at <= ${endDate}
      AND activated_at IS NOT NULL
    ORDER BY ttv
  `;

  const medianIndex = Math.floor(timeToValues.length / 2);
  const medianTimeToFirstValue = timeToValues[medianIndex]?.ttv ?? 0;

  // Drop-off by step
  const stepCounts = await db.onboardingStep.groupBy({
    by: ["stepId"],
    _count: { userId: true },
    where: {
      user: { createdAt: { gte: startDate, lte: endDate } },
    },
  });

  const dropOffByStep: Record<string, number> = {};
  const sortedSteps = stepCounts.sort((a, b) => a._count.userId - b._count.userId);
  for (let i = 0; i < sortedSteps.length; i++) {
    const current = sortedSteps[i]._count.userId;
    const previous = i === 0 ? totalSignups : sortedSteps[i - 1]._count.userId;
    dropOffByStep[sortedSteps[i].stepId] = previous - current;
  }

  return {
    totalSignups,
    completedOnboarding,
    completionRate: totalSignups > 0 ? completedOnboarding / totalSignups : 0,
    medianTimeToFirstValue,
    activationRate: totalSignups > 0 ? activatedUsers / totalSignups : 0,
    dropOffByStep,
  };
}

// --- A/B testing onboarding flows ---
// lib/onboarding/ab-test.ts
interface OnboardingVariant {
  id: string;
  name: string;
  weight: number; // 0-1, must sum to 1 across variants
  config: {
    showWelcomeWizard: boolean;
    showProductTour: boolean;
    showSampleData: boolean;
    checklistItems: string[];
    dripEmailSequence: string;
  };
}

const ONBOARDING_VARIANTS: OnboardingVariant[] = [
  {
    id: "control",
    name: "Standard onboarding",
    weight: 0.5,
    config: {
      showWelcomeWizard: true,
      showProductTour: true,
      showSampleData: false,
      checklistItems: ["create_project", "invite_team", "connect_integration"],
      dripEmailSequence: "standard",
    },
  },
  {
    id: "streamlined",
    name: "Streamlined onboarding",
    weight: 0.5,
    config: {
      showWelcomeWizard: false,
      showProductTour: true,
      showSampleData: true,
      checklistItems: ["explore_sample", "create_own_project"],
      dripEmailSequence: "short",
    },
  },
];

function assignVariant(userId: string): OnboardingVariant {
  // Deterministic assignment based on user ID (consistent across sessions)
  const hash = simpleHash(userId);
  const normalized = (hash % 1000) / 1000;

  let cumulative = 0;
  for (const variant of ONBOARDING_VARIANTS) {
    cumulative += variant.weight;
    if (normalized < cumulative) return variant;
  }

  return ONBOARDING_VARIANTS[0];
}

function simpleHash(str: string): number {
  let hash = 0;
  for (let i = 0; i < str.length; i++) {
    const char = str.charCodeAt(i);
    hash = (hash << 5) - hash + char;
    hash |= 0;
  }
  return Math.abs(hash);
}

// --- Track variant performance ---
async function getVariantPerformance(
  startDate: Date,
  endDate: Date
): Promise<Record<string, { completionRate: number; activationRate: number; medianTtv: number }>> {
  const results: Record<string, { completionRate: number; activationRate: number; medianTtv: number }> = {};

  for (const variant of ONBOARDING_VARIANTS) {
    const users = await db.user.findMany({
      where: {
        createdAt: { gte: startDate, lte: endDate },
        onboardingVariant: variant.id,
      },
      select: {
        onboardingCompletedAt: true,
        activatedAt: true,
        createdAt: true,
      },
    });

    const total = users.length;
    const completed = users.filter((u) => u.onboardingCompletedAt).length;
    const activated = users.filter((u) => u.activatedAt).length;

    const ttvValues = users
      .filter((u) => u.activatedAt)
      .map((u) => (u.activatedAt!.getTime() - u.createdAt.getTime()) / 60_000)
      .sort((a, b) => a - b);

    results[variant.id] = {
      completionRate: total > 0 ? completed / total : 0,
      activationRate: total > 0 ? activated / total : 0,
      medianTtv: ttvValues[Math.floor(ttvValues.length / 2)] ?? 0,
    };
  }

  return results;
}

// --- Dashboard component for metrics ---
// app/admin/onboarding/page.tsx
// (Server component that fetches and displays metrics)
export default async function OnboardingDashboard() {
  const now = new Date();
  const thirtyDaysAgo = new Date(now.getTime() - 30 * 24 * 60 * 60 * 1000);

  const metrics = await getOnboardingMetrics(thirtyDaysAgo, now);
  const variantPerformance = await getVariantPerformance(thirtyDaysAgo, now);

  return (
    <div className="space-y-8 p-8">
      <h1 className="text-2xl font-bold">Onboarding Analytics</h1>

      {/* KPI cards */}
      <div className="grid grid-cols-4 gap-4">
        <MetricCard label="Completion Rate" value={`${(metrics.completionRate * 100).toFixed(1)}%`} />
        <MetricCard label="Activation Rate" value={`${(metrics.activationRate * 100).toFixed(1)}%`} />
        <MetricCard label="Median Time to Value" value={`${metrics.medianTimeToFirstValue.toFixed(0)} min`} />
        <MetricCard label="Total Signups" value={metrics.totalSignups.toString()} />
      </div>

      {/* Drop-off funnel */}
      <div className="rounded-xl border p-6">
        <h2 className="mb-4 text-lg font-semibold">Drop-off by Step</h2>
        {Object.entries(metrics.dropOffByStep).map(([step, dropOff]) => (
          <div key={step} className="flex items-center justify-between border-b py-2">
            <span className="text-sm">{step}</span>
            <span className="text-sm text-red-500">-{dropOff} users</span>
          </div>
        ))}
      </div>

      {/* A/B test results */}
      <div className="rounded-xl border p-6">
        <h2 className="mb-4 text-lg font-semibold">A/B Test Results</h2>
        {Object.entries(variantPerformance).map(([variantId, perf]) => (
          <div key={variantId} className="mb-4 rounded-lg bg-zinc-50 p-4 dark:bg-zinc-800">
            <div className="mb-2 font-medium">{variantId}</div>
            <div className="grid grid-cols-3 gap-4 text-sm">
              <div>Completion: {(perf.completionRate * 100).toFixed(1)}%</div>
              <div>Activation: {(perf.activationRate * 100).toFixed(1)}%</div>
              <div>Median TTV: {perf.medianTtv.toFixed(0)} min</div>
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}

function MetricCard({ label, value }: { label: string; value: string }) {
  return (
    <div className="rounded-xl border border-zinc-200 p-4 dark:border-zinc-800">
      <div className="text-xs text-zinc-500">{label}</div>
      <div className="mt-1 text-2xl font-bold text-zinc-900 dark:text-white">{value}</div>
    </div>
  );
}
```
