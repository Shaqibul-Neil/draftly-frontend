# Draftly — Backend Requirements & API Guideline

Backend contract derived from the frontend routes (see [routing-plan.md](./routing-plan.md)).
Medium-style blogging platform with paid membership. **3 roles:** `reader`, `author`, `admin`.

> This document is the source of truth for the data model and REST API. Frontend features
> (`src/features/**`) consume these endpoints; backend implements them.

---

## 1. Tech Assumptions

| Concern | Choice |
|---------|--------|
| API style | REST, JSON over HTTPS |
| Auth | JWT (access + refresh) or session cookies; role claim in token |
| Passwords | hashed (bcrypt/argon2) |
| Payments | Stripe (Checkout + Connect + Webhooks) |
| DB | Relational (PostgreSQL recommended) |
| IDs | UUID (or cuid) |
| Timestamps | `created_at`, `updated_at` on every table (UTC) |

**Base URL:** `/api/v1`

---

## 2. Database Tables

### Core (must-have)

| # | Table | Key columns | Notes / relations |
|---|-------|-------------|-------------------|
| 1 | **users** | id, name, username(unique), email(unique), password_hash, role(`reader`\|`author`\|`admin`), avatar_url, bio, email_verified, is_active, created_at, updated_at | Central identity. `role` drives access. |
| 2 | **posts** | id, author_id→users, title, slug(unique), subtitle, content, cover_image, status(`draft`\|`published`\|`archived`), is_member_only, reading_time, views_count, claps_count, published_at, created_at, updated_at | The article. `is_member_only` = paywall. |
| 3 | **tags** | id, name, slug(unique) | |
| 4 | **post_tags** | post_id→posts, tag_id→tags | Many-to-many join. |
| 5 | **comments** | id, post_id→posts, user_id→users, parent_id→comments(nullable), content, is_deleted, created_at, updated_at | `parent_id` enables threads. |
| 6 | **claps** | id, post_id→posts, user_id→users, count, created_at | Reaction/like (Medium claps). |
| 7 | **bookmarks** | id, user_id→users, post_id→posts, created_at | Reading list. Unique(user, post). |
| 8 | **follows** | id, follower_id→users, following_id→users, created_at | Author follow. Unique pair. |
| 9 | **reading_history** | id, user_id→users, post_id→posts, progress, read_at | Feeds reader analytics. |
| 10 | **notifications** | id, user_id→users, type, actor_id→users, entity_type, entity_id, is_read, created_at | In-app alerts. |

### Membership & Payments

| # | Table | Key columns | Notes |
|---|-------|-------------|-------|
| 11 | **plans** | id, name, price, currency, interval(`month`\|`year`), stripe_price_id, features(json), is_active | Membership tiers. |
| 12 | **subscriptions** | id, user_id→users, plan_id→plans, status(`active`\|`canceled`\|`past_due`\|`trialing`), stripe_customer_id, stripe_subscription_id, current_period_end, created_at | One active per user. Drives paywall unlock. |
| 13 | **transactions** | id, user_id→users, subscription_id→subscriptions, amount, currency, status, stripe_payment_intent, created_at | Payment ledger (admin view). |
| 14 | **earnings** | id, author_id→users, post_id→posts, amount, source(`member_read`), period, created_at | Author accrual per period. |
| 15 | **payouts** | id, author_id→users, amount, status(`pending`\|`paid`\|`failed`), period, stripe_transfer_id, created_at | Author withdrawal. |

### Moderation

| # | Table | Key columns | Notes |
|---|-------|-------------|-------|
| 16 | **reports** | id, reporter_id→users, entity_type(`post`\|`comment`\|`user`), entity_id, reason, status(`open`\|`resolved`\|`dismissed`), created_at | Abuse flags → admin queue. |

### Optional (differentiators — phase 2)

| # | Table | Purpose |
|---|-------|---------|
| 17 | **series** / **series_posts** | Multi-part collections. |
| 18 | **highlights** | Inline text highlights + margin notes on posts. |
| 19 | **post_revisions** | Draft version history / restore. |

---

## 3. API Conventions

