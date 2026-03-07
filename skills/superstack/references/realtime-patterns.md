# Realtime Patterns (2026)

## Supabase Realtime Setup

### Client Configuration

```typescript
import { createClient } from "@supabase/supabase-js";
import type { Database } from "@/types/supabase";

export const supabase = createClient<Database>(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  {
    realtime: {
      params: {
        eventsPerSecond: 10, // throttle to prevent flooding
      },
    },
  }
);
```

### Channel Subscription & Cleanup

```typescript
"use client";

import { useEffect, useRef } from "react";
import { supabase } from "@/lib/supabase";
import type { RealtimeChannel } from "@supabase/supabase-js";

export function useRealtimeChannel(channelName: string) {
  const channelRef = useRef<RealtimeChannel | null>(null);

  useEffect(() => {
    const channel = supabase.channel(channelName);
    channelRef.current = channel;

    channel.subscribe((status) => {
      if (status === "SUBSCRIBED") {
        console.log(`Connected to ${channelName}`);
      }
    });

    return () => {
      supabase.removeChannel(channel);
    };
  }, [channelName]);

  return channelRef;
}
```

---

## Postgres Changes

### Listen to INSERT/UPDATE/DELETE

```typescript
import { useEffect, useState } from "react";
import { supabase } from "@/lib/supabase";
import type { RealtimePostgresChangesPayload } from "@supabase/supabase-js";
import type { Database } from "@/types/supabase";

type Message = Database["public"]["Tables"]["messages"]["Row"];

export function useRealtimeMessages(conversationId: string) {
  const [messages, setMessages] = useState<Message[]>([]);

  useEffect(() => {
    // Initial fetch
    const fetchMessages = async () => {
      const { data } = await supabase
        .from("messages")
        .select("*")
        .eq("conversation_id", conversationId)
        .order("created_at", { ascending: true });
      if (data) setMessages(data);
    };
    fetchMessages();

    // Realtime subscription
    const channel = supabase
      .channel(`messages:${conversationId}`)
      .on<Message>(
        "postgres_changes",
        {
          event: "INSERT",
          schema: "public",
          table: "messages",
          filter: `conversation_id=eq.${conversationId}`,
        },
        (payload) => {
          setMessages((prev) => [...prev, payload.new]);
        }
      )
      .on<Message>(
        "postgres_changes",
        {
          event: "UPDATE",
          schema: "public",
          table: "messages",
          filter: `conversation_id=eq.${conversationId}`,
        },
        (payload) => {
          setMessages((prev) =>
            prev.map((m) => (m.id === payload.new.id ? payload.new : m))
          );
        }
      )
      .on<Message>(
        "postgres_changes",
        {
          event: "DELETE",
          schema: "public",
          table: "messages",
          filter: `conversation_id=eq.${conversationId}`,
        },
        (payload) => {
          setMessages((prev) =>
            prev.filter((m) => m.id !== payload.old.id)
          );
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [conversationId]);

  return messages;
}
```

### Typed Payload Helper

```typescript
type PostgresPayload<T extends Record<string, unknown>> = {
  new: T;
  old: Partial<T>;
  eventType: "INSERT" | "UPDATE" | "DELETE";
};

function handlePayload<T extends Record<string, unknown>>(
  payload: RealtimePostgresChangesPayload<T>
): PostgresPayload<T> {
  return {
    new: payload.new as T,
    old: payload.old as Partial<T>,
    eventType: payload.eventType,
  };
}
```

### Enable Realtime on Table (SQL)

```sql
-- Enable realtime for a table
ALTER PUBLICATION supabase_realtime ADD TABLE messages;

-- Enable with column filter (reduces payload size)
ALTER PUBLICATION supabase_realtime ADD TABLE messages (
  id, conversation_id, content, sender_id, created_at
);
```

---

## Broadcast

Ephemeral messages between clients — no database persistence.

### Typing Indicator

