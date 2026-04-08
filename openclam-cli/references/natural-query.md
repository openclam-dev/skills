# Natural Query Reference

Cross-domain SQL queries via `clam sql`. Use when a request can't be answered by a single domain command — e.g. "find all persons with open deals in Q1 that haven't been emailed in 30 days".

## Workflow

```
1. Check which extensions are enabled
   clam --json status
   → look for enabledExtensions in output

2. Based on what the user wants, identify relevant extension(s) from the table map below.
   Core tables are always available.

3. Look up the columns for those tables in this reference — do NOT guess column names.

4. Construct a read-only SQL query (SELECT only).

5. Execute:
   clam --json sql "SELECT ..."
   clam --json sql --input-json '{"query":"SELECT ..."}'
   → { "rows": [...] }

6. If the data the user wants doesn't exist in any available table, say so.
   Never fabricate table or column names.
```

## Rules

- Only SELECT, WITH, EXPLAIN, PRAGMA are allowed — service rejects everything else
- Do not use INSERT, UPDATE, DELETE, DROP, ALTER, CREATE
- Always check columns before writing the query — the schema is below
- If an extension is not in enabledExtensions, its tables don't exist

---

## Table Reference by Extension

### Core (always available)

**users** — id (text PK), name (text), type (text), identifier (text UNIQUE), is_active (int/bool), secret_hash (text), metadata (text), created_at (text), updated_at (text), deleted_at (text)

**log** — id (text PK), person_id (text), org_id (text), user_id (text FK→users), type (text), subject (text), body (text), source (text), created_at (text)

**config** — key (text PK), value (text), updated_at (text)

**events** — id (text PK), source (text), event_type (text), entity_type (text), entity_id (text), payload (text), user_id (text FK→users), created_at (text), processed_at (text), status (text DEFAULT 'pending'), failure_reason (text), attempts (int DEFAULT 0)

**rules** — id (text PK), slug (text UNIQUE), name (text), event_pattern (text), conditions (text), action (text), priority (int), is_active (int/bool), created_at (text), updated_at (text), deleted_at (text)

**rule_actions** — id (text PK), rule_id (text FK→rules), extension (text), action (text), params_json (text), execution_order (int), created_at (text)

**roles** — id (text PK), slug (text UNIQUE), name (text), is_system (int/bool DEFAULT false), created_at (text), updated_at (text), deleted_at (text)

**role_permissions** — role_id (text FK→roles), permission (text) — composite PK (role_id, permission)

**user_roles** — user_id (text FK→users), role_id (text FK→roles) — composite PK (user_id, role_id)

**projects** — id (text PK), name (text), description (text), status (text), owner_id (text), owner_type (text), created_at (text), updated_at (text), deleted_at (text)

**tags** — id (text PK), label (text), slug (text UNIQUE), color (text), created_by (text FK→users), created_at (text), updated_at (text)

**custom_field_definitions** — id (text PK), entity_type (text), slug (text), label (text), description (text), field_type (text), options_json (text), required (boolean), position (integer), created_at (text), updated_at (text) — UNIQUE(entity_type, slug)

**custom_field_values** — id (text PK), definition_id (text FK→custom_field_definitions CASCADE), entity_type (text), entity_id (text), value_text (text), value_number (integer), value_boolean (boolean), value_date (text), created_at (text), updated_at (text) — UNIQUE(definition_id, entity_id)

---

### Contacts (extension: contacts)

**persons** — id (text PK), org_id (text), first_name (text), last_name (text), title (text), summary (text), image_url (text), created_at (text), updated_at (text), deleted_at (text)

**person_emails** — id (text PK), person_id (text FK→persons CASCADE), email (text), label (text), is_primary (int/bool), created_at (text)

**person_phones** — id (text PK), person_id (text FK→persons CASCADE), phone (text), label (text), is_primary (int/bool), created_at (text)

