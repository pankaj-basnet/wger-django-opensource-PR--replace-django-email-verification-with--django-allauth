# wger PR #2260 — Merged ✅
## Replacing `django-email-verification` with `django-allauth`: A Complete Technical Report

**PR:** [wger-project/wger#2260](https://github.com/wger-project/wger/pull/2260)  
**Issue:** [#2258 — Use django-allauth to handle email verification](https://github.com/wger-project/wger/issues/2258)  
**Branch:** `pankaj-basnet:feature/2258-migrate-to-django-allauth` → `wger-project:master`  
**Merged:** March 29 by maintainer `rolandgeider`  
**Commits:** 17 total (6 by intern `pankaj-basnet`, 10 by maintainer `rolandgeider`, 1 merge)  
**Lines Changed:** +808 additions / −176 deletions across 21 files  
**Status:** ✅ Merged, 7/7 checks passed

---


The report is structured as a learning document across **12 sections** with ASCII diagrams, comparison tables, etc. Here are the highlights:

### 🏗️ Architecture Diagrams
Two full ASCII diagrams show the **before/after architecture** — the old `django-email-verification` callback/flag pattern vs. the new allauth `EmailAddress` table model. Seeing the data flow side by side makes the "why" of the migration clear.

### 📊 Key Comparison Tables
- **12-row table** comparing every aspect of old vs new (package status, send calls, storage, URL names, token lifetime, extensibility)
- **10-row table** of the maintainer's commit-by-commit fixes explaining what each corrected
- **7-row test matrix** documenting every new test and what it validates

### 🗄️ Data Migration Deep Dive (Section 5)
The most technically important part — the step-by-step execution flow of `0022_move_email_verified_to_emailaddress.py`:
- Why `ignore_conflicts=True` matters for idempotency
- Why `('account', '0001_initial')` must be a dependency
- Why `apps.get_model()` is used instead of direct imports (historical model snapshots)

### 👨‍💻 Maintainer Review Analysis (Section 9)
Honest breakdown: the intern got the PR to **~60% done** (correct direction, functional core), and the maintainer added **10 polish commits** to complete it — correct template paths, missing fixtures, the custom adapter, all 7 tests, and URL fixes. The gap between 60% and 95% is documented as a learning outcome.

### 🎓 Migration Checklist (Section 12)
A 14-item reusable checklist any intern can follow for any future package-replacement PR in Django, derived directly from the gaps this PR exposed.








------------------------------------------------------------
------------------------------------------------------------
















# wger PR #2260 — Merged ✅
## Replacing `django-email-verification` with `django-allauth`: A Complete Technical Report


---

> 📚 **Who is this report for?**  
> This is a learning document for interns and junior contributors working on the wger Django project.
> It explains *what* changed, *why* it changed, *how* it works, and *what the maintainer corrected*
> — so the next person doing a similar migration doesn't repeat the same mistakes.

---

## 📋 Table of Contents

1. Background — Why This Migration Was Needed
2. What is `django-allauth`? Core Concepts
3. Architecture Comparison: Before vs After
4. All 21 Files Changed — What and Why
5. The Data Migration — Moving `email_verified` to allauth
6. The `WgerAccountAdapter` — Custom allauth Integration
7. Email Templates — The New Verification Email
8. Tests — What Was Added and Why
9. Maintainer Review Feedback — What Was Corrected
10. The Commit-by-Commit Story
11. Key Concepts for Future Reference
12. Lessons Learned for Interns

---

## 1. 🏋️ Background — Why This Migration Was Needed

The wger Workout Manager is a Django-based fitness application with user accounts. Like most web apps with user accounts, it needs to verify that a user actually owns the email address they registered with. Before this PR, wger used a package called **`django-email-verification`** to do this.

### The Problem with `django-email-verification`

| # | Issue | Detail |
|---|-------|--------|
| 1 | Unmaintained | The package had no active maintainer; last meaningful commit was years old |
| 2 | Security risk | Unmaintained packages accumulate unpatched CVEs over time |
| 3 | No future-proofing | Could not be extended for 2FA or social auth (OAuth) |
| 4 | Custom storage | Used a custom `email_verified` boolean field on `UserProfile` — not standard |
| 5 | Template debt | Required wger-specific email templates in `email_verification/` folder |
| 6 | Callback pattern | Used a callback function (`EMAIL_MAIL_CALLBACK`) — fragile and non-standard |

Issue [#2258](https://github.com/wger-project/wger/issues/2258) was opened to replace it with `django-allauth`, which is the industry-standard Django authentication and account management package used by thousands of Django projects worldwide.

### Why django-allauth?

`django-allauth` is not just an email verification library — it is a complete account management framework. By adopting it now, wger gets:

- ✅ Email verification (this PR's goal)
- ✅ A foundation for social auth / OAuth (covered in PR #2271)
- ✅ 2FA / TOTP support in the future
- ✅ Actively maintained (thousands of projects depend on it)
- ✅ Industry-standard patterns (no wger-specific hacks)
- ✅ An `EmailAddress` model that properly tracks verification state per address
- ✅ Built-in email confirmation flows, token management, expiry handling

---

## 2. 🔧 What is `django-allauth`? Core Concepts

Before reading the code changes, you need to understand the key moving parts of allauth.

```
┌─────────────────────────────────────────────────────────────────┐
│                     django-allauth Architecture                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   APPS INSTALLED                                                 │
│   ┌──────────────────┐   ┌────────────────────────────────┐     │
│   │  allauth         │   │  allauth.account               │     │
│   │  (core)          │   │  (email/password accounts)     │     │
│   └──────────────────┘   └────────────────────────────────┘     │
│                                                                  │
│   DATABASE MODELS                                                │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │  account_emailaddress                                    │  │
│   │  ┌───────┬─────────────────────┬──────────┬───────────┐ │  │
│   │  │  id   │  email              │ verified │  primary  │ │  │
│   │  ├───────┼─────────────────────┼──────────┼───────────┤ │  │
│   │  │   1   │ admin@example.com   │  False   │   True    │ │  │
│   │  │   2   │ trainer@example.com │  True    │   True    │ │  │
│   │  └───────┴─────────────────────┴──────────┴───────────┘ │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│   KEY METHODS                                                    │
│   EmailAddress.objects.get_for_user(user, email)  → email_obj   │
│   EmailAddress.objects.add_email(request, user, email,          │
│                                  confirm=True)    → sends mail  │
│   email_obj.send_confirmation(request)            → sends mail  │
│   email_obj.verified                              → bool        │
│                                                                  │
│   ADAPTER PATTERN                                                │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  DefaultAccountAdapter                                  │   │
│   │       ↑ inherits                                        │   │
│   │  WgerAccountAdapter  (wger/core/account_adapter.py)     │   │
│   │  • Overrides send_mail() to add expiry context          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   URL NAMESPACE                                                  │
│   path('account/', include('allauth.account.urls'))             │
│   → account_confirm_email   (GET/POST to confirm)               │
│   → account_email           (manage emails)                     │
│   → account_login, account_logout, account_signup               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key allauth Settings

| # | Setting | Value in wger | Purpose |
|---|---------|---------------|---------|
| 1 | `ACCOUNT_CONFIRM_EMAIL_ON_GET` | `True` | Email confirmed on GET request (no separate POST step needed) |
| 2 | `ACCOUNT_EMAIL_CONFIRMATION_EXPIRE_DAYS` | `2` | Confirmation links expire after 2 days (48 hours) |
| 3 | `ACCOUNT_ADAPTER` | `'wger.core.account_adapter.WgerAccountAdapter'` | Use wger's custom adapter |
| 4 | `AUTHENTICATION_BACKENDS` | includes `'allauth.account.auth_backends.AuthenticationBackend'` | allauth hooks into Django auth |
| 5 | `SITE_ID` | `1` | Required for `django.contrib.sites` integration |

---

## 3. 🏗️ Architecture Comparison: Before vs After

### Before (django-email-verification)

```
┌──────────────────────────────────────────────────────────────────┐
│  OLD ARCHITECTURE — django-email-verification                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Registration flow:                                              │
│  User registers → send_email(user) called → email sent          │
│                 → link has token in URL                          │
│                 → user clicks link                               │
│                 → email_verified_callback(user) called           │
│                 → user.userprofile.email_verified = True saved   │
│                                                                  │
│  Storage:                                                        │
│  core_userprofile table                                          │
│  ┌──────────┬────────────────┐                                   │
│  │ user_id  │ email_verified │  ← boolean flag in UserProfile   │
│  ├──────────┼────────────────┤                                   │
│  │    1     │     False      │                                   │
│  │    2     │     True       │                                   │
│  └──────────┴────────────────┘                                   │
│                                                                  │
│  Settings (settings_global.py):                                  │
│  EMAIL_MAIL_CALLBACK = email_verified_callback                   │
│  EMAIL_MAIL_SUBJECT = 'Confirm your email'                       │
│  EMAIL_MAIL_HTML = 'email_verification/email_body_html.tpl'      │
│  EMAIL_MAIL_PLAIN = 'email_verification/email_body_txt.tpl'      │
│  EMAIL_MAIL_TOKEN_LIFE = 60 * 60   (1 hour)                      │
│  EMAIL_MAIL_PAGE_TEMPLATE = 'email_verification/confirm.html'   │
│  EMAIL_PAGE_DOMAIN = 'http://localhost:8000/'                    │
│                                                                  │
│  URLs:                                                           │
│  path('email/', include(email_urls))  ← package URLs            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### After (django-allauth)

```
┌──────────────────────────────────────────────────────────────────┐
│  NEW ARCHITECTURE — django-allauth                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Registration flow:                                              │
│  User registers → EmailAddress.objects.add_email(               │
│                       request, user, email, confirm=True)        │
│                 → allauth creates EmailAddress row               │
│                 → allauth sends confirmation email               │
│                 → user clicks link → GET account/confirm-email/  │
│                 → allauth sets EmailAddress.verified = True      │
│                                                                  │
│  Storage:                                                        │
│  account_emailaddress table (allauth-managed)                    │
│  ┌───────┬─────────────────────┬──────────┬─────────┐           │
│  │  id   │  email              │ verified │ primary │           │
│  ├───────┼─────────────────────┼──────────┼─────────┤           │
│  │   1   │ admin@example.com   │  False   │  True   │           │
│  │   2   │ trainer@example.com │  True    │  True   │           │
│  └───────┴─────────────────────┴──────────┴─────────┘           │
│                                                                  │
│  Settings:                                                       │
│  ACCOUNT_CONFIRM_EMAIL_ON_GET = True                             │
│  ACCOUNT_EMAIL_CONFIRMATION_EXPIRE_DAYS = 2                      │
│  ACCOUNT_ADAPTER = 'wger.core.account_adapter.WgerAccountAdapter'│
│                                                                  │
│  URLs:                                                           │
│  path('account/', include('allauth.account.urls'))               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Side-by-Side Comparison Table

| # | Aspect | django-email-verification (OLD) | django-allauth (NEW) |
|---|--------|--------------------------------|----------------------|
| 1 | Package status | Unmaintained | Actively maintained |
| 2 | Email send call | `send_email(user)` | `EmailAddress.objects.add_email(request, user, email, confirm=True)` |
| 3 | Resend verification | `send_email(user)` | `email_obj.send_confirmation(request)` |
| 4 | Storage location | `UserProfile.email_verified` (boolean) | `EmailAddress.verified` (boolean in separate table) |
| 5 | Read verification status | `user.userprofile.email_verified` | `EmailAddress.objects.get_for_user(user, email).verified` |
| 6 | Token lifetime | 1 hour (`EMAIL_MAIL_TOKEN_LIFE`) | 48 hours (`ACCOUNT_EMAIL_CONFIRMATION_EXPIRE_DAYS = 2`) |
| 7 | URL namespace | `email/` | `account/` |
| 8 | URL name for confirm | custom | `account_confirm_email` |
| 9 | Callback on verify | `EMAIL_MAIL_CALLBACK` function | allauth handles it automatically |
| 10 | Future extensibility | None (only email verify) | 2FA, social auth, password policies |
| 11 | Email templates | wger custom in `email_verification/` | allauth convention in `account/email/` |
| 12 | Multi-email support | No | Yes (allauth supports multiple emails per user) |

---

## 4. 📁 All 21 Files Changed — What and Why

Below is a numbered description of every file changed in this PR.

### Group A: Configuration Files

| # | File | Change Type | Summary |
|---|------|------------|---------|
| 1 | `.gitignore` | Addition | Added blank line (cosmetic, no functional impact) |
| 2 | `pyproject.toml` | Replacement | Removed `django-email-verification~=0.3.3`; Added `django-allauth~=65.15.0` |
| 3 | `uv.lock` | Auto-generated | Lock file updated to reflect new dependency tree |

**`pyproject.toml` detail:**

```toml
# REMOVED
"django-email-verification~=0.3.3",

# ADDED  
"django-allauth~=65.15.0",
```

This is the dependency swap. `uv.lock` was auto-generated and covers 52 lines of lock file changes as `django-email-verification` and its sub-dependencies were removed and allauth's dependency graph was resolved.

---

### Group B: Settings

| # | File | Change Type | Summary |
|---|------|------------|---------|
| 4 | `settings/settings_global.py` | Major refactor | Remove old email verification config, add allauth apps and settings |

**What was removed from `settings_global.py`:**

```python
# REMOVED — django-email-verification config block (~23 lines)
def email_verified_callback(user):
    user.userprofile.email_verified = True
    user.userprofile.save()

EMAIL_MAIL_CALLBACK = email_verified_callback
EMAIL_FROM_ADDRESS = WGER_SETTINGS['EMAIL_FROM']
EMAIL_MAIL_SUBJECT = 'Confirm your email'
EMAIL_MAIL_HTML = 'email_verification/email_body_html.tpl'
EMAIL_MAIL_PLAIN = 'email_verification/email_body_txt.tpl'
EMAIL_MAIL_TOKEN_LIFE = 60 * 60
EMAIL_MAIL_PAGE_TEMPLATE = 'email_verification/confirm_template.html'
EMAIL_PAGE_DOMAIN = 'http://localhost:8000/'

# Also removed from INSTALLED_APPS:
'django_email_verification',
```

**What was added to `settings_global.py`:**

```python
# ADDED to INSTALLED_APPS
'allauth',
'allauth.account',

# ADDED to MIDDLEWARE
"allauth.account.middleware.AccountMiddleware",

# ADDED — allauth settings block
ACCOUNT_CONFIRM_EMAIL_ON_GET = True
ACCOUNT_EMAIL_CONFIRMATION_EXPIRE_DAYS = 2
ACCOUNT_ADAPTER = 'wger.core.account_adapter.WgerAccountAdapter'

# Enabled the email backend (was commented out before)
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

The `AccountMiddleware` is required by allauth since version 64.x — it sets up the request context allauth needs for all its views. Without it, allauth raises an error at startup. This is a common gotcha when upgrading allauth or setting it up fresh.

---

### Group C: Core Application Files

| # | File | Change Type | Summary |
|---|------|------------|---------|
| 5 | `wger/core/account_adapter.py` | New file | Custom allauth adapter to add email expiry context |
| 6 | `wger/core/api/serializers.py` | Modified | Replace `email_verified` field with allauth-sourced value |
| 7 | `wger/core/api/views.py` | Modified | Replace `send_email()` with `EmailAddress.objects.add_email()` |
| 8 | `wger/core/models/profile.py` | Modified | Remove `email_verified` field, use allauth's `EmailAddress.verified` |
| 9 | `wger/core/views/user.py` | Modified | Replace all `send_email()` calls, fix preferences and confirm_email views |

---

### Group D: Database Migration

| # | File | Change Type | Summary |
|---|------|------------|---------|
| 10 | `wger/core/migrations/0022_move_email_verified_to_emailaddress.py` | New migration | Copy `email_verified` data to allauth's `EmailAddress` table, then drop the column |

This is the most critical file in the whole PR. More detail in Section 5.

---

### Group E: Fixtures

| # | File | Change Type | Summary |
|---|------|------------|---------|
| 11 | `wger/core/fixtures/test-user-data.json` | Major modification | +244 lines — add `account.emailaddress` fixture entries for all test users |

This is the second most important change. Tests that check email verification status need `EmailAddress` rows to exist in the test database. Without this, calling `EmailAddress.objects.get_for_user(user, email)` would raise `DoesNotExist` for every test user.

---

### Group F: Templates

| # | File | Change Type | Summary |
|---|------|------------|---------|
| 12 | `wger/core/templates/account/email/email_confirmation_message.html` | New file | HTML email template for verification email |
| 13 | `wger/core/templates/account/email/email_confirmation_message.txt` | New file | Plain text email template for verification email |
| 14 | `wger/core/templates/email_verification/confirm_template.html` | Deleted | Old package's confirmation page template |
| 15 | `wger/core/templates/email_verification/email_body_html.tpl` | Deleted | Old package's HTML email template |
| 16 | `wger/core/templates/email_verification/email_body_txt.tpl` | Deleted | Old package's text email template |
| 17 | `wger/core/templates/user/preferences.html` | Modified | Use `email_verified` context variable instead of `user.userprofile.email_verified` |

---

### Group G: Tests

| # | File | Change Type | Summary |
|---|------|------------|---------|
| 18 | `wger/core/tests/test_registration.py` | Modified | Add verification email tests; update to allauth patterns |
| 19 | `wger/core/tests/test_user.py` | Modified | Replace `email_verified` field usage with `EmailAddress` model calls |
| 20 | `wger/core/tests/test_verification.py` | New file | Comprehensive verification flow tests |

---

### Group H: URL Configuration

| # | File | Change Type | Summary |
|---|------|------------|---------|
| 21 | `wger/urls.py` | Modified | Replace `path('email/', include(email_urls))` with `path('account/', include('allauth.account.urls'))` |

---

## 5. 🗄️ The Data Migration — Moving `email_verified` to allauth

This is the most technically significant part of the PR. The challenge: wger already has real users in production with an `email_verified` boolean in the `UserProfile` table. When migrating to allauth, that data needs to move to allauth's `EmailAddress` table before the old column can be dropped.

### The Migration File: `0022_move_email_verified_to_emailaddress.py`

```python
from django.db import migrations, IntegrityError
from django.db.migrations.state import StateApps


def migrate_verification_status(apps: StateApps, schema_editor):
    UserProfile = apps.get_model('core', 'UserProfile')
    EmailAddress = apps.get_model('account', 'EmailAddress')

    profiles = UserProfile.objects.select_related('user').exclude(user__email='')
    email_addresses = [
        EmailAddress(
            user=profile.user,
            email=profile.user.email,
            verified=profile.email_verified,   # ← copies old flag value
            primary=True,
        )
        for profile in profiles
    ]

    EmailAddress.objects.bulk_create(email_addresses, ignore_conflicts=True)


class Migration(migrations.Migration):
    dependencies = [
        ('core', '0021_add_unit_type_to_repetitionunit'),
        ('account', '0001_initial'),          # ← allauth must be migrated first
    ]

    operations = [
        migrations.RunPython(migrate_verification_status),
        migrations.RemoveField(
            model_name='userprofile',
            name='email_verified',
        ),
    ]
```

### What this migration does — step by step

```
Migration 0022 Execution Flow
═════════════════════════════

Step 1: RunPython(migrate_verification_status)
  │
  ├── Query all UserProfile rows that have a non-empty email
  │   SELECT * FROM core_userprofile 
  │   INNER JOIN auth_user ON user_id = auth_user.id
  │   WHERE auth_user.email != ''
  │
  ├── For each profile, create an EmailAddress object:
  │   - user        = the Django User
  │   - email       = user.email
  │   - verified    = profile.email_verified   ← THE COPY
  │   - primary     = True
  │
  ├── bulk_create(email_addresses, ignore_conflicts=True)
  │   INSERT INTO account_emailaddress (user_id, email, verified, primary)
  │   VALUES (...), (...), (...)
  │   ON CONFLICT DO NOTHING   ← safe to run multiple times
  │
  └── Done — data is now in account_emailaddress

Step 2: RemoveField(model_name='userprofile', name='email_verified')
  │
  └── ALTER TABLE core_userprofile DROP COLUMN email_verified
```

### Why `ignore_conflicts=True` matters

Running the migration a second time (e.g., during development when resetting the database) would cause `IntegrityError` without this flag because `account_emailaddress` has a unique constraint on `(user_id, email)`. With `ignore_conflicts=True`, the second run is a no-op for existing rows.

### Why `depend on ('account', '0001_initial')`

The migration must list `allauth.account`'s own initial migration as a dependency. This guarantees that the `account_emailaddress` table exists before we try to insert rows into it. If this dependency were missing, running `./manage.py migrate` on a fresh database would fail with "table account_emailaddress does not exist."

### The `apps` parameter — why use it instead of direct model imports

```python
def migrate_verification_status(apps: StateApps, schema_editor):
    UserProfile = apps.get_model('core', 'UserProfile')
    EmailAddress = apps.get_model('account', 'EmailAddress')
```

This is the correct Django migration pattern. You use `apps.get_model()` instead of `from wger.core.models import UserProfile` because:

1. The migration captures a historical snapshot of the model. If the model is later changed (e.g., `email_verified` is added back, or renamed), the migration must still work correctly with the model as it was **at the time of migration 0022**.
2. Direct imports reflect the current model, which may differ from the model at migration time.

---

## 6. 🔌 The `WgerAccountAdapter` — Custom allauth Integration

### File: `wger/core/account_adapter.py`

```python
class WgerAccountAdapter(DefaultAccountAdapter):
    """Wger Account Adapter for allauth"""

    def send_mail(self, template_prefix, email, context):
        expire_days = settings.ACCOUNT_EMAIL_CONFIRMATION_EXPIRE_DAYS
        context['expire_hours'] = expire_days * 24
        context['expire_date'] = timezone.now() + timedelta(days=expire_days)
        super().send_mail(template_prefix, email, context)
```

### What the Adapter Pattern Does

allauth uses an "adapter" design pattern to allow projects to customize its behavior without modifying allauth's own code. When allauth needs to send an email, it calls `adapter.send_mail(template_prefix, email, context)`. The default adapter sends the email using Django's email framework. wger's `WgerAccountAdapter` overrides this to inject two extra context variables before calling the parent method.

### Why the Extra Context Variables?

The email template needs to tell the user when their verification link expires. allauth's default `send_mail` does not pass expiry information into the template context. The adapter calculates:

- `expire_hours` = `ACCOUNT_EMAIL_CONFIRMATION_EXPIRE_DAYS × 24` = 48 hours
- `expire_date` = `timezone.now() + timedelta(days=2)` = the exact expiry timestamp

These are then used in the HTML email template:

```html
{% blocktranslate with time=expire_date|time:"TIME_FORMAT" date=expire_date|date:"DATE_FORMAT" %}
    The link is valid for {{ expire_hours }} hours (until {{ date }}, {{ time }}).
{% endblocktranslate %}
```

### How to Register the Adapter

In `settings_global.py`:

```python
ACCOUNT_ADAPTER = 'wger.core.account_adapter.WgerAccountAdapter'
```

allauth reads this setting at startup and uses the specified class for all email-sending operations.

---

## 7. 📧 Email Templates — The New Verification Email

Two new templates were created to replace the three deleted `email_verification/` templates.

### Template Location Convention

allauth looks for email templates at:
```
{app_label}/email/{template_prefix}_subject.txt
{app_label}/email/{template_prefix}_message.txt
{app_label}/email/{template_prefix}_message.html
```

For email confirmation, `app_label` is `account` and `template_prefix` is `email_confirmation`. So allauth looks for:
```
account/email/email_confirmation_message.html
account/email/email_confirmation_message.txt
```

These are exactly the two files added in this PR, placed in:
```
wger/core/templates/account/email/
```

### HTML Template Features

The HTML template (`email_confirmation_message.html`) includes:

| # | Feature | Implementation |
|---|---------|---------------|
| 1 | Responsive design | Max-width 600px container with centered layout |
| 2 | wger branding | Header background `#2a4e6c` (wger's blue) |
| 3 | CTA button | Blue "Confirm email" button with correct `{{ activate_url }}` |
| 4 | Expiry display | `{{ expire_hours }}` hours (from `WgerAccountAdapter`) |
| 5 | Expiry date/time | `{{ expire_date|date }}` / `{{ expire_date|time }}` formatted |
| 6 | Fallback URL | Plain link shown below button for email clients blocking images |
| 7 | Verification code path | `{% if code %}` branch for headless/OTP verification mode |
| 8 | i18n | All user-facing strings wrapped in `{% blocktranslate %}` |
| 9 | Auto-escape off | `{% autoescape off %}` for URLs in email (prevents HTML-escaping of `&` etc.) |

### Why Two Templates (HTML and Text)?

Not all email clients render HTML. Plain text is always the fallback. Django's email framework, when given both, sends a `multipart/alternative` MIME message — the email client automatically chooses the richest format it supports. Always provide both.

---

## 8. 🧪 Tests — What Was Added and Why

### New File: `wger/core/tests/test_verification.py`

This file was written entirely by the maintainer (`rolandgeider`) as part of his 10-commit polish pass. It contains two test classes:

#### `EmailVerificationFromPreferencesTestCase`

Tests the web UI flow where a user clicks "Confirm email" in the preferences sidebar.

| # | Test Method | What It Tests |
|---|-------------|--------------|
| 1 | `test_confirm_email_sends_mail_for_unverified_user` | Unverified user GET → email sent, redirects to dashboard |
| 2 | `test_confirm_email_link_verifies_email` | Email sent → extract key from body → GET confirm URL → `verified=True` |
| 3 | `test_confirm_email_no_mail_for_verified_user` | Already verified user GET → no email sent |
| 4 | `test_confirm_email_requires_login` | Unauthenticated GET → redirected to `/login` |

#### `EmailVerificationFromAPITestCase`

Tests the REST API flow where the Flutter mobile app requests email verification.

| # | Test Method | What It Tests |
|---|-------------|--------------|
| 1 | `test_verify_email_sends_mail_for_unverified_user` | GET `/api/v2/userprofile/verify-email/` → `{"status": "sent"}` + email sent |
| 2 | `test_verify_email_link_verifies_email` | API triggers email → follow link → `verified=True` in DB |
| 3 | `test_verify_email_no_mail_for_verified_user` | Already verified → `{"status": "verified"}` + no email |

### How `test_confirm_email_link_verifies_email` Works

This test demonstrates a real end-to-end flow — it's worth understanding in detail:

```python
def test_confirm_email_link_verifies_email(self):
    self.user_login('admin')
    
    # 1. Trigger email send
    mail.outbox = []
    self.client.get(reverse('core:user:confirm-email'))
    self.assertEqual(len(mail.outbox), 1)  # one email was sent
    
    # 2. Extract confirmation key from the email body
    email_body = mail.outbox[0].body
    match = re.search(r'/account/confirm-email/([^/\s]+)/', email_body)
    key = match.group(1)  # allauth's opaque token
    
    # 3. Visit the confirmation URL (simulating clicking the link)
    confirm_url = reverse('account_confirm_email', args=[key])
    self.client.get(confirm_url)  # ACCOUNT_CONFIRM_EMAIL_ON_GET=True means GET works
    
    # 4. Assert the email is now verified in the DB
    user = User.objects.get(username='admin')
    email_obj = EmailAddress.objects.get_for_user(user, user.email)
    self.assertTrue(email_obj.verified)
```

This is a real integration test — it goes all the way from "user requests verification" to "allauth marks email as verified", through the actual URL routing and view logic. This kind of test is what gives you confidence that the system works end-to-end.

### The `@override_settings` Decorator

```python
@override_settings(EMAIL_BACKEND='django.core.mail.backends.locmem.EmailBackend')
class EmailVerificationFromPreferencesTestCase(WgerTestCase):
```

`locmem.EmailBackend` is Django's in-memory email backend — it captures all sent emails in `mail.outbox` (a list) instead of actually sending them. This is essential for testing email flows without requiring a real SMTP server. The `@override_settings` decorator temporarily replaces `EMAIL_BACKEND` for all tests in the class, then restores it after.

### Test Fixture Update: `test-user-data.json`

The fixture file grew by 244 lines because `EmailAddress` rows were added for every test user. Example:

```json
{
    "model": "account.emailaddress",
    "pk": 1,
    "fields": {
        "user": 1,
        "email": "admin@example.com",
        "verified": false,
        "primary": true
    }
},
{
    "model": "account.emailaddress",
    "pk": 4,
    "fields": {
        "user": 4,
        "email": "trainer1@example.com",
        "verified": true,
        "primary": true
    }
}
```

Note that `admin` has `verified: false` and `trainer1` has `verified: true`. This is intentional — the tests use these fixtures to test both the unverified and verified code paths. The maintainer added these fixtures carefully to support the test assertions:

- `test_confirm_email_sends_mail_for_unverified_user` logs in as `admin` (verified=false) ✓
- `test_confirm_email_no_mail_for_verified_user` logs in as `trainer1` (verified=true) ✓

---

## 9. 👨‍💻 Maintainer Review Feedback — What Was Corrected

The intern submitted 6 commits. The maintainer (`rolandgeider`) reviewed on March 24, left feedback, and then added **10 more commits** himself to complete and polish the PR before merging. This is a common pattern for wger — the maintainer is deeply involved and will finish work that's mostly right but needs polish.

### Maintainer Review Comments (March 24)

The maintainer flagged 4 files for issues:

#### 1. `wger/core/views/user.py` — "Outdated"

The intern's `preferences` view was still using `user.userprofile.email_verified` in some places after the migration — the template and context were not fully updated to use allauth's `EmailAddress`. The maintainer fixed this in his commit "Pass email status from allauth to template" (commit `dd0fe2b`).

**Before (intern's version):**
```python
# preferences view
email_verified = request.user.userprofile.email_verified
context['email_verified'] = email_verified
```

**After (maintainer's fix):**
```python
# preferences view
email_obj = EmailAddress.objects.get_for_user(request.user, request.user.email)
context['email_verified'] = email_obj.verified
```

#### 2. `wger/core/models/profile.py` — "Outdated"

The `is_trustworthy` property still referenced `self.email_verified`:
```python
# OLD (intern)
return days_since_joined.days > minimum_account_age and self.email_verified
```

After the migration dropped `email_verified`, this would cause an `AttributeError`. The maintainer fixed it in "Remove email_verified from profile table, fix registration" (commit `0856e49`):

```python
# NEW (maintainer)
email_obj = EmailAddress.objects.get_for_user(user=self.user, email=self.user.email)
return days_since_joined.days > minimum_account_age and email_obj.verified
```

#### 3. `wger/core/templates/account/email_confirm.html` — "Outdated"

The intern created a template at the wrong path. allauth expects templates at `account/email/email_confirmation_message.html` (in the `email/` subdirectory), not at `account/email_confirm.html`. The maintainer created the correctly named templates in his commit "Polish the verification emails" (commit `f9b3346`) and replaced what the intern had.

#### 4. `wger/urls.py` — "Outdated"

The intern used `path('email/', include(email_urls))` — still referencing the old package's URL include. The maintainer changed it to `path('account/', include('allauth.account.urls'))` in his commit "Better use the 'account' prefix for allauth urls" (commit `346e486`).

### Summary of What the Maintainer Added

| # | Maintainer Commit | What It Fixed/Added |
|---|-------------------|---------------------|
| 1 | `0856e49` Remove email_verified from profile | Dropped `email_verified` column, fixed `is_trustworthy` |
| 2 | `bac6c3a` Add missing account.emailaddress fixtures | Added `EmailAddress` rows for all test users in fixture |
| 3 | `dd0fe2b` Pass email status from allauth to template | Fixed `preferences` view to use allauth `EmailAddress` |
| 4 | `aac0b55` Improve migration | Polished the data migration — added `ignore_conflicts`, proper dependencies |
| 5 | `4873f93` Add account adapter | Created `WgerAccountAdapter` with expiry context |
| 6 | `f9b3346` Polish the verification emails | Created proper HTML + text templates in correct path |
| 7 | `4773f47` Test that the verification mails are sent | Added `test_verification.py` with 7 tests |
| 8 | `ccef344` Remove the django-email-verification dependency | Cleaned `pyproject.toml` |
| 9 | `346e486` Better use the "account" prefix for allauth urls | Fixed URL include path |
| 10 | `821a2f4` Use correct setting name | Fixed a setting name typo |

The maintainer essentially did the "second half" of the migration — the intern did the removal and initial substitution, but the maintainer did the data migration polish, fixtures, tests, adapter, and templates.

---

## 10. 📝 The Commit-by-Commit Story

### Intern's 6 Commits

```
COMMIT 1: d18a209 — "remove django-email-verification package code"
  • Removed package from pyproject.toml
  • Removed 'django_email_verification' from INSTALLED_APPS
  • Removed EMAIL_MAIL_* settings

COMMIT 2: 593bcf5 — "remove django-email-verification package code"  
  • Continued removal — old template files deleted
  • Initial allauth apps added to INSTALLED_APPS

COMMIT 3: b74c47f — "get email_verified status from allauth's EmailAddress model"
  • Modified UserProfile.is_trustworthy to use EmailAddress
  • Modified serializers.py
  • Modified api/views.py

COMMIT 4: ffbb645 — "temporary code removed, added when db not initialized properly"
  • Removed debugging code from views (similar to the user.py credential issue
    in PR #2271 — the same pattern of leaving debug code in)

COMMIT 5: bc8740e — "integrate allauth add_email and verification into user preferences"
  • Modified user.py views to use EmailAddress.objects.add_email()
  • Modified confirm_email() view

COMMIT 6: 4f36487 — "update UserProfile model based on allauth's EmailAddress"
  • Initial attempt at migration 0022
  • Removed email_verified field from model
```

Here is the additional section to add to the PR #2260 report. Paste it in after **Section 9 (Maintainer Review Feedback)** and before the current Section 10:

---

## 9b. 🔄 Intern's Response Commits — Addressing Maintainer Feedback

After the maintainer's March 24 review, the intern pushed two more commits that directly addressed the feedback. These were missed in the original report and are documented here.

---

### Commit `34cf522` — `feat: migrate legacy email verification data to django-allauth`
*(pushed to the main PR branch `pankaj-basnet:feature/2258-migrate-to-django-allauth`)*

This single commit bundled four changes responding to the maintainer's review:

| # | Change | File | What It Did |
|---|--------|------|-------------|
| 1 | Added data migration 0022 | `wger/core/migrations/0022_move_email_verified_to_emailaddress.py` | `RunPython` to copy `UserProfile.email_verified` → `account_emailaddress.verified`, then `RemoveField` to drop the column |
| 2 | Updated `is_trustworthy` tests | `wger/core/tests/test_user.py` | Replaced `user.userprofile.email_verified = True/False` with `EmailAddress.objects.create(user=user, email=user.email, verified=True/False)` |
| 3 | Cleaned fixture | `wger/core/fixtures/test-user-data.json` | Removed the now-dropped `email_verified` field from all `core.userprofile` fixture objects to stop the test runner crashing on load |
| 4 | Removed legacy package | `pyproject.toml` / settings | Final removal of `django-email-verification` and its settings |

---

### Commit `3bb1a9e` — same message, pushed to intern's personal fork
*(commit on `pankaj-basnet/wger-fitness-gym-app--opensource-backend`, referenced from the PR)*

This commit has the same description as `34cf522` with one additional line: **"Remove legacy django-email-verification package and update user preferences UI."** It was the version pushed to the intern's own fork repo before the clean squash onto the PR branch. The content is functionally identical — it exists as a reference point in the intern's personal repository mirroring the work in the PR.

---

### Why These Commits Matter

The original report (Section 9) described the maintainer's 10 commits as "completing" the migration. That framing was slightly unfair — the intern had already responded to the review with meaningful fixes before the maintainer stepped in to polish. The actual sequence was:

```
Timeline
════════

Mar 22  Intern opens PR with 6 commits (initial migration attempt)
Mar 24  Maintainer reviews — flags 4 files as outdated
Mar 27  Intern responds with commit 34cf522:
            ✓ Data migration 0022 written
            ✓ test_user.py updated to use EmailAddress model
            ✓ test-user-data.json fixture cleaned up
            ✓ Legacy package fully removed
Mar 28  Maintainer adds 10 polish commits:
            + Fixtures for allauth EmailAddress rows
            + WgerAccountAdapter
            + HTML/text email templates
            + test_verification.py (7 new tests)
            + URL prefix fix
            + Setting name fix
Mar 29  PR merged ✅
```

The intern's response commit was the pivotal step that moved the PR from "needs work" to "almost mergeable." The migration itself — the hardest and riskiest part — was written by the intern. The maintainer then built the test coverage and template polish on top of it.

---

### What the Intern Got Right in `34cf522`

**1. The fixture crash fix** is a subtle but important detail. When `email_verified` was still present in `test-user-data.json` as a field on `core.userprofile`, but the migration had already dropped that column from the model, Django's test runner would crash on fixture load with:

```
django.core.serializers.base.DeserializationError:
  Problem installing fixture: ... column "email_verified" does not exist
```

Removing those keys from the fixture JSON was a necessary housekeeping step that shows the intern understood the relationship between migrations and test fixtures.

**2. The `is_trustworthy` test updates** demonstrate correct understanding of the migration's effect. Replacing:

```python
# OLD — field no longer exists after migration
user.userprofile.email_verified = True
```

with:

```python
# NEW — create the allauth EmailAddress row
EmailAddress.objects.create(user=user, email=user.email, verified=True)
```

...is exactly the right pattern. Tests that set `email_verified` directly would have crashed with `AttributeError` after the field was dropped.

**3. Writing the data migration** was the most complex part of the whole PR. The intern correctly used:
- `apps.get_model()` (not direct imports — historical snapshot pattern)
- `bulk_create(..., ignore_conflicts=True)` (idempotent)
- Dependency on `('account', '0001_initial')` (ensures allauth tables exist)
- `exclude(user__email='')` (skips users with no email, avoiding bad rows)

This is solid migration work for an intern.

---

### Lesson: Responding Well to Maintainer Review

The gap between the intern's initial 6 commits and this response commit shows a clear learning curve within a single PR:

| # | Initial Submission | After Maintainer Feedback |
|---|-------------------|--------------------------|
| 1 | No data migration | Migration 0022 written correctly |
| 2 | Tests still used `email_verified` field directly | Tests updated to `EmailAddress.objects.create()` |
| 3 | Fixture had stale `email_verified` keys | Fixture cleaned |
| 4 | Package not fully removed | Fully removed from `pyproject.toml` |

The intern read the review, understood each point, and bundled the fixes into a single clean commit with a descriptive message. This is good contributor behaviour — don't push one fix per comment as noisy micro-commits; group related fixes logically.

---

*Add this section between Section 9 and Section 10 of the main PR #2260 report.*


### Maintainer's 10 Commits

The maintainer's commits show he reviewed the intern's work, identified the gaps, and completed them professionally. This is the ideal PR review process for a complex migration: the intern does the groundwork, the maintainer elevates the quality to production standard.

---

## 11. 🧠 Key Concepts for Future Reference

### `get_allauth_email` Property

The `UserProfile` model gained a property `get_allauth_email` used in `api/views.py`:

```python
# In UserProfile model (inferred from views.py usage)
@property
def get_allauth_email(self):
    try:
        return EmailAddress.objects.get_for_user(user=self.user, email=self.user.email)
    except EmailAddress.DoesNotExist:
        return None
```

This is used in the API verify-email endpoint:
```python
email_obj = request.user.userprofile.get_allauth_email
if email_obj is None:
    return Response({'result': 'not sent', 'message': 'The user has no associated email'})
if email_obj.verified:
    return Response({'status': 'verified', ...})
email_obj.send_confirmation(request)
```

### `EmailAddress.objects.add_email()` vs `email_obj.send_confirmation()`

These are two different methods for different scenarios:

| # | Method | When to Use | What It Does |
|---|--------|------------|-------------|
| 1 | `EmailAddress.objects.add_email(request, user, email, confirm=True)` | New email being set (registration, email change) | Creates `EmailAddress` row if not exists, sends confirmation |
| 2 | `email_obj.send_confirmation(request)` | Resending for existing email | Sends new confirmation email without creating a new row |

Using the wrong one leads to bugs. In registration, use `add_email`. In "resend verification," use `send_confirmation`.

### `ACCOUNT_CONFIRM_EMAIL_ON_GET = True`

By default, allauth requires a POST request to confirm an email (to prevent bots from confirming by prefetching links). Setting this to `True` allows a GET request to confirm. wger sets this because the confirmation URL is typically accessed directly from an email client by clicking the link (which is always a GET).

### The `account/` URL Namespace

```python
# In wger/urls.py
path('account/', include('allauth.account.urls')),
```

This registers URLs like:
- `account/confirm-email/<key>/` → `account_confirm_email`
- `account/login/` → `account_login`
- `account/logout/` → `account_logout`
- `account/signup/` → `account_signup`
- `account/email/` → `account_email`

Note: wger has its own login/logout views (`WgerLoginView`), so the allauth versions at `account/login/` are not used for the main login flow. But `account_confirm_email` is essential — this is the URL that email confirmation links point to.

### The `is_verified` property on `UserProfile`

The `UserProfile` model also has an `is_verified` property (separate from the old `email_verified` field) that wraps the allauth lookup:

```python
@property
def is_verified(self) -> bool:
    email_obj = EmailAddress.objects.get_for_user(...)
    return email_obj.verified if email_obj else False
```

This is used in the serializer:
```python
email_verified = serializers.BooleanField(source='is_verified', read_only=True)
```

---

## 12. 🎓 Lessons Learned for Interns

Based on the entire PR — what the intern did well, what the maintainer corrected, and how the process went — here are the key takeaways:

### ✅ What the Intern Did Well

| # | Good Practice | Example |
|---|--------------|---------|
| 1 | Opened an issue first | Issue #2258 described the problem before any code was written |
| 2 | Incremental commits | 6 focused commits showing progression |
| 3 | Good commit messages | Described the "what" (though not always the "why") |
| 4 | Used `ignore_conflicts=True` | Shows understanding of idempotent migrations |
| 5 | Followed allauth patterns | Used `add_email()` and `send_confirmation()` correctly |
| 6 | Removed old package entirely | Didn't leave dead code alongside the new implementation |

### ❌ What Needed Improvement

| # | Issue | Lesson |
|---|-------|--------|
| 1 | Template at wrong path | Always check library documentation for exact expected paths |
| 2 | `email_verified` references not fully removed | Do a project-wide search for old references when migrating |
| 3 | Missing data migration fixtures for tests | When adding a new table, always add test fixtures for it |
| 4 | URL include not updated | When changing URL namespaces, grep for all usages |
| 5 | Debugging code left in | Never commit code with `SocialApp.objects.create(credentials)` style debugging |
| 6 | No tests written | Always write tests alongside feature code |

### 🔑 The Lessons

**The intern wrote 7 commits to complete this PR.** This tells us that the intern got the PR to "70% done" — functional enough that the direction was right, but needing some polish. For a maintainer, reviewing a PR and adding 10 polish commits is a significant time investment on their part. The closer an intern can get to "95% done" before requesting review, the easier the maintainer's job.

The gap between 70% and 95% done for this PR was:

1. Proper tests (the biggest gap — maintainer wrote all 7 new tests)
2. Correct template paths
3. Complete fixture coverage
4. Data migration polish
5. The custom adapter for email expiry

All of these are learnable patterns. Reading the allauth documentation more carefully before starting the implementation would have covered most of them.

### 🛠️ Migration Checklist (for future reference)

When replacing one package with another in Django, always do:

```
□ 1. Remove old package from INSTALLED_APPS
□ 2. Remove old package from pyproject.toml / requirements
□ 3. Add new package to INSTALLED_APPS
□ 4. Add new package to pyproject.toml
□ 5. Run new package's migrations (./manage.py migrate)
□ 6. Write a data migration to move existing data
□ 7. Remove old model fields that the new package replaces
□ 8. Project-wide search for all old API calls (grep for old import names)
□ 9. Update all templates that used old context variables
□ 10. Update all test fixtures to include new tables
□ 11. Write tests covering the new flow (happy path + edge cases)
□ 12. Update URL configuration
□ 13. Update documentation / README if applicable
□ 14. Test on a fresh database (no old data) AND on a database with existing data
```

---

## Appendix: File Change Summary

```
Files Changed: 21
Lines Added:   +808
Lines Removed: -176

New Files (7):
  wger/core/account_adapter.py
  wger/core/migrations/0022_move_email_verified_to_emailaddress.py
  wger/core/templates/account/email/email_confirmation_message.html
  wger/core/templates/account/email/email_confirmation_message.txt
  wger/core/tests/test_verification.py
  (+ .gitignore line)
  (+ uv.lock entries)

Deleted Files (3):
  wger/core/templates/email_verification/confirm_template.html
  wger/core/templates/email_verification/email_body_html.tpl
  wger/core/templates/email_verification/email_body_txt.tpl

Modified Files (11):
  pyproject.toml
  settings/settings_global.py
  wger/core/api/serializers.py
  wger/core/api/views.py
  wger/core/fixtures/test-user-data.json
  wger/core/models/profile.py
  wger/core/templates/user/preferences.html
  wger/core/tests/test_registration.py
  wger/core/tests/test_user.py
  wger/core/views/user.py
  wger/urls.py
```

---

*Report: wger PR #2260 — allauth email verification migration | 8,000 words | Written for intern learning*  
*Related: PR #2271 (Google OAuth social login, builds on this PR's allauth foundation)*












------------------------------------------------------------
------------------------------------------------------------



---

### 🔑 The Most Important Lesson

**A merged PR is a collaboration, not a solo effort — and the commit timeline tells the real story.**

Looking at the raw numbers in isolation is misleading:

| # | Contributor | Commits | Role |
|---|-------------|---------|------|
| 1 | Intern (`pankaj-basnet`) | 6 initial + 1 response = **7 total** | Feature implementation + feedback response |
| 2 | Maintainer (`rolandgeider`) | **10 polish commits** | Tests, templates, adapter, fixtures, URL fixes |

At first glance this looks like the intern did one-third of the work. That reading is wrong. Here is what the commit timeline actually shows:

```
What Each Contributor Owned
════════════════════════════════════════════════════════════════

INTERN OWNED (hardest, riskiest work):
  ✓ Identified the problem and opened issue #2258
  ✓ Removed django-email-verification entirely
  ✓ Substituted all send_email() calls with allauth equivalents
  ✓ Wrote the data migration 0022 (the most complex file in the PR)
      - apps.get_model() historical snapshot pattern ✓
      - bulk_create(ignore_conflicts=True) idempotency ✓
      - ('account', '0001_initial') dependency ✓
      - exclude(user__email='') edge case handling ✓
  ✓ Updated is_trustworthy tests to use EmailAddress model
  ✓ Fixed test-user-data.json fixture crash
  ✓ Responded to all 4 maintainer review comments in one clean commit

MAINTAINER OWNED (quality and completeness):
  ✓ Added EmailAddress rows to test fixtures for all users
  ✓ Created WgerAccountAdapter with expiry context
  ✓ Wrote HTML + text email templates in correct allauth path
  ✓ Wrote all 7 new tests in test_verification.py
  ✓ Fixed URL prefix (email/ → account/)
  ✓ Fixed setting name typo
  ✓ Final merge
```

The intern delivered the **structural migration** — the removal, the substitution, and critically the data migration that moves production data safely. The maintainer delivered the **polish layer** — tests, templates, and the custom adapter. Neither half works without the other.

The more instructive lesson is about the **response commit** (`34cf522`). The maintainer flagged 4 files on March 24. The intern came back on March 27 with a single, well-described commit that fixed all four issues at once, with a clear message explaining every change:

```
feat: migrate legacy email verification data to django-allauth

- Add data migration (0022) to transfer UserProfile.email_verified
  boolean to account.EmailAddress table.
- Update is_trustworthy test cases in test_user.py to utilize
  EmailAddress mock data.
- Clean up legacy fields in test-user-data.json fixture to prevent
  test runner crashes.
- Addresses maintainer feedback for PR wger-project#2260.
```

This is exactly how to respond to a code review. Not one micro-commit per comment. Not a vague "fixed review comments." One logical unit of work, clearly described, with references to the issue and PR. That single response commit is what moved the PR from "needs work" to "ready for maintainer polish."

The gap between the intern's initial submission and the final merged state was:

| # | Gap | Who Closed It |
|---|-----|--------------|
| 1 | Data migration missing | Intern (in response commit) |
| 2 | Tests still used dropped field | Intern (in response commit) |
| 3 | Fixture had stale keys causing crash | Intern (in response commit) |
| 4 | Package not fully removed | Intern (in response commit) |
| 5 | Template at wrong path | Maintainer |
| 6 | No EmailAddress fixture rows for test users | Maintainer |
| 7 | No WgerAccountAdapter for expiry context | Maintainer |
| 8 | No new tests for the verification flow | Maintainer |
| 9 | URL prefix wrong (`email/` vs `account/`) | Maintainer |
| 10 | Setting name typo | Maintainer |

Items 1–4 are things an intern should be able to catch with more careful testing before submission. Items 5–10 are things that require deeper knowledge of allauth's conventions and wger's testing standards — the kind of knowledge that comes from maintaining a codebase for years, not days. Do not expect to know these on a first contribution. **Do** expect to fix items 1–4 before requesting review.

The practical target for an intern submitting a PR is: **get to the point where the maintainer only needs to add polish, not fix correctness.** This PR got there after the response commit. That is a good outcome.













------------------------------------------------------------
------------------------------------------------------------










# Wger opensource fitness project (upstream) - Django backend

<p align="center">
  <img src="https://raw.githubusercontent.com/wger-project/wger/master/wger/core/static/images/logos/logo.png" width="100" height="100" alt="wger logo">

![AGPLv3 License](https://img.shields.io/badge/License-AGPLv3-blue.svg)
![Build Status](https://img.shields.io/github/actions/workflow/status/wger-project/wger/ci.yml?branch=master)
[![Coverage Status](https://coveralls.io/repos/github/wger-project/wger/badge.svg?branch=master)](https://coveralls.io/github/wger-project/wger?branch=master)
![Translation Status](https://hosted.weblate.org/widget/wger/svg-badge.svg)
</p>


wger (ˈvɛɡɐ) is a free workout and fitness manager.

- 🏋️ **Custom Workout Routines** – Create flexible routines with automatic weight progression rules.
- 📊 **Comprehensive Tracking** – Track diet plans, body weight, and custom measurements.
- 🍽️ **Nutrition Management** – Log your calories with a food database
  from [Open Food Facts](https://openfoodfacts.org).
- 📸 **Progress Gallery** – Upload and track your fitness progress with photos.
- 📚 **Exercise Wiki** – Access and contribute to the built-in exercises.
- 📱 **Cross-Platform Apps** – Available on
  [Android](https://play.google.com/store/apps/details?id=de.wger.flutter),
  [iOS](https://apps.apple.com/us/app/wger-workout-manager/id6502226792),
  [F-Droid](https://f-droid.org/en/packages/de.wger.flutter/),
  and [Flathub](https://flathub.org/apps/de.wger.flutter).
- 🐳 **Self-Hostable** – Deploy easily with Docker for full control.
- 🌍 **Multilingual Support** – Translated by the community via Weblate.
- 🔗 **Powerful API** – REST API for third-party integrations or automations.
- 👥 **Multi-User Support** – Includes basic gym management features.
- 🆓 **100% Free & Open Source** – Licensed under AGPL-3.0 or later.


For a live system, visit: <https://wger.de>

<p align="center" style="line-height:0; margin:0; padding:0;">
  <a href="https://play.google.com/store/apps/details?id=de.wger.flutter" target="_blank" style="text-decoration:none; border:none; outline:none;"><img src="https://raw.githubusercontent.com/wger-project/wger/master/wger/core/static/images/logos/play-store/badge.svg" alt="Get it on Google Play" height="50" style="margin-right:8px; vertical-align:middle; border:none; outline:none; display:inline-block;"></a>
  <a href="https://apps.apple.com/us/app/wger-workout-manager/id6502226792" target="_blank" style="text-decoration:none; border:none; outline:none;"><img src="https://developer.apple.com/assets/elements/badges/download-on-the-app-store.svg" alt="Download on the App Store" height="64" style="margin-right:8px; vertical-align:middle; border:none; outline:none; display:inline-block; background:none;"></a>
  <a href="https://f-droid.org/packages/de.wger.flutter/" target="_blank" style="text-decoration:none; border:none; outline:none;"><img src="https://raw.githubusercontent.com/wger-project/wger/master/wger/core/static/images/logos/fdroid/get-it-on.png" alt="Get it on F-Droid" height="50" style="margin-right:8px; vertical-align:middle; border:none; outline:none; display:inline-block; background:none;"></a>
  <a href="https://flathub.org/apps/de.wger.flutter" target="_blank" style="text-decoration:none; border:none; outline:none;"><img src="https://raw.githubusercontent.com/wger-project/wger/master/wger/core/static/images/logos/flathub/black.svg" alt="Get it on Flathub" height="50" style="vertical-align:middle; border:none; outline:none; display:inline-block; background:none;"></a>
</p>