```typescript
"use client";

import { useEffect, useState, useCallback } from "react";
import { supabase } from "@/lib/supabase";

type TypingPayload = {
  userId: string;
  username: string;
  isTyping: boolean;
};

export function useTypingIndicator(channelName: string, currentUser: { id: string; name: string }) {
  const [typingUsers, setTypingUsers] = useState<Map<string, string>>(new Map());

  useEffect(() => {
    const channel = supabase
      .channel(channelName)
      .on("broadcast", { event: "typing" }, ({ payload }: { payload: TypingPayload }) => {
        if (payload.userId === currentUser.id) return;

        setTypingUsers((prev) => {
          const next = new Map(prev);
          if (payload.isTyping) {
            next.set(payload.userId, payload.username);
          } else {
            next.delete(payload.userId);
          }
          return next;
        });
      })
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [channelName, currentUser.id]);

  const sendTyping = useCallback(
    (isTyping: boolean) => {
      supabase.channel(channelName).send({
        type: "broadcast",
        event: "typing",
        payload: {
          userId: currentUser.id,
          username: currentUser.name,
          isTyping,
        } satisfies TypingPayload,
      });
    },
    [channelName, currentUser]
  );

  return { typingUsers: Array.from(typingUsers.values()), sendTyping };
}
```

### Cursor Sharing (Broadcast)

```typescript
type CursorPayload = {
  userId: string;
  x: number;
  y: number;
  color: string;
};

export function useCursorBroadcast(channelName: string, userId: string, color: string) {
  const channelRef = useRef<RealtimeChannel | null>(null);

  useEffect(() => {
    channelRef.current = supabase.channel(channelName).subscribe();
    return () => {
      if (channelRef.current) supabase.removeChannel(channelRef.current);
    };
  }, [channelName]);

  const broadcastCursor = useCallback(
    (x: number, y: number) => {
      channelRef.current?.send({
        type: "broadcast",
        event: "cursor",
        payload: { userId, x, y, color } satisfies CursorPayload,
      });
    },
    [userId, color]
  );

  return { broadcastCursor };
}
```

---

## Presence

### Online Users List

```typescript
"use client";

import { useEffect, useState } from "react";
import { supabase } from "@/lib/supabase";

type UserPresence = {
  userId: string;
  username: string;
  avatarUrl: string;
  onlineAt: string;
  status: "online" | "away" | "busy";
};

export function usePresence(roomId: string, currentUser: UserPresence) {
  const [onlineUsers, setOnlineUsers] = useState<UserPresence[]>([]);

  useEffect(() => {
    const channel = supabase.channel(`presence:${roomId}`);

    channel
      .on("presence", { event: "sync" }, () => {
        const state = channel.presenceState<UserPresence>();
        const users = Object.values(state)
          .flat()
          .filter((u) => u.userId !== currentUser.userId);
        setOnlineUsers(users);
      })
      .on("presence", { event: "join" }, ({ newPresences }) => {
        console.log("Joined:", newPresences);
      })
      .on("presence", { event: "leave" }, ({ leftPresences }) => {
        console.log("Left:", leftPresences);
      })
      .subscribe(async (status) => {
        if (status === "SUBSCRIBED") {
          await channel.track({
            userId: currentUser.userId,
            username: currentUser.username,
            avatarUrl: currentUser.avatarUrl,
            onlineAt: new Date().toISOString(),
            status: currentUser.status,
          });
        }
      });

    return () => {
      channel.untrack();
      supabase.removeChannel(channel);
    };
  }, [roomId, currentUser.userId]);

  return onlineUsers;
}
```

### Presence UI Component

```typescript
function OnlineIndicator({ users }: { users: UserPresence[] }) {
  return (
    <div className="flex -space-x-2">
      {users.slice(0, 5).map((user) => (
        <div key={user.userId} className="relative">
          <img
            src={user.avatarUrl}
            alt={user.username}
            className="h-8 w-8 rounded-full border-2 border-white"
          />
          <span
            className={cn(
              "absolute bottom-0 right-0 h-2.5 w-2.5 rounded-full border-2 border-white",
              user.status === "online" && "bg-green-500",
              user.status === "away" && "bg-yellow-500",
              user.status === "busy" && "bg-red-500"
            )}
          />
        </div>
      ))}
      {users.length > 5 && (
        <div className="flex h-8 w-8 items-center justify-center rounded-full border-2 border-white bg-muted text-xs">
          +{users.length - 5}
        </div>
      )}
    </div>
  );
}
```

---

## Chat Implementation

### Full Chat Component

