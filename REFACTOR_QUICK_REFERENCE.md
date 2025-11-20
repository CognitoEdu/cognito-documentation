# SubtopicElements Refactoring - Quick Reference

**One-page reference for key information**

---

## 📊 By The Numbers

| Metric | Value |
|--------|-------|
| **Current Size** | 6,461 lines |
| **Redundant Code** | 820 lines (14%) |
| **After Cleanup** | ~5,545 lines |
| **Target (Final)** | ~2,500 lines core |
| **Total Reduction** | ~61% |
| **Functional Areas** | 12 distinct areas |
| **GraphQL Queries** | 11 queries (9 after homework removal) |
| **Contexts Used** | 11 contexts |
| **Props** | 50+ props |

---

## 🗑️ What's Being Removed

### Immediate Removals (820 lines)

1. **Batch Saving** (~300 lines)
   - CREATE_COURSE_*_RESULTS mutations
   - saveResults/continueToSave state
   - excludeXP logic

2. **Homework** (~200 lines)  
   - 11 homework props
   - 2 homework queries
   - All homework conditionals

3. **Day-Based Limits** (~95 lines)
   - currentFreeType: 'days' branch
   - getDifferenceInDays() function
   - freeDayAllowance state

4. **Commented Code** (~120 lines)
   - Old handleRedoHard function
   - Flashcard complete dialog
   - Various setSaveResultDetails calls

5. **Other** (~105 lines)
   - Maintenance mode
   - Missing config fallback
   - Always-false toggle
   - Teacher save checks

---

## ✅ What's Being Kept

### Current Mechanisms (Working)
- ✅ saveCourseElementResult (per-element)
- ✅ saveCourseFlashcardResult (per-flashcard)
- ✅ markPremiumUserCourseAsComplete (activity summary)
- ✅ Current activity limit system
- ✅ All 10 cohort configurations
- ✅ isProxyPremiumAccount (emulates premium)
- ✅ isGuestAccount (restrictions)

---

## 🏗️ Cohort Configurations (Keep All)

| ID | Period | Elements | Activities | Across? | By Activity? |
|----|--------|----------|------------|---------|--------------|
| Guest | daily | 10 | 1 | No | No |
| Cohort 1 | daily | 15 | 1 | No | No |
| Cohort 2 | daily | 10 | 1 | No | No |
| Cohort 3 | daily | 15 | 1 | **Yes** | No |
| Cohort 4 | weekly | 25 | - | No | No |
| Cohort 5 | weekly | 50 | - | **Yes** | No |
| Cohort 6 | weekly | 10 | 7 | **Yes** | **Yes** |
| Cohort 7 | weekly | 10 | 10 | **Yes** | **Yes** |
| Cohort 8 | weekly | 10 | 3 | No | **Yes** |
| Cohort 9 | weekly | 10 | 5 | No | **Yes** |

**All have**: `includeLessons: true`, `initialFreeType: 'elements'`  
**None have**: `currentFreeType: 'days'`

---

## 🔄 Refactoring Phases

### Phase 0: Cleanup (1-2 weeks) ⭐ START HERE
Remove 820 lines of redundant code

### Phase 1: Hooks (2-3 weeks)
Extract 7 custom hooks:
1. useElementFetching
2. useFlashcardLogic
3. useExamQLogic  
4. useActivityLifecycle
5. useActivityAnalytics
6. useScoreTracking
7. useElementSaving

### Phase 2: Components (3-4 weeks)
Split into 5 components:
1. ActivityContainer (shared)
2. LessonActivity
3. QuizActivity
4. FlashcardActivity
5. ExamQActivity

### Phase 3: Contexts (2-3 weeks)
Consolidate 11 → 3 contexts

**Total**: 10-14 weeks

---

## ⚠️ Pre-Removal Checklist

### Before Removing Anything

- [ ] Read Executive Summary
- [ ] Read Redundancy Report
- [ ] Run DB verification queries:
  ```javascript
  // Day-based users
  db.users.find({ 'activityAllowance.currentFreeType': 'days', isPremiumAccount: false }).count()
  
  // Users without config
  db.users.find({ activityAllowanceConfig: null, isPremiumAccount: false, isGuestAccount: false }).count()
  ```
- [ ] If any found: Run migration script
- [ ] If zero found: Proceed with removal
- [ ] Create backup branch
- [ ] Get approval from tech lead

---

## 🧪 Testing Checklist

### After Removals

**Automated**:
- [ ] All unit tests pass
- [ ] All integration tests pass
- [ ] No console errors
- [ ] Build successful

**Manual** (each activity type):
- [ ] Start lesson (focus mode)
- [ ] Start lesson (all-at-once)
- [ ] Start quiz
- [ ] Start flashcards
- [ ] Start exam qs
- [ ] Hit activity limit
- [ ] Complete activity
- [ ] Retry mistakes
- [ ] Restart activity
- [ ] Resume activity

