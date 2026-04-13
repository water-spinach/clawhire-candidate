---
name: clawhire-candidate
description: >
  Help your owner job-hunt on ClawHire via WeChat — SMS login plus a
  single chat-routed conversation that handles profile intake, job
  recommendations, role recommendations, and applying. All substantive
  owner interaction flows through /chat/profile-intake; direct REST is
  only used for auth, account metadata, speech preprocessing, profile
  activation, and optional snapshots. Turn-based text / voice / PDF.
  No streaming.
capabilities:
  - name: sms-onboarding
    description: Register or log in an owner via SMS MFA
    endpoint: /auth/sms/send-code + /auth/sms/verify
    method: POST
    triggers: ["注册", "登录", "login", "register", "sign in", "手机号"]
  - name: chat-loop
    description: >
      Main conversational path. Routes every owner turn through
      /chat/profile-intake and renders content_list / jobs / roles
      from the structured response. Covers profile intake, job
      recommendations, match-like ranked deck, role suggestions,
      and career chat.
    endpoint: /chat/profile-intake
    method: POST
    triggers: ["(any non-auth, non-admin owner message)"]
  - name: voice-input
    description: Transcribe a WeChat voice message and feed it into the chat loop
    endpoint: /speech/transcribe
    method: POST
    triggers: ["(voice message received)"]
  - name: pdf-resume
    description: Extract text from a WeChat PDF attachment and feed it into the chat loop
    endpoint: /chat/profile-intake (with <PDF_CV_CONTENT> wrapper)
    method: POST
    triggers: ["(PDF attachment received)"]
  - name: apply
    description: Apply to a job via chat action (no separate conversations/initiate call)
    endpoint: /chat/profile-intake (action=apply)
    method: POST
    triggers: ["投递", "申请", "apply", "投第一个"]
  - name: chat-history
    description: Fetch prior A2C session history for context resume
    endpoint: /chat/history
    method: POST
    triggers: ["我们之前聊到哪了", "上次聊到哪"]
  - name: extract-cv
    description: Read the server's current candidate draft snapshot
    endpoint: /chat/extract-cv
    method: GET
    triggers: ["确认简历", "看看我的简历", "show my cv"]
  - name: activate-profile
    description: Toggle profile visibility to recruiters (the ONLY PATCH still needed)
    endpoint: /candidates/profiles/{id}
    method: PATCH
    triggers: ["激活简历", "activate", "让招聘方看到我", "隐藏简历", "下架"]
  - name: view-profile
    description: Show the owner's saved profile snapshot
    endpoint: /candidates/profiles?per_page=1
    method: GET
    triggers: ["我的简历", "show my profile"]
  - name: account
    description: Show the owner's account info (phone masked)
    endpoint: /account
    method: GET
    triggers: ["我的账号", "account"]
env:
  - CLAWHIRE_BASE_URL   # e.g. https://metalink.cc/clawhire/api/v1
  - CLAWHIRE_PHONE      # owner's phone in E.164 form, e.g. +8613800001111
---

# ClawHire Candidate

## ⛔ STOP — Read before doing ANYTHING

**You MUST read BOTH files below BEFORE responding to your owner. Do NOT respond, ask questions, or take any action until you have read them completely.**

1. [references/workflows.md](references/workflows.md) — Exact HTTP calls for every action, numbered 1–10
2. [references/state.md](references/state.md) — What you must remember across turns (session token, last job deck, etc.)

**If you skip these files and answer on your own, you WILL give wrong answers.** The ClawHire A2C server handles profile intake, job recommendations, role suggestions, and apply — you are a proxy that routes owner messages in and relays structured responses out. You are not the interviewer, the career coach, or the job-matching engine.

## The one rule you must internalize

> **Every substantive owner message goes through workflow 2 (`POST /chat/profile-intake`).**
>
> Profile talk, background questions, "find me Java jobs in Shenzhen", "tell me about job #2", "investor pitch advice", "which role fits me", "apply to the first one" — all of it. You do NOT have a separate search endpoint, a separate match endpoint, or a separate apply endpoint. The chat endpoint returns `content_list`, `jobs`, `roles`, and `phase` in one structured response, and you render each field to the owner.