**person_addresses** — id (text PK), person_id (text FK→persons CASCADE), label (text), street (text), city (text), state (text), postal_code (text), country (text), is_primary (int/bool), created_at (text)

**person_urls** — id (text PK), person_id (text FK→persons CASCADE), url (text), label (text), is_primary (int/bool), created_at (text)

**person_notes** — id (text PK), person_id (text FK→persons CASCADE), title (text), body (text), source (text), created_at (text), updated_at (text), deleted_at (text)

**person_tags** — id (text PK), person_id (text FK→persons CASCADE), tag_id (text), created_at (text)

**person_comments** — id (text PK), person_id (text FK→persons CASCADE), author_id (text), parent_id (text), body (text), status (text), created_at (text), updated_at (text)

**orgs** — id (text PK), name (text), website (text), industry (text), employees (integer), location (text), country (text), image_url (text), accelerator (text), tagline (text), description (text), founded_year (integer), status (text), created_at (text), updated_at (text), deleted_at (text)

**org_emails** — id (text PK), org_id (text FK→orgs CASCADE), email (text), label (text), is_primary (int/bool), created_at (text)

**org_phones** — id (text PK), org_id (text FK→orgs CASCADE), phone (text), label (text), is_primary (int/bool), created_at (text)

**org_addresses** — id (text PK), org_id (text FK→orgs CASCADE), label (text), street (text), city (text), state (text), postal_code (text), country (text), is_primary (int/bool), created_at (text)

**org_urls** — id (text PK), org_id (text FK→orgs CASCADE), url (text), label (text), is_primary (int/bool), created_at (text)

**org_notes** — id (text PK), org_id (text FK→orgs CASCADE), title (text), body (text), source (text), created_at (text), updated_at (text), deleted_at (text)

**org_tags** — id (text PK), org_id (text FK→orgs CASCADE), tag_id (text), created_at (text)

**org_comments** — id (text PK), org_id (text FK→orgs CASCADE), author_id (text), parent_id (text), body (text), status (text), created_at (text), updated_at (text)

**relationships** — id (text PK), person_id_1 (text FK→persons), person_id_2 (text FK→persons), type (text), notes (text), created_at (text), deleted_at (text)

**segments** — id (text PK), name (text), entity_type (text), filter_json (text), created_at (text), updated_at (text), deleted_at (text)

**project_persons** — id (text PK), project_id (text), person_id (text FK→persons CASCADE), added_at (text), added_by (text)

**project_orgs** — id (text PK), project_id (text), org_id (text FK→orgs CASCADE), added_at (text), added_by (text)

---

### Deals (extension: deals, requires: contacts)

**pipelines** — id (text PK), name (text), description (text), is_default (int/bool), created_at (text), updated_at (text), deleted_at (text)

**pipeline_stages** — id (text PK), pipeline_id (text FK→pipelines), name (text), stage_order (int), probability (int), is_win (int/bool), is_loss (int/bool), created_at (text), updated_at (text)

**deals** — id (text PK), name (text), pipeline_id (text FK→pipelines), person_id (text), org_id (text), stage (text), value (real), currency (text), probability (int), expected_close_date (text), notes (text), created_at (text), updated_at (text), deleted_at (text)

**deal_notes** — id (text PK), deal_id (text FK→deals CASCADE), title (text), body (text), source (text), created_at (text), updated_at (text)

**deal_tags** — id (text PK), deal_id (text FK→deals CASCADE), tag_id (text), created_at (text)

**deal_comments** — id (text PK), deal_id (text FK→deals CASCADE), author_id (text), parent_id (text), body (text), status (text), created_at (text), updated_at (text)

**project_deals** — id (text PK), project_id (text), deal_id (text FK→deals CASCADE), added_at (text), added_by (text)

---

### Tasks (extension: tasks)

**tasks** — id (text PK), title (text), description (text), project_id (text), parent_task_id (text), assignee_id (text), assignee_type (text), creator_id (text), creator_type (text), person_id (text), org_id (text), due_date (text), priority (text), status (text), position (int DEFAULT 0), estimated_effort (text), actual_effort (text), source (text), created_at (text), updated_at (text), completed_at (text), deleted_at (text)

