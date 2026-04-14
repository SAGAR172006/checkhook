# Implementation Guide
# n8n Webhook Tester — Step-by-Step Build Plan

---

## 1. Implementation Overview

This document is the developer's blueprint for building the n8n Webhook Tester. It translates the PRD, Design, and Architecture documents into concrete coding steps.

**Estimated build time:** 45–60 minutes  
**Output:** A single `.html` file (or React `.jsx` artifact) that is fully functional.

---

## 2. Pre-Implementation Checklist

- [ ] Read the PRD (requirements are clear)
- [ ] Read the Design doc (know what it looks like)
- [ ] Read the Architecture doc (know how data flows)
- [ ] Decide on format: **Single HTML file** (simplest) or **React JSX** (if deploying as Claude artifact)
- [ ] Have a test n8n webhook URL ready for validation

---

## 3. Implementation Phases

### Phase 1: HTML Structure (10 minutes)

**Goal:** Build the semantic skeleton of the page.

#### Steps:

1. Create `index.html` with HTML5 doctype
2. Add `<meta>` tags for charset, viewport (responsive), and description
3. Link Google Fonts (JetBrains Mono) in `<head>`
4. Build the page structure:

```html
<body>
  <main class="container">
    <header class="header">
      <!-- Title + subtitle -->
    </header>

    <section class="form-section">
      <!-- URL input with label -->
      <!-- Message textarea with label -->
      <!-- Submit button -->
    </section>

    <section class="result-section" id="result" aria-live="polite" hidden>
      <!-- Dynamic result content injected by JS -->
    </section>

    <footer class="footer">
      <!-- One-liner credit/description -->
    </footer>
  </main>
</body>
```

#### Key HTML Details:

| Element | Attribute Notes |
|---------|----------------|
| URL input | `type="url"`, `required`, `id="webhookUrl"`, `autocomplete="url"` |
| Textarea | `required`, `id="messagePayload"`, `rows="3"` |
| Button | `type="submit"` if inside `<form>`, or `type="button"` with onclick |
| Result section | `aria-live="polite"` for screen reader announcements |

#### Checkpoint:
> Open the file in a browser. You should see unstyled text, inputs, and a button. Everything should be readable and all inputs should be tab-navigable.

---

### Phase 2: CSS Styling (15 minutes)

**Goal:** Apply the design system from the Design Document.

#### Steps:

1. Define CSS custom properties (variables) in `:root`
2. Apply base styles: reset, body background, font
3. Style the card container (centered, max-width, padding, border-radius)
4. Style inputs (background, border, focus states, transition)
5. Style the button (colors, hover, active, disabled, loading states)
6. Style the result panel (3 variants: success, waiting, error)
7. Add responsive breakpoints
8. Add animations (fade-in, slide-down, countdown pulse)

#### CSS Variables Block:

```css
:root {
  --bg-primary: #0a0a0f;
  --bg-card: #12121a;
  --bg-input: #1a1a2e;
  --text-primary: #e4e4e7;
  --text-muted: #71717a;
  --accent: #f97316;
  --accent-hover: #ea580c;
  --success: #22c55e;
  --error: #ef4444;
  --warning: #eab308;
  --border: #27272a;
  --radius: 8px;
  --font-mono: 'JetBrains Mono', 'Fira Code', monospace;
  --font-sans: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}
```

#### Animation Definitions:

```css
@keyframes fadeSlideIn {
  from { opacity: 0; transform: translateY(12px); }
  to   { opacity: 1; transform: translateY(0); }
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50%      { opacity: 0.5; }
}

@keyframes shake {
  0%, 100% { transform: translateX(0); }
  20%, 60% { transform: translateX(-4px); }
  40%, 80% { transform: translateX(4px); }
}
```

#### Checkpoint:
> The page should now look polished. Dark background, styled card, orange button, proper spacing. Resize the browser window — everything should remain usable at 360px width.

---

### Phase 3: JavaScript Logic (20 minutes)

**Goal:** Implement the core functionality: validate, send, handle responses.

#### Step 3.1: Input Validation

```javascript
function validateUrl(urlString) {
  try {
    const url = new URL(urlString);
    return url.protocol === 'http:' || url.protocol === 'https:';
  } catch {
    return false;
  }
}

function validateForm(url, message) {
  const errors = [];
  if (!validateUrl(url)) errors.push('Please enter a valid webhook URL');
  if (!message.trim()) errors.push('Please enter a message payload');
  return errors;
}
```

