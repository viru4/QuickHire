# QHire Interview — Basic Questions

## 1) Explain QuickHire
Answer:
QuickHire is a lightweight job portal web application that connects employers and candidates. It provides features to post and manage jobs, create and view company profiles, submit and track applications, save jobs, and run simple onboarding flows. Key functional components include:

- Public job listings and search (filters, categories, remote/location)
- Employer dashboard to create and manage job posts and view applications
- Candidate features: apply to jobs, upload resumes, save jobs, view application status
- Company profiles with logo, description, and open roles
- Authentication and role separation (candidates, employers, admins)
- Notifications for new applications, status changes, and messages

Example:
User story — As an employer I create a job posting, candidates apply, and I review applications in a dedicated dashboard. The UI shows application cards with candidate info and resume links.

## 2) Why Supabase?
Answer:
Supabase is chosen for its combination of a hosted Postgres database, realtime capabilities, storage, and built-in auth helpers — all accessible through a simple client SDK. Benefits:

- Postgres power: ACID transactions, joins, full SQL, referential integrity.
- Realtime updates: listen to table changes for in-app notifications and live lists.
- Storage: store uploaded resumes, company logos, and other assets.
- Row-Level Security (RLS): implement fine-grained access policies server-side.
- Quick to prototype: SDKs for web/mobile reduce backend code and hosting needs.

Example implementation:

- Jobs table: jobs(id, company_id, title, description, posted_at, is_active) stored in Postgres.
- Applications table with foreign keys and RLS policy so only the employer and the applicant can access specific rows.

## 3) Why Clerk authentication?
Answer:
Clerk is a modern user management/auth provider that speeds up building secure auth flows with pre-built UI components (sign-in, sign-up, sessions), passwordless and social login support, and user management APIs. Reasons to pick Clerk:

- Fast developer experience: drop-in UI components and SDKs.
- Secure session management and refresh flows handled by the provider.
- Rich user metadata: store custom fields (role, company_id) in user profiles.
- Works well alongside Supabase (Clerk for identity, Supabase for data), avoiding building auth from scratch.

Example integration pattern:

- Use Clerk for user sign-in. On new user sign-up, create a corresponding profiles row in Supabase containing id: clerkUserId, role: candidate|employer, and display_name.
- Use Clerk session tokens to authenticate API requests to server endpoints. For Supabase policies, reference the stored profiles.role when evaluating permissions.

## 4) What is role-based access?
Answer:
Role-Based Access Control (RBAC) assigns permissions to users based on roles (e.g., candidate, employer, admin). Instead of granting permissions per-user, roles group permissions and simplify authorization logic.

How it’s implemented in QuickHire:

- Store a role on each user profile (in Clerk metadata or a Supabase profiles table).
- Enforce permissions both client-side (UI) and server-side (database policies / edge functions).
- Use Supabase RLS policies to protect rows: e.g., employers can edit jobs they own; candidates can only view their own applications.

Example RLS policy (conceptual SQL):

-- Allow employers to update jobs they own
CREATE POLICY "employer_update_own_job" ON jobs
FOR UPDATE USING (
  EXISTS (
    SELECT 1 FROM profiles p WHERE p.id = auth.uid AND p.role = 'employer' AND p.company_id = jobs.company_id
  )
);

## 5) How notifications work?
Answer:
Notifications combine realtime in-app updates, email, and optional push notifications:

- Realtime in-app: use Supabase Realtime (or websockets) to subscribe to table changes (e.g., new application rows). The client updates UI immediately when events arrive.
- Server-side/email: an Edge Function or webhook sends transactional emails (new application received, status changed) via an email provider (SendGrid, Postmark).
- Push notifications: optionally integrate with FCM or OneSignal for mobile/desktop push.

Example flow — New application:

1. Candidate submits application -> applications row is inserted into Supabase.
2. Supabase Realtime pushes an INSERT event to the employer's dashboard; UI shows a live badge.
3. An Edge Function (trigger or Change Listener) sends an email to the employer with applicant details and a link to the dashboard.

Snippet (client listen):

supabase.from('applications').on('INSERT', payload => {
  // show in-app notification and increment badge
  showNotification(`New application for ${payload.new.job_title}`)
})


