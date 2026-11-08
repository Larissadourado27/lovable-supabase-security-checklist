# Lovable + Supabase Security Checklist

A practical checklist to secure apps built with Lovable, Bolt, Cursor or v0 on top of Supabase.

AI builders ship fast, but they rarely finish the security work. Every item below comes from real audits of AI-built React + Supabase apps. Go through it before you put real users and real data on your app.

> Only run these checks on apps you own or are authorized to test.

---

## 1. Row Level Security (RLS)

- [ ] **RLS is enabled on every table** in the `public` schema (and any other schema exposed through the API).
- [ ] **No table is readable or writable by everyone.** Look for policies with `using (true)` or `with check (true)` on tables that hold user data.
- [ ] **Policies are scoped to the user**, usually `user_id = auth.uid()` (or through a helper like `is_member_of(org_id)`).
- [ ] **Insert and update policies have a `with check`.** Without it, a user may create rows for someone else or move a row to another owner.
- [ ] **Foreign keys are checked too.** A user should not be able to attach their record to another user's record (for example, an order pointing to someone else's customer). Use `exists (...)` checks in `with check`.
- [ ] **Users cannot edit privileged columns.** If users can update their own profile row, make sure they cannot change `role`, `plan`, `balance` or similar fields. Use column privileges or a trigger.

Find tables without RLS:

```sql
select schemaname, tablename
from pg_tables
where schemaname = 'public' and rowsecurity = false;
```

List every policy and read it:

```sql
select tablename, policyname, roles, cmd, qual, with_check
from pg_policies
where schemaname = 'public'
order by tablename;
```

Lock privileged columns (example):

```sql
revoke update on public.profiles from authenticated;
grant update (full_name, avatar_url) on public.profiles to authenticated;
```

## 2. Views and database functions

- [ ] **Views respect RLS.** A normal view runs with its owner's rights and can bypass RLS. Create views with `security_invoker = true`.
- [ ] **`SECURITY DEFINER` functions are locked down.** They run with admin rights and bypass RLS. Pin the `search_path`, check ownership inside the function, and remove API access if only triggers use it.
- [ ] **No admin function is callable by `anon`.** Functions in `public` can be called through `/rest/v1/rpc/<name>`.

```sql
create view public.my_view with (security_invoker = true) as
select ...;

alter function public.my_admin_fn(uuid) set search_path = '';
revoke execute on function public.my_admin_fn(uuid) from public, anon, authenticated;
```

## 3. Keys and secrets

- [ ] **You understand which key is public.** The Supabase anon / publishable key is meant to be in the frontend. It is safe only if RLS is correct.
- [ ] **The service role / secret key is never in the frontend.** In Vite, any variable starting with `VITE_` ends up in the browser bundle.
- [ ] **No secrets in Git.** `.env` is in `.gitignore`, and no key was committed in the past (check the history). If one was, rotate it.
- [ ] **Third-party keys (OpenAI, Stripe secret, etc.) live only on the server** (Edge Functions or your backend).

## 4. Edge Functions and backend

- [ ] **Every function checks who is calling.** If `verify_jwt = false`, the function must do its own authentication.
- [ ] **Never trust IDs from the request body.** If the client sends `notebook_id`, `order_id` or `file_path`, check that it belongs to the caller before using it with the service role. This is the most common serious bug in AI-built backends.
- [ ] **Webhooks are verified.** Payment and automation webhooks (Stripe, Kiwify, n8n, etc.) must check a signature or a secret token. Otherwise anyone can send fake events.
- [ ] **Callbacks are not public.** Endpoints that receive results from background jobs must require a secret.
- [ ] **No personal data or secrets in logs.**
- [ ] **AI endpoints have limits.** Cap input size and rate-limit calls, or one user can run up your bill.

## 5. Storage

- [ ] **Private files live in private buckets.**
- [ ] **Storage policies check ownership**, for example by folder: `(storage.foldername(name))[1] = auth.uid()::text`.
- [ ] **Public buckets hold only public content**, and upload is restricted.

## 6. Authentication settings

- [ ] **Sign-ups are disabled** if the app is internal or invite-only.
- [ ] **Email confirmation is on** for public apps.
- [ ] **Redirect URLs are restricted** to your own domains.

## 7. Frontend

- [ ] **Role checks in React are only for the UI.** Hiding a button does not protect data. The database (RLS) must enforce it.
- [ ] **Cached data is cleared on logout.** If you persist React Query or app state in `localStorage`, clear it on sign-out, or the next user on the same device may see it.
- [ ] **Sensitive data is not hard-coded** in the source (real financial data, customer lists, seed data). Everything in `src/` is shipped to the browser.
- [ ] **HTML from users or AI is sanitized** before rendering (`dangerouslySetInnerHTML`, Markdown renderers).

## 8. Dependencies and tooling

- [ ] `npm audit --omit=dev` shows no critical or high issues you have not reviewed.
- [ ] **Supabase Security Advisor** (Dashboard → Advisors) shows no warnings left unexplained.

---

## Need help?

I audit and fix AI-built React + Supabase apps: a clear report of every issue, how severe it is, and tested fixes.

**Fiverr:** https://www.fiverr.com/larissadoura380
**GitHub:** https://github.com/Larissadourado27

## License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Share and adapt freely with credit.
