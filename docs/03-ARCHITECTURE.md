# Architecture Document
# n8n Webhook Tester — Technical Architecture

---

## 1. Architecture Overview

This is a **client-side only** (no backend) single-page application. All logic runs in the user's browser.

```
┌─────────────────────────────────────────────────────┐
│                    USER'S BROWSER                    │
│                                                      │
│  ┌──────────────┐    ┌──────────────┐               │
│  │   UI Layer    │───▶│  App Logic   │               │
│  │  (HTML/CSS)   │◀───│  (JavaScript)│               │
│  └──────────────┘    └──────┬───────┘               │
│                             │                        │
│                    ┌────────▼────────┐               │
│                    │  Fetch API      │               │
│                    │  (HTTP Client)  │               │
│                    └────────┬────────┘               │
│                             │                        │
└─────────────────────────────┼────────────────────────┘
                              │  HTTP POST (JSON)
                              ▼
                    ┌──────────────────┐
                    │  n8n Webhook     │
                    │  (External)      │
                    └──────────────────┘
```

**Key architectural decision:** No server, no build step, no framework dependency. The entire app is a single `.html` file (or a single React `.jsx` artifact) that can be opened directly in a browser.

---

## 2. Technology Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| **Markup** | HTML5 | Semantic, accessible, no build needed |
| **Styling** | CSS3 (embedded) | CSS variables for theming, no preprocessor needed |
| **Logic** | Vanilla JavaScript (ES6+) OR React (JSX) | Fetch API for HTTP, no dependencies |
| **Fonts** | Google Fonts CDN (JetBrains Mono) | Single font load, cached by browser |
| **HTTP Client** | `fetch()` API | Built into every modern browser |
| **Deployment** | Static file | Open locally, or deploy to any static host |

### Why no framework?

For a single form with 3 response states, a framework adds complexity without benefit. If deploying as a Claude artifact (React JSX), React is available by default. Either approach works — the architecture is the same.

---

## 3. Component Architecture

Even without a framework, the code is organized into logical components:

```
App
├── FormComponent
│   ├── WebhookUrlInput
│   ├── MessagePayloadTextarea
│   └── SubmitButton
├── ResultComponent
│   ├── SuccessDisplay
│   ├── TimerDisplay (countdown)
│   └── ErrorDisplay
└── Utilities
    ├── validateUrl()
    ├── sendWebhookRequest()
    └── formatErrorMessage()
```

### Component Responsibilities

| Component | Responsibility |
|-----------|---------------|
| `FormComponent` | Captures user input, validates fields, manages form state |
| `ResultComponent` | Displays the outcome of the POST request in one of 3 states |
| `SubmitButton` | Manages its own loading/disabled state |
| `sendWebhookRequest()` | Core logic — sends fetch, handles all 3 response scenarios |

---

## 4. Data Flow

### 4.1 Happy Path (Scenario 5a: Success with body)

```
User fills form
      │
      ▼
User clicks "Test Webhook"
      │
      ▼
validateInputs()
      │ ✅ valid
      ▼
Set UI to LOADING state
      │
      ▼
fetch(webhookUrl, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ message: payload })
})
      │
      ▼
Response received (HTTP 200)
      │
      ▼
Read response body → response.text()
      │
      ▼
Body is NOT empty?
      │ ✅ yes
      ▼
Set UI to SUCCESS state
Show: "Message received – webhook is working with n8n workflow"
```

### 4.2 Empty Body Path (Scenario 5b)

```
Response received (HTTP 200)
      │
      ▼
Body IS empty
      │
      ▼
Set UI to TIMER state
Start 10-second countdown
      │
      ▼
Timer expires (10s)
      │
      ▼
Set UI to ERROR state
Show: "n8n workflow and webhook is not responding properly"
```

### 4.3 Error Path (Scenario 5c)

```
fetch() throws error (network)
  OR
Response status is 4xx/5xx
      │
      ▼
Set UI to ERROR state (immediately)
Show: "n8n workflow and webhook is not responding properly"
Show: error details (status code, message, or network error)
```

---

## 5. State Machine

The app has exactly 5 states. Only valid transitions are shown.

