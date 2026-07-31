# Neuron Nursing — Student Mobile App (Play Store + App Store)

## Context

Neuron Nursing currently has no way to serve *already-admitted* students after they enroll. The public website only tracks a person through the admission/payment funnel (`Application` model, status: pending → awaiting_verification → paid → admitted). Once a student is admitted, the system stops modeling them entirely — there is no login, no ongoing academic tracking, nothing. Two seeded marketing claims on the homepage ("Digital attendance & weekly progress report to parents", "Attendance and result SMS sent to parents after every exam" — `FacilitySeeder.php`, `WhyUsItemSeeder.php`) describe features that **do not exist in code today**. The owner wants a real, installable mobile app (Play Store + App Store) so enrolled students can check notices, batch/payment info, and (in later phases) attendance and exam results — turning an unfulfilled sales pitch into an actual feature and giving the coaching center a "we have an app" credibility marker that matters in the local market.

Decided with the owner (confirmed, not to be re-litigated):
- **Real native app**, not a web-only portal — needs Play Store + App Store presence.
- **Login = Application Number + PIN** (a short numeric PIN the admin sets at enrollment), not SMS OTP — avoids ongoing SMS gateway costs, consistent with this project's established preference for free/low-cost approaches (Web Speech API over paid ElevenLabs, Gemini free tier).

This plan covers the **technical foundation** (auth, API, app framework, push notifications) once, and then a **phased feature list** on top of it, so the work can be shipped and verified in stages exactly like the recent admin-panel redesign (build → test locally → deploy → confirm live → next phase).

## Design System (locked)

Approved by the owner against a live HTML mockup (2 phone-frame concept: Dashboard + Quest Detail), inspired by a "quest/XP" gamified-app treatment but built entirely on the site's own existing brand tokens (`resources/css/app.css` `@theme` block) — not a generic gamified palette. This is now the canonical visual reference for the Flutter build; the Flutter app's own design tokens/theme file should be derived from this, not re-invented.

- **Color** (reuse the exact hex values already defined in `resources/css/app.css`): Ink `#071620` (dark surfaces, level ribbon), Teal `#0FB8A9` / Teal-dim `#0A8A80` (primary actions, XP fill, active tab), Coral `#FF5D6C` (streak flame, urgent state), Gold `#FFC24B` (XP/reward accents), Mint `#E9F7F4` (soft tile backgrounds), Paper `#F4FAF8` (screen background), Border `#DDECE8`.
- **Type**: a rounded/display face for headlines, level numbers, and quest titles (character + friendliness — matches the site's `Baloo 2` display face in spirit); a plain system sans for body/UI text with first-class Bangla rendering; a monospace face for XP counts, countdown digits, and streak numbers (`tabular-nums`, matches the site's existing `JetBrains Mono` mono role for data/labels).
- **Iconography**: 24×24 outline SVG, ~1.75–2px stroke, round caps — identical convention to the admin-panel icons already shipped (`resources/views/partials/admin-nav-links.blade.php`), so the student app and admin panel feel like the same product family.
- **Gamification pattern**: "Quest" = the existing To-Do item, reframed with an XP value; "Level ribbon" = an aggregate progress bar (e.g. total quests completed over time); tiles use one pastel tint per quest on a shared neutral surface, never more than one bold accent moment per screen.
- **Gradient hero cards**: the Login screen and Profile header use a soft mint-to-teal gradient card (inspired by a reference doctor-booking app mockup the owner shared) instead of a flat surface — reserved for exactly these two moments (first impression, identity), not overused elsewhere.
- **Slide-to-login**: the login action is a drag-to-confirm slider, not a tap button — see Login/Auth UX note below.
- **Reference artifact**: the approved mockup is preserved at the published Artifact URL from this conversation — pull it up again when briefing whoever builds the Flutter UI.

**Login/Auth UX — slide-to-confirm, not tap-to-submit.** The Login screen's submit control is a drag slider ("slide to login") rather than a plain button, at the owner's request as a bot-deterrence measure. Be honest about what this does and doesn't do: it adds light friction against casual/naive automated form-fill scripts (a drag gesture is one extra step a simple bot script wouldn't bother replicating), but it is **not a real security control** — a bot targeted specifically at this login endpoint can still POST directly to the API and skip the UI entirely. The actual protection is the `RateLimiter::for('student-login', ...)` already specified in the Foundation section below; the slider is a UX nicety layered on top, not a substitute for it.