**task_comments** — id (text PK), task_id (text FK→tasks CASCADE), author_id (text), parent_id (text), body (text), status (text), created_at (text), updated_at (text)

**project_tasks** — id (text PK), project_id (text), task_id (text FK→tasks CASCADE), added_at (text), added_by (text)

---

### Messaging (extension: messaging, requires: contacts)

**transports** — id (text PK), name (text UNIQUE), type (text), channel (text), config_json (text), is_default (int/bool), created_at (text), updated_at (text), deleted_at (text)

**messages** — id (text PK), person_id (text), transport_name (text), channel (text), to_address (text), subject (text), body (text), status (text), sent_at (text), delivered_at (text), opened_at (text), replied_at (text), bounced_at (text), external_id (text), created_at (text), deleted_at (text)

---

### Marketing (extension: marketing, requires: contacts, messaging)

**sequences** — id (text PK), name (text), description (text), status (text), created_at (text), updated_at (text), deleted_at (text)

**sequence_steps** — id (text PK), sequence_id (text FK→sequences), step_order (int), channel (text), subject (text), body (text), delay_days (int), created_at (text), updated_at (text)

**enrollments** — id (text PK), person_id (text), sequence_id (text FK→sequences), status (text), current_step_order (int), enrolled_at (text), paused_at (text), completed_at (text), cancelled_at (text)

**enrollment_messages** — id (text PK), enrollment_id (text FK→enrollments), message_id (text), created_at (text)

**sequence_comments** — id (text PK), sequence_id (text FK→sequences CASCADE), author_id (text), parent_id (text), body (text), status (text), created_at (text), updated_at (text)

---

### Content (extension: content)

**content_types** — id (text PK), slug (text UNIQUE), name (text), description (text), icon (text), body_format (text), schema_json (text), default_status (text), created_at (text), updated_at (text), deleted_at (text)

**content_items** — id (text PK), type (text), slug (text UNIQUE), status (text), author_id (text), person_id (text), org_id (text), published_version_id (text), scheduled_at (text), published_at (text), created_at (text), updated_at (text), deleted_at (text)

**content_versions** — id (text PK), content_id (text FK→content_items CASCADE), version (int), title (text), body (text), summary (text), metadata_json (text), author_id (text), change_note (text), created_at (text)

**assets** — id (text PK), filename (text), mime_type (text), size_bytes (int), url (text), alt_text (text), width (int), height (int), duration_seconds (int), metadata_json (text), uploaded_by (text), created_at (text), deleted_at (text)

**content_assets** — id (text PK), content_id (text FK→content_items CASCADE), asset_id (text FK→assets), role (text), position (int), created_at (text)

**collections** — id (text PK), name (text), slug (text UNIQUE), type (text), description (text), parent_id (text), created_at (text), updated_at (text), deleted_at (text)

**collection_items** — id (text PK), collection_id (text FK→collections CASCADE), content_id (text FK→content_items CASCADE), position (int), created_at (text)

**content_tags** — id (text PK), content_id (text FK→content_items CASCADE), tag_id (text), created_at (text)

**content_notes** — id (text PK), content_id (text FK→content_items CASCADE), title (text), body (text), source (text), created_at (text), updated_at (text)

**content_comments** — id (text PK), content_id (text FK→content_items CASCADE), version_id (text FK→content_versions), author_id (text), parent_id (text), body (text), anchor (text), status (text), created_at (text), updated_at (text)

**content_channels** — id (text PK), content_id (text FK→content_items CASCADE), channel (text), external_id (text), status (text), published_at (text), created_at (text)

**content_relations** — id (text PK), source_id (text FK→content_items CASCADE), target_id (text FK→content_items CASCADE), type (text), position (int), created_at (text)

**content_locales** — id (text PK), content_id (text FK→content_items CASCADE), version_id (text FK→content_versions CASCADE), locale (text), created_at (text)