```typescript
"use client";

import { useEffect, useRef, useState, useCallback, useOptimistic } from "react";
import { supabase } from "@/lib/supabase";
import type { Database } from "@/types/supabase";

type Message = Database["public"]["Tables"]["messages"]["Row"];

type OptimisticMessage = Message & { pending?: boolean };

export function ChatRoom({ conversationId, userId }: { conversationId: string; userId: string }) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [optimisticMessages, addOptimistic] = useOptimistic<
    OptimisticMessage[],
    OptimisticMessage
  >(messages, (state, newMessage) => [...state, newMessage]);

  const bottomRef = useRef<HTMLDivElement>(null);
  const containerRef = useRef<HTMLDivElement>(null);
  const [isAtBottom, setIsAtBottom] = useState(true);
  const [unreadCount, setUnreadCount] = useState(0);

  // Scroll to bottom
  const scrollToBottom = useCallback((behavior: ScrollBehavior = "smooth") => {
    bottomRef.current?.scrollIntoView({ behavior });
    setUnreadCount(0);
  }, []);

  // Track scroll position
  const handleScroll = useCallback(() => {
    const container = containerRef.current;
    if (!container) return;
    const threshold = 100;
    const atBottom =
      container.scrollHeight - container.scrollTop - container.clientHeight < threshold;
    setIsAtBottom(atBottom);
    if (atBottom) setUnreadCount(0);
  }, []);

  // Fetch initial messages
  useEffect(() => {
    const fetch = async () => {
      const { data } = await supabase
        .from("messages")
        .select("*")
        .eq("conversation_id", conversationId)
        .order("created_at", { ascending: true })
        .limit(50);
      if (data) {
        setMessages(data);
        // Instant scroll on initial load
        setTimeout(() => scrollToBottom("instant"), 0);
      }
    };
    fetch();
  }, [conversationId]);

  // Realtime subscription
  useEffect(() => {
    const channel = supabase
      .channel(`chat:${conversationId}`)
      .on<Message>(
        "postgres_changes",
        {
          event: "INSERT",
          schema: "public",
          table: "messages",
          filter: `conversation_id=eq.${conversationId}`,
        },
        (payload) => {
          setMessages((prev) => {
            // Deduplicate (optimistic message already present)
            if (prev.some((m) => m.id === payload.new.id)) {
              return prev.map((m) => (m.id === payload.new.id ? payload.new : m));
            }
            return [...prev, payload.new];
          });

          if (isAtBottom) {
            scrollToBottom();
          } else {
            setUnreadCount((c) => c + 1);
          }
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [conversationId, isAtBottom]);

  // Optimistic send
  const sendMessage = useCallback(
    async (content: string) => {
      const tempId = crypto.randomUUID();
      const optimistic: OptimisticMessage = {
        id: tempId,
        conversation_id: conversationId,
        sender_id: userId,
        content,
        created_at: new Date().toISOString(),
        pending: true,
      };

      addOptimistic(optimistic);
      scrollToBottom();

      const { error } = await supabase.from("messages").insert({
        conversation_id: conversationId,
        sender_id: userId,
        content,
      });

      if (error) {
        // Remove failed optimistic message
        setMessages((prev) => prev.filter((m) => m.id !== tempId));
        console.error("Failed to send message:", error);
      }
    },
    [conversationId, userId, addOptimistic, scrollToBottom]
  );

  return (
    <div className="flex h-full flex-col">
      <div ref={containerRef} onScroll={handleScroll} className="flex-1 overflow-y-auto p-4">
        {optimisticMessages.map((msg) => (
          <div
            key={msg.id}
            className={cn(
              "mb-2 max-w-[70%] rounded-lg px-3 py-2",
              msg.sender_id === userId
                ? "ml-auto bg-primary text-primary-foreground"
                : "bg-muted",
              msg.pending && "opacity-60"
            )}
          >
            {msg.content}
          </div>
        ))}
        <div ref={bottomRef} />
      </div>

      {/* Unread badge */}
      {unreadCount > 0 && (
        <button
          onClick={() => scrollToBottom()}
          className="absolute bottom-20 left-1/2 -translate-x-1/2 rounded-full bg-primary px-3 py-1 text-sm text-primary-foreground shadow-lg"
        >
          {unreadCount} new message{unreadCount > 1 ? "s" : ""}
        </button>
      )}

      <ChatInput onSend={sendMessage} />
    </div>
  );
}
```

### Chat Input with Typing Indicator

