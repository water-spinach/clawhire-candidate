# Workflows

Every runtime HTTP call the `clawhire-candidate` skill makes, numbered 1–10.

**Core principle:** the go-clawhire `/chat/profile-intake` endpoint is a **full candidate router**, not just a profile collector. Its response carries `content_list`, `jobs` (ranked + enriched), `roles`, `candidate_profile` (auto-synced to DB by the server), `phase`, and an optional `interactive` a2ui payload. **Every owner turn — job search, apply, profile talk, role questions — routes through workflow 2.** Direct REST calls are used only for the five things the chat endpoint can't do: SMS auth, account metadata, voice preprocessing, profile activation toggle, and optional snapshot reads.

**Conventions used throughout this file:**
- `${BASE}` = `${CLAWHIRE_BASE_URL}` env var, e.g. `https://metalink.cc/clawhire/api/v1`
- Every authed call sends `Authorization: Bearer ${state.session_token}` and `Content-Type: application/json` unless noted.
- State keys are defined in [state.md](state.md).
- When a step says "relay to owner", send each item in `content_list` as a **separate** WeChat message.

---

## 1. SMS MFA onboarding / login

**When:** first turn of a session, or any authed call returns 401.

**Trigger phrases:** "登录", "注册", "login", "register", or implicit (no `state.session_token`).

### Step 1 — collect phone

If `CLAWHIRE_PHONE` is empty, say to owner:

> "请告诉我你的 ClawHire 手机号（格式 +86…），我给你发验证码。"

Store the reply as `state.phone`.

### Step 2 — send the code

```
POST ${BASE}/auth/sms/send-code
Content-Type: application/json

{
  "phone": "${state.phone}",
  "account_type": "candidate",
  "name": "(owner display name or empty)"
}
```

One endpoint covers both register and login — the server decides based on whether `phone` is already registered.

Expected response:
```json
{ "data": { "message": "verification code sent" } }
```

Say to owner:

> "验证码已发送到 +86138****1111。请把收到的 6 位数字发给我。"

(Local testing note: the mock provider logs the code via `slog.Info("[SMS_MOCK] to=%s code=%s")`. Tell the owner to grep the server log if they run go-clawhire themselves.)

### Step 3 — verify

When the owner sends a 6-digit code:

```
POST ${BASE}/auth/sms/verify
Content-Type: application/json

{
  "phone": "${state.phone}",
  "code": "${owner_reply}",
  "account_type": "candidate",
  "name": "(same as send-code step)"
}
```

Expected response:
```json
{
  "data": {
    "message": "account created" | "login successful",
    "session_token": "eyJ…",
    "api_key": "ck_…",
    "account": { "id": "acc_abc", "phone": "…", "account_type": "candidate" }
  }
}
```

Store:
- `state.session_token` ← `data.session_token`
- `state.account_id` ← `data.account.id`
- `state.account_type` ← `"candidate"`
- `state.phone` ← `data.account.phone`

Say to owner:

> "✅ 登录成功。聊聊你最近的求职情况吧？"

Then immediately transition to workflow 2 with whatever the owner says next.

### Step 4 — handle errors

- 4xx from `/auth/sms/verify` → wrong/expired code. Say: "验证码不对，再发一次给我？" Allow up to 3 retries on the same challenge, then re-run Step 2.
- 5xx → apologize and suggest retry in a minute.

---

## 2. Main chat loop — ROUTE EVERY OWNER TURN THROUGH HERE

**When:** any owner message that is not a pure auth code, a voice/PDF attachment (those preprocess through workflows 3/4 first), or an explicit admin toggle (workflow 7). **This is the default path for everything — profile talk, background questions, "find me jobs", "apply to the first one", match review, role questions, career chat.**

**⛔ You are a PROXY. Do NOT generate your own answers to the owner's substantive questions — forward them to the server and render what it returns. Your only freeform prose is: short acknowledgements ("好的"), status messages ("正在搜索…"), and the fixed templates defined in workflows 1, 5, 7–10.**

### Step 1 — forward the owner's message

```
POST ${BASE}/chat/profile-intake
Authorization: Bearer ${state.session_token}
Content-Type: application/json

{
  "user_input": "${owner_message}"
}
```

The backend derives the session ID from the account — do NOT send one.

### Step 2 — parse the structured response

The response wraps a rich `a2cResponse` structure. Read these fields:

