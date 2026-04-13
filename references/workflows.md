# Workflows

Every runtime HTTP call the `clawhire-candidate` skill makes, numbered 1–16.

**Conventions used throughout this file:**
- `${BASE}` = `${CLAWHIRE_BASE_URL}` env var, e.g. `https://metalink.cc/clawhire/api/v1`
- Every authed call sends `Authorization: Bearer ${session_token}` and `Content-Type: application/json` unless noted.
- State keys (`session_token`, `profile_id`, etc.) are defined in [state.md](state.md).
- When a step says "say to owner", send that text as a WeChat message. If `content_list` has multiple items, send each as a **separate** WeChat message.

---

## 1. First-time onboarding / returning login (SMS MFA)

**When:** first turn of a session, or any authed call returns 401.

**Trigger phrases:** "登录", "注册", "login", "register", or implicit (no `session_token`).

### Step 1 — ask for the phone (if not in env)

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

Expected response:
```json
{ "data": { "message": "verification code sent" } }
```

Say to owner:

> "验证码已发送到 +86138****1111。请把收到的 6 位数字发给我。"

(During local testing with the mock SMS provider, tell the owner to check the go-clawhire server log for `[SMS_MOCK]` — see `SKILL.md` → Mock SMS provider.)

### Step 3 — verify

When the owner sends a 6-digit code, call:

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

> "✅ 登录成功。要更新简历还是看看有什么新岗位？"

### Step 4 — handle errors

- 4xx from `/auth/sms/verify` → wrong/expired code. Say: "验证码不对，再发一次给我？" Allow up to 3 retries on the same challenge, then re-run Step 2 (fresh `send-code`).
- 5xx → apologize and suggest retry in a minute.

---

## 2. Returning login

Same as workflow 1. The option-B `/auth/sms/send-code` endpoint collapses register and login — the server decides based on whether `${state.phone}` is already registered. No separate workflow.

---

## 3. Profile intake (A2C conversation loop) — MAIN FLOW

**When:** owner wants to build or update their profile, or says anything about their background, skills, or job preferences.

**Trigger phrases:** "简历", "找工作", "更新简历", "我之前做…", "我会…".

**⛔ You are a PROXY here, not the interviewer. Do NOT generate your own questions about background, skills, or preferences. The server does that.**

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

Expected response:
```json
{
  "data": {
    "content_list": [
      "你好~想了解一下你的求职意向",
      "之前做什么工作呀"
    ],
    "agent_type": "a2c"
  }
}
```

### Step 2 — relay each `content_list` item as a SEPARATE WeChat message

If `content_list` has 3 items, send 3 WeChat messages. Do NOT merge them, do NOT paraphrase, do NOT add your own commentary.

### Step 3 — wait for the owner's next message, then loop back to Step 1

The A2C server will naturally collect, over multiple turns:
- name, age, gender, phone
- education, school, major
- work history and experience years
- skills and certifications
- desired roles, industries, cities
- salary expectations
- availability and deal-breakers

After ~4+ exchanges the server will start summarizing. When it does, proceed to **workflow 7 (extract CV)**.

### Step 4 — handle empty content_list

If `content_list` is empty or `content_list: null`, say to owner: "还在么？你上一句说到…" and retry workflow 3 step 1 once. If still empty, tell owner: "服务器好像没反应，稍后再聊？"

---

## 4. WeChat voice message → transcribe → feed profile-intake

**When:** WeChat delivers an audio message to the agent.

### Step 1 — transcribe the audio

```
POST ${BASE}/speech/transcribe
Authorization: Bearer ${state.session_token}
Content-Type: multipart/form-data

audio=<voice-input.wav or .amr blob>
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

### Step 2 — feed the transcript into workflow 3

Use `data.text` as the `user_input` and run workflow 3 Step 1. From the owner's perspective the voice message Just Works as if they typed it.

---

## 5. WeChat PDF resume → extract → feed profile-intake

**When:** WeChat delivers a PDF attachment to the agent.

### Step 1 — extract text from the PDF

Use openclaw's PDF extraction capability (not a ClawHire endpoint) to get the plain text of the resume.

### Step 2 — wrap and forward to profile-intake

```
POST ${BASE}/chat/profile-intake
Authorization: Bearer ${state.session_token}
Content-Type: application/json

{
  "user_input": "<PDF_CV_CONTENT>\n${extracted_text}\n</PDF_CV_CONTENT>"
}
```

The `<PDF_CV_CONTENT>…</PDF_CV_CONTENT>` tag tells the A2C backend to treat this as a resume dump instead of a conversational reply. Relay the `content_list` response as usual.

---

## 6. Load prior chat history

**When:** owner asks "我们之前聊到哪了" or you want to resume context after a session break.

```
POST ${BASE}/chat/history
Authorization: Bearer ${state.session_token}
Content-Type: application/json