**Profile as a shareable Digital V-Card.** The Profile screen doubles as a personal digital business card: alongside the CV-style academic/enrollment summary (see Phase 1 below), the student can add their own social links (Facebook/WhatsApp/LinkedIn/Instagram) — self-managed, same pattern as the To-Do/Notes/Bookmarks features. New `student_social_links` table (`application_id` FK, `platform` enum, `url`).

## Technical Foundation (build once, before any feature)

**Status: Auth + API layer — ✅ DONE, tested locally (18→27 passing tests), not yet deployed live.** Push notifications (FCM) below are still pending — blocked on the owner creating a free Firebase project (external account setup, same category of action as the Play Store/App Store accounts).

**Auth — extends `Application`, no new Student model.** `Application` already *is* the student identity record (`application_no` is its route key; it has `student_name`, `mobile`, `status`). Added a hashed `pin` + `pin_set_at` column directly to it (mirrors `AdminUser::password` with `'hashed'` cast) via `2026_07_27_203858_add_pin_to_applications_table.php`. A student logs into **one specific Application**, not an abstract cross-application identity — if someone re-enrolls after a cancelled application, they get a new `application_no` + new PIN, exactly matching how payments/batch/course already hang off `application_no` today. Login is blocked unless `status` is `paid` or `admitted`.

- `Application` (`app/Models/Application.php`) gains `Illuminate\Auth\Authenticatable` + Sanctum's `HasApiTokens` traits, `pin` hashed + hidden.
- **Correction from the original plan**: no separate `student` guard/provider was added to `config/auth.php`. Sanctum's `auth:sanctum` middleware resolves the tokenable model polymorphically from the `personal_access_tokens` row itself (`Laravel\Sanctum\Guard::__invoke()`) — it doesn't need a named guard per model. A dedicated guard entry would have been redundant; simpler to omit it.
- `RateLimiter::for('student-login', ...)` added in `AppServiceProvider::boot()` — copy of the existing `admin-login` limiter, keyed on `application_no` instead of `email`.
- **Laravel Sanctum v4.3 installed** (`composer require laravel/sanctum`, migration published).
- **Gotcha discovered and fixed**: `bootstrap/app.php` already called `$middleware->throttleApi()`, but the `api` rate limiter it references was never actually *defined* anywhere (nothing had used the `api` middleware group before this). First hit on any api-group route threw `MissingRateLimiterException`. Fixed by adding the standard Laravel default `RateLimiter::for('api', ...)` (60/min per user-or-IP) to `AppServiceProvider::boot()`.

**API layer — built.** `bootstrap/app.php`'s `withRouting()` now has `api: __DIR__.'/../routes/api.php'`. New `routes/api.php` with `Route::prefix('v1/student')->name('api.student.')`, mirroring `routes/admin.php`'s convention: `POST /login` (throttled `student-login`), and `auth:sanctum`-gated `POST /logout` + `GET /me`. New `App\Http\Controllers\Api\Student\AuthController`, `App\Http\Requests\Api\Student\LoginRequest`, `App\Http\Resources\Api\Student\ApplicationResource` (never exposes `pin` or the internal admin `note` field).

**Login error handling**: identical "invalid_credentials" response (401) whether `application_no` doesn't exist, has no pin set, or the pin is wrong — prevents application_no enumeration. A separate "not_eligible" response (403) for `pending`/`awaiting_verification`/`cancelled` status — that's not a security leak, just a status the applicant already knows.

**Tests**: `tests/Feature/StudentAuthTest.php` — login success/failure, status-gating, rate limiting (429 after 5 attempts/min), token issuance, `/me` auth requirement, logout token revocation (asserted against the `personal_access_tokens` row count directly, not a follow-up API call — Sanctum's `RequestGuard` memoizes the resolved user per guard instance, which persists across sequential `postJson()`/`getJson()` calls *within one test method* since they don't reboot the app the way two real separate HTTP requests do; confirmed via manual `curl` with genuinely separate requests that a logged-out token really does 401 on the next real request).