- **Auth header:** `Authorization: Bearer <token>`.
- **Pagination:** `?page=1&limit=20` → response `{ data, meta: { page, limit, total, totalPages } }`.
- **Filtering/sort (list):** `?tag=&author=&status=&q=&sort=latest|trending|top`.
- **Errors:** `{ error: { code, message, details? } }`.
- **Status codes:** 200 ok · 201 created · 204 no content · 400 validation · 401 unauth · 403 forbidden · 404 not found · 409 conflict · 422 unprocessable · 429 rate-limit · 500 server.
- **Ownership rule:** author-only mutations verify `post.author_id === currentUser.id` (admin bypass).

---

## 4. API Endpoints

Access legend: 🌐 public · 👤 authenticated (any role) · ✍️ author · 🛡️ admin · 🔒 owner-only

### 4.1 Auth — `/auth`

| Method | Endpoint | Purpose | Access | Frontend |
|--------|----------|---------|:------:|----------|
| POST | `/auth/register` | Create account (default role `reader`) | 🌐 | `/register` |
| POST | `/auth/login` | Issue tokens | 🌐 | `/login` |
| POST | `/auth/logout` | Invalidate session/refresh | 👤 | navbar |
| POST | `/auth/refresh` | Rotate access token | 🌐 | — |
| POST | `/auth/forgot-password` | Send reset link | 🌐 | `/forgot-password` |
| POST | `/auth/reset-password` | Set new password via token | 🌐 | reset page |
| GET | `/auth/verify-email` | Confirm email token | 🌐 | verify page |
| GET | `/auth/me` | Current user + role | 👤 | app-wide |

### 4.2 Users & Social — `/users`

| Method | Endpoint | Purpose | Access | Frontend |
|--------|----------|---------|:------:|----------|
| GET | `/users/:username` | Public author profile | 🌐 | `/profile/[username]` |
| PATCH | `/users/me` | Update own profile/bio/avatar | 👤 | `/*/profile`, settings |
| PATCH | `/users/me/password` | Change password | 👤 | `/*/settings` |
| GET | `/users/:username/followers` | Follower list | 🌐 | `/author/followers` |
| GET | `/users/:username/following` | Following list | 🌐 | `/reader/following` |
| POST | `/users/:username/follow` | Follow author | 👤 | profile |
| DELETE | `/users/:username/follow` | Unfollow | 👤 | profile |

### 4.3 Posts — `/posts`

| Method | Endpoint | Purpose | Access | Frontend |
|--------|----------|---------|:------:|----------|
| GET | `/posts` | Feed/list (filter, sort, paginate) | 🌐 | `/home`, `/blogs`, `/explore` |
| GET | `/posts/:slug` | Single article (paywall-aware) | 🌐 | `/blogs/[id]` |
| GET | `/me/posts` | Own drafts + published | ✍️ | `/author/my-posts` |
| POST | `/posts` | Create draft | ✍️ | `/author/new` |
| PATCH | `/posts/:id` | Update post | 🔒✍️ | `/author/my-posts/[id]/edit` |
| DELETE | `/posts/:id` | Delete post | 🔒✍️/🛡️ | my-posts |
| POST | `/posts/:id/publish` | Publish draft | 🔒✍️ | editor |
| POST | `/posts/:id/unpublish` | Revert to draft | 🔒✍️ | editor |
| GET | `/posts/:id/analytics` | Per-post stats (views, claps, read %) | 🔒✍️ | `/author/analytics` |
| POST | `/posts/:id/read` | Track view + reading progress | 👤 | `/blogs/[id]` |
| POST | `/posts/:id/clap` | Clap/react | 👤 | `/blogs/[id]` |
| DELETE | `/posts/:id/clap` | Remove clap | 👤 | `/blogs/[id]` |

### 4.4 Tags — `/tags`

| Method | Endpoint | Purpose | Access | Frontend |
|--------|----------|---------|:------:|----------|
| GET | `/tags` | All tags / topics | 🌐 | `/explore` |
| GET | `/tags/:slug/posts` | Posts by tag | 🌐 | `/tag/[slug]` |

### 4.5 Comments — `/comments`