{ "agent_type": "a2c" }
```

Expected response:
```json
{
  "data": {
    "messages": [
      { "role": "user", "content": "…" },
      { "role": "assistant", "content": "…" }
    ]
  }
}
```

Use this to reconstruct where the profile intake left off. Do NOT re-send the history to `/chat/profile-intake` — the server already has it. Just use it locally to remind yourself (and the owner, if asked) of context.

---

## 7. Extract structured CV

**When:** the profile intake has gone on for 4+ owner replies and the server has started asking confirming/summary questions, or the owner says "确认简历".

```
GET ${BASE}/chat/extract-cv
Authorization: Bearer ${state.session_token}
```

Expected response:
```json
{
  "data": {
    "name": "赵杰",
    "education": "本科",
    "school": "XX大学",
    "major": "计算机",
    "experience_years": 3,
    "skills": ["Python", "RAG", "Agent"],
    "certifications": [],
    "desired_roles": ["AI应用开发"],
    "desired_industries": ["互联网"],
    "desired_cities": ["深圳"],
    "desired_salary": "25K",
    "available_date": "immediately",
    "work_history": [],
    "summary": "…",
    "city": "深圳"
  }
}
```

Store the whole object as `state.latest_cv_snapshot`.

Present the summary to the owner for confirmation:

> "帮你整理好了：
> 📋 姓名：赵杰
> 🎓 本科 XX大学 计算机
> 💼 3年经验，Python / RAG / Agent
> 🎯 意向：AI应用开发，深圳，25K
> 看着对吗？我帮你存进简历库。"

If `data` is empty `{}`, say "咱们再多聊几句吧，我还没完全了解你" and do 2–3 more turns of workflow 3 before retrying.

On confirmation (or owner correction), proceed to **workflow 8**.

---

## 8. Save profile to ClawHire

**When:** owner confirms the extracted CV in workflow 7.

### Case A — no existing profile (`state.profile_id` is empty)

```
POST ${BASE}/candidates/profiles
Authorization: Bearer ${state.session_token}
Content-Type: application/json

{
  "name":                "${snapshot.name}",
  "summary":             "${snapshot.summary}",
  "city":                "${snapshot.city}",
  "education":           "${snapshot.education}",
  "school":              "${snapshot.school}",
  "major":               "${snapshot.major}",
  "total_experience_yrs": ${snapshot.experience_years},
  "skills":              ${snapshot.skills},
  "certifications":      ${snapshot.certifications},
  "desired_roles":       ${snapshot.desired_roles},
  "desired_industries":  ${snapshot.desired_industries},
  "desired_cities":      ${snapshot.desired_cities},
  "desired_salary":      "${snapshot.desired_salary}",
  "available_date":      "${snapshot.available_date}",
  "work_history":        ${snapshot.work_history},
  "job_status":          "open_to_offers",
  "custom_tags":         ${snapshot.skills}
}
```

Expected response:
```json
{ "data": { "id": "prof_abc", "active": false } }
```

Store `state.profile_id` ← `data.id`.

### Case B — existing profile

```
PATCH ${BASE}/candidates/profiles/${state.profile_id}
Authorization: Bearer ${state.session_token}
Content-Type: application/json

{ ...same fields... }
```

Say to owner:

> "✅ 简历已保存。要激活让招聘方看到吗？"

Then wait for explicit confirmation before running workflow 9.

---

## 9. Activate / deactivate profile

**When:** owner explicitly says "激活", "让招聘方看到我", "yes activate" (or "隐藏", "下架").

**Only run this after explicit owner confirmation. Never activate implicitly.**

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

## 10. View own profile

**When:** owner asks "我的简历", "show my profile".

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

If `data` is empty, run workflow 3 (profile intake) first. Otherwise:
- Store `state.profile_id` ← `data[0].id` (in case it was missing).
- Render the profile as a compact WeChat message: name, city, experience, top skills, desired role, active status.

---

---

## 11. Search jobs

**When:** owner asks "有什么工作", "搜深圳的后端", "search jobs in Beijing".

```
GET ${BASE}/jobs/search?city=深圳&job_type=full_time&industry=互联网&salary_min=20000&salary_max=50000&page=1&per_page=20
Authorization: Bearer ${state.session_token}
```

All query params are optional. Extract whatever filters the owner mentioned from their natural-language query. Defaults: `page=1&per_page=10`.

Expected response:
```json
{
  "data": [
    {
      "id": "job_1",
      "title": "Java高级开发",
      "company_name": "XX科技",
      "city": "深圳",
      "salary": "25K-40K",
      "job_type": "full_time"
    }
  ],
  "total": 42,
  "page": 1,
  "per_page": 10
}
```

Render as a numbered WeChat message (keep it short):

```
🔍 找到 3 个匹配职位