**Native app framework — Flutter.** One Dart codebase → both Play Store and App Store builds, free/open-source, strong Bangla text rendering, and the largest local Bangladeshi freelance-developer pool to hire from later. Pairs directly with the Sanctum token API: `dio`/`http` + `flutter_secure_storage` for the token, `firebase_messaging` for push.

**Push notifications — Firebase Cloud Messaging, riding on infra that already exists.** The queue is already `QUEUE_CONNECTION=database` in production, and `DEPLOY-BANGLA.md` already documents a cron entry running `queue:work --stop-when-empty` every minute — so a queued FCM job is not new infrastructure, just a new job class. Add a `Notice` model Observer (`app/Observers/NoticeObserver.php`) that dispatches a `SendFcmPushNotification` job on `created`, calling FCM's HTTP v1 API directly via `Http::withToken(...)` (skip the full `kreait/firebase-php` SDK to keep vendor size down for cPanel deploys). New `config/services.php` block (`fcm.project_id`, `fcm.credentials_path`) — service-account JSON stored outside the web root, never in `public/`.

**New foundation migrations:**
1. Add `pin` (hashed, nullable) + `pin_set_at` to `applications`.
2. Sanctum's `personal_access_tokens` migration (polymorphic, `tokenable` = `Application`).
3. New `device_tokens` table: `application_id` FK, `fcm_token` (unique), `platform` enum, `last_seen_at` — one Application can have multiple device tokens (phone + tablet, or reinstalls).
4. Add `Schedule::command('sanctum:prune-expired --hours=24')->daily()` to `routes/console.php` so the token table doesn't grow unbounded on shared hosting.

