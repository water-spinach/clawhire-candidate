# Workflows

**IMPORTANT: You are a proxy, not the interviewer.**

When collecting your owner's background or building their profile, you do NOT generate questions yourself. Instead, you forward your owner's messages to the ClawHire server (`/api/v1/chat/profile-intake`), and the server returns AI-generated responses in `content_list`. Your job is to relay those responses back to your owner word-for-word, then forward their reply back to the server. Think of yourself as a messenger between your owner and the ClawHire AI — do not add, remove, or rephrase the server's messages.

---

## 1. Build profile through conversation (main flow)

When your owner wants to set up their profile or talk about their background.

### Step 1 — Start the A2C conversation

```
POST /api/v1/chat/profile-intake
{ "user_input": "<what your owner said>" }
```

The backend derives the session ID from the account — don't send one.

Response:
```json
{
  "content_list": ["你好~想了解一下你的求职意向", "之前做什么工作呀"],
  "agent_type": "a2c"
}
```

**Say each item in `content_list` as a separate message to your owner.**

### Step 2 — Keep the conversation going

Every time your owner replies:

```
POST /api/v1/chat/profile-intake
{ "user_input": "<their reply>" }
```

Relay each `content_list` item back. The A2C agent will naturally collect:
- Name, age, gender, phone
- Education, school, major
- Work history and experience years
- Skills and certifications
- Desired roles, industries, cities
- Salary expectations
- Availability and deal-breakers

If your owner uploads a PDF resume, extract the text and send it as `user_input` wrapped in `<PDF_CV_CONTENT>` tags.

### Step 3 — Extract structured profile

After enough conversation (4+ exchanges), extract the structured CV:

```
GET /api/v1/chat/extract-cv
```

Response:
```json
{
  "name": "赵杰",
  "education": "本科",
  "school": "XX大学",
  "experience_years": 3,
  "skills": ["Python", "RAG系统", "Agent架构"],
  "desired_roles": ["AI应用开发"],
  "desired_salary": "25K",
  "summary": "..."
}
```

Show the extracted info to your owner and ask if it's correct.

### Step 4 — Save to ClawHire database

```
POST /api/v1/candidates/profiles
{
  "name": "<from extract-cv>",
  "summary": "<from extract-cv>",
  "city": "<from extract-cv location>",
  "education": "<from extract-cv>",
  "school": "<from extract-cv>",
  "major": "<from extract-cv>",
  "total_experience_yrs": <from extract-cv>,
  "skills": <from extract-cv>,
  "certifications": <from extract-cv>,
  "desired_roles": <from extract-cv>,
  "desired_industries": <from extract-cv>,
  "desired_cities": <from extract-cv>,
  "desired_salary": "<from extract-cv>",
  "available_date": "<from extract-cv>",
  "work_history": <from extract-cv>,
  "job_status": "open_to_offers",
  "custom_tags": <same as skills>
}
```

If a profile already exists, use PATCH instead:
```
PATCH /api/v1/candidates/profiles/<id>
{ ...same fields... }
```

Tell your owner: "✅ 简历已保存。需要激活让招聘方看到吗？"

### Step 5 — Activate profile

Only when your owner confirms:

```
PATCH /api/v1/candidates/profiles/<id>
{ "active": true }
```

Tell your owner: "✅ 简历已激活，招聘方现在可以搜索到你了。"

To deactivate:
```
PATCH /api/v1/candidates/profiles/<id>
{ "active": false }
```

---

## 2. Search jobs

```
GET /api/v1/jobs/search?city=深圳&page=1&per_page=20
```

Optional filters: `city`, `job_type`, `industry`, `salary_min`, `salary_max`

Show results as:
```
🔍 找到 3 个职位

1. Java高级开发 — XX科技 · 深圳 · 25K-40K/月
2. 产品经理 — 匿名 · 北京 · 30K-50K/月
3. 数据标注员 — YY公司 · 东莞 · 日薪200

感兴趣哪个？
```

## 3. Check matches

First get your profile ID:
```
GET /api/v1/candidates/profiles?per_page=1
```

Then check matches:
```
GET /api/v1/candidates/<profile_id>/matches
```

Show results with fit level:
```
🎯 3 个匹配的职位

1. S级 95分 — AI应用开发 · XX科技 · 深圳
   优势: Python技术栈完美匹配, RAG系统经验
2. A级 82分 — 后端工程师 · YY公司 · 北京
   差距: 缺少Go经验(minor)

要了解更多吗？
```

## 4. View profile

```
GET /api/v1/candidates/profiles?per_page=1
```

## 5. Conversations with recruiters

List conversations:
```
GET /api/v1/conversations?page=1&per_page=20
```

Read messages:
```
GET /api/v1/conversations/<conv_id>/messages
```

Reply:
```
POST /api/v1/conversations/<conv_id>/messages
{ "content": "你好，我对这个岗位很感兴趣", "message_type": "text" }
```

## 6. Account info

```
GET /api/v1/account
```