**project_content** — id (text PK), project_id (text), content_id (text FK→content_items CASCADE), added_at (text), added_by (text)

---

### Calendar (extension: calendar, requires: contacts)

**calendars** — id (text PK), name (text), description (text), color (text), owner_id (text), owner_type (text), is_default (int/bool), created_at (text), updated_at (text), deleted_at (text)

**calendar_events** — id (text PK), calendar_id (text FK→calendars CASCADE), title (text), description (text), location (text), start_at (text), end_at (text), all_day (int/bool), status (text), recurrence_rule (text), created_by (text), created_at (text), updated_at (text), deleted_at (text)

**calendar_event_attendees** — id (text PK), event_id (text FK→calendar_events CASCADE), person_id (text), user_id (text), status (text), is_organizer (int/bool), created_at (text)

**calendar_event_comments** — id (text PK), event_id (text FK→calendar_events CASCADE), author_id (text), parent_id (text), body (text), status (text), created_at (text), updated_at (text)

**calendar_event_tags** — id (text PK), event_id (text FK→calendar_events CASCADE), tag_id (text), created_at (text)

**project_calendar_events** — id (text PK), project_id (text), event_id (text FK→calendar_events CASCADE), added_at (text), added_by (text)

---

### Research (extension: research)

**research_topics** — id (text PK), name (text), description (text), color (text), status (text), created_by (text), created_at (text), updated_at (text), deleted_at (text)

**research_topic_persons** — id (text PK), topic_id (text FK→research_topics CASCADE), person_id (text), created_at (text)

**research_topic_orgs** — id (text PK), topic_id (text FK→research_topics CASCADE), org_id (text), created_at (text)

**research_runs** — id (text PK), topic_id (text FK→research_topics CASCADE), status (text), summary (text), started_at (text), completed_at (text), created_at (text)

**research_sources** — id (text PK), run_id (text FK→research_runs CASCADE), url (text), source_type (text), title (text), author (text), published_at (text), summary (text), created_at (text)

**research_findings** — id (text PK), run_id (text FK→research_runs CASCADE), source_id (text FK→research_sources SET NULL), title (text), content (text), confidence (int), embedding (text), embedding_model (text), embedding_dimensions (int), created_at (text)

**research_briefings** — id (text PK), topic_id (text FK→research_topics CASCADE), title (text), summary (text), created_at (text)

**research_briefing_insights** — id (text PK), briefing_id (text FK→research_briefings CASCADE), question (text), content (text), order (int), created_at (text)

**research_briefing_insight_findings** — id (text PK), insight_id (text FK→research_briefing_insights CASCADE), finding_id (text FK→research_findings CASCADE), relevance_score (int), created_at (text)

---

### Storage (extension: storage)

**storage_buckets** — id (text PK), name (text), slug (text UNIQUE), description (text), uri (text), config (text), created_by (text), created_at (text), updated_at (text)

**storage_files** — id (text PK), bucket_id (text FK→storage_buckets CASCADE), path (text), filename (text), mime_type (text), size (int), created_by (text), created_at (text), updated_at (text)

**storage_bucket_permissions** — id (text PK), bucket_id (text FK→storage_buckets CASCADE), user_id (text), permission (text), granted_by (text), created_at (text)

---

### Cron (extension: cron)

**cron_jobs** — id (text PK), slug (text UNIQUE), name (text), schedule (text), handler (text), params_json (text), is_active (int/bool), consecutive_errors (int), running_since (text), timezone (text), last_run_at (text), next_run_at (text), created_at (text), updated_at (text), deleted_at (text)

**cron_runs** — id (text PK), job_id (text FK→cron_jobs), status (text), started_at (text), finished_at (text), duration_ms (int), error (text), created_at (text)

---

### Webhooks (extension: webhooks)