**Gotchas specific to this cPanel deployment** (already learned the hard way elsewhere in this project): any new `env('FCM_...')` call must live only inside `config/services.php`, never inline — `config:cache` in the deploy step silently breaks it otherwise. FCM credentials JSON must sit outside the web root. Confirm the existing `queue:work` cron line is actually active on the live server before shipping push notifications (it's documented but easy for a non-technical owner to have skipped).

## Phased Feature List

### Phase 1 — MVP (login + read-only info, no new academic-tracking data)
Everything here reuses data that already exists — the app is just a new read surface on top of it.
- **Login**: Application Number + PIN.
- **Home dashboard**: student name, course, batch, admission status at a glance.
- **Topper of the Week banner**: an admin-managed spotlight card at the top of every student's Dashboard — same broadcast-to-all-students visibility as Notices, not personal to the viewer. New `weekly_topper` table (or a single-row settings-style record) with `student_name`, `photo` (nullable — falls back to a monogram avatar, exactly like the existing Testimonial/Topper photo convention), `achievement_note`, `week_of`. **Deliberately admin-curated, not auto-computed from the Leaderboard** — the batch Leaderboard ranks by app-engagement (quests/streak), which isn't the same thing as "who actually deserves recognition this week," and putting a student's real name/photo in front of every other student needs a human check each time, not an automatic algorithm. Same photo-consent requirement already established for Testimonial/Topper uploads applies here.
- **Notices & Notice Ticker**: existing `Notice`/`NoticeTickerItem` content, surfaced in-app with push alerts on new notices.
- **Payment history**: list of `Payment` rows for their application (amount, status, date) — "have I paid, what's outstanding."
- **Course & batch info**: fee breakdown, schedule text, admission session — from `Course`/`Batch`.
- **Profile (CV-style)**: a single scrollable profile screen — photo, contact, academic background (SSC/HSC year/group/GPA), father's/mother's name, address, enrolled course/batch, admitted-since date. Confirmed via the `applications` migration that **every one of these fields already exists** on `Application` (`father_name`, `mother_name`, `dob`, `gender`, `address`, `district`, `ssc_year`/`ssc_group`/`ssc_gpa`, `hsc_year`/`hsc_group`/`hsc_gpa`, `photo_path`) — this screen is almost entirely a read-only view over data collected at admission time, not new data entry. A "Download as PDF" action is a nice-to-have, not required for v1. (An "Attendance % / latest mock test score / batch rank" summary row shown on this screen depends on Phase 2's Attendance/MockTestResult models — show it only once Phase 2 ships, omit it in the Phase 1 version.)
- **Results & Toppers**: existing public `Result`/`Topper` data, shown in-app too (motivational, low build cost).
- **Contact/support**: call and WhatsApp buttons from `BranchProfile` contact info; optionally surface the existing AI voice-agent widget for quick Q&A.
- **PIN self-service**: change PIN from within the app after first login.
- **Forgot PIN**: student calls/visits the office; admin resets the PIN from the admin panel (same `pin`/`pin_set_at` columns from the foundation) — no self-serve SMS/email reset needed, matching the no-SMS-cost decision already made for login itself.
- **Language toggle**: English/Bangla switch inside the app, reusing the bilingual data (`_en`/`_bn` columns) already present on every model.

**Student self-managed features** (no admin involvement at all — the student creates/controls this data entirely themselves; kept separate here because every other feature in this plan is admin-authored content the student only reads):
- **Personal To-Do List**: a study checklist/planner (e.g. "revise Chapter 4", "submit assignment Friday"). New `student_todos` table (`application_id` FK, `title`, `is_done`, `due_date`, timestamps) + simple CRUD API under `/api/v1/student/todos`.
- **Exam Countdown**: a big "X days left" counter to the admission-test date — motivational, purely a display calculation (either from `BranchProfile.early_bird_deadline`/admission session, or a per-student target date field), no new heavy data model needed.
- **Daily Study Streak**: student taps "studied today ✓"; app tracks a consecutive-day streak count. New `study_log_entries` table (`application_id` FK, `date`, unique per day) — a habit-building/retention hook, same pattern as Duolingo-style streaks.
- **Personal Notes**: free-form text notes the student jots down per subject/topic, purely private. New `student_notes` table (`application_id` FK, `subject`, `body`, timestamps).
- **Bookmarks/Save**: let the student save a Notice or (once Phase 2 ships) a Study Material for quick access later. New `student_bookmarks` table (`application_id` FK, polymorphic `bookmarkable_type`/`bookmarkable_id`).
- **Social links (Digital V-Card)**: student adds their own Facebook/WhatsApp/LinkedIn/Instagram links to their Profile screen, turning it into a shareable digital business card alongside the CV-style academic/enrollment summary. New `student_social_links` table (`application_id` FK, `platform` enum, `url`).
- **To-Do due-date reminders**: reuses the Phase 1 FCM push-notification pipeline already being built for Notices — a scheduled job checks `student_todos.due_date` and sends a push the day before, no new infrastructure beyond one more queued job type.

### Phase 2 — Academic tracking (delivers on the existing marketing promise)
Requires new data models and a small admin-side data-entry UI (reuses the CRUD patterns already built for Courses/Batches/etc. in the admin panel).
- **Attendance**: new `Attendance` model (per class-date, per application), marked by admin/teacher, shown as a calendar/percentage view in-app.
- **Exam / mock-test results**: new `MockTestResult` model (subject-wise marks, batch rank), shown as a list + trend view.
- **Class routine**: structured weekly timetable (new `ClassRoutine` model: batch, day, time, subject, teacher) replacing the current free-text `schedule_en` field.
- **Study materials**: downloadable PDFs/notes per course/batch (new `StudyMaterial` model + admin file upload).

### Phase 3 — Engagement & growth
- **Live class links**: Zoom/YouTube Live URL surfaced on class day, admin-editable per batch.
- **Ask a teacher / doubt-clearing**: either a simple message form routed to the admin's contact inbox, or extend `VoiceAgentController` to accept the logged-in student's application context so it can answer personalized questions (fee due, next class).
- **In-app fee payment**: reuse the existing bKash payment-gateway code so a student can pay their next installment directly from the app instead of visiting the office.
- **Referral tracking**: existing students referring new admissions — common growth lever for coaching centers.
- **Push notification preferences, profile photo, secondary guardian contact.**
- **Optional free-API candidates** (from a general public-APIs directory the owner shared — most of that directory is irrelevant novelty content for this app; these two entries are the only ones with a real, narrow fit): [Free Dictionary API](https://dictionaryapi.dev) (no key required) could auto-generate definitions/example sentences for the English Vocabulary quest type; [Nager.Date](https://date.nager.at) (no key required, public-holiday data) could make the Exam Countdown / future Class Routine skip known holidays automatically — verify Bangladesh coverage before relying on it. Neither is committed to a phase yet; revisit only if/when those specific quest types are built.
- **Badges/Achievements**: unlock a badge at milestones (7-day streak, 50 quests done, first mock test 80%+) — new `student_badges` table (`application_id` FK, `badge_key`, `earned_at`); badge rules evaluated in the same job/observer that already updates streaks/XP, no separate infrastructure.
- **Batch Leaderboard**: opt-in ranking of quest/streak activity among batch-mates — a read-only query over existing `student_todos`/`study_log_entries` grouped by `batch_code`, no new core data; must be opt-in (default off) since it surfaces one student's activity to others.
- **Achievement Share**: generate a shareable image of a badge/streak milestone for WhatsApp/Facebook status — free organic marketing for the coaching center every time a student shares one. Share action lives directly on each unlocked badge tile (not a separate flow).
- **IA note**: Badges, Leaderboard, and Share are not standalone/disconnected screens — Profile is the hub. Profile shows a compact preview (3 recent badges + current batch rank) with a "View All" link into the full Achievement grid and a separate Leaderboard screen. This mirrors how the mockup was built and should carry into the real Flutter navigation (Profile tab → Achievement/Leaderboard as sub-pages, not top-level tabs — keeps the 5-item bottom nav limit intact).
- **Quick Flashcards**: student creates simple front/back flashcards per subject for self-revision — new `student_flashcards` table (`application_id` FK, `front`, `back`), same self-managed pattern as Personal Notes.
- **Home-screen widget**: Android/iOS widget showing the Exam Countdown or Study Streak without opening the app — a Flutter-side platform-widget feature (`home_widget` package), no backend change beyond the data already served by existing endpoints.

## Cross-Cutting: Standard App Polish (applies across phases, not tied to one)

These are expected of any well-built app regardless of the coaching-specific content — mostly Flutter-side work, minimal-to-no new backend:
- **Settings screen**: language toggle, notification on/off, change PIN, log out — the home for the Phase 1 self-service items above.
- **Dark mode**: a second theme derived from the same locked design-system tokens (see Design System section) — not a from-scratch palette.
- **Offline access**: cache the last-fetched Notices/Materials/quests locally (Flutter's standard local-db/cache patterns) so the app is still useful with no signal — common in Bangladesh mobile-data conditions.
- **Biometric unlock**: fingerprint/Face ID as a convenience layer on top of the PIN (device-local, via Flutter's `local_auth` — no backend change, the PIN/token still authenticates the account).
- **Pull-to-refresh** on every list screen (Notices, Payments, Quests) — standard mobile interaction, no backend change.
- **Search**: client-side filter over already-loaded Notices/Materials — no new search infrastructure needed at this data scale.
- **Rate us / feedback prompt**: ask for a Play Store/App Store review after a positive moment (e.g. right after completing a quest or hitting a streak milestone), not on a timer — standard app-growth practice.

## Verification

- **Phase 0 (foundation)**: `php artisan test` must keep passing (18 existing tests); add feature tests for student login (correct PIN succeeds, wrong PIN fails, cancelled-status login blocked, rate limiting kicks in after N attempts) and for token issuance/revocation.
- **Each feature phase**: build and test the API endpoints locally with `php artisan test` + manual `curl`/Postman checks before any Flutter work consumes them; once the Flutter app can call them, verify end-to-end on a real device (or emulator) against the local API before pointing it at the live server; deploy backend changes to the live cPanel server following the established zip-upload + secret-script pattern, verify with the `browse` skill against the real endpoints, then move to the next phase — same one-phase-at-a-time cadence as the admin-panel redesign.
- Store credentials (FCM service account, Sanctum config) the same careful way `.env` secrets are already handled in this project — never commit them, never paste them in chat.
