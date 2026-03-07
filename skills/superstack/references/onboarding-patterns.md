# User Onboarding Patterns for SaaS/Web Apps

## Table of Contents
- [1. Product Tours](#1-product-tours)
  - [Using react-joyride](#using-react-joyride)
  - [Custom Tooltip Tour (no library)](#custom-tooltip-tour-no-library)
- [2. Onboarding Checklist](#2-onboarding-checklist)
  - [Checklist Component](#checklist-component)
  - [Supabase Schema for Persistent State](#supabase-schema-for-persistent-state)
  - [API Route for Progress](#api-route-for-progress)
- [3. Activation Metrics](#3-activation-metrics)
  - [Activation Event Tracking](#activation-event-tracking)
  - [Time-to-Value Tracking](#time-to-value-tracking)
  - [Drop-off Analysis Dashboard](#drop-off-analysis-dashboard)
- [4. Welcome Wizard](#4-welcome-wizard)
- [5. Sample Data](#5-sample-data)
  - [Seed Function](#seed-function)
  - [Sample Data Banner](#sample-data-banner)
  - [Cleanup API](#cleanup-api)
- [6. Empty State CTAs](#6-empty-state-ctas)
- [7. Progressive Disclosure](#7-progressive-disclosure)
- [8. Email Onboarding Drip](#8-email-onboarding-drip)
  - [Drip Configuration and Scheduler](#drip-configuration-and-scheduler)
  - [Cron-based Email Sender](#cron-based-email-sender)
- [9. Help System](#9-help-system)
  - [Contextual Tooltip](#contextual-tooltip)
  - [Slide-in Help Panel](#slide-in-help-panel)
  - [Chat Widget Integration](#chat-widget-integration)
- [10. Measuring Success](#10-measuring-success)
  - [Analytics Dashboard Component](#analytics-dashboard-component)
  - [A/B Testing Assignment](#ab-testing-assignment)
  - [Metrics API Route](#metrics-api-route)
  - [Supabase RPC Functions for Metrics](#supabase-rpc-functions-for-metrics)
- [Quick Reference: Key Decisions](#quick-reference-key-decisions)

Reference for implementing user onboarding flows with React/Next.js, Supabase, and related tooling.

---

## 1. Product Tours

Step-by-step tooltips that guide users through key features on first visit.

### Using react-joyride

```tsx
"use client";

import Joyride, { CallBackProps, STATUS, Step } from "react-joyride";
import { useState, useEffect } from "react";

const tourSteps: Step[] = [
  {
    target: "#dashboard-header",
    content: "Welcome! This is your dashboard where you can see all key metrics.",
    placement: "bottom",
    disableBeacon: true,
  },
  {
    target: "#create-campaign-btn",
    content: "Click here to create your first campaign.",
    placement: "right",
    spotlightClicks: true,
  },
  {
    target: "#analytics-panel",
    content: "Track performance in real-time from this panel.",
    placement: "left",
  },
];

export function ProductTour() {
  const [run, setRun] = useState(false);

  useEffect(() => {
    const hasSeenTour = localStorage.getItem("tour_completed");
    if (!hasSeenTour) setRun(true);
  }, []);

  function handleCallback(data: CallBackProps) {
    const { status } = data;
    if (status === STATUS.FINISHED || status === STATUS.SKIPPED) {
      setRun(false);
      localStorage.setItem("tour_completed", "true");
      // Persist to backend
      fetch("/api/onboarding/tour", { method: "POST" });
    }
  }

  return (
    <Joyride
      steps={tourSteps}
      run={run}
      continuous
      showSkipButton
      showProgress
      callback={handleCallback}
      styles={{
        options: {
          primaryColor: "#6366f1",
          zIndex: 10000,
        },
        tooltipContent: {
          fontSize: 14,
        },
      }}
      locale={{
        back: "Back",
        close: "Close",
        last: "Done",
        next: "Next",
        skip: "Skip tour",
      }}
    />
  );
}
```

### Custom Tooltip Tour (no library)

```tsx
"use client";

import { useState, useEffect } from "react";
import { createPortal } from "react-dom";
import { AnimatePresence, motion } from "framer-motion";

interface TourStep {
  targetSelector: string;
  title: string;
  content: string;
  placement: "top" | "bottom" | "left" | "right";
}

const steps: TourStep[] = [
  {
    targetSelector: "#sidebar-nav",
    title: "Navigation",
    content: "Use the sidebar to switch between sections.",
    placement: "right",
  },
  {
    targetSelector: "#data-table",
    title: "Your Data",
    content: "All your records appear here. Click any row to edit.",
    placement: "bottom",
  },
];

export function CustomTour({ onComplete }: { onComplete: () => void }) {
  const [currentStep, setCurrentStep] = useState(0);
  const [targetRect, setTargetRect] = useState<DOMRect | null>(null);

  useEffect(() => {
    const el = document.querySelector(steps[currentStep].targetSelector);
    if (!el) return;
    const rect = el.getBoundingClientRect();
    setTargetRect(rect);
    el.scrollIntoView({ behavior: "smooth", block: "center" });
  }, [currentStep]);

  function next() {
    if (currentStep < steps.length - 1) {
      setCurrentStep((s) => s + 1);
    } else {
      onComplete();
    }
  }

  if (!targetRect) return null;

  const step = steps[currentStep];

  return createPortal(
    <>
      {/* Overlay with spotlight cutout */}
      <div className="fixed inset-0 z-[9998]">
        <svg className="h-full w-full">
          <defs>
            <mask id="spotlight">
              <rect width="100%" height="100%" fill="white" />
              <rect
                x={targetRect.left - 8}
                y={targetRect.top - 8}
                width={targetRect.width + 16}
                height={targetRect.height + 16}
                rx={8}
                fill="black"
              />
            </mask>
          </defs>
          <rect
            width="100%"
            height="100%"
            fill="rgba(0,0,0,0.5)"
            mask="url(#spotlight)"
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
          className="fixed z-[9999] w-72 rounded-lg bg-white p-4 shadow-xl"
          style={{
            top: targetRect.bottom + 12,
            left: targetRect.left,
          }}
        >
          <p className="text-sm font-semibold text-gray-900">{step.title}</p>
          <p className="mt-1 text-sm text-gray-600">{step.content}</p>
          <div className="mt-3 flex items-center justify-between">
            <span className="text-xs text-gray-400">
              {currentStep + 1} / {steps.length}
            </span>
            <div className="flex gap-2">
              <button
                onClick={onComplete}
                className="text-xs text-gray-400 hover:text-gray-600"
              >
                Skip
              </button>
              <button
                onClick={next}
                className="rounded bg-indigo-600 px-3 py-1 text-xs text-white hover:bg-indigo-700"
              >
                {currentStep < steps.length - 1 ? "Next" : "Done"}
              </button>
            </div>
          </div>
        </motion.div>
      </AnimatePresence>
    </>,
    document.body
  );
}
```

---

## 2. Onboarding Checklist

A persistent progress checklist that guides users through setup tasks.

### Checklist Component

```tsx
"use client";

import { useState, useEffect } from "react";
import { motion, AnimatePresence } from "framer-motion";
import { CheckCircle2, Circle, ChevronDown, Gift } from "lucide-react";

interface ChecklistItem {
  id: string;
  title: string;
  description: string;
  action: () => void;
  actionLabel: string;
}

const checklistItems: ChecklistItem[] = [
  {
    id: "profile",
    title: "Complete your profile",
    description: "Add your name and company details.",
    action: () => (window.location.href = "/settings/profile"),
    actionLabel: "Go to profile",
  },
  {
    id: "first_project",
    title: "Create your first project",
    description: "Set up a project to start tracking data.",
    action: () => (window.location.href = "/projects/new"),
    actionLabel: "Create project",
  },
  {
    id: "invite_team",
    title: "Invite a team member",
    description: "Collaborate by inviting your team.",
    action: () => (window.location.href = "/settings/team"),
    actionLabel: "Invite team",
  },
  {
    id: "connect_source",
    title: "Connect a data source",
    description: "Link your marketing platforms.",
    action: () => (window.location.href = "/integrations"),
    actionLabel: "Connect",
  },
];

export function OnboardingChecklist() {
  const [completed, setCompleted] = useState<Set<string>>(new Set());
  const [expanded, setExpanded] = useState(true);
  const [showReward, setShowReward] = useState(false);

  useEffect(() => {
    // Load progress from backend
    fetch("/api/onboarding/progress")
      .then((r) => r.json())
      .then((data) => setCompleted(new Set(data.completedSteps)));
  }, []);

  const progress = (completed.size / checklistItems.length) * 100;
  const allDone = completed.size === checklistItems.length;

  useEffect(() => {
    if (allDone) {
      setShowReward(true);
      fetch("/api/onboarding/complete", { method: "POST" });
    }
  }, [allDone]);

  return (
    <div className="w-80 rounded-lg border border-gray-200 bg-white shadow-sm">
      {/* Header */}
      <button
        onClick={() => setExpanded(!expanded)}
        className="flex w-full items-center justify-between p-4"
      >
        <div>
          <h3 className="text-sm font-semibold text-gray-900">
            Getting Started
          </h3>
          <p className="text-xs text-gray-500">
            {completed.size} of {checklistItems.length} complete
          </p>
        </div>
        <ChevronDown
          className={`h-4 w-4 text-gray-400 transition-transform ${
            expanded ? "rotate-180" : ""
          }`}
        />
      </button>

      {/* Progress bar */}
      <div className="mx-4 mb-2 h-1.5 overflow-hidden rounded-full bg-gray-100">
        <motion.div
          className="h-full rounded-full bg-indigo-600"
          initial={{ width: 0 }}
          animate={{ width: `${progress}%` }}
          transition={{ duration: 0.5 }}
        />
      </div>

      {/* Items */}
      <AnimatePresence>
        {expanded && (
          <motion.div
            initial={{ height: 0 }}
            animate={{ height: "auto" }}
            exit={{ height: 0 }}
            className="overflow-hidden"
          >
            <div className="space-y-1 p-2">
              {checklistItems.map((item) => {
                const isDone = completed.has(item.id);
                return (
                  <div
                    key={item.id}
                    className={`rounded-md p-3 ${
                      isDone ? "opacity-60" : "hover:bg-gray-50"
                    }`}
                  >
                    <div className="flex items-start gap-3">
                      {isDone ? (
                        <CheckCircle2 className="mt-0.5 h-5 w-5 shrink-0 text-green-500" />
                      ) : (
                        <Circle className="mt-0.5 h-5 w-5 shrink-0 text-gray-300" />
                      )}
                      <div className="flex-1">
                        <p
                          className={`text-sm font-medium ${
                            isDone
                              ? "text-gray-400 line-through"
                              : "text-gray-900"
                          }`}
                        >
                          {item.title}
                        </p>
                        {!isDone && (
                          <>
                            <p className="mt-0.5 text-xs text-gray-500">
                              {item.description}
                            </p>
                            <button
                              onClick={item.action}
                              className="mt-2 text-xs font-medium text-indigo-600 hover:text-indigo-700"
                            >
                              {item.actionLabel} &rarr;
                            </button>
                          </>
                        )}
                      </div>
                    </div>
                  </div>
                );
              })}
            </div>
          </motion.div>
        )}
      </AnimatePresence>

      {/* Completion reward */}
      <AnimatePresence>
        {showReward && (
          <motion.div
            initial={{ opacity: 0, scale: 0.9 }}
            animate={{ opacity: 1, scale: 1 }}
            className="m-4 rounded-lg bg-gradient-to-r from-indigo-500 to-purple-500 p-4 text-center text-white"
          >
            <Gift className="mx-auto h-8 w-8" />
            <p className="mt-2 text-sm font-semibold">Setup Complete!</p>
            <p className="mt-1 text-xs opacity-90">
              You've unlocked 30 extra days on your trial.
            </p>
          </motion.div>
        )}
      </AnimatePresence>
    </div>
  );
}
```

### Supabase Schema for Persistent State

```sql
create table onboarding_progress (
  user_id uuid references auth.users primary key,
  completed_steps text[] default '{}',
  checklist_dismissed boolean default false,
  completed_at timestamptz,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

alter table onboarding_progress enable row level security;

create policy "Users manage own onboarding"
  on onboarding_progress for all
  using (auth.uid() = user_id);
```

### API Route for Progress

```tsx
// app/api/onboarding/progress/route.ts
import { createClient } from "@/lib/supabase/server";
import { NextResponse } from "next/server";

export async function GET() {
  const supabase = await createClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const { data } = await supabase
    .from("onboarding_progress")
    .select("completed_steps")
    .eq("user_id", user.id)
    .single();

  return NextResponse.json({ completedSteps: data?.completed_steps ?? [] });
}

export async function POST(req: Request) {
  const supabase = await createClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const { stepId } = await req.json();

  const { data: current } = await supabase
    .from("onboarding_progress")
    .select("completed_steps")
    .eq("user_id", user.id)
    .single();

  const steps = new Set(current?.completed_steps ?? []);
  steps.add(stepId);

  await supabase.from("onboarding_progress").upsert({
    user_id: user.id,
    completed_steps: Array.from(steps),
    updated_at: new Date().toISOString(),
    ...(steps.size === 4 ? { completed_at: new Date().toISOString() } : {}),
  });

  return NextResponse.json({ ok: true });
}
```

---

## 3. Activation Metrics

Define what "activated" means, track time-to-value, and identify drop-off points.

### Activation Event Tracking

```tsx
// lib/activation.ts

export interface ActivationEvent {
  event: string;
  userId: string;
  timestamp: string;
  metadata?: Record<string, unknown>;
}

const ACTIVATION_MILESTONES = [
  "signed_up",
  "completed_profile",
  "created_first_project",
  "connected_data_source",
  "viewed_first_report",
] as const;

type Milestone = (typeof ACTIVATION_MILESTONES)[number];

export async function trackActivation(
  userId: string,
  milestone: Milestone,
  metadata?: Record<string, unknown>
) {
  const event: ActivationEvent = {
    event: milestone,
    userId,
    timestamp: new Date().toISOString(),
    metadata,
  };

  // Store in Supabase
  const { createClient } = await import("@/lib/supabase/server");
  const supabase = await createClient();

  await supabase.from("activation_events").insert({
    user_id: userId,
    event_name: milestone,
    metadata,
  });

  // Also send to analytics (Posthog, Mixpanel, etc.)
  await fetch(process.env.ANALYTICS_WEBHOOK_URL!, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(event),
  });
}
```

### Time-to-Value Tracking

```tsx
// lib/time-to-value.ts

export async function calculateTimeToValue(userId: string) {
  const { createClient } = await import("@/lib/supabase/server");
  const supabase = await createClient();

  const { data: events } = await supabase
    .from("activation_events")
    .select("event_name, created_at")
    .eq("user_id", userId)
    .order("created_at", { ascending: true });

  if (!events?.length) return null;

  const signedUp = events.find((e) => e.event_name === "signed_up");
  const firstValue = events.find((e) => e.event_name === "viewed_first_report");

  if (!signedUp || !firstValue) return null;

  const ttv =
    new Date(firstValue.created_at).getTime() -
    new Date(signedUp.created_at).getTime();

  return {
    userId,
    timeToValueMs: ttv,
    timeToValueHours: ttv / (1000 * 60 * 60),
    milestonesReached: events.map((e) => e.event_name),
    dropOffPoint: getDropOffPoint(events.map((e) => e.event_name)),
  };
}

function getDropOffPoint(reached: string[]): string | null {
  const funnel = [
    "signed_up",
    "completed_profile",
    "created_first_project",
    "connected_data_source",
    "viewed_first_report",
  ];

  for (let i = 0; i < funnel.length; i++) {
    if (!reached.includes(funnel[i])) return funnel[i];
  }
  return null; // Fully activated
}
```

### Drop-off Analysis Dashboard

```tsx
"use client";

import { useEffect, useState } from "react";

interface FunnelStep {
  name: string;
  count: number;
  percentage: number;
}

export function ActivationFunnel() {
  const [funnel, setFunnel] = useState<FunnelStep[]>([]);

  useEffect(() => {
    fetch("/api/admin/activation-funnel")
      .then((r) => r.json())
      .then(setFunnel);
  }, []);

  return (
    <div className="space-y-3">
      <h3 className="text-lg font-semibold">Activation Funnel</h3>
      {funnel.map((step, i) => (
        <div key={step.name} className="space-y-1">
          <div className="flex justify-between text-sm">
            <span className="text-gray-700">{step.name}</span>
            <span className="text-gray-500">
              {step.count} ({step.percentage}%)
            </span>
          </div>
          <div className="h-3 overflow-hidden rounded-full bg-gray-100">
            <div
              className="h-full rounded-full bg-indigo-500 transition-all"
              style={{ width: `${step.percentage}%` }}
            />
          </div>
          {i < funnel.length - 1 && (
            <p className="text-xs text-red-500">
              Drop-off: {(funnel[i].percentage - funnel[i + 1].percentage).toFixed(1)}%
            </p>
          )}
        </div>
      ))}
    </div>
  );
}
```

---

## 4. Welcome Wizard

Multi-step setup flow with personalization questions, skip option, and progress indicator.

```tsx
"use client";

import { useState } from "react";
import { motion, AnimatePresence } from "framer-motion";
import { useRouter } from "next/navigation";

interface WizardStep {
  id: string;
  title: string;
  subtitle: string;
}

const wizardSteps: WizardStep[] = [
  { id: "role", title: "What's your role?", subtitle: "We'll customize your experience." },
  { id: "company", title: "Tell us about your company", subtitle: "Help us set things up." },
  { id: "goals", title: "What are your goals?", subtitle: "We'll suggest the right features." },
  { id: "integrations", title: "Connect your tools", subtitle: "Import your existing data." },
];

export function WelcomeWizard() {
  const router = useRouter();
  const [step, setStep] = useState(0);
  const [answers, setAnswers] = useState<Record<string, unknown>>({});

  function updateAnswer(key: string, value: unknown) {
    setAnswers((prev) => ({ ...prev, [key]: value }));
  }

  async function handleComplete() {
    await fetch("/api/onboarding/wizard", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(answers),
    });
    router.push("/dashboard");
  }

  function handleSkip() {
    fetch("/api/onboarding/wizard", {
      method: "POST",
      body: JSON.stringify({ ...answers, skippedAtStep: step }),
    });
    router.push("/dashboard");
  }

  const currentStep = wizardSteps[step];

  return (
    <div className="flex min-h-screen items-center justify-center bg-gray-50">
      <div className="w-full max-w-lg">
        {/* Progress indicator */}
        <div className="mb-8 flex gap-2">
          {wizardSteps.map((_, i) => (
            <div
              key={i}
              className={`h-1 flex-1 rounded-full transition-colors ${
                i <= step ? "bg-indigo-600" : "bg-gray-200"
              }`}
            />
          ))}
        </div>

        <AnimatePresence mode="wait">
          <motion.div
            key={step}
            initial={{ opacity: 0, x: 20 }}
            animate={{ opacity: 1, x: 0 }}
            exit={{ opacity: 0, x: -20 }}
            className="rounded-xl bg-white p-8 shadow-sm"
          >
            <h2 className="text-xl font-bold text-gray-900">
              {currentStep.title}
            </h2>
            <p className="mt-1 text-sm text-gray-500">
              {currentStep.subtitle}
            </p>

            <div className="mt-6">
              {currentStep.id === "role" && (
                <RoleSelector
                  value={answers.role as string}
                  onChange={(v) => updateAnswer("role", v)}
                />
              )}
              {currentStep.id === "company" && (
                <CompanyForm
                  value={answers.company as Record<string, string>}
                  onChange={(v) => updateAnswer("company", v)}
                />
              )}
              {currentStep.id === "goals" && (
                <GoalsPicker
                  value={answers.goals as string[]}
                  onChange={(v) => updateAnswer("goals", v)}
                />
              )}
              {currentStep.id === "integrations" && (
                <IntegrationConnector
                  connected={answers.integrations as string[]}
                  onChange={(v) => updateAnswer("integrations", v)}
                />
              )}
            </div>

            {/* Navigation */}
            <div className="mt-8 flex items-center justify-between">
              <button
                onClick={handleSkip}
                className="text-sm text-gray-400 hover:text-gray-600"
              >
                Skip setup
              </button>
              <div className="flex gap-3">
                {step > 0 && (
                  <button
                    onClick={() => setStep((s) => s - 1)}
                    className="rounded-lg border border-gray-200 px-4 py-2 text-sm text-gray-700 hover:bg-gray-50"
                  >
                    Back
                  </button>
                )}
                <button
                  onClick={() => {
                    if (step < wizardSteps.length - 1) {
                      setStep((s) => s + 1);
                    } else {
                      handleComplete();
                    }
                  }}
                  className="rounded-lg bg-indigo-600 px-4 py-2 text-sm text-white hover:bg-indigo-700"
                >
                  {step < wizardSteps.length - 1 ? "Continue" : "Get Started"}
                </button>
              </div>
            </div>
          </motion.div>
        </AnimatePresence>
      </div>
    </div>
  );
}

// --- Sub-components ---

function RoleSelector({
  value,
  onChange,
}: {
  value?: string;
  onChange: (v: string) => void;
}) {
  const roles = [
    { id: "founder", label: "Founder / CEO" },
    { id: "marketer", label: "Marketing Manager" },
    { id: "developer", label: "Developer" },
    { id: "agency", label: "Agency" },
  ];

  return (
    <div className="grid grid-cols-2 gap-3">
      {roles.map((role) => (
        <button
          key={role.id}
          onClick={() => onChange(role.id)}
          className={`rounded-lg border-2 p-4 text-left transition-colors ${
            value === role.id
              ? "border-indigo-600 bg-indigo-50"
              : "border-gray-200 hover:border-gray-300"
          }`}
        >
          <p className="text-sm font-medium text-gray-900">{role.label}</p>
        </button>
      ))}
    </div>
  );
}

function CompanyForm({
  value,
  onChange,
}: {
  value?: Record<string, string>;
  onChange: (v: Record<string, string>) => void;
}) {
  const data = value ?? { name: "", size: "", industry: "" };

  return (
    <div className="space-y-4">
      <input
        type="text"
        placeholder="Company name"
        value={data.name}
        onChange={(e) => onChange({ ...data, name: e.target.value })}
        className="w-full rounded-lg border border-gray-300 px-3 py-2 text-sm"
      />
      <select
        value={data.size}
        onChange={(e) => onChange({ ...data, size: e.target.value })}
        className="w-full rounded-lg border border-gray-300 px-3 py-2 text-sm"
      >
        <option value="">Company size</option>
        <option value="1-10">1-10</option>
        <option value="11-50">11-50</option>
        <option value="51-200">51-200</option>
        <option value="200+">200+</option>
      </select>
      <select
        value={data.industry}
        onChange={(e) => onChange({ ...data, industry: e.target.value })}
        className="w-full rounded-lg border border-gray-300 px-3 py-2 text-sm"
      >
        <option value="">Industry</option>
        <option value="saas">SaaS</option>
        <option value="ecommerce">E-Commerce</option>
        <option value="agency">Agency</option>
        <option value="other">Other</option>
      </select>
    </div>
  );
}

function GoalsPicker({
  value,
  onChange,
}: {
  value?: string[];
  onChange: (v: string[]) => void;
}) {
  const goals = [
    "Track marketing KPIs",
    "Automate reporting",
    "Reduce manual data work",
    "Improve ROI visibility",
    "Collaborate with team",
  ];
  const selected = value ?? [];

  function toggle(goal: string) {
    onChange(
      selected.includes(goal)
        ? selected.filter((g) => g !== goal)
        : [...selected, goal]
    );
  }

  return (
    <div className="space-y-2">
      {goals.map((goal) => (
        <label
          key={goal}
          className="flex cursor-pointer items-center gap-3 rounded-lg border border-gray-200 p-3 hover:bg-gray-50"
        >
          <input
            type="checkbox"
            checked={selected.includes(goal)}
            onChange={() => toggle(goal)}
            className="h-4 w-4 rounded border-gray-300 text-indigo-600"
          />
          <span className="text-sm text-gray-700">{goal}</span>
        </label>
      ))}
    </div>
  );
}

function IntegrationConnector({
  connected,
  onChange,
}: {
  connected?: string[];
  onChange: (v: string[]) => void;
}) {
  const integrations = [
    { id: "google_ads", name: "Google Ads" },
    { id: "meta_ads", name: "Meta Ads" },
    { id: "google_analytics", name: "Google Analytics" },
  ];
  const list = connected ?? [];

  return (
    <div className="space-y-3">
      {integrations.map((int) => (
        <div
          key={int.id}
          className="flex items-center justify-between rounded-lg border border-gray-200 p-3"
        >
          <span className="text-sm font-medium text-gray-900">{int.name}</span>
          {list.includes(int.id) ? (
            <span className="text-xs font-medium text-green-600">Connected</span>
          ) : (
            <button
              onClick={() => onChange([...list, int.id])}
              className="rounded-md bg-gray-100 px-3 py-1 text-xs font-medium text-gray-700 hover:bg-gray-200"
            >
              Connect
            </button>
          )}
        </div>
      ))}
      <p className="text-xs text-gray-400">You can add more integrations later.</p>
    </div>
  );
}
```

---

## 5. Sample Data

Seed demo data for new users so they see a populated interface immediately.

### Seed Function

```tsx
// lib/seed-sample-data.ts

import { SupabaseClient } from "@supabase/supabase-js";

const SAMPLE_CAMPAIGNS = [
  {
    name: "Summer Sale 2025",
    platform: "meta_ads",
    spend: 1250.0,
    impressions: 145000,
    clicks: 3200,
    conversions: 85,
    status: "active",
    is_sample: true,
  },
  {
    name: "Brand Awareness Q3",
    platform: "google_ads",
    spend: 890.0,
    impressions: 230000,
    clicks: 4100,
    conversions: 42,
    status: "active",
    is_sample: true,
  },
  {
    name: "Retargeting - Cart Abandoners",
    platform: "meta_ads",
    spend: 340.0,
    impressions: 45000,
    clicks: 1800,
    conversions: 63,
    status: "paused",
    is_sample: true,
  },
];

export async function seedSampleData(
  supabase: SupabaseClient,
  userId: string
) {
  // Check if already seeded
  const { count } = await supabase
    .from("campaigns")
    .select("*", { count: "exact", head: true })
    .eq("user_id", userId)
    .eq("is_sample", true);

  if (count && count > 0) return;

  // Insert sample campaigns
  const campaigns = SAMPLE_CAMPAIGNS.map((c) => ({
    ...c,
    user_id: userId,
    created_at: new Date().toISOString(),
  }));

  await supabase.from("campaigns").insert(campaigns);

  // Insert sample daily metrics for the last 30 days
  const { data: inserted } = await supabase
    .from("campaigns")
    .select("id")
    .eq("user_id", userId)
    .eq("is_sample", true);

  if (!inserted) return;

  const metrics = inserted.flatMap((campaign) =>
    Array.from({ length: 30 }, (_, i) => {
      const date = new Date();
      date.setDate(date.getDate() - i);
      return {
        campaign_id: campaign.id,
        date: date.toISOString().split("T")[0],
        spend: Math.round(Math.random() * 100 + 20),
        impressions: Math.round(Math.random() * 8000 + 2000),
        clicks: Math.round(Math.random() * 200 + 50),
        conversions: Math.round(Math.random() * 10 + 1),
        is_sample: true,
      };
    })
  );

  await supabase.from("campaign_metrics").insert(metrics);
}
```

### Sample Data Banner

```tsx
"use client";

import { useState } from "react";
import { Info, X } from "lucide-react";

export function SampleDataBanner({
  onCleanup,
}: {
  onCleanup: () => Promise<void>;
}) {
  const [visible, setVisible] = useState(true);
  const [cleaning, setCleaning] = useState(false);

  if (!visible) return null;

  async function handleCleanup() {
    setCleaning(true);
    await onCleanup();
    setVisible(false);
  }

  return (
    <div className="flex items-center justify-between rounded-lg border border-amber-200 bg-amber-50 px-4 py-3">
      <div className="flex items-center gap-3">
        <Info className="h-4 w-4 text-amber-600" />
        <p className="text-sm text-amber-800">
          You're viewing <strong>sample data</strong>. Connect your own data
          source or{" "}
          <button
            onClick={handleCleanup}
            disabled={cleaning}
            className="font-medium underline hover:no-underline"
          >
            {cleaning ? "Removing..." : "remove sample data"}
          </button>
          .
        </p>
      </div>
      <button
        onClick={() => setVisible(false)}
        className="text-amber-400 hover:text-amber-600"
      >
        <X className="h-4 w-4" />
      </button>
    </div>
  );
}
```

### Cleanup API

```tsx
// app/api/onboarding/cleanup-sample-data/route.ts
import { createClient } from "@/lib/supabase/server";
import { NextResponse } from "next/server";

export async function POST() {
  const supabase = await createClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  // Delete sample metrics first (FK constraint)
  await supabase
    .from("campaign_metrics")
    .delete()
    .eq("is_sample", true)
    .in(
      "campaign_id",
      (
        await supabase
          .from("campaigns")
          .select("id")
          .eq("user_id", user.id)
          .eq("is_sample", true)
      ).data?.map((c) => c.id) ?? []
    );

  // Delete sample campaigns
  await supabase
    .from("campaigns")
    .delete()
    .eq("user_id", user.id)
    .eq("is_sample", true);

  return NextResponse.json({ ok: true });
}
```

---

## 6. Empty State CTAs

Guide users from an empty dashboard to their first meaningful action.

```tsx
import { Plus, Upload, BookOpen, PlayCircle } from "lucide-react";

interface EmptyStateProps {
  title: string;
  description: string;
  icon?: React.ReactNode;
  primaryAction: { label: string; onClick: () => void };
  secondaryAction?: { label: string; onClick: () => void };
  showTutorial?: boolean;
}

export function EmptyState({
  title,
  description,
  icon,
  primaryAction,
  secondaryAction,
  showTutorial,
}: EmptyStateProps) {
  return (
    <div className="flex flex-col items-center justify-center rounded-xl border-2 border-dashed border-gray-200 py-16">
      {icon && (
        <div className="mb-4 rounded-full bg-indigo-50 p-4 text-indigo-600">
          {icon}
        </div>
      )}
      <h3 className="text-lg font-semibold text-gray-900">{title}</h3>
      <p className="mt-1 max-w-sm text-center text-sm text-gray-500">
        {description}
      </p>

      <div className="mt-6 flex gap-3">
        <button
          onClick={primaryAction.onClick}
          className="flex items-center gap-2 rounded-lg bg-indigo-600 px-4 py-2 text-sm font-medium text-white hover:bg-indigo-700"
        >
          <Plus className="h-4 w-4" />
          {primaryAction.label}
        </button>
        {secondaryAction && (
          <button
            onClick={secondaryAction.onClick}
            className="flex items-center gap-2 rounded-lg border border-gray-200 px-4 py-2 text-sm font-medium text-gray-700 hover:bg-gray-50"
          >
            <Upload className="h-4 w-4" />
            {secondaryAction.label}
          </button>
        )}
      </div>

      {showTutorial && (
        <div className="mt-8 flex gap-4">
          <button className="flex items-center gap-2 text-sm text-gray-500 hover:text-gray-700">
            <PlayCircle className="h-4 w-4" />
            Watch tutorial (2 min)
          </button>
          <button className="flex items-center gap-2 text-sm text-gray-500 hover:text-gray-700">
            <BookOpen className="h-4 w-4" />
            Read the docs
          </button>
        </div>
      )}
    </div>
  );
}

// --- Usage ---

export function CampaignsDashboard({ campaigns }: { campaigns: unknown[] }) {
  if (campaigns.length === 0) {
    return (
      <EmptyState
        icon={<Plus className="h-8 w-8" />}
        title="No campaigns yet"
        description="Create your first campaign to start tracking performance across your marketing channels."
        primaryAction={{
          label: "Create Campaign",
          onClick: () => (window.location.href = "/campaigns/new"),
        }}
        secondaryAction={{
          label: "Import from CSV",
          onClick: () => (window.location.href = "/campaigns/import"),
        }}
        showTutorial
      />
    );
  }

  return <div>{/* Render campaign list */}</div>;
}

// Inline tutorial empty state with step-by-step guidance
export function InlineTutorialEmptyState() {
  return (
    <div className="rounded-xl border border-gray-200 bg-white p-8">
      <h3 className="text-lg font-semibold text-gray-900">
        Get started in 3 steps
      </h3>
      <div className="mt-6 space-y-4">
        {[
          {
            step: 1,
            title: "Connect your ad platform",
            description: "Link Google Ads or Meta Ads.",
            action: "Connect",
            href: "/integrations",
          },
          {
            step: 2,
            title: "Create a dashboard",
            description: "Choose metrics and build your view.",
            action: "Create",
            href: "/dashboards/new",
          },
          {
            step: 3,
            title: "Share with your team",
            description: "Invite colleagues to collaborate.",
            action: "Invite",
            href: "/settings/team",
          },
        ].map((item) => (
          <div
            key={item.step}
            className="flex items-center gap-4 rounded-lg border border-gray-100 p-4"
          >
            <div className="flex h-8 w-8 shrink-0 items-center justify-center rounded-full bg-indigo-100 text-sm font-bold text-indigo-600">
              {item.step}
            </div>
            <div className="flex-1">
              <p className="text-sm font-medium text-gray-900">{item.title}</p>
              <p className="text-xs text-gray-500">{item.description}</p>
            </div>
            <a
              href={item.href}
              className="rounded-md bg-indigo-50 px-3 py-1.5 text-xs font-medium text-indigo-600 hover:bg-indigo-100"
            >
              {item.action}
            </a>
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## 7. Progressive Disclosure

Show features gradually as users become more experienced. Unlock advanced features over time.

```tsx
"use client";

import { useState, createContext, useContext, useEffect } from "react";
import { Lock, Sparkles } from "lucide-react";

// --- Feature gating system ---

type FeatureTier = "beginner" | "intermediate" | "advanced";

interface FeatureGate {
  id: string;
  tier: FeatureTier;
  requiredActions: string[];
  label: string;
}

const FEATURE_GATES: FeatureGate[] = [
  {
    id: "custom_reports",
    tier: "intermediate",
    requiredActions: ["created_first_project", "viewed_first_report"],
    label: "Custom Reports",
  },
  {
    id: "api_access",
    tier: "advanced",
    requiredActions: [
      "created_first_project",
      "connected_data_source",
      "created_custom_report",
    ],
    label: "API Access",
  },
  {
    id: "automations",
    tier: "advanced",
    requiredActions: [
      "created_first_project",
      "connected_data_source",
      "invited_team_member",
    ],
    label: "Automations",
  },
];

interface DisclosureContextType {
  completedActions: Set<string>;
  isFeatureUnlocked: (featureId: string) => boolean;
  getUnlockProgress: (featureId: string) => { done: number; total: number };
}

const DisclosureContext = createContext<DisclosureContextType | null>(null);

export function DisclosureProvider({ children }: { children: React.ReactNode }) {
  const [completedActions, setCompletedActions] = useState<Set<string>>(
    new Set()
  );

  useEffect(() => {
    fetch("/api/onboarding/actions")
      .then((r) => r.json())
      .then((data) => setCompletedActions(new Set(data.actions)));
  }, []);

  function isFeatureUnlocked(featureId: string) {
    const gate = FEATURE_GATES.find((g) => g.id === featureId);
    if (!gate) return true;
    return gate.requiredActions.every((a) => completedActions.has(a));
  }

  function getUnlockProgress(featureId: string) {
    const gate = FEATURE_GATES.find((g) => g.id === featureId);
    if (!gate) return { done: 1, total: 1 };
    const done = gate.requiredActions.filter((a) =>
      completedActions.has(a)
    ).length;
    return { done, total: gate.requiredActions.length };
  }

  return (
    <DisclosureContext.Provider
      value={{ completedActions, isFeatureUnlocked, getUnlockProgress }}
    >
      {children}
    </DisclosureContext.Provider>
  );
}

export function useDisclosure() {
  const ctx = useContext(DisclosureContext);
  if (!ctx) throw new Error("useDisclosure must be inside DisclosureProvider");
  return ctx;
}

// --- Gated feature wrapper ---

export function GatedFeature({
  featureId,
  children,
  fallback,
}: {
  featureId: string;
  children: React.ReactNode;
  fallback?: React.ReactNode;
}) {
  const { isFeatureUnlocked, getUnlockProgress } = useDisclosure();
  const unlocked = isFeatureUnlocked(featureId);
  const progress = getUnlockProgress(featureId);

  if (unlocked) return <>{children}</>;

  if (fallback) return <>{fallback}</>;

  return (
    <div className="relative rounded-lg border border-gray-200 bg-gray-50 p-4">
      <div className="flex items-center gap-3">
        <Lock className="h-5 w-5 text-gray-400" />
        <div>
          <p className="text-sm font-medium text-gray-700">Feature locked</p>
          <p className="text-xs text-gray-500">
            Complete {progress.total - progress.done} more action
            {progress.total - progress.done > 1 ? "s" : ""} to unlock.
          </p>
        </div>
      </div>
      <div className="mt-2 h-1.5 overflow-hidden rounded-full bg-gray-200">
        <div
          className="h-full rounded-full bg-indigo-500"
          style={{ width: `${(progress.done / progress.total) * 100}%` }}
        />
      </div>
    </div>
  );
}

// --- Feature hint tooltip ---

export function FeatureHint({
  children,
  hint,
  showOnce = true,
}: {
  children: React.ReactNode;
  hint: string;
  showOnce?: boolean;
}) {
  const [visible, setVisible] = useState(false);
  const storageKey = `hint_seen_${hint.slice(0, 20)}`;

  useEffect(() => {
    if (showOnce && localStorage.getItem(storageKey)) return;
    const timer = setTimeout(() => setVisible(true), 1000);
    return () => clearTimeout(timer);
  }, [showOnce, storageKey]);

  function dismiss() {
    setVisible(false);
    if (showOnce) localStorage.setItem(storageKey, "true");
  }

  return (
    <div className="relative">
      {children}
      {visible && (
        <div className="absolute -top-2 left-full z-50 ml-2 w-56 rounded-lg border border-indigo-200 bg-indigo-50 p-3 shadow-sm">
          <div className="flex items-start gap-2">
            <Sparkles className="mt-0.5 h-4 w-4 shrink-0 text-indigo-600" />
            <p className="text-xs text-indigo-700">{hint}</p>
          </div>
          <button
            onClick={dismiss}
            className="mt-2 text-xs font-medium text-indigo-600 hover:text-indigo-700"
          >
            Got it
          </button>
        </div>
      )}
    </div>
  );
}

// --- Usage ---

export function DashboardSidebar() {
  return (
    <nav className="space-y-1">
      <a href="/dashboard" className="block rounded-md px-3 py-2 text-sm">
        Dashboard
      </a>
      <a href="/campaigns" className="block rounded-md px-3 py-2 text-sm">
        Campaigns
      </a>

      <GatedFeature featureId="custom_reports">
        <FeatureHint hint="Build custom reports by dragging metrics. Try it!">
          <a href="/reports" className="block rounded-md px-3 py-2 text-sm">
            Custom Reports
          </a>
        </FeatureHint>
      </GatedFeature>

      <GatedFeature featureId="api_access">
        <a href="/api-keys" className="block rounded-md px-3 py-2 text-sm">
          API Access
        </a>
      </GatedFeature>

      <GatedFeature featureId="automations">
        <a href="/automations" className="block rounded-md px-3 py-2 text-sm">
          Automations
        </a>
      </GatedFeature>
    </nav>
  );
}
```

---

## 8. Email Onboarding Drip

Structured email sequence for the first 14 days after signup.

### Drip Configuration and Scheduler

```tsx
// lib/email-drip.ts

interface DripEmail {
  day: number;
  templateId: string;
  subject: string;
  purpose: string;
  condition?: (userState: UserOnboardingState) => boolean;
}

interface UserOnboardingState {
  signedUpAt: string;
  completedSteps: string[];
  lastActiveAt: string;
  emailsSent: string[];
}

export const DRIP_SEQUENCE: DripEmail[] = [
  {
    day: 0,
    templateId: "welcome",
    subject: "Welcome to {{appName}} -- here's how to get started",
    purpose: "Welcome, set expectations, link to first action",
  },
  {
    day: 1,
    templateId: "quick_win",
    subject: "Your first report is 2 minutes away",
    purpose: "Drive first value moment",
    condition: (state) => !state.completedSteps.includes("viewed_first_report"),
  },
  {
    day: 3,
    templateId: "feature_highlight",
    subject: "Did you know you can connect Google Ads?",
    purpose: "Highlight key integration",
    condition: (state) => !state.completedSteps.includes("connected_data_source"),
  },
  {
    day: 5,
    templateId: "social_proof",
    subject: "How {{companyName}} increased ROAS by 40%",
    purpose: "Case study / social proof",
  },
  {
    day: 7,
    templateId: "checklist_reminder",
    subject: "You're 60% done with setup -- finish it today",
    purpose: "Nudge to complete onboarding checklist",
    condition: (state) => state.completedSteps.length < 4,
  },
  {
    day: 10,
    templateId: "advanced_feature",
    subject: "Unlock automations: save 5 hours/week",
    purpose: "Introduce advanced feature",
    condition: (state) => state.completedSteps.length >= 3,
  },
  {
    day: 14,
    templateId: "re_engage_or_celebrate",
    subject: "Your 2-week milestone -- here's what you've achieved",
    purpose: "Re-engagement or celebration based on activity",
  },
];

export function getNextDripEmail(
  state: UserOnboardingState
): DripEmail | null {
  const daysSinceSignup = Math.floor(
    (Date.now() - new Date(state.signedUpAt).getTime()) / (1000 * 60 * 60 * 24)
  );

  for (const email of DRIP_SEQUENCE) {
    if (email.day > daysSinceSignup) continue;
    if (state.emailsSent.includes(email.templateId)) continue;
    if (email.condition && !email.condition(state)) continue;
    return email;
  }

  return null;
}
```

### Cron-based Email Sender

```tsx
// app/api/cron/drip-emails/route.ts
import { createClient } from "@supabase/supabase-js";
import { getNextDripEmail } from "@/lib/email-drip";
import { NextResponse } from "next/server";

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function GET(req: Request) {
  // Verify cron secret
  const authHeader = req.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  // Get all users who signed up in the last 14 days
  const fourteenDaysAgo = new Date();
  fourteenDaysAgo.setDate(fourteenDaysAgo.getDate() - 14);

  const { data: users } = await supabase
    .from("onboarding_progress")
    .select("user_id, completed_steps, created_at, emails_sent")
    .gte("created_at", fourteenDaysAgo.toISOString());

  if (!users) return NextResponse.json({ sent: 0 });

  let sentCount = 0;

  for (const user of users) {
    const state = {
      signedUpAt: user.created_at,
      completedSteps: user.completed_steps ?? [],
      lastActiveAt: user.created_at,
      emailsSent: user.emails_sent ?? [],
    };

    const nextEmail = getNextDripEmail(state);
    if (!nextEmail) continue;

    // Get user email
    const {
      data: { user: authUser },
    } = await supabase.auth.admin.getUserById(user.user_id);
    if (!authUser?.email) continue;

    // Send via email provider (Resend, Postmark, etc.)
    await fetch("https://api.resend.com/emails", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.RESEND_API_KEY}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        from: "onboarding@yourapp.com",
        to: authUser.email,
        subject: nextEmail.subject,
        html: `<p>Email template: ${nextEmail.templateId}</p>`,
      }),
    });

    // Record sent email
    await supabase
      .from("onboarding_progress")
      .update({
        emails_sent: [...(user.emails_sent ?? []), nextEmail.templateId],
      })
      .eq("user_id", user.user_id);

    sentCount++;
  }

  return NextResponse.json({ sent: sentCount });
}
```

---

## 9. Help System

Contextual tooltips, help center integration, and chat widget.

### Contextual Tooltip

```tsx
"use client";

import { useState } from "react";
import { HelpCircle, X } from "lucide-react";
import { AnimatePresence, motion } from "framer-motion";

export function ContextualHelp({
  children,
  helpText,
  learnMoreUrl,
}: {
  children: React.ReactNode;
  helpText: string;
  learnMoreUrl?: string;
}) {
  const [open, setOpen] = useState(false);

  return (
    <div className="relative inline-flex items-center gap-1">
      {children}
      <button
        onClick={() => setOpen(!open)}
        className="text-gray-400 hover:text-gray-600"
      >
        <HelpCircle className="h-4 w-4" />
      </button>

      <AnimatePresence>
        {open && (
          <motion.div
            initial={{ opacity: 0, y: 4 }}
            animate={{ opacity: 1, y: 0 }}
            exit={{ opacity: 0, y: 4 }}
            className="absolute left-0 top-full z-50 mt-2 w-64 rounded-lg border border-gray-200 bg-white p-3 shadow-lg"
          >
            <div className="flex items-start justify-between gap-2">
              <p className="text-xs text-gray-600">{helpText}</p>
              <button
                onClick={() => setOpen(false)}
                className="shrink-0 text-gray-400 hover:text-gray-600"
              >
                <X className="h-3 w-3" />
              </button>
            </div>
            {learnMoreUrl && (
              <a
                href={learnMoreUrl}
                target="_blank"
                rel="noopener noreferrer"
                className="mt-2 inline-block text-xs font-medium text-indigo-600 hover:text-indigo-700"
              >
                Learn more &rarr;
              </a>
            )}
          </motion.div>
        )}
      </AnimatePresence>
    </div>
  );
}
```

### Slide-in Help Panel

```tsx
"use client";

import { useState } from "react";
import { HelpCircle, X, MessageCircle, Book, Search } from "lucide-react";
import { AnimatePresence, motion } from "framer-motion";

interface HelpArticle {
  id: string;
  title: string;
  summary: string;
  url: string;
  category: string;
}

const HELP_ARTICLES: HelpArticle[] = [
  {
    id: "1",
    title: "Getting started with campaigns",
    summary: "Learn how to create and manage your first campaign.",
    url: "/docs/campaigns",
    category: "Getting Started",
  },
  {
    id: "2",
    title: "Connecting data sources",
    summary: "Link Google Ads, Meta Ads, and more.",
    url: "/docs/integrations",
    category: "Integrations",
  },
  {
    id: "3",
    title: "Understanding your dashboard",
    summary: "Navigate metrics, charts, and filters.",
    url: "/docs/dashboard",
    category: "Getting Started",
  },
];

export function HelpPanel() {
  const [open, setOpen] = useState(false);
  const [search, setSearch] = useState("");

  const filtered = HELP_ARTICLES.filter(
    (a) =>
      a.title.toLowerCase().includes(search.toLowerCase()) ||
      a.summary.toLowerCase().includes(search.toLowerCase())
  );

  return (
    <>
      {/* Floating trigger */}
      <button
        onClick={() => setOpen(true)}
        className="fixed bottom-6 right-6 z-40 flex h-12 w-12 items-center justify-center rounded-full bg-indigo-600 text-white shadow-lg hover:bg-indigo-700"
      >
        <HelpCircle className="h-6 w-6" />
      </button>

      {/* Slide-in panel */}
      <AnimatePresence>
        {open && (
          <>
            <motion.div
              initial={{ opacity: 0 }}
              animate={{ opacity: 0.3 }}
              exit={{ opacity: 0 }}
              className="fixed inset-0 z-40 bg-black"
              onClick={() => setOpen(false)}
            />
            <motion.div
              initial={{ x: "100%" }}
              animate={{ x: 0 }}
              exit={{ x: "100%" }}
              transition={{ type: "spring", damping: 25, stiffness: 200 }}
              className="fixed right-0 top-0 z-50 h-full w-96 overflow-y-auto bg-white shadow-xl"
            >
              <div className="sticky top-0 border-b border-gray-200 bg-white p-4">
                <div className="flex items-center justify-between">
                  <h2 className="text-lg font-semibold">Help Center</h2>
                  <button
                    onClick={() => setOpen(false)}
                    className="text-gray-400 hover:text-gray-600"
                  >
                    <X className="h-5 w-5" />
                  </button>
                </div>

                <div className="relative mt-3">
                  <Search className="absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-gray-400" />
                  <input
                    type="text"
                    placeholder="Search help articles..."
                    value={search}
                    onChange={(e) => setSearch(e.target.value)}
                    className="w-full rounded-lg border border-gray-200 py-2 pl-9 pr-3 text-sm"
                  />
                </div>
              </div>

              <div className="p-4">
                {/* Quick actions */}
                <div className="mb-6 grid grid-cols-2 gap-2">
                  <button className="flex flex-col items-center gap-2 rounded-lg border border-gray-200 p-3 text-center hover:bg-gray-50">
                    <Book className="h-5 w-5 text-indigo-600" />
                    <span className="text-xs font-medium">Docs</span>
                  </button>
                  <button
                    onClick={() => {
                      // Open chat widget (Intercom, Crisp, etc.)
                      if (typeof window !== "undefined" && (window as any).Intercom) {
                        (window as any).Intercom("show");
                      }
                    }}
                    className="flex flex-col items-center gap-2 rounded-lg border border-gray-200 p-3 text-center hover:bg-gray-50"
                  >
                    <MessageCircle className="h-5 w-5 text-indigo-600" />
                    <span className="text-xs font-medium">Live Chat</span>
                  </button>
                </div>

                {/* Articles */}
                <div className="space-y-3">
                  {filtered.map((article) => (
                    <a
                      key={article.id}
                      href={article.url}
                      className="block rounded-lg border border-gray-100 p-3 hover:bg-gray-50"
                    >
                      <span className="text-xs text-indigo-600">
                        {article.category}
                      </span>
                      <p className="text-sm font-medium text-gray-900">
                        {article.title}
                      </p>
                      <p className="mt-0.5 text-xs text-gray-500">
                        {article.summary}
                      </p>
                    </a>
                  ))}
                </div>
              </div>
            </motion.div>
          </>
        )}
      </AnimatePresence>
    </>
  );
}
```

### Chat Widget Integration

For Intercom or Crisp, load the widget script in your root layout. Pass the `appId` from environment variables (never hardcoded) and user data from your auth context. Both providers offer React SDKs as well:

- **Intercom**: `@intercom/messenger-js-sdk`
- **Crisp**: `crisp-sdk-web`

```tsx
// Example: Intercom via their React SDK
import Intercom from "@intercom/messenger-js-sdk";

export function IntercomProvider({ appId, user }: { appId: string; user?: { name: string; email: string; id: string } }) {
  Intercom({
    app_id: appId,
    user_id: user?.id,
    name: user?.name,
    email: user?.email,
  });

  return null;
}
```

---

## 10. Measuring Success

Track onboarding completion rate, time to first value, drop-off analysis, and A/B testing.

### Analytics Dashboard Component

```tsx
"use client";

import { useEffect, useState } from "react";

interface OnboardingMetrics {
  totalSignups: number;
  completionRate: number;
  avgTimeToFirstValue: number; // hours
  avgTimeToComplete: number; // hours
  dropOffByStep: { step: string; dropOff: number }[];
  variantPerformance: {
    variant: string;
    completionRate: number;
    avgTTV: number;
    sampleSize: number;
  }[];
}

export function OnboardingAnalytics() {
  const [metrics, setMetrics] = useState<OnboardingMetrics | null>(null);
  const [period, setPeriod] = useState<"7d" | "30d" | "90d">("30d");

  useEffect(() => {
    fetch(`/api/admin/onboarding-metrics?period=${period}`)
      .then((r) => r.json())
      .then(setMetrics);
  }, [period]);

  if (!metrics) return <div>Loading...</div>;

  return (
    <div className="space-y-8">
      {/* Period selector */}
      <div className="flex gap-2">
        {(["7d", "30d", "90d"] as const).map((p) => (
          <button
            key={p}
            onClick={() => setPeriod(p)}
            className={`rounded-md px-3 py-1.5 text-sm ${
              period === p
                ? "bg-indigo-600 text-white"
                : "bg-gray-100 text-gray-700"
            }`}
          >
            {p}
          </button>
        ))}
      </div>

      {/* KPI cards */}
      <div className="grid grid-cols-4 gap-4">
        <MetricCard
          label="Completion Rate"
          value={`${metrics.completionRate.toFixed(1)}%`}
          trend={metrics.completionRate > 60 ? "good" : "bad"}
        />
        <MetricCard
          label="Avg Time to First Value"
          value={`${metrics.avgTimeToFirstValue.toFixed(1)}h`}
          trend={metrics.avgTimeToFirstValue < 2 ? "good" : "bad"}
        />
        <MetricCard
          label="Avg Time to Complete"
          value={`${metrics.avgTimeToComplete.toFixed(1)}h`}
          trend={metrics.avgTimeToComplete < 24 ? "good" : "bad"}
        />
        <MetricCard
          label="Total Signups"
          value={metrics.totalSignups.toString()}
        />
      </div>

      {/* Drop-off analysis */}
      <div>
        <h3 className="mb-4 text-lg font-semibold">Drop-off by Step</h3>
        <div className="space-y-2">
          {metrics.dropOffByStep.map((step) => (
            <div key={step.step} className="flex items-center gap-4">
              <span className="w-48 text-sm text-gray-700">{step.step}</span>
              <div className="flex-1">
                <div className="h-6 overflow-hidden rounded bg-gray-100">
                  <div
                    className={`h-full rounded ${
                      step.dropOff > 30
                        ? "bg-red-400"
                        : step.dropOff > 15
                          ? "bg-amber-400"
                          : "bg-green-400"
                    }`}
                    style={{ width: `${step.dropOff}%` }}
                  />
                </div>
              </div>
              <span className="w-16 text-right text-sm text-gray-500">
                {step.dropOff}%
              </span>
            </div>
          ))}
        </div>
      </div>

      {/* A/B test results */}
      {metrics.variantPerformance.length > 1 && (
        <div>
          <h3 className="mb-4 text-lg font-semibold">A/B Test Results</h3>
          <table className="w-full text-sm">
            <thead>
              <tr className="border-b border-gray-200 text-left text-gray-500">
                <th className="pb-2">Variant</th>
                <th className="pb-2">Completion Rate</th>
                <th className="pb-2">Avg TTV</th>
                <th className="pb-2">Sample Size</th>
                <th className="pb-2">Status</th>
              </tr>
            </thead>
            <tbody>
              {metrics.variantPerformance.map((v) => {
                const best = metrics.variantPerformance.reduce((a, b) =>
                  a.completionRate > b.completionRate ? a : b
                );
                return (
                  <tr key={v.variant} className="border-b border-gray-100">
                    <td className="py-3 font-medium">{v.variant}</td>
                    <td className="py-3">{v.completionRate.toFixed(1)}%</td>
                    <td className="py-3">{v.avgTTV.toFixed(1)}h</td>
                    <td className="py-3">{v.sampleSize}</td>
                    <td className="py-3">
                      {v.variant === best.variant && (
                        <span className="rounded-full bg-green-100 px-2 py-0.5 text-xs font-medium text-green-700">
                          Leading
                        </span>
                      )}
                    </td>
                  </tr>
                );
              })}
            </tbody>
          </table>
        </div>
      )}
    </div>
  );
}

function MetricCard({
  label,
  value,
  trend,
}: {
  label: string;
  value: string;
  trend?: "good" | "bad";
}) {
  return (
    <div className="rounded-lg border border-gray-200 bg-white p-4">
      <p className="text-sm text-gray-500">{label}</p>
      <p
        className={`mt-1 text-2xl font-bold ${
          trend === "good"
            ? "text-green-600"
            : trend === "bad"
              ? "text-red-600"
              : "text-gray-900"
        }`}
      >
        {value}
      </p>
    </div>
  );
}
```

### A/B Testing Assignment

```tsx
// lib/ab-test.ts

import { createClient } from "@/lib/supabase/server";

export type OnboardingVariant = "control" | "wizard_v2" | "checklist_first";

export async function assignOnboardingVariant(
  userId: string
): Promise<OnboardingVariant> {
  const supabase = await createClient();

  // Check existing assignment
  const { data: existing } = await supabase
    .from("ab_test_assignments")
    .select("variant")
    .eq("user_id", userId)
    .eq("experiment", "onboarding_flow_v2")
    .single();

  if (existing) return existing.variant as OnboardingVariant;

  // Assign based on deterministic hash of userId
  const variants: OnboardingVariant[] = [
    "control",
    "wizard_v2",
    "checklist_first",
  ];
  const hash = simpleHash(userId);
  const variant = variants[hash % variants.length];

  await supabase.from("ab_test_assignments").insert({
    user_id: userId,
    experiment: "onboarding_flow_v2",
    variant,
  });

  return variant;
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
```

### Metrics API Route

```tsx
// app/api/admin/onboarding-metrics/route.ts
import { createClient } from "@supabase/supabase-js";
import { NextRequest, NextResponse } from "next/server";

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function GET(req: NextRequest) {
  const period = req.nextUrl.searchParams.get("period") ?? "30d";
  const days = parseInt(period) || 30;
  const since = new Date();
  since.setDate(since.getDate() - days);

  const { count: totalSignups } = await supabase
    .from("onboarding_progress")
    .select("*", { count: "exact", head: true })
    .gte("created_at", since.toISOString());

  const { count: completed } = await supabase
    .from("onboarding_progress")
    .select("*", { count: "exact", head: true })
    .gte("created_at", since.toISOString())
    .not("completed_at", "is", null);

  const completionRate =
    totalSignups && totalSignups > 0
      ? ((completed ?? 0) / totalSignups) * 100
      : 0;

  // Drop-off by step
  const steps = [
    "signed_up",
    "completed_profile",
    "created_first_project",
    "connected_data_source",
    "viewed_first_report",
  ];

  const dropOffByStep = [];
  let previousCount = totalSignups ?? 0;

  for (const step of steps) {
    const { count } = await supabase
      .from("activation_events")
      .select("*", { count: "exact", head: true })
      .eq("event_name", step)
      .gte("created_at", since.toISOString());

    const current = count ?? 0;
    const dropOff =
      previousCount > 0
        ? ((previousCount - current) / previousCount) * 100
        : 0;
    dropOffByStep.push({ step, dropOff: Math.round(dropOff) });
    previousCount = current;
  }

  // A/B test performance via RPC
  const { data: variants } = await supabase.rpc(
    "onboarding_variant_performance",
    { since_date: since.toISOString() }
  );

  return NextResponse.json({
    totalSignups: totalSignups ?? 0,
    completionRate,
    avgTimeToFirstValue: 0, // compute via RPC
    avgTimeToComplete: 0,
    dropOffByStep,
    variantPerformance: variants ?? [],
  });
}
```

### Supabase RPC Functions for Metrics

```sql
-- Average time to first value
create or replace function avg_time_to_first_value(since_date timestamptz)
returns table(avg_hours numeric, avg_complete_hours numeric)
language sql stable
as $$
  with user_times as (
    select
      user_id,
      min(case when event_name = 'signed_up' then created_at end) as signup_time,
      min(case when event_name = 'viewed_first_report' then created_at end) as first_value_time
    from activation_events
    where created_at >= since_date
    group by user_id
  )
  select
    avg(extract(epoch from (first_value_time - signup_time)) / 3600)::numeric as avg_hours,
    avg(extract(epoch from (first_value_time - signup_time)) / 3600)::numeric as avg_complete_hours
  from user_times
  where first_value_time is not null;
$$;

-- Variant performance
create or replace function onboarding_variant_performance(since_date timestamptz)
returns table(variant text, completion_rate numeric, avg_ttv numeric, sample_size bigint)
language sql stable
as $$
  select
    a.variant,
    (count(o.completed_at)::numeric / nullif(count(*)::numeric, 0) * 100) as completion_rate,
    avg(
      extract(epoch from (o.completed_at - o.created_at)) / 3600
    )::numeric as avg_ttv,
    count(*) as sample_size
  from ab_test_assignments a
  join onboarding_progress o on o.user_id = a.user_id
  where a.experiment = 'onboarding_flow_v2'
    and o.created_at >= since_date
  group by a.variant;
$$;
```

---

## Quick Reference: Key Decisions

| Decision | Recommendation |
|---|---|
| Tour library | react-joyride for speed, custom for full control |
| State persistence | Supabase with RLS, localStorage as cache |
| Email provider | Resend (simple API, React email templates) |
| Analytics | PostHog (open-source) or Mixpanel |
| Chat widget | Crisp (free tier) or Intercom (enterprise) |
| A/B testing | Custom (deterministic hash) or PostHog feature flags |
| Activation metric | "Viewed first report" -- the moment user sees value |
| Target completion rate | 60-80% within first 7 days |
| Target TTV | Under 5 minutes for initial value moment |