```typescript
function ChatInput({ onSend }: { onSend: (content: string) => void }) {
  const [value, setValue] = useState("");
  const typingTimeoutRef = useRef<ReturnType<typeof setTimeout>>();

  const handleChange = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
    setValue(e.target.value);

    // Debounced typing indicator
    clearTimeout(typingTimeoutRef.current);
    // sendTyping(true) — broadcast via useTypingIndicator
    typingTimeoutRef.current = setTimeout(() => {
      // sendTyping(false)
    }, 2000);
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (!value.trim()) return;
    onSend(value.trim());
    setValue("");
    // sendTyping(false)
  };

  // Submit on Enter, Shift+Enter for newline
  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === "Enter" && !e.shiftKey) {
      e.preventDefault();
      handleSubmit(e);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="border-t p-4">
      <div className="flex gap-2">
        <textarea
          value={value}
          onChange={handleChange}
          onKeyDown={handleKeyDown}
          placeholder="Type a message..."
          className="flex-1 resize-none rounded-lg border p-2"
          rows={1}
        />
        <button
          type="submit"
          disabled={!value.trim()}
          className="rounded-lg bg-primary px-4 py-2 text-primary-foreground disabled:opacity-50"
        >
          Send
        </button>
      </div>
    </form>
  );
}
```

---

## Live Dashboard

### Realtime KPI Updates

```typescript
"use client";

import { useEffect, useState } from "react";
import { supabase } from "@/lib/supabase";

type KPI = {
  id: string;
  metric: string;
  value: number;
  previous_value: number;
  updated_at: string;
};

export function useRealtimeKPIs(dashboardId: string) {
  const [kpis, setKpis] = useState<Map<string, KPI>>(new Map());

  useEffect(() => {
    // Initial fetch
    const fetch = async () => {
      const { data } = await supabase
        .from("kpi_metrics")
        .select("*")
        .eq("dashboard_id", dashboardId);
      if (data) {
        const map = new Map(data.map((k) => [k.id, k]));
        setKpis(map);
      }
    };
    fetch();

    // Listen for updates
    const channel = supabase
      .channel(`kpi:${dashboardId}`)
      .on<KPI>(
        "postgres_changes",
        {
          event: "UPDATE",
          schema: "public",
          table: "kpi_metrics",
          filter: `dashboard_id=eq.${dashboardId}`,
        },
        (payload) => {
          setKpis((prev) => {
            const next = new Map(prev);
            next.set(payload.new.id, payload.new);
            return next;
          });
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [dashboardId]);

  return Array.from(kpis.values());
}
```

### Chart Data Streaming

```typescript
export function useRealtimeChartData(metricId: string, windowMinutes = 60) {
  const [dataPoints, setDataPoints] = useState<{ timestamp: string; value: number }[]>([]);

  useEffect(() => {
    // Fetch historical window
    const since = new Date(Date.now() - windowMinutes * 60 * 1000).toISOString();
    const fetch = async () => {
      const { data } = await supabase
        .from("metric_events")
        .select("timestamp, value")
        .eq("metric_id", metricId)
        .gte("timestamp", since)
        .order("timestamp", { ascending: true });
      if (data) setDataPoints(data);
    };
    fetch();

    // Stream new data points
    const channel = supabase
      .channel(`chart:${metricId}`)
      .on(
        "postgres_changes",
        {
          event: "INSERT",
          schema: "public",
          table: "metric_events",
          filter: `metric_id=eq.${metricId}`,
        },
        (payload) => {
          setDataPoints((prev) => {
            const cutoff = new Date(Date.now() - windowMinutes * 60 * 1000).toISOString();
            const filtered = prev.filter((p) => p.timestamp > cutoff);
            return [...filtered, { timestamp: payload.new.timestamp, value: payload.new.value }];
          });
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [metricId, windowMinutes]);

  return dataPoints;
}
```

### Connection Status Indicator

