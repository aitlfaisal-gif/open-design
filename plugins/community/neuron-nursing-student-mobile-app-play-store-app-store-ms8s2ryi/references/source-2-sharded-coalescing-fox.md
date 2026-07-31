# MCQ Quiz Feature — Duolingo-style Practice for the Student App

## Context

The student wants a nursing-MCQ practice feature inside the `neuron-student-app` Flutter app, styled after Duolingo: a lesson path with locked/unlocked units, hearts/lives per quiz session, instant right/wrong feedback, and integration with the app's existing XP/streak/badge/leaderboard gamification (already live for the todo-quest system). Content will be authored by the branch admin through the admin panel — no bulk content import needed.

This is a new first-class feature spanning the Laravel backend (`/Users/niazgdm/Downloads/Neuron`) and the Flutter app (`/Users/niazgdm/Downloads/neuron-student-app`), and must slot into two established conventions already audited in this session:
- **Backend admin CRUD pattern**: hand-rolled Blade CRUD (no Form Requests/services), `auth:admin` session guard, manual routes in `routes/admin.php` (see `Admin/MockTestResultController`, `Admin/ClassRoutineController`, `Admin/StudyMaterialController`).
- **Backend student API pattern**: stateless `auth:sanctum`, `Application` itself is the authenticated model, flat `JsonResource` responses, read-time-derived stats everywhere (attendance %, streak, and — critically — **XP has no persisted source of truth today**, it's duplicated inline arithmetic in `LeaderboardController` and in `dashboard_screen.dart`).
- **Flutter pattern**: `StatefulWidget` + `setState`, Provider only for DI, `Navigator.push(MaterialPageRoute(...))` (no named routes), plain `fromJson` models, `AppTypography`/`AppColors` design tokens, zero animation packages (hand-rolled `AnimationController`, e.g. `lib/widgets/slide_to_action.dart`).

**Deliberate architecture decision**: XP is currently duplicated as inline `completed_quests * 20` in two places with no shared source of truth. Rather than add a *third* parallel MCQ-XP formula, this plan consolidates XP into one shared `Application::xp()` method used everywhere, folding in quiz-correct-answers alongside completed todos. This is a small, well-justified refactor riding along with the new feature, not scope creep — the alternative (a third duplicated formula) would make the existing smell worse.

**Explicitly out of scope for v1** (flag if the student wants these later — each is a natural fast-follow, not part of this build): heart refill timers/monetization, combo multipliers, daily-goal mechanics separate from the existing streak, spaced repetition / SRS scheduling, confetti/celebration animation packages (hand-rolled feedback only for v1).

---

## Backend (Laravel) — `/Users/niazgdm/Downloads/Neuron`

### Schema — 4 new tables, one new column source, no changes to existing tables

Real FKs between the new entities (they're first-class, not references into existing Batch/Course/Application tables — matches the codebase's own convention: plain-string scoping like `batch_code` is reserved for *referencing* pre-existing tables, real FKs are used between genuinely-new related tables).

```
quiz_subjects
  id, title (string), course_slug (nullable string, indexed, validated exists:courses,slug — optional
    course-scoping like StudyMaterial's broadcast-or-scoped pattern; null = shown to everyone),
  sort_order (unsignedInteger default 0), timestamps

quiz_units   -- one "lesson node" on the path; unlock sequence = sort_order within a subject
  id, quiz_subject_id (FK cascadeOnDelete), title (string), sort_order (unsignedInteger default 0),
  pass_threshold (unsignedInteger default 70) -- % correct required to pass/unlock the next unit,
  timestamps, index(quiz_subject_id, sort_order)

quiz_questions
  id, quiz_unit_id (FK cascadeOnDelete), question_text (text), options (json — array of 4 strings),
  correct_option_index (unsignedTinyInteger, 0-3), explanation (text, nullable),
  sort_order (unsignedInteger default 0), timestamps, index(quiz_unit_id, sort_order)

quiz_question_attempts   -- the ONE persisted table; everything else (pass/fail, unlock, XP, badges)
                          -- is derived read-time from this, matching attendancePercentage()/currentStreak()
  id, application_id (FK cascadeOnDelete), quiz_question_id (FK cascadeOnDelete),
  selected_option_index (unsignedTinyInteger), is_correct (boolean), answered_at (timestamp),
  timestamps, unique(application_id, quiz_question_id) -- re-answering upserts, most recent attempt counts,
  index(application_id)
```

No separate "attempt summary" table — unit pass/fail, correct/total counts, and unlock state are all computed by joining `quiz_question_attempts` to `quiz_questions` filtered by `quiz_unit_id` + `application_id`, exactly like `attendancePercentage()` joins `attendances`.

### Models

- `app/Models/QuizSubject.php`, `QuizUnit.php`, `QuizQuestion.php`, `QuizQuestionAttempt.php` — `$fillable`/`$casts` (`options` cast to `array`) + `belongsTo`/`hasMany` relations, matching `MockTestResult.php`'s trivial style.
- `app/Models/Application.php` additions:
  - `quizQuestionAttempts(): HasMany` relation (alongside the existing `todos()`/`badges()`/`studyLogEntries()` block at lines 60-95).
  - **New `xp(): int` method** — consolidates the two duplicated inline formulas: `return ($this->todos()->where('is_done', true)->count() + $this->quizQuestionAttempts()->where('is_correct', true)->count()) * 20;`
  - **Extract `logStudyToday(): array`** out of `StudyLogController::store()`'s inline body (idempotent insert into `study_log_entries` for today + return `['streak' => ..., 'logged_today' => true]`) so both `StudyLogController::store()` and the new quiz-answer endpoint can call the same method instead of duplicating the idempotent-insert logic.
  - `evaluateBadges()`: add one new rule to the existing `$rules` array (same pattern as `streak_7`/`quests_50`/`mock_80`): `'quiz_25_correct' => fn () => $this->quizQuestionAttempts()->where('is_correct', true)->count() >= 25`.

### Admin CRUD (matches `Admin/ClassRoutineController`'s drill-down-by-query-param pattern exactly)

- `Admin/QuizSubjectController` — full CRUD (index/create/store/edit/update/destroy), fields: title, course_slug (nullable select), sort_order.
- `Admin/QuizUnitController` — full CRUD, `index(Request $request)` takes `?subject=` (default: first subject by sort_order) exactly like `ClassRoutineController`'s `?batch=` — fields: title, sort_order, pass_threshold, quiz_subject_id (from the query param).
- `Admin/QuizQuestionController` — full CRUD, `index(Request $request)` takes `?unit=` — fields: question_text, 4 option text inputs, radio-select for correct option, explanation (optional), sort_order.
- Routes: append to the existing `auth:admin` group in `routes/admin.php` (no new middleware needed), same 6-route-per-resource shape as the three reference controllers, params `{quizSubject}`/`{quizUnit}`/`{quizQuestion}`.
- Views: `resources/views/admin/quiz-subjects/{index,form}.blade.php`, `admin/quiz-units/{index,form}.blade.php`, `admin/quiz-questions/{index,form}.blade.php` — copy the exact Tailwind input classes, Alpine `x-data="{submitting:false}"` spinner-button pattern, and `admin.partials.empty-state` include used by the three reference `form.blade.php`/`index.blade.php` files verbatim.
- Nav: **one** new entry "MCQ কুইজ" in `resources/views/partials/admin-nav-links.blade.php` linking to the subjects index (matches `request()->routeIs('admin.quiz-subjects.*')`), with units/questions reached by drilling in from there — avoids cluttering the sidebar with 3 top-level entries for what's really one feature.
- Lang keys: add to both `lang/en/site.php` and `lang/bn/site.php` in lockstep (title/labels for all three CRUD screens) — required for the existing `lang:check` test to keep passing.

### Student API (all under the existing `auth:sanctum` group in `routes/api.php`)

Security-critical design point: **the questions payload never includes `correct_option_index` or `explanation`** — those only come back in the per-question answer response, after the student has committed to an answer. This is a deliberate anti-cheat measure (a naive design would leak the answer key in the initial question list, defeating the quiz).

```
GET  /quiz-subjects                          -> QuizSubjectController@index
     scoped like StudyMaterial (course_slug null-or-match), each with a computed `progress`
     (units passed / total units for this application) via QuizSubjectResource

GET  /quiz-subjects/{quizSubject}/units      -> QuizUnitController@index
     ordered by sort_order; each unit's `locked` is computed read-time: false if it's the
     first unit in the subject, else true unless the previous unit (sort_order - 1) is
     "passed" (correct/total for that unit's questions >= pass_threshold); also returns
     `best_score` (correct_count/total_count) from existing attempts if any

GET  /quiz-units/{quizUnit}/questions        -> QuizQuestionController@index
     QuizQuestionResource strips correct_option_index and explanation — id, question_text,
     options only

POST /quiz-units/{quizUnit}/questions/{quizQuestion}/answer   -> QuizAnswerController@store
     body: { selected_option_index }
     server grades against the REAL correct_option_index, upserts one
     quiz_question_attempts row (application_id, quiz_question_id unique), calls
     $application->logStudyToday() (answering counts as "studied today", same as the
     existing manual study-log button), returns:
       { correct, correct_option_index, explanation, streak }
     this is the "instant feedback" call — one per question, mirrors the existing
     PATCH /todos/{id}/toggle per-item pattern

POST /quiz-units/{quizUnit}/complete          -> QuizUnitController@complete
     no body — aggregates this application's quiz_question_attempts for the unit's
     questions (read-time, not client-trusted), computes passed via pass_threshold,
     calls $application->evaluateBadges(), returns:
       { correct_count, total_count, passed, xp, newly_earned_badges }
```

### Wiring XP into existing endpoints (the consolidation)

- `LeaderboardController::index()` — replace the inline `'xp' => $a->completed_quests * 20` with a call to the new `$a->xp()` method. Also change `orderByDesc('completed_quests')` to sort the fetched collection by `xp()` in PHP after fetch (batch size is small — a coaching batch, not a global leaderboard — so an in-memory sort per request is fine and avoids a complex SQL rewrite), so quiz performance actually affects rank, not just the displayed number.
- `DashboardController::show()` — add `'xp' => $application->xp()` to the response array.

### Demo seeding (dev/admin-testing convenience only — NOT the real content source)

Extend `php artisan demo:seed` with a small fixed set (1-2 subjects, 2-3 units each, 5 questions each) of realistic dummy nursing MCQs, purely so the admin can see the admin CRUD screens populated and the student app flow can be smoke-tested end-to-end before real content is authored. Matches the existing `demo:seed` convention (6 teachers, 12 toppers, etc. — realistic placeholder data, swapped for real content later).

---

## Flutter (`/Users/niazgdm/Downloads/neuron-student-app`)

### Models (plain `fromJson`, no code-gen — matches every existing model)

- `lib/models/quiz_subject.dart` — id, title, progress (0.0-1.0 or unit counts).
- `lib/models/quiz_unit.dart` — id, title, sortOrder, locked (bool), bestScore (nullable record/fields).
- `lib/models/quiz_question.dart` — id, questionText, options (`List<String>`) — no correct-answer field, matches the stripped API resource.

### Service methods — `lib/services/student_api_service.dart`

```dart
Future<List<QuizSubject>> quizSubjects()
Future<List<QuizUnit>> quizUnits(int subjectId)
Future<List<QuizQuestion>> quizQuestions(int unitId)
Future<({bool correct, int correctOptionIndex, String? explanation, int streak})>
    answerQuizQuestion(int unitId, int questionId, int selectedOptionIndex)
Future<({bool passed, int correctCount, int totalCount, int xp, List<String> newlyEarnedBadges})>
    completeQuizUnit(int unitId)
```
Composite-payload methods return Dart records, matching the existing `badges()`/`leaderboard()` convention exactly (not a new wrapper model class).

### Screens (all pushed via `Navigator.push(MaterialPageRoute(...))`, no named routes — matches every existing sub-screen)

1. **Entry point**: `_AcademicQuickLinks` in `dashboard_screen.dart` gets a 5th tile ("MCQ কুইজ") alongside the existing Attendance/Mock Results/Class Routine/Materials tiles, pushing the new subjects screen.
2. `lib/screens/quiz_subjects_screen.dart` — list/grid of subjects with a progress indicator per subject. Standard `StatefulWidget` + nullable-list-as-loading-flag + `_load()` in `initState` + `RefreshIndicator` + `EmptyState` — identical skeleton to `flashcards_screen.dart`/`achievement_screen.dart`.
3. `lib/screens/quiz_units_screen.dart` — the lesson path: vertical list of unit nodes in order. Locked units: `Opacity(0.42)` + lock icon (exact precedent from `achievement_screen.dart`'s locked-badge treatment), not tappable. Unlocked-not-passed: normal, tappable. Passed: checkmark/star accent (`AppColors.gold`). Tapping an unlocked unit pushes the quiz-taking screen.
4. `lib/screens/quiz_taking_screen.dart` — the core session:
   - Local `StatefulWidget` fields: `int _questionIndex`, `int _hearts = 5`, `List<QuizQuestion> _questions`, per-question answered state.
   - One question shown at a time, 4 option buttons. Tapping an option calls `answerQuizQuestion(...)` immediately, then shows instant feedback: correct → teal/mint flash on the selected option (matches `flashcards_screen.dart`'s flip-color convention); incorrect → coral flash + `_hearts--` + reveal `correct_option_index` (highlight the right one in teal) + optional `explanation` text beneath. A "Continue" button advances to the next question (or ends the session).
   - Hearts row (❤️×5) at the top, rendered as filled/outline icons in `AppColors.coral`, decremented locally — **purely client-side, ephemeral per session, resets to 5 every time a unit is opened** (matches "no precedent for mid-session persistence" finding; this is also how Duolingo's practice-session hearts actually work, not a cross-session ledger).
   - Hearts hitting 0 ends the session early → straight to the results screen with whatever was answered so far.
   - Feedback animation: hand-rolled `AnimationController` + `AnimatedContainer`/`AnimatedSwitcher` for the color flash (matches `lib/widgets/slide_to_action.dart`'s existing precedent) — zero new package dependency.
   - When all questions are answered (or hearts hit 0), call `completeQuizUnit(unitId)` and push the results screen with its response.
5. `lib/screens/quiz_results_screen.dart` — pass/fail banner, score (`correct/total`), XP earned this session, streak (already updated via the per-question calls), any newly-earned badge callout (reuse `achievement_screen.dart`'s badge visual). Buttons: "আবার চেষ্টা করো" (pop back into a fresh `quiz_taking_screen` instance, hearts reset) / "চালিয়ে যাও" (pop back to the lesson path, which re-fetches and shows the next unit unlocked if passed).

### Typography / bilingual copy

All new screen copy goes through the existing `tr(context, bn, en)` helper (`lib/l10n/tr.dart`) and `AppTypography`'s HIG-named styles (`headline` for section titles, `bodyHig`/`callout` for question text, `caption1`/`caption2` for meta labels, `mono` for the hearts/score counters — matching this session's earlier typography migration exactly, so the new screens are consistent with the rest of the app from day one instead of needing a follow-up migration).

---

## Verification

1. **Backend**: `php artisan migrate` (new tables), `php artisan test --filter=Quiz` (new Feature tests to write, following the existing `StudentPhase2ApiTest`/`StudentPhase3ApiTest` style: login as a demo application, hit each new endpoint, assert grading correctness, assert the answer-key is never present in the pre-answer questions payload, assert unlock logic, assert XP/badge/streak side effects). Also re-run the full existing suite (`php artisan test`) to confirm the `Application::xp()` refactor didn't break `LeaderboardControllerTest`-equivalent coverage.
2. **Admin**: manually walk through creating a subject → unit → question via the new admin screens (`php artisan serve`), confirm it appears correctly, confirm validation errors render, confirm delete cascades correctly (deleting a subject removes its units/questions via cascadeOnDelete).
3. **Flutter**: `flutter analyze` (zero issues, matching the standard this session already established), then run on Chrome/emulator, log in as the seeded demo application, walk the full path: subjects → units (confirm lock/unlock rendering) → take a quiz (confirm instant feedback, hearts depleting, correct answer reveal on miss) → results (confirm XP/streak/badge numbers match what the backend actually returned) → back to lesson path (confirm next unit now shows unlocked). Screenshot each screen via the `browse` skill headless flow already used earlier in this session for visual QA.
