# State the agent must carry across turns

The `clawhire-candidate` skill does not persist state itself — openclaw's runtime memory (SQLite at `~/.openclaw/memory/main.sqlite`) does. This file defines the **contract**: what keys the workflows read from and write to, and why each matters. If a workflow step uses a state key not listed here, either the workflow or this file has a bug.

## Keys

### `session_token` (string, sensitive)

- **Source:** Response of workflow 1 step 3 (`/auth/sms/verify`).
- **Why:** Every authed call needs it as `Authorization: Bearer ${session_token}`. Without it, every request returns 401.
- **How to apply:** Send on every request except `/auth/sms/*`. On any 401 response, drop the value and re-run workflow 1.
- **Rotation:** Tokens may expire server-side after an unspecified period — treat 401 as "time to re-auth", not an error.

### `account_id` (string)

- **Source:** `account.id` field of workflow 1 step 3 response.
- **Why:** Workflow 13 (`GET /candidates/:id/matches`) requires it in the path.
- **How to apply:** Substitute into match-list URLs. Do not confuse with `profile_id` — an account is the login identity, a profile is the CV.

### `phone` (string, E.164)

- **Source:** Either `CLAWHIRE_PHONE` env var or the owner's first reply in workflow 1 step 1.
- **Why:** Needed to re-run workflow 1 after token expiry.
- **How to apply:** Never share with recruiters, never echo in full back to the owner in other workflows (mask as `+86138****1111` when displaying in workflow 16).

### `account_type` (constant: `"candidate"`)

- **Why:** The go-clawhire backend routes requests by account type. Using the wrong type returns wrong matches or 403s.
- **How to apply:** Always `"candidate"` for this skill. If the owner asks about recruiter features, tell them they'd need the `clawhire-recruiter` skill instead (future work).

### `profile_id` (string, nullable)

- **Source:** `data.id` field of workflow 8 step 1 response (POST), or `data[0].id` from workflow 10.
- **Why:** PATCH calls in workflows 8/9 need the id in the URL. Also needed for pre-flight checks before workflow 15 (apply).
- **How to apply:** Capture after the first successful save. If nullable and the owner wants to apply/activate, run workflow 10 first to fetch an existing profile, or workflow 3 to create one.

### `latest_cv_snapshot` (object, nullable)

- **Source:** Full response data of workflow 7 (`/chat/extract-cv`).
- **Why:** Workflow 8 builds its save body from this snapshot without re-extracting. Also lets the agent diff between a new extract and the prior version to describe changes to the owner.
- **How to apply:** Overwrite on every successful workflow 7 call. Clear only when the owner says "重新开始" / "start over" on their profile.

### `active_job_id` / `active_match_id` (strings, nullable, transient)

- **Source:** Set by workflows 11/12 (search + detail) and workflow 14 (match interest).
- **Why:** Workflow 15 (apply) needs to know which job the owner is currently looking at. Without this, the agent can't disambiguate "apply" / "投递" when the owner is several turns away from the original job-detail view.
- **How to apply:** Set in workflows 12 (job detail) or 14 (match → interested). Clear after workflow 15 completes, or after 5 turns of unrelated conversation.

### `last_search_results` / `last_matches` (arrays, nullable, transient)

- **Source:** Workflow 11 (`/jobs/search`) and workflow 13 (`/candidates/:id/matches`) response arrays.
- **Why:** Lets the owner pick by index ("第一个详情", "跳过第三个"). Without this, the agent has to re-query every time.
- **How to apply:** Overwrite on every search/match call. Safe to clear after the user moves to an unrelated topic.

## What NOT to persist

- **API key** (`data.api_key` from workflow 1 step 3) — only returned on first-time register; not needed by any workflow (all runtime auth uses `session_token`). If you do capture it, store it only at the openclaw memory layer, never echo it back in logs or messages.
- **Full profile-intake conversation history** — the server owns this, the agent reads it on demand via workflow 6.
- **Recruiter messages** — not handled by this skill at all (see workflow 15 rationale).
