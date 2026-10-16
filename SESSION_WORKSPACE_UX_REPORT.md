# Session Workspace UX Report

## Screens Changed

- `admin_app/lib/ui/teacher/session_workspace/session_workspace_screen.dart`
  - Keeps the session summary and workspace visible in one route.
  - Flushes Smart Save before exit and lifecycle actions.
  - Presents missing work in a side panel with direct student jump actions.
- `admin_app/lib/ui/teacher/session_workspace/teacher_workspace.dart`
  - Rebuilt the roster as the primary control surface.
  - Student panel is now two clean fields only: student picker + filter picker.
  - Student picker opens a searchable list sheet; filter opens a simple option list.
  - Removed always-visible roster, filter chips, bulk buttons, and extra clutter from the left panel.
  - Attendance lives in a separate center card with direct حاضر/غائب buttons.
  - Secondary bulk actions moved into a compact overflow menu.
  - Added attendance summary: الحضور، الغياب، الإجمالي.
  - Retained the selected student as the only center-panel subject.
  - Kept all four compact score-only evaluation cards visible together.
  - Added Enter save and Ctrl+Enter save-and-next actions.
- `admin_app/lib/ui/teacher/session_workspace/session_workspace_controller.dart`
  - Replaced timer autosave with an explicit Smart Save dirty queue.
  - Flushes pending edits on student switch, keyboard navigation, lifecycle actions, completion, and exit.
  - Tracks saved, saving, pending, and failed states per evaluation and attendance.
- `api/app/Features/LearningRecords/Actions/ValidatesLearningRecord.php`
  - Supports the new `score_only` contract for memorization, revision, matn, and listening.
  - Preserves legacy range/topic validation for existing clients that do not send `score_only`.
- `api/database/migrations/2026_08_05_000001_add_homework_driven_evaluation.php`
  - Adds homework type, homework-to-evaluation linkage, and not-memorized result persistence.
- `admin_app/lib/ui/teacher/session_workspace/session_workspace_controller.dart`
  - Adds optional per-student تسميع and مراجعة assignments with Quran ranges.
- `admin_app/lib/ui/teacher/widgets/grade_chips.dart`
  - Uses the requested quick score set: 100, 95, 90, 85, 80, 75, 70.
- `admin_app/lib/ui/teacher/session_workspace/widgets/session_summary_bar.dart`
  - Keeps session name, student totals, present, absent, evaluated, and remaining counters visible.

## Components Changed

- `TeacherWorkspace`: fixed-area responsive shell and keyboard action map.
- `_Roster`: local search/filter state and compact roster rendering.
- `_AttendanceChoice`: one-click attendance and completion state.
- `_EvaluationCard`: score chips, custom score, `لم يحفظ المطلوب`, `لا تقيم`, and per-record save status.
- `_HomeworkAssignmentsCard`: independent optional تسميع and مراجعة homework per student.
- `_SessionEvaluationCard`: student history grouped by session with attendance, homework ranges, and evaluation results.
- `_Progress`: live attendance and evaluation completion counts.
- `SessionWorkspaceController`: Smart Save queue and lifecycle flush boundaries.

## Before vs After Workflow

### Before

1. Navigate through attendance and evaluation tabs.
2. Mark attendance and press a separate save action.
3. Select a type tab.
4. Select a student from a switcher.
5. Fill one score-only evaluation and press save.
6. Repeat across four tabs and 15-40 students.
7. Navigate to summary to locate missing items.

### After

1. Press `تحديد الجميع حاضر` or `تحديد الجميع غائب`.
2. Click `غائب` only for exceptions.
3. Filter/search the roster or select the next present student.
4. Mark the required homework cards plus matn/listening without changing views.
5. Press Down, or Ctrl+Enter, to save and move forward.
6. Use `عرض النواقص` to open the side panel and jump directly to a student.

## Save Behavior

Local state updates immediately. Each changed attendance/evaluation is added to a dirty set. Smart Save flushes that set at navigation and lifecycle boundaries instead of running a timer request after every change. It also runs before completion and exit. Failed records remain dirty and display `⚠ فشل الحفظ` with retry; lifecycle actions are blocked while unsaved work remains.

## Click Reduction Estimate

For 25 students with all four evaluation types:

| Operation | Previous | New frequent path |
|---|---:|---:|
| Attendance | 25 row choices + save | 1 bulk action + absent corrections |
| Evaluation saves | Up to 100 explicit saves | 0 save clicks; Ctrl+Enter is optional |
| Type navigation | Up to 100 tab selections | 0 |
| Student switching | Repeated switcher interactions | Down key or one roster click |
| Missing work | Summary navigation | 1 side-panel button |

Estimated frequent-path interaction drops from roughly 150-225 clicks to roughly 5-35, depending on the number of absences and whether keyboard navigation is used. This is an estimated 75-95% interaction reduction for the common case.

## Evaluation Time Estimate

For 15-40 students, the former tab/form flow is estimated at 8-20 minutes. The roster workflow is estimated at 3-9 minutes for score-only sessions. The only content-entry exception is the optional memorization homework range, entered once in the separate homework card. This represents an expected 55-75% time reduction.

## Performance Impact

- Roster, attendance, reference data, and session evaluations remain preloaded.
- Student changes are local and do not navigate or fetch.
- Attendance still uses one bulk API write.
- Evaluation writes are limited to dirty records and are serialized through one Smart Save flush.
- Completion status is not refetched after every evaluation write.
- No timer continues running after the teacher stops editing.
- Pending homework is loaded by circle and student, not by a target-session id.

## Homework-Driven Business Logic

- Homework now has an explicit type: `memorization` (تسميع) or `revision` (مراجعة).
- Both types are optional per student and can be cancelled before the next session.
- Homework remains active until it is evaluated; the next session consumes the active assignment without requiring a target session id.
- Memorization and revision evaluation cards appear only when that student has an active assignment of the matching type.
- Matn and listening remain regular evaluation types for every present student.
- A homework evaluation stores `homework_id` and either a score or `result_status=not_memorized` (`لم يحفظ المطلوب`).
- `لا تقيم` remains reserved for matn/listening and is not used to represent a missing homework assignment.
- Session completion now requires attendance, matn/listening results, and only the active memorization/revision assignments.
- Assignments created in the current session are intentionally excluded from the current session's completion requirements and become active on the next visit.

## Mobile Responsiveness Review

- Desktop and tablet widths use a persistent two-area layout with the roster fixed at the left and the selected student in the center.
- Tablet widths retain the roster as a narrow, scrollable panel, avoiding the previous full-width student carousel.
- Large mobile widths stack a compact horizontal roster above the selected-student work area.
- Evaluation cards remain vertically scrollable on mobile, while all four types remain on the same screen and no tabs are introduced.
- Search, filters, attendance buttons, and score chips remain touch-sized and usable without a mouse.

## Verification

- `flutter analyze` passes with no issues.
- `flutter test` passes. Live API integration tests skip when the local API is unavailable, as designed by the existing test suite.
