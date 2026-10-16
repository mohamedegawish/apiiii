# Teacher Workspace UX Redesign

## Outcome

The teacher session screen is now a single marking workspace. The roster, attendance controls, progress, missing-items access, and all four evaluation types are available without a page transition or tab switch.

## Screens Changed

- `admin_app/lib/ui/teacher/session_workspace/session_workspace_screen.dart`
  - Removed the six-tab session flow.
  - Hosts one persistent workspace and keeps lifecycle actions at the top.
- `admin_app/lib/ui/teacher/session_workspace/teacher_workspace.dart`
  - Added the roster-plus-evaluation layout.
  - Desktop: roster on the left and evaluation cards in the center.
  - Compact layouts: horizontal roster followed by the same evaluation surface.
  - Added keyboard Up/Down student navigation.
  - Added live progress and `عرض النواقص` jump dialog.
  - Attendance is editable directly in each roster row.
  - Added bulk present and bulk N/A actions.
- `admin_app/lib/ui/teacher/session_workspace/widgets/session_summary_bar.dart`
  - Added students and remaining counters to the session summary.
- `admin_app/lib/ui/teacher/widgets/grade_chips.dart`
  - Changed common scores to 100, 95, 90, 85, 80, 75, and 70.

## Components Changed

- `SessionWorkspaceController`
  - Keeps all roster, attendance, and evaluation drafts in memory.
  - Adds per-record debounced autosave with pending/saved/failed state.
  - Adds retry for failed evaluation saves.
  - Advances to the next incomplete student after a successful save.
  - Exposes evaluated, remaining, and incomplete counts for live UI updates.
  - Stops refreshing server completion status after every individual evaluation.
- Existing `SurahRangeField` and `MatnRangeField` remain available inside the cards, so required content validation is preserved.

## API Changes

No new API endpoint was required. Existing contracts are reused:

- `GET /sessions/{session}/roster` preloads students.
- `GET /attendance?circle_session_id=...` preloads attendance.
- `GET /learning-records?circle_session_id=...` preloads all evaluation records.
- `POST /attendance/bulk` saves the attendance roster in one request.
- `POST /learning-records` and `PUT /learning-records/{id}` save only changed evaluation records.
- `GET /sessions/{session}/completion-status` remains available for authoritative completion checks, but is no longer called after every score change.

## Performance Improvements

- Student switching is local state only and does not fetch or navigate.
- All four evaluation types are rendered together; tab switching is eliminated.
- Attendance is one bulk action plus only the absent-student corrections.
- Common scores are one click.
- N/A is one click and has no modal.
- Autosave uses a 650ms debounce, preventing a request for every keystroke.
- Save state is visible as `✓ تم الحفظ` or `لم يتم الحفظ` with retry.
- Keyboard Up/Down changes students; normal Tab order moves through fields.

## Click Count Estimate

The following comparison uses a 25-student session, all students present, and four evaluation types.

| Activity | Previous flow | Workspace flow |
|---|---:|---:|
| Open attendance/evaluation views | 6 tab selections | 0 |
| Mark attendance | 25 row actions + save | 1 bulk action + absent corrections |
| Select students | Up to 25 switcher selections | 0 with keyboard, or 25 roster clicks |
| Save evaluations | 100 explicit save clicks | 0 autosave clicks |
| Missing-items inspection | Summary tab + navigation | 1 button |
| Estimated frequent-path clicks | 157+ | 5 to 30 |

The lower new range assumes keyboard navigation and one bulk-present action. Even with roster clicks, removing tab switches and 100 save clicks produces a reduction above 50% before accounting for reduced scrolling and context loss.

## Flow Comparison

### Previous

1. Open attendance tab.
2. Mark or bulk-mark attendance.
3. Press save attendance.
4. Switch to a type tab.
5. Find/select a student.
6. Fill the type form.
7. Press save.
8. Repeat steps 4-7 for four types and every student.
9. Switch to summary to find missing items.

### New

1. Press `تحديد الجميع حاضر`.
2. Change only absent students in their roster rows.
3. Select the first present student once, or use the selected student.
4. Click score chips or `غير منطبق` on the four visible cards.
5. Use Down to move to the next incomplete student; autosave runs in the background.
6. Use `عرض النواقص` only when a targeted correction list is needed.

## Estimated Time Reduction

For 15-30 students, the old interaction pattern is approximately 8-15 minutes depending on range-entry complexity. Removing 100 save actions, four repeated view changes per student, and repeated student-list navigation is estimated to reduce the frequent-path session to approximately 3-7 minutes, a 50-65% reduction. Range and topic entry remain intentionally available where required by the evaluation data model.