**event_subscribers** — id (text PK), name (text), url (text), event_pattern (text), conditions (text), secret (text), headers_json (text), is_active (int/bool), retry_policy (text), max_retries (int), created_at (text), updated_at (text), deleted_at (text)

**event_deliveries** — id (text PK), subscriber_id (text FK→event_subscribers), event_id (text FK→events), status (text), attempts (int), last_attempt_at (text), next_retry_at (text), status_code (int), response_body (text), failure_reason (text), created_at (text)

---

### Workflows (execution layer)

**workflow_definitions** — id (text PK), name (text UNIQUE), description (text), file_path (text), triggers_json (text), is_valid (int/bool), error (text), is_active (int/bool), loaded_at (text), created_at (text), updated_at (text)

**workflow_runs** — id (text PK), workflow_name (text), status (text), input_json (text), output_json (text), error (text), trigger_type (text), trigger_event_id (text), parent_run_id (text), parent_step_name (text), worker_id (text), available_at (text), consecutive_errors (int), attempt_number (int), max_attempts (int), started_at (text), finished_at (text), created_at (text)

**workflow_step_attempts** — id (text PK), workflow_run_id (text FK→workflow_runs), name (text), ordinal (int), step_type (text), status (text), output_json (text), error (text), started_at (text), finished_at (text), duration_ms (int)

**workflow_event_waiters** — id (text PK), workflow_run_id (text FK→workflow_runs), step_name (text), event_pattern (text), match_json (text), timeout_at (text), created_at (text)

---

## Examples

### Persons with most deals
```sql
SELECT p.first_name, p.last_name, COUNT(d.id) as deal_count
FROM persons p
LEFT JOIN deals d ON d.person_id = p.id AND d.deleted_at IS NULL
WHERE p.deleted_at IS NULL
GROUP BY p.id
ORDER BY deal_count DESC
LIMIT 10
```

### Open deals with person and org details
```sql
SELECT d.name as deal, d.value, d.stage,
       p.first_name, p.last_name,
       o.name as org_name
FROM deals d
LEFT JOIN persons p ON d.person_id = p.id
LEFT JOIN orgs o ON d.org_id = o.id
WHERE d.deleted_at IS NULL
ORDER BY d.value DESC
LIMIT 20
```

### Persons not emailed in the last 30 days
```sql
SELECT p.id, p.first_name, p.last_name, MAX(m.sent_at) as last_emailed
FROM persons p
LEFT JOIN messages m ON m.person_id = p.id AND m.channel = 'email'
WHERE p.deleted_at IS NULL
GROUP BY p.id
HAVING last_emailed IS NULL OR last_emailed < datetime('now', '-30 days')
LIMIT 20
```

### Tasks by status with assignee
```sql
SELECT t.title, t.status, t.priority, t.due_date, u.name as assignee
FROM tasks t
LEFT JOIN users u ON t.assignee_id = u.id
WHERE t.deleted_at IS NULL
ORDER BY t.priority, t.due_date
```

### Content items with latest version title
```sql
SELECT ci.id, ci.status, cv.title, cv.version, ci.published_at
FROM content_items ci
JOIN content_versions cv ON cv.content_id = ci.id
WHERE ci.deleted_at IS NULL
  AND cv.version = (SELECT MAX(v2.version) FROM content_versions v2 WHERE v2.content_id = ci.id)
ORDER BY ci.updated_at DESC
LIMIT 20
```

### Cross-domain: deals + tasks for a project
```sql
SELECT 'deal' as type, d.name as title, d.stage as status, d.value
FROM project_deals pd
JOIN deals d ON d.id = pd.deal_id
WHERE pd.project_id = '<project-id>' AND d.deleted_at IS NULL
UNION ALL
SELECT 'task' as type, t.title, t.status, NULL as value
FROM project_tasks pt
JOIN tasks t ON t.id = pt.task_id
WHERE pt.project_id = '<project-id>' AND t.deleted_at IS NULL
```

### PRAGMA introspection (SQLite only)
```sql
PRAGMA table_info(persons)
```