```typescript
"use client";

import { useEffect, useState } from "react";
import { supabase } from "@/lib/supabase";

type ConnectionStatus = "connecting" | "connected" | "disconnected" | "reconnecting";

export function useConnectionStatus() {
  const [status, setStatus] = useState<ConnectionStatus>("connecting");

  useEffect(() => {
    const channel = supabase.channel("connection-monitor");

    channel.subscribe((s) => {
      switch (s) {
        case "SUBSCRIBED":
          setStatus("connected");
          break;
        case "TIMED_OUT":
        case "CHANNEL_ERROR":
          setStatus("disconnected");
          break;
        case "CLOSED":
          setStatus("disconnected");
          break;
      }
    });

    return () => {
      supabase.removeChannel(channel);
    };
  }, []);

  return status;
}

function ConnectionBadge() {
  const status = useConnectionStatus();

  return (
    <div className="flex items-center gap-2 text-sm">
      <span
        className={cn(
          "h-2 w-2 rounded-full",
          status === "connected" && "bg-green-500",
          status === "connecting" && "animate-pulse bg-yellow-500",
          status === "reconnecting" && "animate-pulse bg-orange-500",
          status === "disconnected" && "bg-red-500"
        )}
      />
      <span className="text-muted-foreground capitalize">{status}</span>
    </div>
  );
}
```

---

## Multiplayer Cursors

### Full Implementation

```typescript
"use client";

import { useEffect, useRef, useState, useCallback } from "react";
import { supabase } from "@/lib/supabase";
import { throttle } from "@/lib/utils";

type CursorPosition = {
  userId: string;
  username: string;
  color: string;
  x: number;
  y: number;
  lastUpdate: number;
};

export function useMultiplayerCursors(
  roomId: string,
  currentUser: { id: string; name: string; color: string }
) {
  const [cursors, setCursors] = useState<Map<string, CursorPosition>>(new Map());
  const channelRef = useRef<ReturnType<typeof supabase.channel> | null>(null);

  useEffect(() => {
    const channel = supabase.channel(`cursors:${roomId}`, {
      config: { broadcast: { self: false } },
    });
    channelRef.current = channel;

    channel
      .on("broadcast", { event: "cursor-move" }, ({ payload }) => {
        setCursors((prev) => {
          const next = new Map(prev);
          next.set(payload.userId, {
            ...payload,
            lastUpdate: Date.now(),
          });
          return next;
        });
      })
      .on("broadcast", { event: "cursor-leave" }, ({ payload }) => {
        setCursors((prev) => {
          const next = new Map(prev);
          next.delete(payload.userId);
          return next;
        });
      })
      .subscribe();

    // Cleanup stale cursors every 5s
    const cleanupInterval = setInterval(() => {
      setCursors((prev) => {
        const next = new Map(prev);
        const staleThreshold = Date.now() - 10_000;
        for (const [id, cursor] of next) {
          if (cursor.lastUpdate < staleThreshold) next.delete(id);
        }
        return next;
      });
    }, 5000);

    return () => {
      channel.send({
        type: "broadcast",
        event: "cursor-leave",
        payload: { userId: currentUser.id },
      });
      supabase.removeChannel(channel);
      clearInterval(cleanupInterval);
    };
  }, [roomId, currentUser.id]);

  // Throttled broadcast (50ms = ~20fps)
  const broadcastPosition = useCallback(
    throttle((x: number, y: number) => {
      channelRef.current?.send({
        type: "broadcast",
        event: "cursor-move",
        payload: {
          userId: currentUser.id,
          username: currentUser.name,
          color: currentUser.color,
          x,
          y,
        },
      });
    }, 50),
    [currentUser]
  );

  return { cursors: Array.from(cursors.values()), broadcastPosition };
}
```

### Cursor Renderer with Smooth Interpolation

```typescript
"use client";

import { useEffect, useRef } from "react";

function AnimatedCursor({ cursor }: { cursor: CursorPosition }) {
  const ref = useRef<HTMLDivElement>(null);
  const posRef = useRef({ x: cursor.x, y: cursor.y });

  useEffect(() => {
    // Smooth interpolation via requestAnimationFrame
    let animId: number;
    const targetX = cursor.x;
    const targetY = cursor.y;

    const animate = () => {
      const lerp = 0.15; // interpolation factor
      posRef.current.x += (targetX - posRef.current.x) * lerp;
      posRef.current.y += (targetY - posRef.current.y) * lerp;

      if (ref.current) {
        ref.current.style.transform = `translate(${posRef.current.x}px, ${posRef.current.y}px)`;
      }

      const dx = Math.abs(targetX - posRef.current.x);
      const dy = Math.abs(targetY - posRef.current.y);
      if (dx > 0.5 || dy > 0.5) {
        animId = requestAnimationFrame(animate);
      }
    };

    animId = requestAnimationFrame(animate);
    return () => cancelAnimationFrame(animId);
  }, [cursor.x, cursor.y]);

  return (
    <div
      ref={ref}
      className="pointer-events-none fixed left-0 top-0 z-50"
      style={{ transform: `translate(${cursor.x}px, ${cursor.y}px)` }}
    >
      {/* Cursor SVG */}
      <svg width="16" height="16" viewBox="0 0 16 16" fill="none">
        <path d="M0 0L16 6L8 8L6 16L0 0Z" fill={cursor.color} />
      </svg>
      {/* Name label */}
      <span
        className="ml-4 -mt-1 whitespace-nowrap rounded px-1.5 py-0.5 text-xs text-white"
        style={{ backgroundColor: cursor.color }}
      >
        {cursor.username}
      </span>
    </div>
  );
}

export function CursorOverlay({ cursors }: { cursors: CursorPosition[] }) {
  return (
    <>
      {cursors.map((cursor) => (
        <AnimatedCursor key={cursor.userId} cursor={cursor} />
      ))}
    </>
  );
}
```