## 6) How AI chatbot integrated?
Answer:
The application embeds a Botpress Cloud webchat widget (see `index.html`) which injects the chat UI and connects clients to a remote Botpress bot instance. The chatbot logic, connectors to LLMs, and any RAG/retrieval pipelines run on the Botpress server (hosted/managed), not in this repo.

How it works in this project:

- The app loads `https://cdn.botpress.cloud/webchat/.../inject.js` and a Botpress content script that initializes the widget in the browser.
- The widget opens a websocket/HTTP connection to the Botpress cloud instance which handles incoming messages, dialogue flows, and any configured LLM or retrieval integrations.
- The local codebase does not contain `/api/chat` or embedding logic; changing bot behavior requires editing the Botpress bot (dashboard) or the remote content script.

Implications:

- The repo only provides the client-side widget integration; all prompt/LLM configuration, prompt engineering, RAG pipelines, and fallback behaviour are controlled in Botpress.
- Security and keys for LLM providers are kept on the Botpress server. To add an in-repo RAG endpoint, implement a server route (e.g., `/api/chat`) and wire it to an embeddings + models provider.

---

If you want, I can add more technical snippets (Clerk <> Supabase session exchange, full Edge Function code for notifications, or a working RAG endpoint) into this QHire-Interview.md file.

## Supabase Questions

### 1) What is Supabase?
Answer:
Supabase is an open-source Backend-as-a-Service that provides a set of composable backend building blocks on top of PostgreSQL: a hosted Postgres database, realtime change streaming, object storage, authentication, and edge/Serverless functions. It exposes these features through a unified client SDK (supabase-js) and admin tools (Dashboard, SQL editor). Supabase aims to be an open-source alternative to proprietary BaaS platforms while keeping the developer experience friendly.

### 2) Difference between Supabase and Firebase?
Answer:
- Database model: Supabase uses PostgreSQL (relational, SQL); Firebase Realtime Database / Firestore are NoSQL document stores. SQL enables complex joins, transactions, and familiar tooling.
- Realtime approach: Supabase streams Postgres WAL changes (via replication) to clients; Firebase uses its proprietary realtime protocol. Both provide low-latency updates, but Supabase's model fits SQL schemas.
- Querying: Supabase offers full SQL and Postgres ecosystem (extensions, functions); Firebase uses document queries with limited relational features.
- Security model: Supabase supports Postgres Row-Level Security (RLS) for fine-grained server-side policies. Firebase uses security rules tied to its DB structure.
- Storage: both provide object storage; Supabase Storage is backed by Postgres ecosystem and S3-compatible storage.
- Open source & portability: Supabase components are open source and can be self-hosted; Firebase is proprietary to Google.

Example: choosing Supabase is beneficial when your app needs relational integrity, complex queries, and server-side policies (RLS). Choose Firebase when you prefer schema-less documents and deep GCP integration.

### 3) What database does Supabase use?
Answer:
Supabase uses PostgreSQL (Postgres) as its core database. Supabase leverages Postgres features like transactions, joins, JSONB, extensions (pgcrypto, pg_trgm, pgvector), and RLS. You can run raw SQL, create functions, and use Postgres indexes and extensions for advanced functionality (e.g., vector search with `pgvector`).

Example schema (SQL):

```sql
CREATE TABLE companies (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL,
  logo_url text
);

CREATE TABLE jobs (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  company_id uuid REFERENCES companies(id),
  title text,
  description text,
  posted_at timestamptz DEFAULT now()
);
```

### 4) What are realtime features?
Answer:
Supabase Realtime lets clients subscribe to database changes and receive INSERT/UPDATE/DELETE events in near real-time. It's implemented by tailing Postgres WAL and broadcasting events to listening clients. Common uses: live feeds, notifications, collaborative UIs, and caches invalidation.

Client subscription example (supabase-js):

```js
supabase
  .channel('public:applications')
  .on('postgres_changes', { event: 'INSERT', schema: 'public', table: 'applications' }, payload => {
    console.log('New application:', payload.new);
  })
  .subscribe();
```

You can also listen to row-level changes for a specific filter (e.g., company_id) and broadcast updates only to relevant users.

