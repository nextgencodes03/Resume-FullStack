# Frontend Integration Handoff — TNPP RAG Chatbot API changes

Document version 1.1 — 2026-08-22 — For the TNPolicyPortal frontend team

## 1. Purpose

This document lists everything that changed on the chatbot API since the frontend last integrated with it, with exactly what the frontend needs to implement and what is fully backward-compatible and needs no frontend change.

### 1.1 Status: Ready for Integration

**The live deployment has been tested end-to-end, including the two issues an earlier version of this document flagged as known live-environment problems — both have since been found, fixed, and re-verified live (see Section 4). There are no known open issues blocking frontend integration as of this version.**

### 1.2 Live Endpoint

| Item | Value |
|---|---|
| Base URL | `http://13.51.17.38` |
| Primary endpoint | `POST /chat` |

*This is a live, EC2-hosted deployment of the corpus and API described below. Treat it as the integration-testing target. The address is an AWS Elastic IP (stable across the instance being stopped and started), not the instance's default dynamic address — it will not change on its own.*

---

## 2. Action Required: Conversation / Session Support

`POST /chat` now supports multi-turn conversations. This is the main frontend implementation item.

### 2.1 What Changed

| Field | Where | Behavior |
|---|---|---|
| `conversation_id` | Request (optional) | Omit on the first message of a new conversation. To continue an existing conversation, echo back the `conversation_id` value from the previous response. |
| `conversation_id` | Response (always present) | Every response includes a `conversation_id` — either newly generated (first message) or the same one that was sent in the request (follow-up). |

### 2.2 What the Frontend Needs to Do

1. Store the `conversation_id` returned by the first `/chat` response in the current chat session's state (e.g. component state, not necessarily persisted beyond the browser session).
2. Include that `conversation_id` in the request body for every subsequent message in the same visible conversation thread.
3. When the user starts a new conversation (e.g. clicking "New chat"), simply stop sending a `conversation_id` — the next request will be treated as a fresh conversation and a new id will come back.
4. No special handling is needed for an invalid or expired `conversation_id` — the API treats it as the start of a new conversation rather than returning an error, so the frontend does not need to catch or distinguish this case.

### 2.3 Example Exchange

```
Request 1:  { "message": "What is the Health Policy Vision 2030?" }
Response 1: { ..., "conversation_id": "31c37a76-..." }

Request 2:  { "message": "Summarize that in one sentence.", "conversation_id": "31c37a76-..." }
Response 2: { ..., "conversation_id": "31c37a76-..." }  (same id, and the answer correctly stays on-topic)
```

### 2.4 Known Behavior to Be Aware Of (not a frontend concern, informational)

A conversation's history currently expires after a period of inactivity (30 minutes by default) and only the most recent few turns are kept — the frontend does not need to manage this, it happens automatically. Also, a follow-up message is answered with the benefit of conversation history, but the underlying document search for that message still runs on the follow-up text alone. In practice this means a very bare follow-up with no topic words at all may occasionally retrieve less precisely than a follow-up that restates a little context — not a defect to fix on the frontend side, just useful to know if a follow-up answer ever seems to have drifted off-topic.

---

## 3. No Frontend Change Needed (informational only)

### 3.1 CORS

The API now restricts which origins may call it directly, rather than accepting any origin. The production TNPolicyPortal origin and the local Angular dev server origin (`http://localhost:4200`) are both already allowed. If the frontend is ever served from a different origin than `https://tnpolicy.in` (e.g. a new staging domain), that origin will need to be added to the API's allowlist — flag this to the backend team rather than expecting it to work automatically.

### 3.2 Rate Limiting

The API now limits each caller (by IP address) to a small number of `/chat` requests per minute. If the frontend ever fires rapid, automated requests (e.g. a test script looping quickly), it may receive HTTP `429 Too Many Requests` — this is expected behavior, not a bug, at the current deliberately conservative limit. Real user interaction (typing and sending messages one at a time) is very unlikely to hit it.

### 3.3 Source / Citation Object Shape — Unchanged in Structure, Worth Re-Confirming

Each item in the `sources` array still has the same shape as before: `title`, `url` (nullable), `category` (nullable), `filename` (nullable), and `download` (nullable, a structured POST action for sources served via the existing S3-fetch flow). What's worth re-confirming on the frontend side: a source may now resolve either via the existing `download` action (S3-backed sources) or via a direct `url` pointing at this API's own document endpoint (sources with no S3 file) — the frontend's existing logic for "if `download` present, use it; else if `url` present, use it" should already handle both cases without change, but this is the first time `url` has been populated with a same-origin-style API link rather than always being either absent or a full S3 link, so worth a quick visual check during integration testing.

---

## 4. Issues Found and Resolved Before Handoff (informational, no action needed)

Two issues were found while testing the EC2 deployment, and both are now fixed and re-verified live as of 2026-08-22. Neither was a frontend concern or an API contract change — both were deployment-environment problems on the backend side. Kept here for transparency and in case either resurfaces after a future redeploy.

### 4.1 [Resolved] Some document download links pointed at the wrong address

A configuration value on the EC2 instance was set to an internal address rather than the public one, so some source `url` values in `/chat` responses did not resolve for a real browser. Fixed by correcting the configuration and restarting the service. Re-verified: a live `/chat` response's source `url` now correctly starts with `http://13.51.17.38`, and fetching it returns the real file.

### 4.2 [Resolved] Some AI-reference-document downloads returned 404

Three of the AI-reference document category's subfolders (roughly 55% of the full corpus by chunk count) returned 404 on download, even though the chatbot could still search and answer from their content correctly. Root cause turned out to be a directory permission issue on the EC2 filesystem (a missing execute bit on those three folders specifically), not a missing file transfer as first suspected. Fixed by correcting the permissions. Re-verified: all three subfolders now serve their files correctly, confirmed by downloading a real file from each.

---

## 5. Unchanged / Not Part of This Handoff

- The core `POST /chat` request/response contract (`message`, `status`, `overview`, `sections`) is unchanged.
- `GET /health` and `GET /live` are unchanged.
- No authentication or API-key requirement has been added to `POST /chat` itself.
