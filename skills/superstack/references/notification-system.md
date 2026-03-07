# Notification System

## Table of Contents
- [1. In-App Notifications](#1-in-app-notifications)
  - [Notification Center Component](#notification-center-component)
- [2. Email Notifications](#2-email-notifications)
  - [React Email Template](#react-email-template)
  - [Sending with Resend](#sending-with-resend)
- [3. Push Notifications](#3-push-notifications)
  - [VAPID Key Generation](#vapid-key-generation)
  - [Service Worker](#service-worker)
  - [Permission Request and Subscription](#permission-request-and-subscription)
  - [Server-Side Push](#server-side-push)
  - [Permission Request UX](#permission-request-ux)
- [4. SMS Notifications](#4-sms-notifications)
  - [Twilio Integration](#twilio-integration)
  - [Usage Guidelines](#usage-guidelines)
- [5. Preference Management](#5-preference-management)
  - [Notification Settings UI](#notification-settings-ui)
- [6. Batching and Digests](#6-batching-and-digests)
  - [Notification Queue with Batching](#notification-queue-with-batching)
  - [Digest Cron Job (API Route or Edge Function)](#digest-cron-job-api-route-or-edge-function)
- [7. Database Schema](#7-database-schema)
- [8. Realtime Delivery](#8-realtime-delivery)
  - [Supabase Realtime for Instant In-App Notifications](#supabase-realtime-for-instant-in-app-notifications)
- [9. Templates](#9-templates)
  - [Notification Template System](#notification-template-system)
  - [Localization Support](#localization-support)
- [10. Webhook Notifications](#10-webhook-notifications)
  - [Webhook Dispatcher](#webhook-dispatcher)
  - [Webhook Management API](#webhook-management-api)
  - [Verifying Webhooks (Consumer Side)](#verifying-webhooks-consumer-side)
- [Unified Notification Dispatcher](#unified-notification-dispatcher)
  - [Usage](#usage)

Complete guide for building a multi-channel notification system in a Next.js + Supabase stack.

---

## 1. In-App Notifications

### Notification Center Component

```tsx
// components/notifications/notification-center.tsx
"use client";

import { useState, useEffect } from "react";
import { Bell } from "lucide-react";
import {
  Popover,
  PopoverContent,
  PopoverTrigger,
} from "@/components/ui/popover";
import { ScrollArea } from "@/components/ui/scroll-area";
import { Button } from "@/components/ui/button";
import { cn } from "@/lib/utils";
import { createClient } from "@/lib/supabase/client";

interface Notification {
  id: string;
  type: string;
  title: string;
  body: string;
  read: boolean;
  created_at: string;
  metadata: Record<string, unknown>;
  action_url?: string;
}

export function NotificationCenter() {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const [open, setOpen] = useState(false);
  const supabase = createClient();

  const unreadCount = notifications.filter((n) => !n.read).length;

  useEffect(() => {
    fetchNotifications();

    // Realtime subscription
    const channel = supabase
      .channel("notifications")
      .on(
        "postgres_changes",
        {
          event: "INSERT",
          schema: "public",
          table: "notifications",
          filter: `user_id=eq.${userId}`,
        },
        (payload) => {
          setNotifications((prev) => [payload.new as Notification, ...prev]);
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, []);

  async function fetchNotifications() {
    const { data } = await supabase
      .from("notifications")
      .select("*")
      .order("created_at", { ascending: false })
      .limit(50);
    if (data) setNotifications(data);
  }

  async function markAsRead(id: string) {
    await supabase.from("notifications").update({ read: true }).eq("id", id);
    setNotifications((prev) =>
      prev.map((n) => (n.id === id ? { ...n, read: true } : n))
    );
  }

  async function markAllAsRead() {
    await supabase
      .from("notifications")
      .update({ read: true })
      .eq("read", false);
    setNotifications((prev) => prev.map((n) => ({ ...n, read: true })));
  }

  return (
    <Popover open={open} onOpenChange={setOpen}>
      <PopoverTrigger asChild>
        <Button variant="ghost" size="icon" className="relative">
          <Bell className="h-5 w-5" />
          {unreadCount > 0 && (
            <span className="absolute -top-1 -right-1 flex h-5 w-5 items-center justify-center rounded-full bg-destructive text-[11px] font-medium text-destructive-foreground">
              {unreadCount > 99 ? "99+" : unreadCount}
            </span>
          )}
        </Button>
      </PopoverTrigger>
      <PopoverContent className="w-[380px] p-0" align="end">
        <div className="flex items-center justify-between border-b px-4 py-3">
          <h3 className="font-semibold">Notifications</h3>
          {unreadCount > 0 && (
            <Button variant="ghost" size="sm" onClick={markAllAsRead}>
              Mark all read
            </Button>
          )}
        </div>
        <ScrollArea className="h-[400px]">
          {notifications.length === 0 ? (
            <div className="flex flex-col items-center justify-center py-12 text-muted-foreground">
              <Bell className="mb-2 h-8 w-8" />
              <p className="text-sm">No notifications yet</p>
            </div>
          ) : (
            <div className="divide-y">
              {notifications.map((notification) => (
                <NotificationItem
                  key={notification.id}
                  notification={notification}
                  onRead={markAsRead}
                />
              ))}
            </div>
          )}
        </ScrollArea>
      </PopoverContent>
    </Popover>
  );
}

function NotificationItem({
  notification,
  onRead,
}: {
  notification: Notification;
  onRead: (id: string) => void;
}) {
  return (
    <button
      className={cn(
        "flex w-full gap-3 px-4 py-3 text-left transition-colors hover:bg-muted/50",
        !notification.read && "bg-primary/5"
      )}
      onClick={() => {
        onRead(notification.id);
        if (notification.action_url) {
          window.location.href = notification.action_url;
        }
      }}
    >
      <NotificationIcon type={notification.type} />
      <div className="flex-1 space-y-1">
        <p className="text-sm font-medium leading-none">{notification.title}</p>
        <p className="text-sm text-muted-foreground">{notification.body}</p>
        <p className="text-xs text-muted-foreground">
          {formatRelativeTime(notification.created_at)}
        </p>
      </div>
      {!notification.read && (
        <div className="mt-1.5 h-2 w-2 shrink-0 rounded-full bg-primary" />
      )}
    </button>
  );
}

function formatRelativeTime(dateString: string): string {
  const date = new Date(dateString);
  const now = new Date();
  const diffMs = now.getTime() - date.getTime();
  const diffMins = Math.floor(diffMs / 60000);
  if (diffMins < 1) return "Just now";
  if (diffMins < 60) return `${diffMins}m ago`;
  const diffHours = Math.floor(diffMins / 60);
  if (diffHours < 24) return `${diffHours}h ago`;
  const diffDays = Math.floor(diffHours / 24);
  if (diffDays < 7) return `${diffDays}d ago`;
  return date.toLocaleDateString();
}
```

---

## 2. Email Notifications

### React Email Template

```tsx
// emails/notification-email.tsx
import {
  Body,
  Container,
  Head,
  Heading,
  Html,
  Link,
  Preview,
  Section,
  Text,
  Button,
  Hr,
} from "@react-email/components";

interface NotificationEmailProps {
  userName: string;
  title: string;
  body: string;
  actionUrl?: string;
  actionLabel?: string;
  unsubscribeUrl: string;
}

export default function NotificationEmail({
  userName,
  title,
  body,
  actionUrl,
  actionLabel = "View Details",
  unsubscribeUrl,
}: NotificationEmailProps) {
  return (
    <Html>
      <Head />
      <Preview>{title}</Preview>
      <Body style={{ backgroundColor: "#f6f9fc", fontFamily: "system-ui, -apple-system, sans-serif" }}>
        <Container style={{ maxWidth: "560px", margin: "0 auto", padding: "20px 0 48px" }}>
          <Heading style={{ fontSize: "24px", fontWeight: "600", color: "#1a1a1a" }}>
            {title}
          </Heading>
          <Text style={{ fontSize: "16px", color: "#4a4a4a", lineHeight: "1.6" }}>
            Hi {userName},
          </Text>
          <Text style={{ fontSize: "16px", color: "#4a4a4a", lineHeight: "1.6" }}>
            {body}
          </Text>
          {actionUrl && (
            <Section style={{ textAlign: "center", marginTop: "24px" }}>
              <Button
                href={actionUrl}
                style={{
                  backgroundColor: "#0f172a",
                  color: "#fff",
                  padding: "12px 24px",
                  borderRadius: "6px",
                  fontSize: "14px",
                  fontWeight: "500",
                  textDecoration: "none",
                }}
              >
                {actionLabel}
              </Button>
            </Section>
          )}
          <Hr style={{ borderColor: "#e5e5e5", margin: "32px 0" }} />
          <Text style={{ fontSize: "12px", color: "#999" }}>
            You received this because of your notification settings.{" "}
            <Link href={unsubscribeUrl} style={{ color: "#999" }}>
              Unsubscribe
            </Link>
          </Text>
        </Container>
      </Body>
    </Html>
  );
}
```

### Sending with Resend

```tsx
// lib/notifications/email.ts
import { Resend } from "resend";
import NotificationEmail from "@/emails/notification-email";

const resend = new Resend(process.env.RESEND_API_KEY);

export async function sendNotificationEmail({
  to,
  userName,
  title,
  body,
  actionUrl,
  actionLabel,
}: {
  to: string;
  userName: string;
  title: string;
  body: string;
  actionUrl?: string;
  actionLabel?: string;
}) {
  const unsubscribeUrl = `${process.env.NEXT_PUBLIC_APP_URL}/settings/notifications`;

  const { data, error } = await resend.emails.send({
    from: "App <notifications@yourdomain.com>",
    to,
    subject: title,
    react: NotificationEmail({
      userName,
      title,
      body,
      actionUrl,
      actionLabel,
      unsubscribeUrl,
    }),
    headers: {
      "List-Unsubscribe": `<${unsubscribeUrl}>`,
    },
  });

  if (error) throw new Error(`Failed to send email: ${error.message}`);
  return data;
}
```

---

## 3. Push Notifications

### VAPID Key Generation

```bash
npx web-push generate-vapid-keys
```

Store `NEXT_PUBLIC_VAPID_PUBLIC_KEY` and `VAPID_PRIVATE_KEY` in `.env`.

### Service Worker

```ts
// public/sw.js
self.addEventListener("push", (event) => {
  const data = event.data?.json() ?? {};
  const { title, body, icon, url } = data;

  event.waitUntil(
    self.registration.showNotification(title, {
      body,
      icon: icon || "/icon-192.png",
      badge: "/badge-72.png",
      data: { url },
      actions: [{ action: "open", title: "Open" }],
    })
  );
});

self.addEventListener("notificationclick", (event) => {
  event.notification.close();
  const url = event.notification.data?.url || "/";
  event.waitUntil(clients.openWindow(url));
});
```

### Permission Request and Subscription

```tsx
// lib/notifications/push.ts
export async function requestPushPermission(): Promise<PushSubscription | null> {
  if (!("Notification" in window) || !("serviceWorker" in navigator)) {
    return null;
  }

  const permission = await Notification.requestPermission();
  if (permission !== "granted") return null;

  const registration = await navigator.serviceWorker.register("/sw.js");
  await navigator.serviceWorker.ready;

  const subscription = await registration.pushManager.subscribe({
    userVisibleOnly: true,
    applicationServerKey: urlBase64ToUint8Array(
      process.env.NEXT_PUBLIC_VAPID_PUBLIC_KEY!
    ),
  });

  // Save subscription to database
  await fetch("/api/push/subscribe", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(subscription),
  });

  return subscription;
}

function urlBase64ToUint8Array(base64String: string): Uint8Array {
  const padding = "=".repeat((4 - (base64String.length % 4)) % 4);
  const base64 = (base64String + padding).replace(/-/g, "+").replace(/_/g, "/");
  const rawData = atob(base64);
  return Uint8Array.from([...rawData].map((char) => char.charCodeAt(0)));
}
```

### Server-Side Push

```ts
// lib/notifications/push-server.ts
import webPush from "web-push";
import { createClient } from "@/lib/supabase/server";

webPush.setVapidDetails(
  "mailto:your@email.com",
  process.env.NEXT_PUBLIC_VAPID_PUBLIC_KEY!,
  process.env.VAPID_PRIVATE_KEY!
);

export async function sendPushNotification(
  userId: string,
  payload: { title: string; body: string; url?: string }
) {
  const supabase = await createClient();
  const { data: subscriptions } = await supabase
    .from("push_subscriptions")
    .select("subscription")
    .eq("user_id", userId);

  if (!subscriptions?.length) return;

  const results = await Promise.allSettled(
    subscriptions.map((sub) =>
      webPush.sendNotification(sub.subscription, JSON.stringify(payload))
    )
  );

  // Remove invalid subscriptions (410 Gone)
  for (let i = 0; i < results.length; i++) {
    const result = results[i];
    if (result.status === "rejected" && result.reason?.statusCode === 410) {
      await supabase
        .from("push_subscriptions")
        .delete()
        .eq("id", subscriptions[i].id);
    }
  }
}
```

### Permission Request UX

Best practices:
- Never request permission on page load. Wait for a user action (e.g., clicking a "Enable notifications" toggle).
- Show a custom pre-prompt explaining the value before triggering the native browser prompt.
- If permission is denied, show instructions for re-enabling in browser settings.

```tsx
// components/notifications/push-prompt.tsx
"use client";

import { useState } from "react";
import { Bell } from "lucide-react";
import { Button } from "@/components/ui/button";
import { requestPushPermission } from "@/lib/notifications/push";

export function PushPrompt() {
  const [status, setStatus] = useState<"idle" | "loading" | "granted" | "denied">("idle");

  async function handleEnable() {
    setStatus("loading");
    const sub = await requestPushPermission();
    setStatus(sub ? "granted" : "denied");
  }

  if (status === "granted") return null;

  return (
    <div className="flex items-center gap-3 rounded-lg border bg-muted/50 p-4">
      <Bell className="h-5 w-5 text-primary" />
      <div className="flex-1">
        <p className="text-sm font-medium">Enable push notifications</p>
        <p className="text-sm text-muted-foreground">
          Get notified about important updates even when you&apos;re not on the site.
        </p>
      </div>
      <Button size="sm" onClick={handleEnable} disabled={status === "loading"}>
        {status === "loading" ? "Enabling..." : "Enable"}
      </Button>
    </div>
  );
}
```

---

## 4. SMS Notifications

### Twilio Integration

```ts
// lib/notifications/sms.ts
import twilio from "twilio";

const client = twilio(
  process.env.TWILIO_ACCOUNT_SID!,
  process.env.TWILIO_AUTH_TOKEN!
);

export async function sendSMS(to: string, body: string) {
  const message = await client.messages.create({
    body,
    from: process.env.TWILIO_PHONE_NUMBER!,
    to,
  });
  return message.sid;
}
```

### Usage Guidelines

- **Use SMS only for critical, time-sensitive alerts**: account security (2FA, password reset), payment failures, service outages, critical threshold breaches.
- **Never use SMS for**: marketing, general updates, social notifications, digest emails. Those belong in email or in-app channels.
- **Cost awareness**: SMS costs ~$0.0079/segment (US). International rates vary widely. Always check Twilio pricing for target countries.
- **Opt-in required**: Collect explicit SMS consent. Include opt-out instructions ("Reply STOP to unsubscribe") in every message.
- **Rate limiting**: Max 1 SMS per event, with a cooldown period (e.g., 15 min) to prevent spam from repeated triggers.

---

## 5. Preference Management

### Notification Settings UI

```tsx
// app/(app)/settings/notifications/page.tsx
"use client";

import { useState, useEffect } from "react";
import { Switch } from "@/components/ui/switch";
import { Label } from "@/components/ui/label";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";
import { Button } from "@/components/ui/button";
import { toast } from "sonner";
import { createClient } from "@/lib/supabase/client";

interface NotificationPreferences {
  email_enabled: boolean;
  push_enabled: boolean;
  sms_enabled: boolean;
  channels: {
    [notificationType: string]: {
      email: boolean;
      push: boolean;
      sms: boolean;
    };
  };
  digest_frequency: "none" | "daily" | "weekly";
  quiet_hours_start: string | null; // "22:00"
  quiet_hours_end: string | null;   // "08:00"
}

const NOTIFICATION_TYPES = [
  { key: "security", label: "Security alerts", description: "Login from new device, password changes" },
  { key: "billing", label: "Billing", description: "Payment confirmations, failed charges" },
  { key: "updates", label: "Product updates", description: "New features, changelog" },
  { key: "mentions", label: "Mentions", description: "When someone mentions you" },
  { key: "comments", label: "Comments", description: "Replies to your posts or threads" },
  { key: "marketing", label: "Tips & offers", description: "Best practices and promotions" },
];

export default function NotificationSettingsPage() {
  const [prefs, setPrefs] = useState<NotificationPreferences | null>(null);
  const [saving, setSaving] = useState(false);
  const supabase = createClient();

  useEffect(() => {
    loadPreferences();
  }, []);

  async function loadPreferences() {
    const { data } = await supabase
      .from("notification_preferences")
      .select("*")
      .single();
    if (data) setPrefs(data.preferences);
  }

  async function savePreferences() {
    if (!prefs) return;
    setSaving(true);
    await supabase
      .from("notification_preferences")
      .upsert({ preferences: prefs });
    toast.success("Notification preferences saved");
    setSaving(false);
  }

  if (!prefs) return null;

  return (
    <div className="space-y-8 max-w-2xl">
      <div>
        <h2 className="text-lg font-semibold">Notification Channels</h2>
        <p className="text-sm text-muted-foreground">
          Choose how you want to be notified for each type of event.
        </p>
      </div>

      {/* Global toggles */}
      <div className="space-y-4 rounded-lg border p-4">
        <div className="flex items-center justify-between">
          <Label>Email notifications</Label>
          <Switch
            checked={prefs.email_enabled}
            onCheckedChange={(v) => setPrefs({ ...prefs, email_enabled: v })}
          />
        </div>
        <div className="flex items-center justify-between">
          <Label>Push notifications</Label>
          <Switch
            checked={prefs.push_enabled}
            onCheckedChange={(v) => setPrefs({ ...prefs, push_enabled: v })}
          />
        </div>
        <div className="flex items-center justify-between">
          <Label>SMS (critical alerts only)</Label>
          <Switch
            checked={prefs.sms_enabled}
            onCheckedChange={(v) => setPrefs({ ...prefs, sms_enabled: v })}
          />
        </div>
      </div>

      {/* Per-type channel matrix */}
      <div className="rounded-lg border">
        <div className="grid grid-cols-[1fr_80px_80px_80px] gap-2 border-b px-4 py-2 text-sm font-medium text-muted-foreground">
          <span>Type</span>
          <span className="text-center">Email</span>
          <span className="text-center">Push</span>
          <span className="text-center">SMS</span>
        </div>
        {NOTIFICATION_TYPES.map((type) => (
          <div
            key={type.key}
            className="grid grid-cols-[1fr_80px_80px_80px] items-center gap-2 border-b px-4 py-3 last:border-0"
          >
            <div>
              <p className="text-sm font-medium">{type.label}</p>
              <p className="text-xs text-muted-foreground">{type.description}</p>
            </div>
            <div className="flex justify-center">
              <Switch
                checked={prefs.channels[type.key]?.email ?? true}
                onCheckedChange={(v) =>
                  setPrefs({
                    ...prefs,
                    channels: {
                      ...prefs.channels,
                      [type.key]: { ...prefs.channels[type.key], email: v },
                    },
                  })
                }
              />
            </div>
            <div className="flex justify-center">
              <Switch
                checked={prefs.channels[type.key]?.push ?? false}
                onCheckedChange={(v) =>
                  setPrefs({
                    ...prefs,
                    channels: {
                      ...prefs.channels,
                      [type.key]: { ...prefs.channels[type.key], push: v },
                    },
                  })
                }
              />
            </div>
            <div className="flex justify-center">
              <Switch
                checked={prefs.channels[type.key]?.sms ?? false}
                disabled={type.key !== "security" && type.key !== "billing"}
                onCheckedChange={(v) =>
                  setPrefs({
                    ...prefs,
                    channels: {
                      ...prefs.channels,
                      [type.key]: { ...prefs.channels[type.key], sms: v },
                    },
                  })
                }
              />
            </div>
          </div>
        ))}
      </div>

      {/* Digest frequency */}
      <div className="flex items-center justify-between rounded-lg border p-4">
        <div>
          <Label>Email digest frequency</Label>
          <p className="text-sm text-muted-foreground">
            Batch non-urgent notifications into a digest
          </p>
        </div>
        <Select
          value={prefs.digest_frequency}
          onValueChange={(v) =>
            setPrefs({ ...prefs, digest_frequency: v as NotificationPreferences["digest_frequency"] })
          }
        >
          <SelectTrigger className="w-[140px]">
            <SelectValue />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="none">Immediate</SelectItem>
            <SelectItem value="daily">Daily</SelectItem>
            <SelectItem value="weekly">Weekly</SelectItem>
          </SelectContent>
        </Select>
      </div>

      <Button onClick={savePreferences} disabled={saving}>
        {saving ? "Saving..." : "Save preferences"}
      </Button>
    </div>
  );
}
```

---

## 6. Batching and Digests

### Notification Queue with Batching

```ts
// lib/notifications/queue.ts
import { createClient } from "@/lib/supabase/server";

/**
 * Queue a notification for potential batching.
 * Urgent notifications (security, billing) are sent immediately.
 * Others are queued and batched based on user digest preferences.
 */
export async function queueNotification({
  userId,
  type,
  title,
  body,
  actionUrl,
  metadata,
}: {
  userId: string;
  type: string;
  title: string;
  body: string;
  actionUrl?: string;
  metadata?: Record<string, unknown>;
}) {
  const supabase = await createClient();

  // Always create the in-app notification immediately
  await supabase.from("notifications").insert({
    user_id: userId,
    type,
    title,
    body,
    action_url: actionUrl,
    metadata,
  });

  // Check if this type is urgent (bypass batching)
  const URGENT_TYPES = ["security", "billing", "system_alert"];
  if (URGENT_TYPES.includes(type)) {
    await sendImmediately({ userId, type, title, body, actionUrl });
    return;
  }

  // Queue for digest
  await supabase.from("notification_queue").insert({
    user_id: userId,
    type,
    title,
    body,
    action_url: actionUrl,
    metadata,
    status: "pending",
  });
}
```

### Digest Cron Job (API Route or Edge Function)

```ts
// app/api/cron/send-digests/route.ts
import { createClient } from "@/lib/supabase/server";
import { sendNotificationEmail } from "@/lib/notifications/email";
import { NextResponse } from "next/server";

export async function GET(request: Request) {
  // Verify cron secret
  const authHeader = request.headers.get("authorization");
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const supabase = await createClient();

  // Get users who want daily digests with pending notifications
  const { data: pendingByUser } = await supabase
    .from("notification_queue")
    .select("user_id, title, body, action_url, type, created_at")
    .eq("status", "pending")
    .order("created_at", { ascending: true });

  if (!pendingByUser?.length) {
    return NextResponse.json({ sent: 0 });
  }

  // Group by user
  const grouped = pendingByUser.reduce(
    (acc, n) => {
      acc[n.user_id] = acc[n.user_id] || [];
      acc[n.user_id].push(n);
      return acc;
    },
    {} as Record<string, typeof pendingByUser>
  );

  let sentCount = 0;

  for (const [userId, notifications] of Object.entries(grouped)) {
    // Check user's digest preference
    const { data: prefs } = await supabase
      .from("notification_preferences")
      .select("preferences")
      .eq("user_id", userId)
      .single();

    const frequency = prefs?.preferences?.digest_frequency ?? "daily";
    if (frequency === "none") continue; // Send individually, not in digest

    // Send digest email
    const { data: user } = await supabase
      .from("profiles")
      .select("email, full_name")
      .eq("id", userId)
      .single();

    if (user) {
      await sendNotificationEmail({
        to: user.email,
        userName: user.full_name,
        title: `You have ${notifications.length} new notifications`,
        body: notifications
          .map((n) => `- ${n.title}: ${n.body}`)
          .join("\n"),
      });
      sentCount++;
    }

    // Mark as sent
    const ids = notifications.map((n) => n.id);
    await supabase
      .from("notification_queue")
      .update({ status: "sent" })
      .in("id", ids);
  }

  return NextResponse.json({ sent: sentCount });
}
```

Configure the cron in `vercel.json`:

```json
{
  "crons": [
    {
      "path": "/api/cron/send-digests",
      "schedule": "0 8 * * *"
    }
  ]
}
```

---

## 7. Database Schema

```sql
-- Notifications table (in-app)
CREATE TABLE notifications (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  type TEXT NOT NULL,           -- 'security' | 'billing' | 'mention' | 'comment' | etc.
  title TEXT NOT NULL,
  body TEXT NOT NULL,
  read BOOLEAN DEFAULT false,
  action_url TEXT,              -- deep link into the app
  metadata JSONB DEFAULT '{}', -- flexible payload (e.g., { "comment_id": "...", "post_id": "..." })
  created_at TIMESTAMPTZ DEFAULT now(),
  read_at TIMESTAMPTZ
);

-- Indexes
CREATE INDEX idx_notifications_user_unread ON notifications (user_id, read) WHERE read = false;
CREATE INDEX idx_notifications_user_created ON notifications (user_id, created_at DESC);

-- RLS
ALTER TABLE notifications ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users read own notifications"
  ON notifications FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "Users update own notifications"
  ON notifications FOR UPDATE
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

-- Notification preferences
CREATE TABLE notification_preferences (
  user_id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  preferences JSONB NOT NULL DEFAULT '{
    "email_enabled": true,
    "push_enabled": false,
    "sms_enabled": false,
    "channels": {},
    "digest_frequency": "daily",
    "quiet_hours_start": null,
    "quiet_hours_end": null
  }',
  updated_at TIMESTAMPTZ DEFAULT now()
);

ALTER TABLE notification_preferences ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users manage own preferences"
  ON notification_preferences FOR ALL
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

-- Push subscriptions
CREATE TABLE push_subscriptions (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  subscription JSONB NOT NULL,  -- Web Push subscription object
  user_agent TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

ALTER TABLE push_subscriptions ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users manage own push subscriptions"
  ON push_subscriptions FOR ALL
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

-- Notification queue (for batching/digests)
CREATE TABLE notification_queue (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  type TEXT NOT NULL,
  title TEXT NOT NULL,
  body TEXT NOT NULL,
  action_url TEXT,
  metadata JSONB DEFAULT '{}',
  status TEXT DEFAULT 'pending', -- 'pending' | 'sent' | 'failed'
  created_at TIMESTAMPTZ DEFAULT now(),
  sent_at TIMESTAMPTZ
);

CREATE INDEX idx_queue_pending ON notification_queue (status, created_at) WHERE status = 'pending';

-- Webhook subscriptions
CREATE TABLE webhook_subscriptions (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  url TEXT NOT NULL,
  secret TEXT NOT NULL,           -- for HMAC signature verification
  events TEXT[] NOT NULL,         -- e.g., ARRAY['invoice.paid', 'user.created']
  active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  last_triggered_at TIMESTAMPTZ,
  failure_count INTEGER DEFAULT 0
);

ALTER TABLE webhook_subscriptions ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users manage own webhooks"
  ON webhook_subscriptions FOR ALL
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);
```

---

## 8. Realtime Delivery

### Supabase Realtime for Instant In-App Notifications

The notification center in section 1 already uses Supabase Realtime via `postgres_changes`. Key points:

```ts
// Enable Realtime on the notifications table in Supabase dashboard:
// Database > Replication > Add table: notifications

// Or via SQL:
ALTER PUBLICATION supabase_realtime ADD TABLE notifications;
```

**Connection management pattern:**

```tsx
// hooks/use-notifications.ts
"use client";

import { useEffect, useState, useCallback } from "react";
import { createClient } from "@/lib/supabase/client";
import type { RealtimeChannel } from "@supabase/supabase-js";

export function useNotifications(userId: string) {
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const [unreadCount, setUnreadCount] = useState(0);
  const supabase = createClient();

  const fetchNotifications = useCallback(async () => {
    const { data } = await supabase
      .from("notifications")
      .select("*")
      .eq("user_id", userId)
      .order("created_at", { ascending: false })
      .limit(50);

    if (data) {
      setNotifications(data);
      setUnreadCount(data.filter((n) => !n.read).length);
    }
  }, [userId]);

  useEffect(() => {
    fetchNotifications();

    const channel: RealtimeChannel = supabase
      .channel(`notifications:${userId}`)
      .on(
        "postgres_changes",
        {
          event: "INSERT",
          schema: "public",
          table: "notifications",
          filter: `user_id=eq.${userId}`,
        },
        (payload) => {
          const newNotification = payload.new as Notification;
          setNotifications((prev) => [newNotification, ...prev]);
          setUnreadCount((prev) => prev + 1);

          // Optional: play sound or show browser notification
          if (document.hidden) {
            new Notification(newNotification.title, {
              body: newNotification.body,
            });
          }
        }
      )
      .on(
        "postgres_changes",
        {
          event: "UPDATE",
          schema: "public",
          table: "notifications",
          filter: `user_id=eq.${userId}`,
        },
        (payload) => {
          const updated = payload.new as Notification;
          setNotifications((prev) =>
            prev.map((n) => (n.id === updated.id ? updated : n))
          );
          // Recalculate unread
          setNotifications((current) => {
            setUnreadCount(current.filter((n) => !n.read).length);
            return current;
          });
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [userId, fetchNotifications]);

  return { notifications, unreadCount };
}
```

---

## 9. Templates

### Notification Template System

```ts
// lib/notifications/templates.ts

type TemplateVariables = Record<string, string | number>;

interface NotificationTemplate {
  title: (vars: TemplateVariables) => string;
  body: (vars: TemplateVariables) => string;
  actionUrl?: (vars: TemplateVariables) => string;
  actionLabel?: string;
  channels: ("in_app" | "email" | "push" | "sms")[];
}

const TEMPLATES: Record<string, NotificationTemplate> = {
  "comment.new": {
    title: (v) => `${v.authorName} commented on "${v.postTitle}"`,
    body: (v) => `"${String(v.commentPreview).slice(0, 100)}..."`,
    actionUrl: (v) => `/posts/${v.postId}#comment-${v.commentId}`,
    actionLabel: "View comment",
    channels: ["in_app", "email", "push"],
  },
  "invoice.paid": {
    title: () => "Payment received",
    body: (v) => `Invoice #${v.invoiceNumber} for ${v.amount} has been paid.`,
    actionUrl: (v) => `/billing/invoices/${v.invoiceId}`,
    actionLabel: "View invoice",
    channels: ["in_app", "email"],
  },
  "security.login_new_device": {
    title: () => "New device login",
    body: (v) =>
      `Your account was accessed from ${v.device} in ${v.location}. If this wasn't you, secure your account immediately.`,
    actionUrl: () => "/settings/security",
    actionLabel: "Review activity",
    channels: ["in_app", "email", "push", "sms"],
  },
  "team.invite": {
    title: (v) => `${v.inviterName} invited you to ${v.teamName}`,
    body: (v) => `You've been invited to join the ${v.teamName} team as ${v.role}.`,
    actionUrl: (v) => `/invites/${v.inviteId}`,
    actionLabel: "Accept invite",
    channels: ["in_app", "email"],
  },
  "usage.threshold": {
    title: () => "Usage limit approaching",
    body: (v) => `You've used ${v.percentage}% of your ${v.resource} quota this month.`,
    actionUrl: () => "/billing/usage",
    actionLabel: "View usage",
    channels: ["in_app", "email"],
  },
};

export function renderTemplate(
  templateKey: string,
  variables: TemplateVariables
) {
  const template = TEMPLATES[templateKey];
  if (!template) throw new Error(`Unknown notification template: ${templateKey}`);

  return {
    title: template.title(variables),
    body: template.body(variables),
    actionUrl: template.actionUrl?.(variables),
    actionLabel: template.actionLabel,
    channels: template.channels,
  };
}
```

### Localization Support

```ts
// lib/notifications/templates-i18n.ts

type Locale = "en" | "de";

const TRANSLATIONS: Record<string, Record<Locale, NotificationTemplate>> = {
  "invoice.paid": {
    en: {
      title: () => "Payment received",
      body: (v) => `Invoice #${v.invoiceNumber} for ${v.amount} has been paid.`,
      channels: ["in_app", "email"],
    },
    de: {
      title: () => "Zahlung erhalten",
      body: (v) => `Rechnung #${v.invoiceNumber} uber ${v.amount} wurde bezahlt.`,
      channels: ["in_app", "email"],
    },
  },
};

export function renderLocalizedTemplate(
  templateKey: string,
  variables: TemplateVariables,
  locale: Locale = "en"
) {
  const templates = TRANSLATIONS[templateKey];
  if (!templates) throw new Error(`Unknown template: ${templateKey}`);

  const template = templates[locale] ?? templates.en;
  return {
    title: template.title(variables),
    body: template.body(variables),
    channels: template.channels,
  };
}
```

---

## 10. Webhook Notifications

### Webhook Dispatcher

```ts
// lib/notifications/webhooks.ts
import crypto from "crypto";
import { createClient } from "@/lib/supabase/server";

interface WebhookPayload {
  event: string;
  data: Record<string, unknown>;
  timestamp: string;
}

export async function dispatchWebhook(
  userId: string,
  event: string,
  data: Record<string, unknown>
) {
  const supabase = await createClient();

  // Find active subscriptions for this event
  const { data: subscriptions } = await supabase
    .from("webhook_subscriptions")
    .select("*")
    .eq("user_id", userId)
    .eq("active", true)
    .contains("events", [event]);

  if (!subscriptions?.length) return;

  const payload: WebhookPayload = {
    event,
    data,
    timestamp: new Date().toISOString(),
  };

  const body = JSON.stringify(payload);

  for (const sub of subscriptions) {
    // Generate HMAC signature
    const signature = crypto
      .createHmac("sha256", sub.secret)
      .update(body)
      .digest("hex");

    try {
      const response = await fetch(sub.url, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "X-Webhook-Signature": `sha256=${signature}`,
          "X-Webhook-Event": event,
          "X-Webhook-Delivery": crypto.randomUUID(),
        },
        body,
        signal: AbortSignal.timeout(10000), // 10s timeout
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }

      // Reset failure count on success
      await supabase
        .from("webhook_subscriptions")
        .update({
          failure_count: 0,
          last_triggered_at: new Date().toISOString(),
        })
        .eq("id", sub.id);
    } catch (error) {
      const newFailureCount = sub.failure_count + 1;

      // Disable after 5 consecutive failures
      await supabase
        .from("webhook_subscriptions")
        .update({
          failure_count: newFailureCount,
          active: newFailureCount < 5,
        })
        .eq("id", sub.id);
    }
  }
}
```

### Webhook Management API

```ts
// app/api/webhooks/route.ts
import { createClient } from "@/lib/supabase/server";
import { NextResponse } from "next/server";
import crypto from "crypto";

export async function POST(request: Request) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const { url, events } = await request.json();

  // Generate a secret for signature verification
  const secret = crypto.randomBytes(32).toString("hex");

  const { data, error } = await supabase
    .from("webhook_subscriptions")
    .insert({
      user_id: user.id,
      url,
      secret,
      events,
    })
    .select()
    .single();

  if (error) return NextResponse.json({ error: error.message }, { status: 400 });

  // Return secret only on creation (never expose it again)
  return NextResponse.json({ ...data, secret });
}

export async function GET(request: Request) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const { data } = await supabase
    .from("webhook_subscriptions")
    .select("id, url, events, active, failure_count, last_triggered_at, created_at")
    .eq("user_id", user.id);

  return NextResponse.json(data);
}
```

### Verifying Webhooks (Consumer Side)

```ts
// Example: how your API consumers verify incoming webhooks
import crypto from "crypto";

function verifyWebhookSignature(
  body: string,
  signature: string,
  secret: string
): boolean {
  const expected = crypto
    .createHmac("sha256", secret)
    .update(body)
    .digest("hex");

  return crypto.timingSafeEqual(
    Buffer.from(`sha256=${expected}`),
    Buffer.from(signature)
  );
}
```

---

## Unified Notification Dispatcher

Ties all channels together:

```ts
// lib/notifications/dispatch.ts
import { renderTemplate } from "./templates";
import { queueNotification } from "./queue";
import { sendNotificationEmail } from "./email";
import { sendPushNotification } from "./push-server";
import { sendSMS } from "./sms";
import { dispatchWebhook } from "./webhooks";
import { createClient } from "@/lib/supabase/server";

export async function notify(
  userId: string,
  templateKey: string,
  variables: Record<string, string | number>
) {
  const rendered = renderTemplate(templateKey, variables);
  const supabase = await createClient();

  // Load user preferences
  const { data: prefRow } = await supabase
    .from("notification_preferences")
    .select("preferences")
    .eq("user_id", userId)
    .single();

  const prefs = prefRow?.preferences;
  const typePrefs = prefs?.channels?.[templateKey.split(".")[0]];

  // In-app: always create
  if (rendered.channels.includes("in_app")) {
    await queueNotification({
      userId,
      type: templateKey,
      title: rendered.title,
      body: rendered.body,
      actionUrl: rendered.actionUrl,
    });
  }

  // Email
  if (
    rendered.channels.includes("email") &&
    prefs?.email_enabled !== false &&
    typePrefs?.email !== false
  ) {
    const { data: user } = await supabase
      .from("profiles")
      .select("email, full_name")
      .eq("id", userId)
      .single();

    if (user) {
      await sendNotificationEmail({
        to: user.email,
        userName: user.full_name,
        title: rendered.title,
        body: rendered.body,
        actionUrl: rendered.actionUrl,
        actionLabel: rendered.actionLabel,
      });
    }
  }

  // Push
  if (
    rendered.channels.includes("push") &&
    prefs?.push_enabled &&
    typePrefs?.push !== false
  ) {
    await sendPushNotification(userId, {
      title: rendered.title,
      body: rendered.body,
      url: rendered.actionUrl,
    });
  }

  // SMS (critical only)
  if (
    rendered.channels.includes("sms") &&
    prefs?.sms_enabled &&
    typePrefs?.sms !== false
  ) {
    const { data: user } = await supabase
      .from("profiles")
      .select("phone")
      .eq("id", userId)
      .single();

    if (user?.phone) {
      await sendSMS(user.phone, `${rendered.title}: ${rendered.body}`);
    }
  }

  // Webhooks
  await dispatchWebhook(userId, templateKey, variables);
}
```

### Usage

```ts
// In any server action or API route:
await notify("user-uuid", "comment.new", {
  authorName: "Alice",
  postTitle: "My first post",
  commentPreview: "Great article! I especially liked the part about...",
  postId: "post-123",
  commentId: "comment-456",
});
```