```json
{
  "data": {
    "session_id": "clawhire-a2c-acc_abc",
    "agent_type": "a2c",
    "content_list": ["…", "…"],
    "phase": "intake" | "job_deck" | "apply" | ...,
    "reasoning_content": "(ignore — debug only)",
    "candidate_profile": {
      "name": "…", "background_summary": "…", "skills": [],
      "desired_roles": [], "preferred_city": "…", "desired_salary": "…"
    },
    "profile_changed": true|false,
    "jobs": [
      {
        "job_id": "job_1",
        "title": "Java高级开发",
        "company": "XX科技",
        "city": "深圳",
        "salary": "25K-40K",
        "score": 0.87,
        "summary": "…",
        "matches": [],
        "gaps": [],
        "detail": { /* full Job object */ }
      }
    ],
    "roles": [
      {
        "id": "role_1",
        "role_family": "AI应用开发",
        "fit_label": "strong",
        "why_fit": "…"
      }
    ],
    "interactive": { /* a2ui payload — IGNORE in WeChat */ }
  }
}
```

### Step 3 — render to the owner in this priority order

**a. Relay `content_list`** — send each string as a separate WeChat message, **byte-for-byte**. Never merge, paraphrase, reorder, skip, translate, or decorate.

⛔ **FORBIDDEN** — any of these is a bug:

Server sent: `["你之前做什么工作呀"]`

- ❌ `嗨！你之前做什么工作呀 😊` (prepended greeting + emoji)
- ❌ `你之前做什么工作呀？（方便的话也说一下公司）` (appended aside)
- ❌ `服务器问：你之前做什么工作呀` ("here's what the system says" framing)
- ❌ `请告诉我你之前的工作经历` (paraphrase for "clarity")
- ❌ `What did you do for work before?` (translation)
- ❌ `你之前做什么工作呀？建议从最近一份讲起` (tip appended)
- ❌ `你之前做什么工作呀？不错的背景！` (encouragement appended)

Server sent: `["整理好了", "看看这些匹配的岗位"]`

- ❌ `整理好了！看看这些匹配的岗位 👇` (merged into one message + emoji)
- ❌ `已为你整理好简历，接下来看看匹配的岗位` (paraphrased + merged)

✅ **ALLOWED** — exactly one thing:

Server sent: `["你之前做什么工作呀"]`  →  WeChat message: `你之前做什么工作呀`

Server sent: `["整理好了", "看看这些匹配的岗位"]`  →  WeChat message 1: `整理好了` · WeChat message 2: `看看这些匹配的岗位`

**Self-check**: before pressing send on each WeChat message, compare byte-for-byte against `content_list[i]`. If there is any difference — even one emoji, one space, one particle — rewrite to match.

**b. If `jobs` is present and non-empty** — render as ONE WeChat message using this **fixed template**. Do not deviate. Do not add commentary above or below. Do not recommend a favorite. Do not tag any entry as "best fit". Just the template:

```
🔍 为你找到 N 个岗位

1. {title} — {company} · {city} · {salary} (匹配度 NN%)
2. {title} — {company} · {city} · {salary} (匹配度 NN%)
…

回复数字查看详情，或说「投递第一个」直接申请。
```

- `N` = `len(data.jobs)`
- `NN%` = `round(data.jobs[i].score * 100)`
- Only the first 10 entries. If `len > 10` the tail is silently dropped.
- The last line is always the literal string `回复数字查看详情，或说「投递第一个」直接申请。` — no variation.

⛔ **FORBIDDEN** additions to the jobs render:
- ❌ `我觉得第二个最适合你` / `根据你的背景建议优先投递第一个`
- ❌ `这些都是 AI 方向的，和你的技能很匹配` (editorial framing)
- ❌ `薪资都不错 👍`
- ❌ `要不要我帮你分析一下？`

Then store:
- `state.last_chat_jobs` ← `data.jobs` (the whole array, so indexed apply works)
- `state.active_job_id` ← `data.jobs[0].job_id` (default selection; overwritten by workflow 5 if owner picks a different index)

**c. If `roles` is present and non-empty** — render as ONE WeChat message using this **fixed template**:

```
💡 推荐方向
• {role_family} — {fit_label}：{why_fit}
• {role_family} — {fit_label}：{why_fit}
```

Fields come straight from `data.roles[i]`. No reordering, no reinterpretation. Do not rank them for the owner. Do not add `我推荐你走第一个方向` or any similar prose.

**d. If `interactive` is present** — **ignore it**. WeChat can't render a2ui. Rely on `content_list` for the actual user-facing text; the backend always sends `content_list` alongside any interactive payload.

**e. If `profile_changed` is true** — the server auto-saved the owner's profile. No action needed from you; do NOT call `POST /candidates/profiles`. If the owner asks "简历存了吗?", say "已帮你保存了". Optionally run workflow 8 to show them the current snapshot.

**f. If `candidate_profile` is present** — treat as reference only. The actual DB write has already happened; this is just the server telling you what it extracted. Do NOT echo it back unless the owner explicitly asks.

### Step 4 — handle empty content_list

