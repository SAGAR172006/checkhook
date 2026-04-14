# Product Requirements Document (PRD)
# n8n Webhook Tester — Single Page Web App

---

## 1. Overview

**Product Name:** n8n Webhook Tester  
**Version:** 1.0  
**Date:** April 14, 2026  
**Author:** Student Developer  
**Status:** Draft  

### 1.1 Purpose

A lightweight, single-page web application that allows users to quickly test whether their n8n webhook workflows are responding correctly. Users provide a webhook URL and a text message, click "Send", and receive clear visual feedback about the webhook's health.

### 1.2 Problem Statement

When building n8n workflows, developers need a fast way to verify that their webhook triggers are:
- Reachable (no network errors)
- Returning a success HTTP status (2xx)
- Actually producing a response body (proof the workflow executed)

Currently, users resort to tools like Postman or cURL, which are overkill for a quick "is my webhook alive?" check. This app provides a purpose-built, zero-setup alternative.

### 1.3 Target Users

| User Type | Description |
|-----------|-------------|
| **Beginner n8n users** | Learning workflows, need a simple way to see if their webhook works |
| **Workflow developers** | Building/debugging automation, need rapid webhook feedback |
| **QA testers** | Validating webhook endpoints before deployment |

---

## 2. Functional Requirements

### FR-1: Webhook URL Input
- **Description:** A text input field where the user pastes their n8n webhook URL.
- **Validation:** Must be a valid URL format (starts with `http://` or `https://`).
- **Placeholder text:** `https://your-n8n-instance.com/webhook/xxxxx`

### FR-2: Message Payload Input
- **Description:** A text input (or textarea) where the user types the message payload.
- **Validation:** Must not be empty.
- **Placeholder text:** `Hello, n8n!`

### FR-3: Submit / Send Button
- **Description:** A button labeled "Send" or "Test Webhook" that triggers the POST request.
- **Behavior while sending:** Button shows a loading state (spinner or "Sending..." text) and is disabled to prevent duplicate requests.

### FR-4: POST Request Execution
- **Description:** On submit, the app sends an HTTP POST request to the provided webhook URL.
- **Request format:**
  ```json
  {
    "message": "<user's text>"
  }
  ```
- **Headers:** `Content-Type: application/json`
- **Method:** POST

### FR-5: Response Handling — Three Scenarios

| # | Condition | User Feedback |
|---|-----------|---------------|
| **5a** | HTTP 2xx + non-empty response body | ✅ Success: *"Message received – webhook is working with n8n workflow"* |
| **5b** | HTTP 2xx + empty response body | ⏳ Start a 10-second timer → then ❌ Error: *"n8n workflow and webhook is not responding properly"* |
| **5c** | HTTP 4xx/5xx or network error | ❌ Immediate error: *"n8n workflow and webhook is not responding properly"* + error details |

### FR-6: Response Details Display
- On success (5a): Optionally show the response body content in a collapsible section.
- On error (5b, 5c): Show HTTP status code, status text, or network error message.

### FR-7: Reset / Retry
- After any result, the user can modify inputs and send again without refreshing the page.

---

## 3. Non-Functional Requirements

| ID | Requirement | Detail |
|----|-------------|--------|
| **NFR-1** | Responsive | Works on mobile (360px+), tablet, and desktop |
| **NFR-2** | No backend | Runs entirely in the browser (static HTML/CSS/JS) |
| **NFR-3** | Fast load | < 1 second to interactive on 3G |
| **NFR-4** | Accessible | Proper labels, focus states, color contrast ≥ 4.5:1 |
| **NFR-5** | No dependencies required | Zero npm install; optionally use a CDN font |
| **NFR-6** | CORS-aware | Gracefully handle CORS errors with a helpful message |

---

## 4. Out of Scope (v1)

- User authentication or accounts
- Saving/bookmarking webhook URLs
- Request history or logging
- Support for other HTTP methods (GET, PUT, DELETE)
- Custom headers or auth tokens in requests
- Hosting/deployment (user opens the HTML file locally or deploys themselves)

---

## 5. Success Metrics

| Metric | Target |
|--------|--------|
| Time to first test | < 30 seconds from page load |
| Correct result display | 100% match to the 3 scenarios above |
| Mobile usability | Fully functional on 360px screen |

---

## 6. Assumptions

1. The user already has a running n8n instance with a webhook node configured.
2. The webhook endpoint has CORS headers enabled (common n8n default), OR the user is okay with CORS limitations being surfaced as errors.
3. The app will be a single `.html` file (or a single React `.jsx` component) with no build step required.
4. No server-side proxy — requests go directly from the browser to the webhook.

---

## 7. Open Questions

| # | Question | Impact |
|---|----------|--------|
| 1 | Should the app support custom request headers? | Could help with authenticated webhooks — deferred to v2 |
| 2 | Should we add a CORS proxy option? | Would solve cross-origin issues but adds complexity |
| 3 | Should response body be shown in full or truncated? | UX decision — current plan: collapsible, truncated to 500 chars |