#### Step 3.2: Core Webhook Request Function

This is the heart of the app. It implements all 3 scenarios from the PRD.

```javascript
async function sendWebhookRequest(webhookUrl, message) {
  // Update UI → LOADING state
  setUIState('loading');

  try {
    const response = await fetch(webhookUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ message: message }),
    });

    // SCENARIO 5c: HTTP error status
    if (!response.ok) {
      setUIState('error', {
        message: 'n8n workflow and webhook is not responding properly',
        details: `HTTP ${response.status}: ${response.statusText}`
      });
      return;
    }

    // HTTP 2xx — now check the body
    const responseBody = await response.text();

    if (responseBody && responseBody.trim().length > 0) {
      // SCENARIO 5a: Success with body
      setUIState('success', {
        message: 'Message received – webhook is working with n8n workflow',
        body: responseBody
      });
    } else {
      // SCENARIO 5b: Success but empty body → start timer
      setUIState('waiting');
      startCountdown(10, () => {
        setUIState('error', {
          message: 'n8n workflow and webhook is not responding properly',
          details: 'Webhook returned an empty response after 10 seconds.'
        });
      });
    }

  } catch (error) {
    // SCENARIO 5c: Network error (includes CORS)
    let details = error.message || 'Unknown network error';
    if (details.includes('Failed to fetch')) {
      details += '\n\nHint: This may be a CORS issue. '
        + 'Ensure your n8n instance allows cross-origin requests, '
        + 'or test from the same domain as your n8n server.';
    }
    setUIState('error', {
      message: 'n8n workflow and webhook is not responding properly',
      details: details
    });
  }
}
```

#### Step 3.3: UI State Manager

```javascript
function setUIState(state, data = {}) {
  const resultEl = document.getElementById('result');
  const button = document.getElementById('submitBtn');

  // Reset
  resultEl.hidden = false;
  resultEl.className = 'result-section';

  switch (state) {
    case 'loading':
      resultEl.hidden = true;
      button.disabled = true;
      button.innerHTML = '<span class="spinner"></span> Sending...';
      break;

    case 'success':
      resultEl.classList.add('result-success');
      resultEl.innerHTML = `
        <span class="result-icon">✅</span>
        <p class="result-message">${data.message}</p>
        ${data.body ? `
          <details class="response-details">
            <summary>View Response</summary>
            <pre class="response-body"></pre>
          </details>
        ` : ''}
      `;
      // Set response body safely (XSS prevention)
      if (data.body) {
        resultEl.querySelector('.response-body').textContent = data.body;
      }
      resetButton();
      break;

    case 'waiting':
      resultEl.classList.add('result-waiting');
      resultEl.innerHTML = `
        <span class="result-icon">⏳</span>
        <p class="result-message">
          Waiting for response...
          <span id="countdown" class="countdown">10</span>s remaining
        </p>
      `;
      break;

    case 'error':
      resultEl.classList.add('result-error');
      resultEl.innerHTML = `
        <span class="result-icon">❌</span>
        <p class="result-message">${data.message}</p>
        ${data.details ? `
          <pre class="error-details"></pre>
        ` : ''}
      `;
      // Set error details safely (XSS prevention)
      if (data.details) {
        resultEl.querySelector('.error-details').textContent = data.details;
      }
      resetButton();
      break;
  }
}
```

#### Step 3.4: Countdown Timer

```javascript
let countdownInterval = null;

function startCountdown(seconds, onExpire) {
  let remaining = seconds;
  const countdownEl = document.getElementById('countdown');

  countdownInterval = setInterval(() => {
    remaining--;
    if (countdownEl) countdownEl.textContent = remaining;

    if (remaining <= 0) {
      clearInterval(countdownInterval);
      countdownInterval = null;
      onExpire();
    }
  }, 1000);
}

// Important: Clear any running timer when user sends a new request
function clearExistingTimer() {
  if (countdownInterval) {
    clearInterval(countdownInterval);
    countdownInterval = null;
  }
}
```

#### Step 3.5: Event Wiring

```javascript
document.addEventListener('DOMContentLoaded', () => {
  const form = document.getElementById('webhookForm');
  const urlInput = document.getElementById('webhookUrl');
  const messageInput = document.getElementById('messagePayload');

  form.addEventListener('submit', (e) => {
    e.preventDefault();
    clearExistingTimer();

    const errors = validateForm(urlInput.value, messageInput.value);
    if (errors.length > 0) {
      // Show inline validation errors
      return;
    }

    sendWebhookRequest(urlInput.value.trim(), messageInput.value.trim());
  });
});
```

