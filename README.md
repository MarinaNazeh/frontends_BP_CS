Here's the restructured version — the detail pane now shows static alert info, and the chat moves into a floating popup triggered by a bubble button at the bottom-right.

## `pages/Alerts/Alerts.tsx`

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
  status: string;
  timestamp: string;
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
    status: "New",
    timestamp: "2 min ago",
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
    status: "New",
    timestamp: "9 min ago",
    messages: [
      { id: "m2", role: "assistant", content: "Isolate the host and run a full AV scan." },
    ],
  },
  {
    id: "3",
    title: "Unusual DNS query",
    asset: "dns-gw",
    severity: "medium",
    status: "New",
    timestamp: "22 min ago",
    messages: [{ id: "m3", role: "assistant", content: "Review query pattern against baseline traffic." }],
  },
];

export function AlertsPage() {
  const { id } = useParams<{ id: string }>();
  const navigate = useNavigate();

  const [alerts] = useState<Alert[]>(MOCK_ALERTS);
  const [draft, setDraft] = useState("");
  const [isResponding, setIsResponding] = useState(false);
  const [isChatOpen, setIsChatOpen] = useState(false);

  const selectedAlert = alerts.find((a) => a.id === id) ?? null;

  useEffect(() => {
    setDraft("");
    setIsChatOpen(false); // close popup when switching alerts
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
            <p>Select an alert to see its details.</p>
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

            <div className={styles.infoGrid}>
              <div className={styles.infoRow}>
                <span className={styles.infoLabel}>Status</span>
                <span className={styles.infoValue}>{selectedAlert.status}</span>
              </div>
              <div className={styles.infoRow}>
                <span className={styles.infoLabel}>Asset</span>
                <span className={styles.infoValue}>{selectedAlert.asset}</span>
              </div>
              {selectedAlert.sourceIp && (
                <div className={styles.infoRow}>
                  <span className={styles.infoLabel}>Source IP</span>
                  <span className={styles.infoValue}>{selectedAlert.sourceIp}</span>
                </div>
              )}
              <div className={styles.infoRow}>
                <span className={styles.infoLabel}>Detected</span>
                <span className={styles.infoValue}>{selectedAlert.timestamp}</span>
              </div>
              <div className={styles.infoRow}>
                <span className={styles.infoLabel}>Severity</span>
                <span className={styles.infoValue}>{selectedAlert.severity}</span>
              </div>
            </div>
          </>
        )}
      </div>

      {/* Floating chat bubble trigger */}
      {selectedAlert && (
        <button
          className={styles.chatFab}
          onClick={() => setIsChatOpen((open) => !open)}
          aria-label="Toggle alert chat"
        >
          💬
        </button>
      )}

      {/* Chat popup */}
      {selectedAlert && isChatOpen && (
        <div className={styles.chatPopup}>
          <div className={styles.chatPopupHeader}>
            <span>{selectedAlert.title}</span>
            <button className={styles.chatPopupClose} onClick={() => setIsChatOpen(false)} aria-label="Close chat">
              ✕
            </button>
          </div>

          <div className={styles.chatPopupMessages}>
            {selectedAlert.messages.map((msg) => (
              <ChatBubble key={msg.id} role={msg.role} content={msg.content} />
            ))}
            {isResponding && <p className={styles.thinking}>Thinking...</p>}
          </div>

          <div className={styles.chatPopupInputRow}>
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
        </div>
      )}
    </div>
  );
}
```

## `pages/Alerts/Alerts.module.css`

```css
.container {
  display: flex;
  height: 100%;
  position: relative; /* anchors the fixed chat popup/fab to this page */
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
  overflow: auto;
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

/* Static alert info panel */
.infoGrid {
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding-top: 14px;
}

.infoRow {
  display: flex;
  justify-content: space-between;
  font-size: 13px;
  padding: 8px 0;
  border-bottom: 1px solid #f2f2f2;
}

.infoLabel {
  color: #888;
}

.infoValue {
  font-weight: 500;
}

/* Floating chat bubble button */
.chatFab {
  position: fixed;
  bottom: 24px;
  right: 24px;
  width: 52px;
  height: 52px;
  border-radius: 50%;
  border: none;
  background: #2563eb;
  color: #fff;
  font-size: 22px;
  cursor: pointer;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
  z-index: 50;
}

.chatFab:hover {
  background: #1d4ed8;
}

/* Chat popup window */
.chatPopup {
  position: fixed;
  bottom: 88px;
  right: 24px;
  width: 340px;
  height: 440px;
  background: #fff;
  border-radius: 10px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.18);
  display: flex;
  flex-direction: column;
  overflow: hidden;
  z-index: 50;
}

.chatPopupHeader {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 12px;
  border-bottom: 1px solid #eee;
  font-size: 13px;
  font-weight: 600;
}

.chatPopupClose {
  border: none;
  background: transparent;
  cursor: pointer;
  font-size: 14px;
  color: #888;
}

.chatPopupMessages {
  flex: 1;
  overflow: auto;
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 12px;
}

.thinking {
  font-size: 12px;
  color: #888;
}

.chatPopupInputRow {
  display: flex;
  gap: 8px;
  border-top: 1px solid #eee;
  padding: 10px;
}

.input {
  flex: 1;
  padding: 8px 10px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 13px;
}
```

## What changed

- **Detail pane** (`.detail`) now shows a static info grid (status, asset, source IP, timestamp, severity) instead of the chat thread.
- **`.chatFab`** — a fixed circular button, bottom-right of the page, only shown once an alert is selected.
- **`.chatPopup`** — the same message list + input from before, now floating above the FAB rather than inline, toggled by `isChatOpen`.
- Switching alerts (`useEffect` on `id`) auto-closes the popup so you don't accidentally send a follow-up to the wrong alert's thread.

One thing worth deciding: right now the popup closes when you navigate to a different alert. If you'd rather it **stay open and just swap to the new alert's thread** as you click through the list (more like a persistent chat widget), that's a one-line change — just remove `setIsChatOpen(false)` from that `useEffect`. Want that instead?