The only owner messages that bypass workflow 2 are:
- Pure auth codes → workflow 1
- Voice messages → workflow 3 (transcribe first, THEN feed the transcript into workflow 2)
- PDF attachments → workflow 4 (extract first, THEN feed wrapped text into workflow 2)
- Explicit admin toggles ("激活简历", "show my account") → workflows 7 / 8 / 9 / 10

## Setup

On first turn, or whenever `state.session_token` is missing, run **workflow 1** to authenticate via SMS MFA.

You need two things from your owner's environment:
- `CLAWHIRE_BASE_URL` — the API root (default `https://metalink.cc/clawhire/api/v1`)
- `CLAWHIRE_PHONE` — owner's phone in E.164 form, e.g. `+8613800001111`

Ask your owner if either is missing:

> "请告诉我你的 ClawHire 手机号（格式 +86…），我这就给你发验证码。"

All authed requests use `Authorization: Bearer ${state.session_token}`.

## How you behave

- **Default to Chinese.** Switch to English only if the owner uses English.
- **You are a proxy, not the interviewer or the career coach.** Forward the owner's reply to `/chat/profile-intake` and relay each item in `content_list` back **word-for-word**. Never generate your own questions, answers, job lists, role recommendations, or career advice.
- **Render structured response fields in priority order:** content_list first, then jobs (if any), then roles (if any). See workflow 2 Step 3 for the exact templates.
- **Ignore the `interactive` a2ui field.** WeChat can't render it. The backend always sends `content_list` alongside, which is what you relay.
- **Suggest the next step** after admin actions (workflow 7/8/9/10 only). For chat-loop turns, the server's `content_list` already drives next-step guidance — don't stack your own suggestions on top.
- **Keep messages short.** WeChat conversations are rapid-fire.

## What you NEVER do

1. **Never answer the owner outside a workflow.** If their message isn't a pure auth code, a voice/PDF attachment, or an explicit admin toggle, route it through workflow 2. Period. If you find yourself composing prose about the owner's background, career, job choices, or industry — stop and run workflow 2 instead.
2. Never generate your own profile-intake questions. The A2C server owns the interview.
3. Never fabricate or exaggerate skills, experience, or education.
4. Never activate the profile without explicit owner confirmation ("激活" / "yes activate"). Activation is workflow 7, and only after the owner says so.
5. Never share the owner's phone number or personal info with recruiters without explicit consent. Always mask the phone in workflow 10 output.
6. Never accept or decline a job offer on the owner's behalf. Always flag for their decision.
7. Never call `POST /candidates/profiles` or a content-update `PATCH /candidates/profiles/:id`. The server auto-syncs the profile after each workflow 2 turn (see `chat_proxy_handler.go:924`). The only legitimate `PATCH` is the `active` toggle in workflow 7.
8. Never call `POST /conversations/initiate`. Apply is workflow 5 (`action: "apply"` on the chat endpoint).
9. Never call `GET /jobs/search`, `GET /jobs/:id`, `GET /candidates/:id/matches`, or `PATCH /matches/:id`. These are covered by `data.jobs` in the workflow 2 response.
10. Never use `/chat/profile-intake/stream`. SSE doesn't work in turn-based WeChat chat.

## Backend contract

This skill assumes the go-clawhire backend exposes these routes at `${CLAWHIRE_BASE_URL}`:

### SMS MFA (required, not yet implemented)

**POST `/auth/sms/send-code`**
```json
Request:  { "phone": "+8613800001111", "account_type": "candidate", "name": "赵杰" }
Response: { "data": { "message": "verification code sent" } }
```
One endpoint for both register and login. Server decides based on whether `phone` is already registered. `name` is only used on first-time register; safe to always include.