### Throttle Utility

```typescript
// lib/utils.ts
export function throttle<T extends (...args: unknown[]) => void>(
  fn: T,
  ms: number
): (...args: Parameters<T>) => void {
  let lastCall = 0;
  let timeoutId: ReturnType<typeof setTimeout> | null = null;

  return (...args: Parameters<T>) => {
    const now = Date.now();
    const remaining = ms - (now - lastCall);

    if (remaining <= 0) {
      if (timeoutId) {
        clearTimeout(timeoutId);
        timeoutId = null;
      }
      lastCall = now;
      fn(...args);
    } else if (!timeoutId) {
      timeoutId = setTimeout(() => {
        lastCall = Date.now();
        timeoutId = null;
        fn(...args);
      }, remaining);
    }
  };
}
```

---

## Connection Management

### Reconnection Strategy with Exponential Backoff

```typescript
"use client";

import { useEffect, useRef, useState, useCallback } from "react";
import { supabase } from "@/lib/supabase";
import type { RealtimeChannel } from "@supabase/supabase-js";

type ConnectionState = "connected" | "connecting" | "disconnected" | "error";

export function useRealtimeWithReconnect(
  channelName: string,
  setupChannel: (channel: RealtimeChannel) => RealtimeChannel
) {
  const [connectionState, setConnectionState] = useState<ConnectionState>("connecting");
  const retryCount = useRef(0);
  const maxRetries = 10;
  const channelRef = useRef<RealtimeChannel | null>(null);

  const connect = useCallback(() => {
    // Clean up existing channel
    if (channelRef.current) {
      supabase.removeChannel(channelRef.current);
    }

    setConnectionState("connecting");
    const channel = supabase.channel(channelName);
    channelRef.current = channel;

    const configuredChannel = setupChannel(channel);

    configuredChannel.subscribe((status, err) => {
      switch (status) {
        case "SUBSCRIBED":
          setConnectionState("connected");
          retryCount.current = 0; // reset on success
          break;

        case "TIMED_OUT":
        case "CHANNEL_ERROR":
          setConnectionState("error");
          handleReconnect();
          break;

        case "CLOSED":
          setConnectionState("disconnected");
          break;
      }
    });
  }, [channelName, setupChannel]);

  const handleReconnect = useCallback(() => {
    if (retryCount.current >= maxRetries) {
      console.error(`Max retries (${maxRetries}) reached for ${channelName}`);
      setConnectionState("error");
      return;
    }

    // Exponential backoff: 1s, 2s, 4s, 8s, 16s... capped at 30s
    const delay = Math.min(1000 * Math.pow(2, retryCount.current), 30_000);
    const jitter = Math.random() * 1000; // add jitter to prevent thundering herd

    console.log(
      `Reconnecting ${channelName} in ${Math.round(delay + jitter)}ms (attempt ${retryCount.current + 1})`
    );

    setTimeout(() => {
      retryCount.current++;
      connect();
    }, delay + jitter);
  }, [channelName, connect]);

  useEffect(() => {
    connect();

    // Reconnect on window focus (handle sleep/wake)
    const handleFocus = () => {
      if (connectionState === "error" || connectionState === "disconnected") {
        retryCount.current = 0;
        connect();
      }
    };
    window.addEventListener("focus", handleFocus);

    // Reconnect on online event
    const handleOnline = () => {
      retryCount.current = 0;
      connect();
    };
    window.addEventListener("online", handleOnline);

    return () => {
      if (channelRef.current) supabase.removeChannel(channelRef.current);
      window.removeEventListener("focus", handleFocus);
      window.removeEventListener("online", handleOnline);
    };
  }, [connect]);

  return {
    connectionState,
    reconnect: () => {
      retryCount.current = 0;
      connect();
    },
  };
}
```