### 5) How authentication works in Supabase?
Answer:
Supabase Auth is built around JWT-based sessions and supports multiple auth flows: email/password, magic links, OAuth (GitHub, Google, etc.), and third-party providers. When a user signs up or signs in, Supabase issues an access token (JWT) and a refresh token. The client SDK stores the session and attaches the access token to requests. On the server (Postgres), `auth.uid()` (or `auth.jwt()` claims) are available in RLS policies to enforce permissions.

Example client sign-up / sign-in (supabase-js):

```js
// Sign up with email
const { data, error } = await supabase.auth.signUp({ email, password });

// Sign in
const { data: session } = await supabase.auth.signInWithPassword({ email, password });
```

Example RLS policy tying rows to authenticated user:

```sql
-- allow users to SELECT their own applications
CREATE POLICY "select_own_applications" ON applications
FOR SELECT USING ( applicant_id = auth.uid() );
```

Notes on best practices:
- Use RLS to enforce server-side access — never rely solely on client-side checks.
- Keep tokens secure and rotate keys when needed. Use refresh tokens and short access token TTLs for sensitive operations.
- Integrate Supabase Auth with external identity providers (or Clerk) by syncing profile metadata into a `profiles` table and referencing it in policies.

### 6) How are uploads handled in QuickHire and how does it work end-to-end?
Answer:
QuickHire handles uploads through Supabase Storage with authenticated client requests. There are two upload flows in the app:

- Resume uploads (candidate job applications) -> `resumes` bucket
- Company logo uploads (company creation) -> `company-logo` bucket

End-to-end flow:

1. User selects a file in the UI form.
2. Client validates input (for resumes: required file + allowed document MIME type checks).
3. App obtains a Clerk session token and exchanges it as a Supabase-authenticated bearer token.
4. File is uploaded to Supabase Storage using `supabase.storage.from(bucket).upload(fileName, file)`.
5. App builds a public object URL and stores that URL in the relational table row:
  - `applications.resume` for resumes
  - `companies.logo_url` for company logos

Why this pattern is useful:

- Binary files stay in object storage (better for cost/performance than DB blobs).
- Postgres tables keep only metadata/URLs, making queries fast and simple.
- With Storage policies + RLS, access control remains enforceable server-side.

Important setup note:

- Buckets must exist in Supabase Storage (`resumes`, `company-logo`).
- Public read or signed URL strategy must match how URLs are generated in code.

---

Let me know if you want these Supabase answers expanded into separate deep-dive sections (examples for RLS policies, Edge Function notification code, or a small reproducible demo). 

## Clerk Questions

### 1) Why Clerk?
Answer:
Clerk is an identity and user management platform focused on developer experience and secure, production-ready auth flows. It provides hosted authentication, pre-built UI components (sign-in, sign-up, user profile), session management, and easy integration with passwordless and social providers. Key reasons to use Clerk:

- Developer speed: drop-in UI components and SDKs for React, Next.js, and other frameworks.
- Security: built-in best-practice session handling, refresh token lifecycles, and secure cookies.
- Flexibility: supports passwordless (magic links), social OAuth, multi-factor, and custom user metadata.
- Extensibility: exposes user and session APIs that make server-side verification and custom onboarding straightforward.

Example: replace a home-grown auth flow with Clerk to avoid reinventing secure session handling and reduce time-to-market for sign-in UX.

### 2) JWT flow in Clerk?
Answer:
Clerk issues short-lived JWT access tokens for authenticated sessions and provides refresh tokens or session cookies depending on the integration. Typical flow:

1. User signs in via Clerk widget or API (email/password, OAuth, magic link).
2. Clerk returns a session object to the client and sets a secure cookie (or returns tokens for SPA flows).
3. For API calls, the client includes the Clerk session token (Bearer JWT) or cookie with requests.
4. Server validates the token with Clerk (via Clerk SDK or by verifying JWT signature and checking claims) and extracts the user ID and metadata.

Example verification (Node/Express using Clerk SDK):

```js
import { clerkClient, requireSession } from '@clerk/clerk-sdk-node';

app.get('/api/protected', requireSession(), (req, res) => {
  // req.auth contains session and user info provided by Clerk middleware
  res.json({ userId: req.auth.userId, sessionId: req.auth.sessionId });
});
```

