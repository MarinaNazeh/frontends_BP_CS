Since you're back on plain React + TypeScript (Vite), I'll wire the Alerts page with `react-router-dom` and CSS Modules throughout. I'll also use local mock data for now so this runs standalone — swap it for your real store/API once that's ready.

## `components/Button/Button.tsx`

```tsx
import type { ButtonHTMLAttributes } from "react";
import styles from "./Button.module.css";

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "secondary" | "danger";
}

export function Button({ variant = "primary", className, children, ...rest }: ButtonProps) {
  return (
    <button
      className={`${styles.button} ${styles[variant]} ${className ?? ""}`}
      {...rest}
    >
      {children}
    </button>
  );
}
```

## `components/Button/Button.module.css`

```css
.button {
  border: none;
  border-radius: 6px;
  padding: 8px 14px;
  font-size: 13px;
  cursor: pointer;
  transition: opacity 0.15s ease;
}

.button:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}

.primary {
  background: #2563eb;
  color: #fff;
}

.secondary {
  background: #f1f1f1;
  color: #111;
}

.danger {
  background: #dc2626;
  color: #fff;
}
```

## `components/Header/Header.tsx`

```tsx
import styles from "./Header.module.css";

export function Header() {
  return (
    <header className={styles.header}>
      <span className={styles.title}>SOC Console</span>
    </header>
  );
}
```

## `components/Header/Header.module.css`

```css
.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 16px;
  border-bottom: 1px solid #eee;
  flex-shrink: 0;
}

.title {
  font-weight: 600;
  font-size: 15px;
}
```

## `components/Footer/Footer.tsx`

```tsx
import styles from "./Footer.module.css";

export function Footer() {
  return (
    <footer className={styles.footer}>
      <span>SOC Console — internal use only</span>
    </footer>
  );
}
```

## `components/Footer/Footer.module.css`

```css
.footer {
  padding: 8px 16px;
  font-size: 12px;
  color: #888;
  border-top: 1px solid #eee;
  flex-shrink: 0;
}
```

## Other components you'll need

Beyond Button/Header/Footer, the Outlook-style Alerts page needs two more:

- **`LogCard`** — the alert list item (title, severity, asset, click-to-select)
- **`ChatBubble`** — a single message in the analysis/follow-up thread

### `components/LogCard/LogCard.tsx`

```tsx
import styles from "./LogCard.module.css";

export type Severity = "critical" | "high" | "medium" | "low";

interface LogCardProps {
  title: string;
  asset: string;
  severity: Severity;
  isSelected?: boolean;
  onClick?: () => void;
}

export function LogCard({ title, asset, severity, isSelected, onClick }: LogCardProps) {
  return (
    <div
      className={`${styles.card} ${styles[severity]} ${isSelected ? styles.selected : ""}`}
      onClick={onClick}
    >
      <p className={styles.title}>
        {severity.toUpperCase()} · {title}
      </p>
      <p className={styles.asset}>{asset}</p>
    </div>
  );
}
```

### `components/LogCard/LogCard.module.css`

```css
.card {
  padding: 10px;
  border-radius: 8px;
  cursor: pointer;
  margin-bottom: 6px;
  border: 1px solid transparent;
}

.title {
  margin: 0;
  font-size: 12px;
  font-weight: 600;
}

.asset {
  margin: 2px 0 0;
  font-size: 11px;
  color: #666;
}

.selected {
  border: 1px solid #6366f1;
  background: #eef2ff !important;
}

.critical {
  background: #fee2e2;
}
.critical .title {
  color: #b91c1c;
}

.high {
  background: #ffedd5;
}
.high .title {
  color: #c2410c;
}

.medium {
  background: #fef9c3;
}
.medium .title {
  color: #a16207;
}

.low {
  background: #dbeafe;
}
.low .title {
  color: #1d4ed8;
}
```

### `components/ChatBubble/ChatBubble.tsx`

```tsx
import styles from "./ChatBubble.module.css";

export type MessageRole = "assistant" | "user";

interface ChatBubbleProps {
  role: MessageRole;
  content: string;
}

export function ChatBubble({ role, content }: ChatBubbleProps) {
  return (
    <div className={`${styles.bubble} ${role === "user" ? styles.user : styles.assistant}`}>
      {content}
    </div>
  );
}
```