| Method | Endpoint | Purpose | Access | Frontend |
|--------|----------|---------|:------:|----------|
| GET | `/posts/:id/comments` | Thread for a post | 🌐 | `/blogs/[id]` |
| POST | `/posts/:id/comments` | Add comment/reply | 👤 | `/blogs/[id]` |
| PATCH | `/comments/:id` | Edit own comment | 🔒 | — |
| DELETE | `/comments/:id` | Delete (owner / post author / admin) | 🔒✍️/🛡️ | `/author/comments` |

### 4.6 Library — bookmarks, history, notifications

| Method | Endpoint | Purpose | Access | Frontend |
|--------|----------|---------|:------:|----------|
| GET | `/me/bookmarks` | Saved posts | 👤 | `/reader/bookmarks` |
| POST | `/posts/:id/bookmark` | Save | 👤 | post card |
| DELETE | `/posts/:id/bookmark` | Unsave | 👤 | post card |
| GET | `/me/history` | Reading history | 👤 | `/reader/history` |
| GET | `/me/notifications` | List notifications | 👤 | `/reader/notifications` |
| PATCH | `/notifications/:id/read` | Mark one read | 👤 | notifications |
| PATCH | `/me/notifications/read-all` | Mark all read | 👤 | notifications |

### 4.7 Search — `/search`

| Method | Endpoint | Purpose | Access | Frontend |
|--------|----------|---------|:------:|----------|
| GET | `/search?q=` | Search posts + authors | 🌐 | `/search` |

### 4.8 Membership & Payments — `/plans`, `/subscriptions`

| Method | Endpoint | Purpose | Access | Frontend |
|--------|----------|---------|:------:|----------|
| GET | `/plans` | List membership plans | 🌐 | `/pricing` |
| GET | `/me/subscription` | Current subscription status | 👤 | `/reader/subscription` |
| POST | `/subscriptions/checkout` | Create Stripe Checkout session | 👤 | `/checkout` |
| POST | `/subscriptions/cancel` | Cancel at period end | 👤 | `/reader/subscription` |
| POST | `/me/billing-portal` | Stripe billing portal session | 👤 | subscription |
| GET | `/me/invoices` | Invoice history | 👤 | `/reader/subscription/invoices` |
| POST | `/webhooks/stripe` | Stripe events (payment, renewal, cancel) | 🌐* | — (server) |

*Webhook is public but signature-verified.

### 4.9 Author Earnings — `/me/earnings`

| Method | Endpoint | Purpose | Access | Frontend |
|--------|----------|---------|:------:|----------|
| GET | `/me/earnings` | Accrued earnings by period | ✍️ | `/author/earnings` |
| GET | `/me/payouts` | Payout history | ✍️ | `/author/earnings` |
| POST | `/me/connect` | Stripe Connect onboarding link | ✍️ | `/author/earnings` |

### 4.10 Analytics (dashboards)

| Method | Endpoint | Purpose | Access | Frontend |
|--------|----------|---------|:------:|----------|
| GET | `/me/analytics/author` | Author aggregate (views, claps, followers, top posts) | ✍️ | `/author/dashboard`, `/author/analytics` |
| GET | `/me/analytics/reading` | Reader aggregate (time read, streak, topics) | 👤 | `/reader/dashboard`, `/reader/analytics` |

### 4.11 Reports — `/reports`

| Method | Endpoint | Purpose | Access | Frontend |
|--------|----------|---------|:------:|----------|
| POST | `/reports` | Flag post/comment/user | 👤 | post/comment menu |

### 4.12 Admin — `/admin`

