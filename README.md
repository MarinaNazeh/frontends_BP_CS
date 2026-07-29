Ah — got it, they're scaffolded but blank. Let's fill them in with reasonable, reusable versions first, then build the Alerts pages on top of them.

## `components/Button.tsx`

```tsx
// components/Button.tsx
"use client";

import type { ButtonHTMLAttributes } from "react";

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "secondary" | "danger";
}

export function Button({ variant = "primary", children, style, ...rest }: ButtonProps) {
  const variantStyles: Record<string, React.CSSProperties> = {
    primary: { background: "#2563eb", color: "#fff" },
    secondary: { background: "#f1f1f1", color: "#111" },
    danger: { background: "#dc2626", color: "#fff" },
  };

  return (
    <button
      {...rest}
      style={{
        border: "none",
        borderRadius: 6,
        padding: "8px 14px",
        fontSize: 13,
        cursor: rest.disabled ? "not-allowed" : "pointer",
        opacity: rest.disabled ? 0.6 : 1,
        ...variantStyles[variant],
        ...style,
      }}
    >
      {children}
    </button>
  );
}
```

## `components/Header.tsx`

```tsx
// components/Header.tsx
export function Header() {
  return (
    <header
      style={{
        display: "flex",
        alignItems: "center",
        justifyContent: "space-between",
        padding: "10px 16px",
        borderBottom: "1px solid #eee",
      }}
    >
      <span style={{ fontWeight: 600 }}>SOC Console</span>
    </header>
  );
}
```

## `components/Footer.tsx`

```tsx
// components/Footer.tsx
export function Footer() {
  return (
    <footer
      style={{
        padding: "8px 16px",
        fontSize: 12,
        color: "#888",
        borderTop: "1px solid #eee",
      }}
    >
      SOC Console — internal use only
    </footer>
  );
}
```

## `components/Layout.tsx`

```tsx
// components/Layout.tsx
import type { ReactNode } from "react";
import { Header } from "./Header";
import { Footer } from "./Footer";

interface LayoutProps {
  children: ReactNode;
}

export function Layout({ children }: LayoutProps) {
  return (
    <div style={{ display: "flex", flexDirection: "column", height: "100vh" }}>
      <Header />
      <main style={{ flex: 1, overflow: "hidden" }}>{children}</main>
      <Footer />
    </div>
  );
}
```

## `components/LogCard.tsx`

This one needs to double as your alert list item — title, severity, asset, and a selected/click state:

```tsx
// components/LogCard.tsx
"use client";

type Severity = "critical" | "high" | "medium" | "low";

interface LogCardProps {
  title: string;
  asset: string;
  severity: Severity;
  isSelected?: boolean;
  onClick?: () => void;
}

const severityColors: Record<Severity, { bg: string; text: string }> = {
  critical: { bg: "#fee2e2", text: "#b91c1c" },
  high: { bg: "#ffedd5", text: "#c2410c" },
  medium: { bg: "#fef9c3", text: "#a16207" },
  low: { bg: "#dbeafe", text: "#1d4ed8" },
};

export function LogCard({ title, asset, severity, isSelected, onClick }: LogCardProps) {
  const colors = severityColors[severity];

  return (
    <div
      onClick={onClick}
      style={{
        padding: 10,
        borderRadius: 8,
        cursor: "pointer",
        background: isSelected ? "#eef2ff" : colors.bg,
        border: isSelected ? "1px solid #6366f1" : "1px solid transparent",
        marginBottom: 6,
      }}
    >
      <p style={{ margin: 0, fontSize: 12, fontWeight: 600, color: colors.text }}>
        {severity.toUpperCase()} · {title}
      </p>
      <p style={{ margin: "2px 0 0", fontSize: 11, color: "#666" }}>{asset}</p>
    </div>
  );
}
```

## Now `app/Alerts/page.tsx`

```tsx
// app/Alerts/page.tsx
"use client";

import { Layout } from "@/components/Layout";
import { LogCard } from "@/components/LogCard";
import { useRouter } from "next/navigation";
import { useAlertsStore } from "@/store/alertsStore";

export default function AlertsPage() {
  const router = useRouter();
  const alerts = useAlertsStore((state) => state.alerts);

  return (
    <Layout>
      <div style={{ display: "flex", height: "100%" }}>
        <div style={{ width: 280, borderRight: "1px solid #eee", padding: 12, overflow: "auto" }}>
          {alerts.map((alert) => (
            <LogCard
              key={alert.id}
              title={alert.title}
              asset={alert.asset}
              severity={alert.severity}
              onClick={() => router.push(`/Alerts/${alert.id}`)}
            />
          ))}
        </div>
        <div style={{ flex: 1, display: "flex", alignItems: "center", justifyContent: "center", color: "#888" }}>
          <p>Select an alert to see its analysis.</p>
        </div>
      </div>
    </Layout>
  );
}
```