### `components/ChatBubble/ChatBubble.module.css`

```css
.bubble {
  max-width: 80%;
  border-radius: 8px;
  padding: 8px 12px;
  font-size: 13px;
  line-height: 1.4;
}

.assistant {
  align-self: flex-start;
  background: #f4f4f4;
}

.user {
  align-self: flex-end;
  background: #e6f0ff;
}
```

## `pages/Alerts/Alerts.tsx`

Single page handling both the list and the detail pane, using `:id` from the URL so it behaves like Outlook (list stays mounted, right pane changes):

```tsx
import { useEffect, useState } from "react";
import { useParams, useNavigate } from "react-router-dom";
import { LogCard, type Severity } from "../../components/LogCard/LogCard";
import { ChatBubble, type MessageRole } from "../../components/ChatBubble/ChatBubble";
import { Button } from "../../components/Button/Button";
import styles from "./Alerts.module.css";

interface ChatMessage {
  id: string;
  role: MessageRole;
  content: string;
}

interface Alert {
  id: string;
  title: string;
  asset: string;
  sourceIp?: string;
  severity: Severity;
  messages: ChatMessage[];
}

// Mock data — replace with your store/API once wired up
const MOCK_ALERTS: Alert[] = [
  {
    id: "1",
    title: "Brute force login attempt",
    asset: "web-srv-03",
    sourceIp: "185.220.101.4",
    severity: "critical",
    messages: [
      {
        id: "m1",
        role: "assistant",
        content:
          "1. Confirm source IP against threat intel feeds.\n2. Check if account lockout policy triggered.\n3. Block source IP if confirmed malicious.",
      },
    ],
  },
  {
    id: "2",
    title: "Malware detected",
    asset: "host-21",
    severity: "high",
    messages: [
      { id: "m2", role: "assistant", content: "Isolate the host and run a full AV scan." },
    ],
  },
  {
    id: "3",
    title: "Unusual DNS query",
    asset: "dns-gw",
    severity: "medium",
    messages: [{ id: "m3", role: "assistant", content: "Review query pattern against baseline traffic." }],
  },
];

export function AlertsPage() {
  const { id } = useParams<{ id: string }>();
  const navigate = useNavigate();

  const [alerts] = useState<Alert[]>(MOCK_ALERTS);
  const [draft, setDraft] = useState("");
  const [isResponding, setIsResponding] = useState(false);

  const selectedAlert = alerts.find((a) => a.id === id) ?? null;

  useEffect(() => {
    setDraft("");
  }, [id]);

  async function handleSend() {
    if (!draft.trim() || !selectedAlert) return;
    const question = draft.trim();
    setDraft("");
    setIsResponding(true);

    // TODO: replace with real LLM call, e.g. POST /api/chat
    await new Promise((resolve) => setTimeout(resolve, 600));
    selectedAlert.messages.push({ id: crypto.randomUUID(), role: "user", content: question });
    selectedAlert.messages.push({
      id: crypto.randomUUID(),
      role: "assistant",
      content: "This is a placeholder response — wire this up to your LLM endpoint.",
    });

    setIsResponding(false);
  }

  return (
    <div className={styles.container}>
      <div className={styles.list}>
        {alerts.map((alert) => (
          <LogCard
            key={alert.id}
            title={alert.title}
            asset={alert.asset}
            severity={alert.severity}
            isSelected={alert.id === id}
            onClick={() => navigate(`/alerts/${alert.id}`)}
          />
        ))}
      </div>

      <div className={styles.detail}>
        {!selectedAlert ? (
          <div className={styles.emptyState}>
            <p>Select an alert to see its analysis.</p>
          </div>
        ) : (
          <>
            <div className={styles.detailHeader}>
              <div>
                <h3 className={styles.detailTitle}>{selectedAlert.title}</h3>
                <p className={styles.detailMeta}>
                  {selectedAlert.asset}
                  {selectedAlert.sourceIp ? ` · ${selectedAlert.sourceIp}` : ""}
                </p>
              </div>
              <span className={`${styles.severityTag} ${styles[selectedAlert.severity]}`}>
                {selectedAlert.severity}
              </span>
            </div>

            <div className={styles.messages}>
              {selectedAlert.messages.map((msg) => (
                <ChatBubble key={msg.id} role={msg.role} content={msg.content} />
              ))}
              {isResponding && <p className={styles.thinking}>Thinking...</p>}
            </div>

            <div className={styles.inputRow}>
              <input
                className={styles.input}
                value={draft}
                onChange={(e) => setDraft(e.target.value)}
                onKeyDown={(e) => e.key === "Enter" && handleSend()}
                placeholder="Ask a follow-up question..."
                disabled={isResponding}
              />
              <Button onClick={handleSend} disabled={isResponding}>
                Send
              </Button>
            </div>
          </>
        )}
      </div>
    </div>
  );
}
```