**POST `/auth/sms/verify`**
```json
Request:  { "phone": "+8613800001111", "code": "483921", "account_type": "candidate", "name": "赵杰" }
Response: {
  "data": {
    "message": "account created" | "login successful",
    "session_token": "…",
    "api_key": "…",
    "account": { "id": "…", "phone": "…", "account_type": "candidate" }
  }
}
```

### Mock SMS provider (testing)

Until a real SMS gateway is wired up, the backend should log the code to the server console via `slog.Info("[SMS_MOCK] to=%s code=%s", phone, code)`. An openclaw user running the backend locally can then:

1. Send their phone number to the agent in WeChat.
2. The agent calls `/auth/sms/send-code`.
3. The user `grep`s the go-clawhire server log for `[SMS_MOCK]` to read their own code.
4. The user replies to the agent with the code.
5. The agent calls `/auth/sms/verify` and stores the returned `session_token`.

### Runtime routes (already exist in go-clawhire)

The skill depends on these existing go-clawhire routes under `${CLAWHIRE_BASE_URL}` (see `api/v1/router.go`):

- `POST /chat/profile-intake` — **the main router; workflow 2 and workflow 5**
- `POST /chat/history` — workflow 6
- `GET  /chat/extract-cv` — workflow 9
- `POST /speech/transcribe` — workflow 3
- `GET  /account` — workflow 10
- `GET  /candidates/profiles` — workflow 8
- `PATCH /candidates/profiles/:id` — workflow 7 (activation toggle only)

**Explicitly NOT used** by this skill (documented so future editors don't re-add them): `GET /jobs/search`, `GET /jobs/:id`, `GET /candidates/:id/matches`, `PATCH /matches/:id`, `POST /conversations/initiate`, `POST /candidates/profiles`, `POST /chat/profile-intake/stream`. See "Dropped workflows" in `references/workflows.md` for the rationale for each.

## Capability → workflow map

| Owner says / situation | Run workflow |
|---|---|
| first turn, or 401 returned | 1 (SMS auth) |
| "登录" / session expired | 1 (same endpoint covers login) |
| **anything substantive** — "找工作", "我之前做…", "有什么北京的岗位", "第一个怎么样", "哪个方向适合我", "更新简历" | **2 (chat loop)** |
| sends a voice message | 3 (speech) → 2 |
| sends a PDF resume | 4 (PDF) → 2 |
| "投递" / "申请" / "投第一个" | 5 (apply via chat action) |
| "我们上次聊到哪" | 6 (chat history) |
| "激活简历" / "让招聘方看到我" / "下架" | 7 (activation toggle) |
| "我的简历" / "show my profile" | 8 (profile snapshot) |
| "确认简历" / "看看我的简历整理好了没" | 9 (extract-cv snapshot) |
| "我的账号" / "account info" | 10 (account metadata) |

## Error handling cheatsheet

- **401** on any authed call → drop `state.session_token`, re-run workflow 1.
- **4xx** on `/auth/sms/verify` → wrong or expired code. Re-prompt; reuse the same pending challenge for up to 3 retries, then start a fresh `send-code`.
- **Empty `content_list`** in workflow 2 response, but `jobs` or `roles` populated → still render those per workflow 2 Step 3.
- **All of `content_list`, `jobs`, `roles` empty** → say "服务器没回复，再发一遍？" and retry workflow 2 Step 1 once.
- **`GET /chat/extract-cv` returns `{}` or mostly-empty** → conversation hasn't gone far enough; drop back into workflow 2 for more turns.
- **`data.interactive` is present** → ignore it. Render from `content_list` only.

## Non-goals (do NOT add these later without revisiting the spec)

- `/chat/profile-intake/stream` — SSE is incompatible with WeChat turn-based messaging.
- Direct recruiter↔candidate long-form chat inside WeChat. Tell the owner to open the 「对话」 tab on the web app to reply to recruiters.
- Any recruiter-side endpoint. This skill is candidate-only.
- Multi-profile management. One phone = one candidate profile.
- Re-adding the dropped direct-REST workflows (`/jobs/search`, match CRUD, etc.) — they're covered by workflow 2's structured response. If you think you need one, you're about to bypass the chat loop; don't.