**Analytics**:
- [ ] Activity started events fire
- [ ] Element response events fire
- [ ] Activity complete events fire
- [ ] Display mode events fire

---

## 🎯 Priority Removals (Safest First)

### Week 1: Low-Risk Removals
1. ✅ Commented code (~120 lines)
2. ✅ Maintenance mode (~10 lines)
3. ✅ Always-false toggle (~25 lines)

**Total**: 155 lines, Zero risk

### Week 1-2: After DB Verification
4. ⚠️ Day-based limits (~95 lines)
5. ⚠️ Missing config fallback (~15 lines)

**Total**: 110 lines, Low risk (after verification)

### Week 2: Medium-Risk Removals
6. 🔸 Homework functionality (~200 lines)
7. 🔸 Batch saving mechanism (~300 lines)
8. 🔸 Teacher save checks (~15 lines)

**Total**: 515 lines, Medium risk (requires thorough testing)

---

## 📞 Quick Links

- **🎯 For Leadership**: [EXECUTIVE_DECISION_BRIEF.md](EXECUTIVE_DECISION_BRIEF.md) ⭐ **START HERE**
- **🚨 Why This Matters**: [REFACTOR_CONTEXT_DEEP_DIVE.md](REFACTOR_CONTEXT_DEEP_DIVE.md) - The 11 contexts problem
- **📄 Full Assessment**: [REFACTOR_README.md](REFACTOR_README.md)
- **📊 Executive Summary**: [REFACTOR_EXECUTIVE_SUMMARY.md](REFACTOR_EXECUTIVE_SUMMARY.md)
- **🗑️ What to Remove**: [REFACTOR_REDUNDANCY_REPORT.md](REFACTOR_REDUNDANCY_REPORT.md)
- **🗺️ Current Architecture**: [REFACTOR_FUNCTIONAL_AREAS.md](REFACTOR_FUNCTIONAL_AREAS.md)
- **⚙️ Config Analysis**: [REFACTOR_CONFIG_ANALYSIS.md](REFACTOR_CONFIG_ANALYSIS.md)
- **💾 DB Migration**: [REFACTOR_DB_MIGRATION_SPEC.md](REFACTOR_DB_MIGRATION_SPEC.md)
- **🛤️ Long-Term Plan**: [REFACTOR_ROADMAP.md](REFACTOR_ROADMAP.md)

---

## 🚦 Status Indicators

### Phase 0: Cleanup
🟢 **READY** - Can begin after DB verification

### Phase 1: Hooks
🟡 **PENDING** - Waiting for Phase 0 completion

### Phase 2: Components
🟡 **PENDING** - Waiting for Phase 1 completion

### Phase 3: Contexts
🟡 **PENDING** - Waiting for Phase 2 completion

---

## 💡 Key Decisions

### Confirmed Removals
- ✅ Batch saving mechanism
- ✅ Homework functionality
- ✅ Day-based activity limits (currentFreeType: 'days' legacy system)
- ✅ Maintenance mode
- ✅ Commented code
- ✅ Always-false toggle
- ✅ Teacher save prevention

### Confirmed Keeps
- ✅ Per-element saving (current)
- ✅ Activity completion tracking
- ✅ All 10 cohort limit logic
- ✅ period: 'daily' vs 'weekly' (NOT the same as day-based system)
- ✅ isProxyPremiumAccount
- ✅ isGuestAccount handling
- ✅ Display mode switching
- ✅ Exam Q reset functionality

### Root Problem to Fix
- 🚨 **11 contexts with flag-based coordination** - See [Context Deep Dive](REFACTOR_CONTEXT_DEEP_DIVE.md)
- This is why the component is unmaintainable
- Phase 3 (context refactoring) is critical

### Pending Verification
- ⚠️ Zero users with currentFreeType: 'days'?
- ⚠️ Zero users without config?
- ⚠️ Homework isolated to school routes?

---

**Last Updated**: November 20, 2025  
**Assessment Status**: ✅ Complete  
**Implementation Status**: 🟡 Ready to begin after verification

---

## 🚨 Critical Discovery: The Context Problem

After deep analysis, we've identified that **11 contexts with flag-based coordination** is the root architectural problem. 

**Read**: [REFACTOR_CONTEXT_DEEP_DIVE.md](REFACTOR_CONTEXT_DEEP_DIVE.md) to understand:
- Why 11 contexts creates maintenance nightmare
- Example: User clicks submit → 8 steps, 4 flags, 3 useEffects
- Why Phase 3 (context refactoring) is essential

**Leadership decision needed**: [EXECUTIVE_DECISION_BRIEF.md](EXECUTIVE_DECISION_BRIEF.md)