If `content_list` is empty/null but `jobs` or `roles` is populated, still render them per Step 3. If all three are empty, say to owner: "服务器没回复，再发一遍？" and retry Step 1 once.

### Step 5 — loop

Wait for the owner's next message, then go back to Step 1. Every subsequent turn of substantive conversation follows this same loop.

---

## 3. Voice message → transcribe → feed chat loop

**When:** WeChat delivers an audio message.

### Step 1 — transcribe

```
POST ${BASE}/speech/transcribe
Authorization: Bearer ${state.session_token}
Content-Type: multipart/form-data

audio=<voice.wav or .amr blob>
```

Expected response:
```json
{
  "data": {
    "text": "我之前在腾讯做后端开发，三年 Java 经验",
    "tagged_text": "…",
    "audio_file": { "id": "…", "url": "…" }
  }
}
```

### Step 2 — feed `data.text` into workflow 2 Step 1

Use the transcript as `user_input`. From the owner's perspective the voice message works exactly like typing.

---

## 4. PDF resume → extract → feed chat loop

**When:** WeChat delivers a PDF attachment.

### Step 1 — extract PDF text

Use openclaw's PDF extraction capability (not a ClawHire endpoint) to get the resume's plain text.

### Step 2 — wrap and feed workflow 2 Step 1

Send the extracted text as `user_input` wrapped in `<PDF_CV_CONTENT>…</PDF_CV_CONTENT>`:

```
POST ${BASE}/chat/profile-intake
Authorization: Bearer ${state.session_token}
Content-Type: application/json

{
  "user_input": "<PDF_CV_CONTENT>\n${extracted_text}\n</PDF_CV_CONTENT>"
}
```

The tag tells the A2C backend to treat this as a resume dump instead of a conversational reply. Then render the response exactly as workflow 2 Step 3. The server will auto-extract and save the profile (`profile_changed: true`).

---

## 5. Apply to a job (chat-routed)

**When:** owner says "投递", "申请", "apply", "投第一个" — either directly after a `jobs` list from workflow 2 (most common), or referencing a specific `job_id`.

**Do NOT call `/conversations/initiate` directly.** The chat endpoint handles apply via an `action` field, and the server enriches + persists everything server-side.

### Step 1 — resolve the target job

- If the owner said a number ("第一个", "投第二个") → `state.last_chat_jobs[n-1]`.
- If no index → use `state.active_job_id` (set by the latest workflow 2 render).
- If neither is set → tell the owner "没看到你在看哪个岗位，先让我给你推荐几个？" and drop back to workflow 2.

Let `job = state.last_chat_jobs[n-1]` (or the matching entry).

### Step 2 — send the apply action to the chat endpoint

```
POST ${BASE}/chat/profile-intake
Authorization: Bearer ${state.session_token}
Content-Type: application/json

{
  "user_input": "",
  "action": "apply",
  "job_id": "${job.job_id}",
  "job_title": "${job.title}",
  "job_company": "${job.company}"
}
```

Note: `user_input` is intentionally empty — the backend explicitly permits `action` without `user_input` (see `chat_proxy_handler.go:525`). The server also looks up missing `job_title`/`job_company` from the DB if you leave them blank, so sending just `job_id` works too.

### Step 3 — render the response

Same as workflow 2 Step 3. The server's `content_list` will typically include the confirmation message ("已为你发起求职申请…"). Relay it verbatim.

Then say:

> "招聘方会在 ClawHire 收到你的简历。他们回消息后，请到网页端「对话」页查看并回复 —— WeChat 里我只能帮你投递。"

### Step 4 — clear active job

Clear `state.active_job_id`. Keep `state.last_chat_jobs` in case the owner wants to apply to another one from the same list.

---

## 6. Resume prior chat history

**When:** owner asks "我们之前聊到哪了" or you want context after a session break.

```
POST ${BASE}/chat/history
Authorization: Bearer ${state.session_token}
Content-Type: application/json

{ "agent_type": "a2c" }
```

Expected response (sanitized by backend, see `chat_proxy_handler.go:1094`):
```json
{
  "data": {
    "messages": [ { "role": "user", "content": "…" }, { "role": "assistant", "content": "…" } ],
    "turn_state": { "latest_job_deck": [] },
    "updated_at": "2026-04-13T…"
  }
}
```

If `turn_state.latest_job_deck` is present, the backend has already enriched each job with sanitized `detail` and stripped recruiter-only fields. You can reuse it to repopulate `state.last_chat_jobs` without calling workflow 2.

Do NOT re-send the history as `user_input` — the server already has it server-side.

---

## 7. Activate / deactivate profile

**When:** owner explicitly says "激活", "让招聘方看到我", "yes activate", "隐藏", "下架".

