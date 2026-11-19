# NirmanEcho

NirmanEcho now has a local browser prototype and an integration-ready FastAPI backend. The browser app has a demo mode; signing in to a configured backend switches voice reports and project records to Supabase and Sarvam.

## Start the API and app

Use Python 3.10 or later. In PowerShell, from this folder:

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env
```

Fill in `.env` using the steps below, then run:

```powershell
uvicorn backend.main:app --reload --port 8000
```

Open `http://localhost:8000`. The backend serves the browser app and API from the same origin. Microphone capture needs localhost or HTTPS; opening the HTML directly from disk will not connect to the API.

## 1. Prepare Supabase

1. Create a Supabase project. Copy its project URL, publishable/anon key, and server-only service role/secret key into `.env` as `SUPABASE_URL`, `SUPABASE_ANON_KEY`, and `SUPABASE_SERVICE_ROLE_KEY`.
2. In **SQL Editor**, run [`backend/schema.sql`](backend/schema.sql). It creates the project tables, membership checks, private voice-evidence bucket, and read policies. If the backend reports `permission denied for table projects` / SQLSTATE `42501`, also run [`backend/service_role_grants.sql`](backend/service_role_grants.sql) to grant the server role access to the app tables.
3. Create a user in **Authentication → Users**. Use the account email/password in NirmanEcho’s **Connect backend** sign-in panel.
4. Create the project, project membership, and sites for that user in the SQL Editor. Replace the email and project/site names below:

```sql
with new_project as (
  insert into public.projects (name, location)
  values ('Green Heights', 'Hyderabad')
  returning id
), project_owner as (
  insert into public.project_members (project_id, user_id, role)
  select new_project.id, auth_user.id, 'owner'
  from new_project
  cross join (select id from auth.users where email = 'manager@example.com' limit 1) auth_user
  returning project_id
)
insert into public.sites (project_id, name, tower, floor)
select project_id, 'Tower B — Third floor', 'Tower B', 'Third floor' from project_owner
union all
select project_id, 'Tower A — Second floor', 'Tower A', 'Second floor' from project_owner
union all
select project_id, 'Clubhouse — Ground floor', 'Clubhouse', 'Ground floor' from project_owner;
```

If the email is wrong or no matching Auth user exists, the project is created without an owner membership. Check the email before running this setup query.

## 2. Connect Sarvam

Create a Sarvam API key and set `SARVAM_API_KEY` in `.env`. The backend sends audio to Saaras v4 batch transcription, then sends the transcript to Sarvam-105B for structured extraction. It stores the original audio in the private Supabase bucket. The API key stays on the server. Extracted changes remain pending until a project user confirms them in the app.

## 3. Connect WhatsApp (optional later stage)

This backend includes a signed Meta webhook at `/api/webhooks/whatsapp`. To receive live voice notes:

1. Deploy the API at a public HTTPS URL.
2. Create/configure a Meta app with the WhatsApp product and a WhatsApp Business number.
3. Set `WHATSAPP_VERIFY_TOKEN`, `WHATSAPP_APP_SECRET`, `WHATSAPP_ACCESS_TOKEN`, `WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_GRAPH_API_VERSION`, and `PUBLIC_BASE_URL` in the deployment’s secret settings.
4. Set the Meta webhook callback URL to `https://YOUR_HOST/api/webhooks/whatsapp`, enter the same verify token, and subscribe to WhatsApp message events.
5. Add each authorized sender to `public.whatsapp_memberships` with their phone number, project, site, and Auth user ID. Unmapped senders are ignored.

The webhook currently receives and processes mapped audio notes; it does not send an automatic WhatsApp reply or place supplier orders. Keep the endpoint on HTTPS and verify the Meta signature. Do not use a development server as the production webhook host.

## What is connected in code

- Supabase Auth login; project membership is checked on every project API request.
- Private audio evidence storage, Sarvam speech-to-text and structured report extraction.
- Human confirmation before labour, material, progress, shortage alert, and draft purchase-request records are written.
- Conflict and clarification alerts, manager-only alert approval, live project lists, and list/create endpoints for labour, materials, and progress.
- Meta webhook verification, HMAC signature checks, audio media download, sender-to-project mapping, and reuse of the same voice processing pipeline.
- Local demo fallback when the API is not configured; demo records remain in browser `localStorage`.

## Current boundaries

No service can process live project data until you create the provider accounts, run the schema, fill in `.env`, and create a project membership and site. I have not deployed the backend or configured external accounts, because those require your account access and secrets. Never commit `.env` or paste API keys into chat. WhatsApp delivery acknowledgements, password recovery, rate limiting, background job persistence, deployment, backups, and production observability still need to be set up before a real-site rollout. The included sample content and figures are illustrative.