```
                ┌──────────┐
                │   IDLE   │ ◀──────────────────────┐
                └────┬─────┘                         │
                     │ click "Send"                   │
                     ▼                                │
                ┌──────────┐                         │
                │ LOADING  │                         │
                └────┬─────┘                         │
                     │                                │
           ┌─────────┼──────────┐                    │
           │         │          │                    │
           ▼         ▼          ▼                    │
      ┌────────┐ ┌────────┐ ┌────────┐              │
      │SUCCESS │ │WAITING │ │ ERROR  │              │
      └────┬───┘ └────┬───┘ └────┬───┘              │
           │          │          │                    │
           │          │ timeout  │                    │
           │          ▼          │                    │
           │     ┌────────┐     │                    │
           │     │ ERROR  │     │                    │
           │     └────┬───┘     │                    │
           │          │         │                    │
           └──────────┴─────────┘                    │
                     │                                │
                     │ user modifies input            │
                     └────────────────────────────────┘
```

### State Definitions

| State | UI | Button | Result Panel |
|-------|----|---------| ------------|
| `IDLE` | Form ready | Enabled: "Test Webhook" | Hidden |
| `LOADING` | Form locked | Disabled: "Sending..." + spinner | Hidden |
| `SUCCESS` | Form unlocked | Enabled: "Test Webhook" | Green success message |
| `WAITING` | Form locked | Disabled | Amber countdown timer |
| `ERROR` | Form unlocked | Enabled: "Test Webhook" | Red error message + details |

---

## 6. API Contract

### Outgoing Request (from app to n8n webhook)

```http
POST {user-provided-webhook-url}
Content-Type: application/json

{
  "message": "string — the user's payload text"
}
```

### Expected Responses

| Scenario | Status | Body | App Behavior |
|----------|--------|------|-------------|
| Workflow active & responding | 200 | Non-empty (any format) | → SUCCESS |
| Workflow active but no "Respond to Webhook" node | 200 | Empty string `""` | → WAITING → ERROR after 10s |
| Workflow inactive or URL wrong | 404 | Any | → ERROR immediately |
| n8n server error | 500 | Any | → ERROR immediately |
| Network unreachable / CORS blocked | N/A | N/A | → ERROR immediately |

---

## 7. Error Handling Strategy

| Error Type | Detection Method | User Message |
|------------|-----------------|--------------|
| **Invalid URL format** | Regex / URL constructor in JS | "Please enter a valid URL" (inline, before send) |
| **Empty message** | `input.value.trim() === ''` | "Please enter a message" (inline) |
| **Network error** | `fetch().catch(err)` | Error message + `err.message` (often "Failed to fetch") |
| **CORS error** | `fetch().catch()` — browsers report as TypeError | Error message + hint: "This may be a CORS issue. Ensure your n8n webhook allows cross-origin requests." |
| **HTTP 4xx/5xx** | `!response.ok` | Error message + `response.status` + `response.statusText` |
| **Empty body timeout** | `setTimeout(10000)` | Error message (timer expired) |

### CORS Special Handling

CORS errors are indistinguishable from network errors in the Fetch API (by browser security design). The app will:
1. Catch the error
2. Check if `err.message` includes "Failed to fetch" or similar
3. Add a helpful hint about CORS alongside the error

---

## 8. Security Considerations

| Concern | Mitigation |
|---------|-----------|
| **XSS via response body** | Render response in `<pre>` with `textContent` (not `innerHTML`) |
| **Sensitive webhook URLs** | URLs are never stored, logged, or sent anywhere except the user's target |
| **No backend** | No server to compromise; no data leaves the browser except to the user's webhook |
| **HTTPS** | The app itself can run locally; webhook URLs should be HTTPS (validated with a warning) |

---

## 9. Performance Budget

| Metric | Target | How |
|--------|--------|-----|
| HTML file size | < 15 KB | Single file, no framework |
| Font load | < 50 KB | Single weight of JetBrains Mono |
| Time to interactive | < 1 second | No JS bundles to parse |
| Lighthouse score | 95+ | Semantic HTML, minimal CSS, no render-blocking scripts |

---

## 10. Deployment Options

Since this is a static single-file app:

| Option | Cost | Complexity |
|--------|------|-----------|
| Open `.html` file locally | Free | None |
| GitHub Pages | Free | Push to repo |
| Netlify / Vercel static | Free tier | Drag and drop |
| Claude Artifact (React JSX) | Free | Already deployed as artifact |

---

## 11. Folder Structure

For the simplest version (single HTML file):

```
n8n-webhook-tester/
└── index.html          # Everything: HTML + CSS + JS
```

For a slightly more organized version:

```
n8n-webhook-tester/
├── index.html          # HTML structure
├── style.css           # Styles
├── app.js              # Application logic
└── README.md           # Usage instructions
```

For this project, we'll use the **single-file approach** since it matches the requirement of a single-page app with zero build steps.