### Connection State UI

```typescript
function ConnectionStatusBar({ state }: { state: ConnectionState }) {
  if (state === "connected") return null;

  return (
    <div
      className={cn(
        "fixed left-0 right-0 top-0 z-50 px-4 py-2 text-center text-sm font-medium",
        state === "connecting" && "bg-yellow-100 text-yellow-800",
        state === "disconnected" && "bg-gray-100 text-gray-800",
        state === "error" && "bg-red-100 text-red-800"
      )}
    >
      {state === "connecting" && "Reconnecting..."}
      {state === "disconnected" && "Connection lost. Attempting to reconnect..."}
      {state === "error" && (
        <>
          Unable to connect.{" "}
          <button className="underline" onClick={() => window.location.reload()}>
            Reload page
          </button>
        </>
      )}
    </div>
  );
}
```

---

## Performance

### Channel Limits

- Supabase allows **max 100 concurrent channels** per client
- Each `postgres_changes` listener = 1 channel slot
- Combine related subscriptions on the same channel when possible

```typescript
// BAD: Separate channels for each listener
const ch1 = supabase.channel("messages-insert").on("postgres_changes", { event: "INSERT", ... });
const ch2 = supabase.channel("messages-update").on("postgres_changes", { event: "UPDATE", ... });

// GOOD: Single channel with multiple listeners
const channel = supabase
  .channel("messages")
  .on("postgres_changes", { event: "INSERT", table: "messages", ... }, handleInsert)
  .on("postgres_changes", { event: "UPDATE", table: "messages", ... }, handleUpdate)
  .on("postgres_changes", { event: "DELETE", table: "messages", ... }, handleDelete)
  .subscribe();
```

### Unsubscribe Cleanup

```typescript
// Always clean up in useEffect return
useEffect(() => {
  const channels: RealtimeChannel[] = [];

  const ch1 = supabase.channel("channel-1").subscribe();
  channels.push(ch1);

  const ch2 = supabase.channel("channel-2").subscribe();
  channels.push(ch2);

  return () => {
    channels.forEach((ch) => supabase.removeChannel(ch));
  };
}, []);

// For component unmount, also remove all channels
useEffect(() => {
  return () => {
    supabase.removeAllChannels();
  };
}, []);
```

### Server-Side Filtering

Reduce payload size by filtering at the database level:

```sql
-- Only publish specific columns (reduces bandwidth)
ALTER PUBLICATION supabase_realtime ADD TABLE messages (
  id, conversation_id, content, sender_id, created_at
);

-- Don't publish large text/binary columns you don't need in realtime
```

```typescript
// Client-side: use filter parameter to reduce events
channel.on("postgres_changes", {
  event: "INSERT",
  schema: "public",
  table: "messages",
  // Only receive events for this conversation
  filter: `conversation_id=eq.${conversationId}`,
}, handler);

// Supported filter operators: eq, neq, lt, lte, gt, gte, in
// Single filter per subscription (combine on server with RLS for complex logic)
```

### Batching Updates

```typescript
// For high-frequency updates (e.g., live metrics), batch state updates
function useBatchedUpdates<T>(intervalMs = 100) {
  const [items, setItems] = useState<T[]>([]);
  const bufferRef = useRef<T[]>([]);

  useEffect(() => {
    const flush = setInterval(() => {
      if (bufferRef.current.length > 0) {
        setItems((prev) => [...prev, ...bufferRef.current]);
        bufferRef.current = [];
      }
    }, intervalMs);

    return () => clearInterval(flush);
  }, [intervalMs]);

  const add = useCallback((item: T) => {
    bufferRef.current.push(item);
  }, []);

  return { items, add };
}
```

### RLS for Realtime Security

```sql
-- Realtime respects Row Level Security policies
-- Users only receive events for rows they can SELECT

CREATE POLICY "Users can read own conversation messages"
  ON messages
  FOR SELECT
  USING (
    conversation_id IN (
      SELECT conversation_id FROM conversation_members
      WHERE user_id = auth.uid()
    )
  );
```