#### Checkpoint:
> Open the page, enter any valid URL (e.g., `https://httpbin.org/post`) and a message. Click "Test Webhook". You should see the loading spinner, then a success or error result.

---

### Phase 4: Testing (10 minutes)

#### Test Cases Table

| # | Test | Input | Expected Result |
|---|------|-------|-----------------|
| 1 | Valid webhook, active workflow with response | Real n8n webhook URL + message | ✅ Success message + response body visible |
| 2 | Valid URL, but no workflow or "Respond to Webhook" node | Real n8n webhook that returns empty | ⏳ Timer counts down → ❌ Error after 10s |
| 3 | Invalid URL format | `not-a-url` | Inline validation error, no request sent |
| 4 | Valid URL but server unreachable | `https://doesnotexist12345.com/webhook` | ❌ Network error with details |
| 5 | HTTP 404 | URL pointing to non-existent n8n endpoint | ❌ Error with "404 Not Found" |
| 6 | Empty message field | Leave textarea blank | Inline validation error |
| 7 | CORS blocked | URL to a server without CORS headers | ❌ Error with CORS hint |
| 8 | Rapid double-click | Click button twice fast | Button disabled during loading — only 1 request sent |
| 9 | Mobile viewport | Resize to 360px width | Everything visible and usable |
| 10 | Screen reader | Navigate with Tab + screen reader | Labels announced, result changes announced |

#### Quick Test with httpbin.org

For testing without a real n8n instance:

- **Success with body:** `https://httpbin.org/post` → returns a JSON body (scenario 5a)
- **HTTP error:** `https://httpbin.org/status/500` → returns 500 (scenario 5c)

---

### Phase 5: Polish & Edge Cases (5 minutes)

- [ ] Add `<noscript>` message: "This app requires JavaScript to function."
- [ ] Trim whitespace from URL input (prevent accidental spaces)
- [ ] Handle response body that's very long — truncate to 2000 chars with a "truncated" note
- [ ] Add `favicon` (optional: inline SVG data URI of a lightning bolt)
- [ ] Test that refreshing the page resets all state cleanly

---

## 4. Common Mistakes to Avoid

| Mistake | Why It's Bad | Prevention |
|---------|-------------|-----------|
| Using `innerHTML` with user/server data | XSS vulnerability | Always use `textContent` for dynamic data |
| Forgetting to `await response.text()` | Body reading is async; you'll get a Promise, not a string | Always await |
| Not clearing the timer on re-submit | Old timer can fire and overwrite new results | Call `clearExistingTimer()` at start of each submit |
| Assuming CORS errors give status codes | Browsers hide CORS details for security | Catch as generic network error + add hint |
| Hardcoding `"http://"` check | Misses `"https://"` or uppercase | Use `new URL()` constructor which handles all cases |

---

## 5. File Output

After implementation, the deliverable is:

```
n8n-webhook-tester/
└── index.html    # Single file, < 15 KB, opens in any browser
```

Or, if building as a Claude artifact:

```
webhook-tester.jsx    # Single React component with embedded styles
```

---

## 6. Future Enhancements (v2 Ideas)

These are NOT in scope for v1, but documented for future reference:

| Feature | Effort | Value |
|---------|--------|-------|
| Custom HTTP headers input | Medium | Supports authenticated webhooks |
| Request history (localStorage) | Medium | Saves past tests for reference |
| Import/export webhook configs | Low | Shareable test configurations |
| CORS proxy toggle | Medium | Solves cross-origin issues automatically |
| Multiple HTTP methods (GET, PUT) | Low | Broader testing capability |
| Syntax-highlighted JSON response | Low | Better readability for JSON responses |
| Dark/light theme toggle | Low | User preference |

---

## 7. Summary: What Gets Built

```
Single HTML file that:
  ├── Shows a form with 2 inputs (URL + message)
  ├── Sends a POST request with JSON body on submit
  ├── Handles 3 response scenarios:
  │   ├── ✅ 2xx + body → success message
  │   ├── ⏳ 2xx + empty → 10s timer → error
  │   └── ❌ 4xx/5xx/network → immediate error
  ├── Is fully responsive (360px to desktop)
  ├── Has zero dependencies (no npm, no build)
  └── Is secure (no innerHTML with dynamic data)
```