Notes:
- You can validate tokens locally by checking the JWT signature and claims, or call Clerk's API to introspect sessions.
- When combining Clerk with another backend (e.g., Supabase), persist the Clerk `userId` in your `profiles` table to map identity to application data.

### 3) How protected routes work?
Answer:
Protected routes restrict access to authenticated users (and often require specific roles). Implementation layers:

- Client-side: hide UI and redirect unauthenticated users to sign-in (convenient UX but not secure by itself).
- Server-side / API: require valid Clerk session token or cookie; middleware enforces authentication before handlers run.
- Database-level: combine Clerk identity with database RLS or server-side checks to ensure only authorized users access data.

Example (React + Clerk + route guard):

```jsx
import { SignedIn, SignedOut, RedirectToSignIn } from '@clerk/clerk-react';

function Dashboard() {
  return (
    <SignedIn>
      <EmployerDashboard />
    </SignedIn>
    <SignedOut>
      <RedirectToSignIn />
    </SignedOut>
  );
}
```

Server example (Express middleware):

```js
app.use('/api/secure', requireSession(), secureRouter);
```

Role checks:
- After verifying the session, fetch the user's role (from Clerk metadata or your `profiles` table) and enforce role-based access (e.g., `if (role !== 'employer') return 403`).

### 4) Session management?
Answer:
Clerk manages sessions with secure cookies or tokens and provides APIs to list, revoke, and inspect sessions. Best practices:

- Use HTTP-only secure cookies for web apps to reduce token leakage risk.
- Keep access tokens short-lived and rely on Clerk's refresh/session management for continuity.
- Revoke sessions on logout or suspicious activity via Clerk's session API.
- Monitor active sessions and offer users a session management UI (list and revoke devices).

Example operations (Clerk server SDK):

```js
// Revoke a session
await clerkClient.sessions.revokeSession(sessionId);

// List sessions for a user
const sessions = await clerkClient.sessions.getUserSessions(userId);
```

Integration notes:
- Sync Clerk `userId` to your application `profiles` row on first sign-up so RLS policies and application data can reference the same identity.
- For SSR frameworks, use Clerk's server components or middleware to retrieve the session during page rendering and protect pages.

---

I finished adding the Clerk Q&A. I can expand any answer with code samples (token verification, RLS examples referencing Clerk IDs, or a demo protected route) — which would you like next?

## AI Chatbot Questions

### 1) Which API / provider is used?
Answer:
The app embeds a Botpress Cloud webchat widget. The widget connects users to a remote Botpress bot instance which runs the chat logic and any configured LLM or retrieval integrations. The repository itself does not directly call OpenAI; the LLM provider (OpenAI, Anthropic, etc.) is configured inside the Botpress instance.

### 2) Prompt engineering basics?
Answer:
Prompting still matters, but prompt/templates live in the Botpress bot (flows, actions, and content). Use system-level instructions, structured context, and clear output formats inside Botpress actions or custom code blocks to guide the backend LLM.

### 3) How chatbot responses are generated?
Answer:
Responses are generated on the Botpress server. The typical Botpress setup:

1. Widget sends user message to Botpress.
2. Botpress runs flow/actions: it may call an embeddings service, perform retrieval, or call an LLM provider according to the bot configuration.
3. Botpress returns the response to the widget which renders it in the client.

### 4) How API calls are handled?
Answer:
All LLM/embedding/third-party calls happen from the Botpress server. The client-side widget only forwards messages. API keys and timeouts are managed on the Botpress side; this repo only includes the widget integration.

### 5) What happens if the bot backend fails?
Answer:
Failures are handled by Botpress configuration: retries, fallback messages, or serving canned responses. The widget will show the bot's fallback reply; client-side you can also hide the widget or show an offline message.

### 6) How did chatbot improve query resolution?
Answer:
Improvements come from updating the Botpress bot: adding documents to retrieval, refining prompts/templates, improving flows, and using analytics from Botpress to iterate on failure cases.

---

I updated the AI Chatbot Q&A to reflect the Botpress widget implementation. If you want, I can extract the remote content script URL and summarize its widget configuration next.