**Only direct REST path left for this — the chat endpoint does not toggle activation. Only run after explicit owner confirmation.**

Requires `state.profile_id`. If empty, run workflow 8 first to fetch it.

### Activate

```
PATCH ${BASE}/candidates/profiles/${state.profile_id}
Authorization: Bearer ${state.session_token}
Content-Type: application/json

{ "active": true }
```

Say: "✅ 简历已激活，招聘方现在可以搜索到你了。"

### Deactivate

```
PATCH ${BASE}/candidates/profiles/${state.profile_id}
Authorization: Bearer ${state.session_token}
Content-Type: application/json

{ "active": false }
```

Say: "✅ 简历已下架，招聘方搜不到你了。"

---

## 8. View own profile snapshot

**When:** owner asks "我的简历", "show my profile", or you need to fetch `state.profile_id` for workflow 7.

```
GET ${BASE}/candidates/profiles?per_page=1
Authorization: Bearer ${state.session_token}
```

Expected response:
```json
{
  "data": [
    { "id": "prof_abc", "name": "…", "active": true, "skills": [] }
  ],
  "total": 1
}
```

Store `state.profile_id` ← `data[0].id`. Render as a compact WeChat message (name, city, experience, top skills, desired role, active status).

If `data` is empty → the server hasn't extracted enough yet. Route the owner back into workflow 2: "我还没收集够信息呢，聊聊你的背景？"

---

## 9. Extract CV snapshot (optional)

**When:** owner says "看看简历整理成什么样", "确认简历", "show me what you've got on me".

```
GET ${BASE}/chat/extract-cv
Authorization: Bearer ${state.session_token}
```

The backend reads the A2C agent's session memory and returns the current candidate draft — no LLM call, just a snapshot of what the server already has.

Expected response (see `extractCVResponseFromSession` in `chat_proxy_handler.go:1366`):
```json
{
  "data": {
    "message_count": 12,
    "session_updated_at": "2026-04-13T…",
    "summary": "…",
    "city": "深圳",
    "education": "本科",
    "age": "28",
    "current_title": "后端开发",
    "current_company": "腾讯",
    "skills": ["Python", "RAG"],
    "desired_roles": ["AI应用开发"],
    "desired_cities": ["深圳"],
    "desired_salary": "25K",
    "available_date": "immediately",
    "deal_breakers": [],
    "risk_signals": [],
    "resume_text": "…"
  }
}
```

Store as `state.latest_cv_snapshot`. Show a compact bulleted summary to the owner.

If `data` is `{}` or missing most fields → conversation hasn't gone far enough; route back into workflow 2 for a few more turns.

**Do NOT use this to drive a separate save step.** The profile is already auto-saved by the server after each workflow 2 turn. This endpoint is read-only.

---

## 10. Account info

**When:** owner asks "我的账号", "account info".

```
GET ${BASE}/account
Authorization: Bearer ${state.session_token}
```

Expected response:
```json
{
  "data": {
    "id": "acc_abc",
    "phone": "+8613800001111",
    "account_type": "candidate",
    "created_at": "2026-01-15T…",
    "name": "赵杰"
  }
}
```

Render as a short WeChat message. **Mask the phone** as `+86138****1111` — do not echo the full number.

---

## Dropped workflows (deliberately NOT included)

These were in the original design but are now covered by workflow 2's structured response. Do NOT add them back without revisiting the spec:

- ❌ `GET /jobs/search` — use workflow 2 (owner says "搜深圳的 Java" → chat returns `jobs` field).
- ❌ `GET /jobs/:id` — job detail is embedded in `data.jobs[n].detail` from workflow 2.
- ❌ `GET /candidates/:id/matches` — ranked matches arrive as `data.jobs` from workflow 2, already scored and enriched.
- ❌ `PATCH /matches/:id` — no chat path for match status. The chat agent naturally moves on when the owner says "跳过".
- ❌ `POST /conversations/initiate` — use workflow 5 (`action: "apply"` on chat endpoint) instead.
- ❌ `POST /candidates/profiles`, most `PATCH /candidates/profiles/:id` — auto-synced by `syncCandidateProfile` after each chat turn (see `chat_proxy_handler.go:924`). The one PATCH that's still needed — the `active` toggle — is workflow 7.
- ❌ `POST /chat/profile-intake/stream` — SSE incompatible with WeChat turn-based messaging.

## Non-goals (do NOT add workflows for these)

- Recruiter↔candidate long-form messaging — handled by the web app's 「对话」 tab.
- Any recruiter-side endpoint (`/recruiter/*`, `/positions/*`). This skill is candidate-only.
- Rendering `interactive` a2ui payloads in WeChat — always fall back to `content_list` text.
