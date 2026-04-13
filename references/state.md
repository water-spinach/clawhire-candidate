# State the agent must carry across turns

The `clawhire-candidate` skill does not persist state itself — openclaw's runtime memory (SQLite at `~/.openclaw/memory/main.sqlite`) does. This file defines the **contract**: what keys the workflows read from and write to, and why each matters. If a workflow step uses a state key not listed here, either the workflow or this file has a bug.

The server owns most of what used to be state: the profile draft, the conversation history, the latest job deck, and the current phase all live server-side and arrive fresh in every workflow 2 response. That's why this list is short.

## Keys

### `session_token` (string, sensitive)

- **Source:** Response of workflow 1 step 3 (`/auth/sms/verify`).
- **Why:** Every authed call needs it as `Authorization: Bearer ${session_token}`. Without it, every request returns 401.
- **How to apply:** Send on every request except `/auth/sms/*`. On any 401 response, drop the value and re-run workflow 1.
- **Rotation:** Tokens may expire server-side — treat 401 as "time to re-auth", not an error.

### `account_id` (string)

- **Source:** `account.id` field of workflow 1 step 3 response.
- **Why:** Used to namespace the server-side A2C session (the backend derives `session_id = "clawhire-a2c-" + account_id`). Also useful for future workflows that need the owner's account id in the path.
- **How to apply:** Stash after verify. You don't put it in URLs — the server derives session IDs from the bearer token's account — but the agent may want to reference it for disambiguation in multi-account testing.

### `phone` (string, E.164)

- **Source:** Either `CLAWHIRE_PHONE` env var or the owner's first reply in workflow 1 step 1.
- **Why:** Needed to re-run workflow 1 after token expiry.
- **How to apply:** Never share with recruiters, never echo in full back to the owner in other workflows (mask as `+86138****1111` when displaying in workflow 10).

### `account_type` (constant: `"candidate"`)

- **Why:** The go-clawhire backend routes requests by account type. Using the wrong type returns wrong matches or 403s.
- **How to apply:** Always `"candidate"` for this skill. If the owner asks about recruiter features, tell them they'd need the `clawhire-recruiter` skill instead (future work).

### `profile_id` (string, nullable)

- **Source:** `data[0].id` from workflow 8 (`GET /candidates/profiles?per_page=1`). The server auto-creates the profile during workflow 2; this key is only needed for the activation toggle in workflow 7.
- **Why:** Workflow 7 (`PATCH /candidates/profiles/:id`) needs the id in the URL.
- **How to apply:** Populate lazily — only when the owner asks to activate/deactivate. Don't fetch it up front.

### `latest_cv_snapshot` (object, nullable)

- **Source:** Full response data of workflow 9 (`GET /chat/extract-cv`).
- **Why:** Lets the agent show the owner what the server currently has on file without re-fetching every turn, and lets it diff against a later snapshot to describe changes.
- **How to apply:** Overwrite on every successful workflow 9 call. Clear only when the owner says "重新开始" / "start over".

### `last_chat_jobs` (array, nullable, transient)

- **Source:** `data.jobs` array from the most recent workflow 2 response.
- **Why:** Lets the owner pick by index ("第一个详情", "投第二个") without the agent re-asking the server. Also lets workflow 5 (apply) resolve the target job without a separate lookup.
- **How to apply:** Overwrite on every workflow 2 call that returns a non-empty `jobs`. Do NOT clear when jobs is empty — the server might be in a conversational phase between deck turns; hold onto the last deck until a new one arrives.

### `active_job_id` (string, nullable, transient)

- **Source:** Set by workflow 2 Step 3b (defaults to `last_chat_jobs[0].job_id`) or workflow 5 Step 1 (when the owner picks a specific index).
- **Why:** Workflow 5 (apply) needs to know which job to send in the `action: "apply"` payload when the owner says just "投递" without specifying an index.
- **How to apply:** Set whenever a fresh `jobs` list arrives. Clear after workflow 5 completes, or after 5 turns of unrelated conversation.

## What NOT to persist

- **API key** (`data.api_key` from workflow 1 step 3) — only returned on first-time register; not needed by any workflow (all runtime auth uses `session_token`). If you do capture it, store it only at the openclaw memory layer, never echo it back in logs or messages.
- **Full profile-intake conversation history** — the server owns this. Re-fetch via workflow 6 on demand.
- **The extracted candidate_profile from workflow 2 responses** — the server already persisted it to the DB via `syncCandidateProfile`. Holding a local copy just risks staleness. Use workflow 9 if you want a canonical read.
- **Recruiter messages** — not handled by this skill at all (see workflow 5 rationale).
- **Match statuses, match ids, ranked match lists** — no chat path for match CRUD. The `data.jobs` array from workflow 2 is the source of truth for "what should I apply to next"; everything else is server-owned.