1. Java高级开发 — XX科技 · 深圳 · 25K-40K
2. 产品经理 — 匿名 · 北京 · 30K-50K
3. 数据标注员 — YY公司 · 东莞 · 日薪200

想看哪个详情？回复数字。
```

Store `state.last_search_results` ← `data` (the array) so the owner can pick by index.

If `data` is empty, say: "没找到匹配的职位，换个条件试试？"

---

## 12. Job detail

**When:** owner replies with a number after a search (workflow 11), or asks "第一个详情", "tell me more about job_1".

Resolve the job id:
- If the owner said a number, use `state.last_search_results[n-1].id`.
- If they said an id, use it directly.

```
GET ${BASE}/jobs/${job_id}
Authorization: Bearer ${state.session_token}
```

Expected response:
```json
{
  "data": {
    "id": "job_1",
    "title": "Java高级开发",
    "company_name": "XX科技",
    "description": "…",
    "requirements": "…",
    "city": "深圳",
    "salary": "25K-40K",
    "benefits": []
  }
}
```

Render as a multi-line WeChat message. End with:

> "感兴趣的话我帮你投递？回复「投递」。"

Store `state.active_job_id` ← `data.id`.

---

## 13. List matches

**When:** owner asks "我的匹配", "推荐岗位", "有什么匹配".

```
GET ${BASE}/candidates/${state.account_id}/matches?page=1&per_page=10
Authorization: Bearer ${state.session_token}
```

Optional filter: `&status=pending|interested|applied|passed`.

Expected response:
```json
{
  "data": [
    {
      "id": "match_1",
      "job": { "id": "job_1", "title": "…", "company_name": "…", "city": "…", "salary": "…" },
      "score": 0.87,
      "fit_code": "strong",
      "status": "pending"
    }
  ],
  "total": 5
}
```

Render as a numbered list showing job title, company, fit score, and status. Store `state.last_matches` ← `data`.

If `data` is empty:
- If `state.profile_id` is empty or profile not activated → tell owner to run workflow 9 first.
- Else → "还没有匹配，过阵子再来看看。"

---

## 14. Update match status (interested / pass)

**When:** owner reacts to a match from workflow 13 ("第一个感兴趣", "跳过第三个").

Resolve `match_id` from `state.last_matches[n-1].id`.

```
PATCH ${BASE}/matches/${match_id}
Authorization: Bearer ${state.session_token}
Content-Type: application/json

{ "status": "interested" }   // or "passed"
```

Expected response: `{ "data": { "id": "match_1", "status": "interested" } }`

Say:
- `interested` → "✅ 已标记为感兴趣。要不要现在投递？"
- `passed` → "好，已跳过。"

If `interested`, set `state.active_match_id` ← `match_id` and `state.active_job_id` ← `state.last_matches[n-1].job.id`. Then wait for the owner to confirm applying (workflow 15).

---

## 15. Apply to a job (start recruiter conversation)

**When:** owner explicitly says "投递", "申请", "apply" — either after a job detail (workflow 12) or a match (workflow 14).

Requires: `state.active_job_id` set, and the owner's profile is activated (workflow 9). If not activated, prompt: "先激活简历招聘方才能看到你，现在激活？"

```
POST ${BASE}/conversations/initiate
Authorization: Bearer ${state.session_token}
Content-Type: application/json

{ "job_id": "${state.active_job_id}" }
```

Expected response:
```json
{ "data": { "conversation_id": "conv_abc", "job_id": "job_1", "recruiter_id": "…" } }
```

If `state.active_match_id` is set, also run workflow 14 with `status: "applied"` to mark it.

Say to owner:

> "✅ 已投递！招聘方会在 ClawHire 收到你的简历。他们回消息后，请到网页「对话」页或 WeChat「我的对话」里查看并回复 —— WeChat 里面我只能帮你投递，后续的面谈请到对话页面。"

Clear `state.active_job_id` and `state.active_match_id`.

**Why we don't relay recruiter messages in WeChat:** recruiter↔candidate threads can be long, have attachments, and need async coordination. A WeChat proxy would either spam the owner or miss messages. Point them at the web app's 「对话」 tab.

---

## 16. Account info

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

Render as a short WeChat message. Do NOT echo the full phone number — mask it as `+86138****1111`.

---

## Non-goals (do NOT add workflows for these)

- `POST /chat/profile-intake/stream` — SSE incompatible with WeChat turn-based chat. Use workflow 3 instead.
- Recruiter↔candidate long-form messaging — handled by the web app's 「对话」 tab (see workflow 15 rationale).
- Any recruiter-side endpoint (`/recruiter/*`, `/positions/*`). This skill is candidate-only.
- Match score/report inspection endpoints beyond what workflow 13 returns.