| Method | Endpoint | Purpose | Access | Frontend |
|--------|----------|---------|:------:|----------|
| GET | `/admin/analytics` | Platform KPIs | 🛡️ | `/admin/dashboard`, `/admin/analytics` |
| GET | `/admin/users` | List/search users | 🛡️ | `/admin/users` |
| PATCH | `/admin/users/:id` | Change role / ban / activate | 🛡️ | `/admin/users` |
| GET | `/admin/posts` | Moderate posts | 🛡️ | `/admin/posts` |
| DELETE | `/admin/posts/:id` | Remove post | 🛡️ | `/admin/posts` |
| GET | `/admin/comments` | Moderate comments | 🛡️ | `/admin/comments` |
| DELETE | `/admin/comments/:id` | Remove comment | 🛡️ | `/admin/comments` |
| GET | `/admin/reports` | Report queue | 🛡️ | `/admin/reports` |
| PATCH | `/admin/reports/:id` | Resolve/dismiss report | 🛡️ | `/admin/reports` |
| GET | `/admin/subscriptions` | All subscriptions | 🛡️ | `/admin/subscriptions` |
| GET | `/admin/transactions` | Payment ledger | 🛡️ | `/admin/transactions` |
| GET | `/admin/payouts` | Payout queue | 🛡️ | `/admin/payouts` |
| PATCH | `/admin/payouts/:id` | Approve/process payout | 🛡️ | `/admin/payouts` |
| GET | `/admin/plans` | Manage plans (list) | 🛡️ | admin |
| POST | `/admin/plans` | Create plan | 🛡️ | admin |
| PATCH | `/admin/plans/:id` | Update plan | 🛡️ | admin |
| DELETE | `/admin/plans/:id` | Deactivate plan | 🛡️ | admin |

---

## 5. Endpoint Count

| Group | Count |
|-------|:-----:|
| Auth | 8 |
| Users & Social | 7 |
| Posts | 12 |
| Tags | 2 |
| Comments | 4 |
| Library (bookmarks/history/notifications) | 7 |
| Search | 1 |
| Membership & Payments | 7 |
| Author Earnings | 3 |
| Analytics | 2 |
| Reports | 1 |
| Admin | 17 |
| **Total** | **71** |

---

## 6. Role-Based API Access Matrix

| Domain | guest | reader | author | admin |
|--------|:-----:|:------:|:------:|:-----:|
| Read posts / tags / profiles / search | ✅ | ✅ | ✅ | ✅ |
| Read **member-only** posts | ❌ | 💳 (if subscribed) | 💳 | ✅ |
| Register / login | ✅ | — | — | — |
| Bookmark / clap / comment / follow | ❌ | ✅ | ✅ | ✅ |
| Notifications / history / reading analytics | ❌ | ✅ | ✅ | ✅ |
| Subscribe / manage membership / invoices | ❌ | ✅ | ✅ | ✅ |
| Create / edit / publish posts | ❌ | ❌ | ✅ (own) | ✅ |
| Post analytics / earnings / payouts | ❌ | ❌ | ✅ (own) | ✅ |
| Report content | ❌ | ✅ | ✅ | ✅ |
| Moderate users/posts/comments/reports | ❌ | ❌ | ❌ | ✅ |
| View subscriptions / transactions / payouts (all) | ❌ | ❌ | ❌ | ✅ |
| Manage plans | ❌ | ❌ | ❌ | ✅ |

💳 = requires active subscription. Author inherits every reader capability; admin inherits everything.

---

## 7. Role → Which APIs You Need

- **Reader** needs: §4.1 auth, §4.2 (read/follow/update-self), §4.3 (read/read-track/clap), §4.5 (read/post comment), §4.6 library, §4.7 search, §4.8 membership, §4.10 reading analytics, §4.11 report.
- **Author** = all reader APIs **+** §4.3 (create/edit/publish/delete own, post analytics), §4.5 (moderate own post comments), §4.9 earnings, §4.10 author analytics.
- **Admin** = everything **+** §4.12 admin.

---

## 8. Webhooks (Stripe)

`POST /webhooks/stripe` handles: `checkout.session.completed` (activate subscription),
`invoice.paid` (record transaction, extend period), `customer.subscription.deleted`
(mark canceled), `invoice.payment_failed` (mark past_due). Always verify signature.

---

## 9. Build Order (suggested)

1. **users + auth** (register/login/me) → unblock everything.
2. **posts + tags + comments** → core content loop.
3. **social** (follow, clap, bookmark, notifications) → engagement.
4. **membership** (plans, subscriptions, Stripe, paywall) → monetization.
5. **author earnings + analytics**.
6. **admin** (moderation, reports, platform analytics).
7. **differentiators** (series, highlights, revisions).

---

_Last updated: 2026-07-16_