## And `app/Alerts/[id]/page.tsx`

Since you want the whole page inlined here (not split into more components), this reuses `Layout`, `LogCard`, and `Button` only:

```tsx
// app/Alerts/[id]/page.tsx
"use client";

import { useEffect, useState } from "react";
import { useRouter } from "next/navigation";
import { Layout } from "@/components/Layout";
import { LogCard } from "@/components/LogCard";
import { Button } from "@/components/Button";
import { useAlertsStore } from "@/store/alertsStore";

interface Props {
  params: { id: string };
}

export default function AlertDetailPage({ params }: Props) {
  const { id } = params;
  const router = useRouter();

  const alerts = useAlertsStore((state) => state.alerts);
  const alert = alerts.find((a) => a.id === id);
  const loadInitialAnalysis = useAlertsStore((state) => state.loadInitialAnalysis);
  const askFollowUp = useAlertsStore((state) => state.askFollowUp);

  const [draft, setDraft] = useState("");
  const [isResponding, setIsResponding] = useState(false);

  useEffect(() => {
    if (alert && alert.messages.length === 0) {
      loadInitialAnalysis(id);
    }
  }, [id, alert, loadInitialAnalysis]);

  if (!alert) {
    return (
      <Layout>
        <p style={{ padding: 16 }}>Alert not found.</p>
      </Layout>
    );
  }

  async function handleSend() {
    if (!draft.trim()) return;
    const question = draft.trim();
    setDraft("");
    setIsResponding(true);
    await askFollowUp(id, question);
    setIsResponding(false);
  }

  return (
    <Layout>
      <div style={{ display: "flex", height: "100%" }}>
        {/* Left: alert list, same as index page */}
        <div style={{ width: 280, borderRight: "1px solid #eee", padding: 12, overflow: "auto" }}>
          {alerts.map((a) => (
            <LogCard
              key={a.id}
              title={a.title}
              asset={a.asset}
              severity={a.severity}
              isSelected={a.id === id}
              onClick={() => router.push(`/Alerts/${a.id}`)}
            />
          ))}
        </div>

        {/* Right: chat pane, inlined */}
        <div style={{ flex: 1, display: "flex", flexDirection: "column", padding: 16 }}>
          <div style={{ borderBottom: "1px solid #eee", paddingBottom: 10 }}>
            <h3 style={{ margin: 0 }}>{alert.title}</h3>
            <p style={{ margin: "2px 0 0", fontSize: 12, color: "#666" }}>
              {alert.asset}
              {alert.sourceIp ? ` · ${alert.sourceIp}` : ""}
            </p>
          </div>

          <div style={{ flex: 1, overflow: "auto", display: "flex", flexDirection: "column", gap: 10, padding: "12px 0" }}>
            {alert.messages.map((msg) => (
              <div
                key={msg.id}
                style={{
                  alignSelf: msg.role === "user" ? "flex-end" : "flex-start",
                  maxWidth: "80%",
                  background: msg.role === "user" ? "#e6f0ff" : "#f4f4f4",
                  borderRadius: 8,
                  padding: "8px 12px",
                  fontSize: 13,
                }}
              >
                {msg.content}
              </div>
            ))}
            {isResponding && <p style={{ fontSize: 12, color: "#888" }}>Thinking...</p>}
          </div>

          <div style={{ display: "flex", gap: 8, borderTop: "1px solid #eee", paddingTop: 10 }}>
            <input
              value={draft}
              onChange={(e) => setDraft(e.target.value)}
              onKeyDown={(e) => e.key === "Enter" && handleSend()}
              placeholder="Ask a follow-up question..."
              disabled={isResponding}
              style={{ flex: 1 }}
            />
            <Button onClick={handleSend} disabled={isResponding}>
              Send
            </Button>
          </div>
        </div>
      </div>
    </Layout>
  );
}
```

Both pages assume `useAlertsStore` exists with `alerts`, `loadInitialAnalysis`, and `askFollowUp` — that's the last missing piece. Want me to write that store now?