## `pages/Alerts/Alerts.module.css`

```css
.container {
  display: flex;
  height: 100%;
}

.list {
  width: 280px;
  border-right: 1px solid #eee;
  padding: 12px;
  overflow: auto;
  flex-shrink: 0;
}

.detail {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding: 16px;
  overflow: hidden;
}

.emptyState {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #888;
}

.detailHeader {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  border-bottom: 1px solid #eee;
  padding-bottom: 10px;
}

.detailTitle {
  margin: 0;
  font-size: 15px;
}

.detailMeta {
  margin: 2px 0 0;
  font-size: 12px;
  color: #666;
}

.severityTag {
  font-size: 12px;
  padding: 4px 10px;
  border-radius: 6px;
  text-transform: capitalize;
}

.severityTag.critical { background: #fee2e2; color: #b91c1c; }
.severityTag.high { background: #ffedd5; color: #c2410c; }
.severityTag.medium { background: #fef9c3; color: #a16207; }
.severityTag.low { background: #dbeafe; color: #1d4ed8; }

.messages {
  flex: 1;
  overflow: auto;
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 12px 0;
}

.thinking {
  font-size: 12px;
  color: #888;
}

.inputRow {
  display: flex;
  gap: 8px;
  border-top: 1px solid #eee;
  padding-top: 10px;
}

.input {
  flex: 1;
  padding: 8px 10px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 13px;
}
```

## `App.tsx`

```tsx
import { BrowserRouter, Routes, Route, Navigate } from "react-router-dom";
import { Header } from "./components/Header/Header";
import { Footer } from "./components/Footer/Footer";
import { AlertsPage } from "./pages/Alerts/Alerts";
import "./App.css";

function App() {
  return (
    <BrowserRouter>
      <div className="app-shell">
        <Header />
        <main className="app-main">
          <Routes>
            <Route path="/" element={<Navigate to="/alerts" replace />} />
            <Route path="/alerts" element={<AlertsPage />} />
            <Route path="/alerts/:id" element={<AlertsPage />} />
          </Routes>
        </main>
        <Footer />
      </div>
    </BrowserRouter>
  );
}

export default App;
```

## `App.css`

```css
.app-shell {
  display: flex;
  flex-direction: column;
  height: 100vh;
}

.app-main {
  flex: 1;
  overflow: hidden;
}
```

## `index.tsx`

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import App from "./App";
import "./index.css";

const rootElement = document.getElementById("root");

if (!rootElement) {
  throw new Error("Root element not found");
}

createRoot(rootElement).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

## `index.css`

```css
* {
  box-sizing: border-box;
}

html, body, #root {
  height: 100%;
  margin: 0;
  padding: 0;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  color: #111;
  background: #fff;
}
```

Don't forget to install the router:

```bash
npm install react-router-dom
```

Two things to swap out when you're ready to make this real:
1. **Replace `MOCK_ALERTS`** with your actual parsed-and-categorized alerts (from the CSV/JSON import flow) — probably lifted into a store or context so `App.tsx` and other pages can access them too.
2. **Replace the `setTimeout` placeholder in `handleSend`** with a real call to your LLM backend endpoint.

Want me to write the import flow (file upload → parse → categorize → populate this alerts list) next, since that's the missing link between what you have now and real data?
