# Activity Allowance Config Analysis

**Date**: 2025-11-20  
**Purpose**: Determine which activity limit code can be safely removed based on current cohort configurations

---

## Current Production Configurations

### Source: `common-server/_helpers/users/activity-allowance-config.js`

### Guest Users
```javascript
{
	id: 'guest_daily_0_1a_10_with_lessons',
	period: 'daily',
	initialFreeAllowance: 0,
	ongoingFreeAllowanceActivity: 1,
	ongoingFreeAllowanceElements: 10,
	includeLessons: true,
	restrictAcrossAllActivities: false,
	// restrictByActivity: undefined (falsy)
}
```

### Cohort 1: Daily 75e/1a/15 with lessons
```javascript
{
	id: 'daily_75e_1a_15_with_lessons',
	period: 'daily',
	initialFreeAllowance: 75,
	initialFreeType: 'elements',
	ongoingFreeAllowanceActivity: 1,
	ongoingFreeAllowanceElements: 15,
	includeLessons: true,
	restrictAcrossAllActivities: false,
	restrictByActivity: false,
}
```

### Cohort 2: Daily 75e/1a/10 with lessons
```javascript
{
	id: 'daily_75e_1a_10_with_lessons',
	period: 'daily',
	initialFreeAllowance: 75,
	initialFreeType: 'elements',
	ongoingFreeAllowanceActivity: 1,
	ongoingFreeAllowanceElements: 10,
	includeLessons: true,
	restrictAcrossAllActivities: false,
	restrictByActivity: false,
}
```

### Cohort 3: Daily 75e/1a/15 across all activities
```javascript
{
	id: 'daily_75e_1a_15_across_all_activities_with_lessons',
	period: 'daily',
	initialFreeAllowance: 75,
	initialFreeType: 'elements',
	ongoingFreeAllowanceActivity: 1,
	ongoingFreeAllowanceElements: 15,
	includeLessons: true,
	restrictAcrossAllActivities: true,
	restrictByActivity: false,
}
```

### Cohort 4: Weekly 50e/25 with lessons
```javascript
{
	id: 'weekly_50e_25_with_lessons',
	period: 'weekly',
	initialFreeAllowance: 50,
	initialFreeType: 'elements',
	ongoingFreeAllowanceElements: 25,
	// ongoingFreeAllowanceActivity: undefined (no activity limit)
	includeLessons: true,
	restrictAcrossAllActivities: false,
	restrictByActivity: false,
}
```

### Cohort 5: Weekly 50e/50 across all activities
```javascript
{
	id: 'weekly_50e_50_across_all_activities_with_lessons',
	period: 'weekly',
	initialFreeAllowance: 50,
	initialFreeType: 'elements',
	ongoingFreeAllowanceElements: 50,
	// ongoingFreeAllowanceActivity: undefined
	includeLessons: true,
	restrictAcrossAllActivities: true,
	restrictByActivity: false,
}
```

### Cohort 6: Weekly 7a across all activities
```javascript
{
	id: 'weekly_7a_across_all_activities_with_lessons',
	period: 'weekly',
	initialFreeAllowance: 0,
	initialFreeType: 'elements',
	ongoingFreeAllowanceElements: 10,
	ongoingFreeAllowanceActivity: 7,
	includeLessons: true,
	restrictAcrossAllActivities: true,
	restrictByActivity: true,
}
```

### Cohort 7: Weekly 10a across all activities
```javascript
{
	id: 'weekly_10a_across_all_activities_with_lessons',
	period: 'weekly',
	initialFreeAllowance: 0,
	initialFreeType: 'elements',
	ongoingFreeAllowanceElements: 10,
	ongoingFreeAllowanceActivity: 10,
	includeLessons: true,
	restrictAcrossAllActivities: true,
	restrictByActivity: true,
}
```

### Cohort 8: Weekly 3a with lessons
```javascript
{
	id: 'weekly_3a_with_lessons',
	period: 'weekly',
	initialFreeAllowance: 0,
	initialFreeType: 'elements',
	ongoingFreeAllowanceElements: 10,
	ongoingFreeAllowanceActivity: 3,
	includeLessons: true,
	restrictAcrossAllActivities: false,
	restrictByActivity: true,
}
```

### Cohort 9: Weekly 5a with lessons
```javascript
{
	id: 'weekly_5a_with_lessons',
	period: 'weekly',
	initialFreeAllowance: 0,
	initialFreeType: 'elements',
	ongoingFreeAllowanceElements: 10,
	ongoingFreeAllowanceActivity: 5,
	includeLessons: true,
	restrictAcrossAllActivities: false,
	restrictByActivity: true,
}
```

---

## Configuration Matrix

| Cohort | period | initialFreeType | restrictAcross | restrictBy | includeLessons |
|--------|--------|----------------|----------------|------------|----------------|
| Guest | daily | N/A (0) | false | false/undef | true |
| 1 | daily | elements | false | false | true |
| 2 | daily | elements | false | false | true |
| 3 | daily | elements | **true** | false | true |
| 4 | weekly | elements | false | false | true |
| 5 | weekly | elements | **true** | false | true |
| 6 | weekly | elements | **true** | **true** | true |
| 7 | weekly | elements | **true** | **true** | true |
| 8 | weekly | elements | false | **true** | true |
| 9 | weekly | elements | false | **true** | true |

### Allowance Values in Use

**initialFreeAllowance**: 0, 50, 75  
**ongoingFreeAllowanceElements**: 10, 15, 25, 50  
**ongoingFreeAllowanceActivity**: 1, 3, 5, 7, 10, undefined

---

## Code Analysis: Can Remove

### ❌ REMOVE: Day-Based Allowance System

**NO cohorts have**: `currentFreeType: 'days'`  
**ALL cohorts have**: `initialFreeType: 'elements'` or `initialFreeAllowance: 0`

**Lines to remove in SubtopicElements.jsx**:

**1332-1369**: Entire `currentFreeType === 'days'` branch
```javascript
} else if (loginContext.activityAllowance.currentFreeType === 'days') {
	const signUpDateAsString = loginContext.activityAllowance.signUpDateAsString;
	const todaysDateAsString = loginContext.activityAllowance.todaysDateAsString;
	const differenceInDays = getDifferenceInDays(todaysDateAsString, signUpDateAsString);
	
	let fieldName = null;
	// ... 30+ lines of field name determination ...
	
	if (differenceInDays < loginContext.activityAllowance[fieldName]) {
		setFreeDayAllowance(true);
	} else {
		setFreeDayAllowance(false);
	}
}
```

**1396-1413**: `getDifferenceInDays()` function

**755**: `const [freeDayAllowance, setFreeDayAllowance] = useState(false);`

**1432**: `if (freeDayAllowance) { usingFreeAllowance = true; }`

### ❌ REMOVE: Missing Config Fallback

**ALL users have**: `activityAllowanceConfig` assigned via cohort

**Lines to remove**:

**1542-1550**: Fallback for `!loginContext.activityAllowanceConfig`
```javascript
if (!loginContext.activityAllowanceConfig) {
	if (
		(numberOfNonFreeScoringComplete || 0) + (currentDailyElementCount || 0) >= maxActivityAllowance
	) {
		setMaxDailyActivityJustReached(true);
		setMaxDailyActivityReached(true);
	}
}
```

**1525**: Fallback to `ActivityAccess.MAX_DAILY_ACTIVITY_ACCESS` no longer needed

**Impact**: Simplifies lines 1519-1574 (maxActivitiesReached logic)

### ❌ REMOVE: includeLessons === false

**ALL cohorts have**: `includeLessons: true`

**Search for and remove**:
- Any checks for `!loginContext.activityAllowanceConfig?.includeLessons`
- Any checks for `includeLessons === false`
- Dead code branches assuming lessons excluded

**Currently used**: All checks assume `includeLessons: true` ✓

---

## Code Analysis: Must Keep

### ✅ KEEP: period (daily vs weekly)

**Used by**:
- Daily: Guest, Cohorts 1-3 (4 configs)
- Weekly: Cohorts 4-9 (6 configs)

**Code locations**: Lines 1551-1574

### ✅ KEEP: restrictAcrossAllActivities

**Used by**:
- `true`: Cohorts 3, 5, 6, 7 (4 configs)
- `false`: Guest, Cohorts 1, 2, 4, 8, 9 (6 configs)

**Purpose**: When true, limits apply to total across all activity types. When false, limits per activity type.

**Code locations**: Lines 1199, 1205, 1246, 1252, 1292, 1299, 1306, 1314, 1322, 1345, 1465, 1470, 1494, 1500, 1504

### ✅ KEEP: restrictByActivity

**Used by**:
- `true`: Cohorts 6, 7, 8, 9 (4 configs)
- `false` or undefined: Guest, Cohorts 1-5 (6 configs)

**Purpose**: When true with restrictAcrossAllActivities, uses activity count limits instead of element limits.

**Code locations**: Lines 1220, 1228, 1266, 1274, 1529

**Special case** (lines 1529-1533):
```javascript
if (
	loginContext.activityAllowanceConfig.restrictByActivity &&
	!examQs
) {
	// We only care about activities, not elements
	maxActivityAllowance = 1;
}
```

### ✅ KEEP: All Tracking States

Required for current cohort functionality:
- `currentDailyElementCount`
- `currentWeeklyElementCount`
- `freeElementAllowance`
- `maxDailyActivityReached`
- `maxDailyActivityJustReached`
- `numberOfCompletedElements`
- `numberOfCompletedScoringElements`
- `completedElements`
- `freeAllowancePeriod`

### ✅ KEEP: Allowance Value Ranges

**Element limits**: 10, 15, 25, 50  
**Activity limits**: 1, 3, 5, 7, 10, undefined (unlimited activities, element-limited only)  
**Initial allowances**: 0, 50, 75

---

## Special Cases Analysis

### Case 1: restrictByActivity = true

**Cohorts**: 6, 7, 8, 9

**Behavior**: Uses activity count limits, not element limits
- Cohort 6: 7 activities/week across all types
- Cohort 7: 10 activities/week across all types
- Cohort 8: 3 activities/week (lessons, quiz, flashcards only)
- Cohort 9: 5 activities/week (lessons, quiz, flashcards only)

**Code implications**:
- Lines 1529-1533: Set `maxActivityAllowance = 1` when `restrictByActivity && !examQs`
- Lines 1220, 1228: `restrictByActivity` excludes exam qs from total element count
- Lines 1266, 1274: Same for weekly

**Exam Q exception**: When `restrictByActivity: true` AND `restrictAcrossAllActivities: false` (cohorts 8, 9), exam qs are NOT restricted by activity limit.

### Case 2: restrictAcrossAllActivities + restrictByActivity

**Cohorts**: 6, 7 (both true)

**Behavior**: Single activity count pool across lessons, quizzes, flashcards, AND exam qs
- Lines 1218-1231: Exam Qs included in total when both flags true

### Case 3: ongoingFreeAllowanceActivity undefined

**Cohorts**: 4, 5 (element-only limits)

**Behavior**: No activity count limit, only element limits
- Lines 1456-1487: Check for `ongoingFreeAllowanceActivity` existence
- When undefined, `maxActivitiesReached = true` (lines 1512-1514) forces element-only checking

---

## Removable vs Required Code

### ✅ Safe to Remove

1. **Day-based allowance** (~80 lines)
   - `currentFreeType === 'days'` branch
   - `getDifferenceInDays()` function
   - `freeDayAllowance` state
   - `signUpDateAsString` / `todaysDateAsString` usage

2. **Missing config fallback** (~15 lines)
   - `!loginContext.activityAllowanceConfig` branch
   - Fallback to `ActivityAccess.MAX_DAILY_ACTIVITY_ACCESS`

3. **includeLessons: false handling** (~0 lines)
   - No code found checking for false
   - All checks assume true ✓

### ⚠️ Must Keep (Required by Cohorts)

1. **period switching** (daily/weekly)
2. **restrictAcrossAllActivities** (true/false)
3. **restrictByActivity** (true/false/undefined)
4. **Element count tracking**
5. **Activity count tracking**
6. **All current allowance value ranges**

---

## Frontend Code Cross-Reference

### Lines Using Removable Config

**Day-based system**:
- 755: State declaration
- 1332-1369: Main conditional block
- 1396-1413: Utility function
- 1432: Usage in allowance calculation

**Missing config fallback**:
- 1542-1550: Conditional block
- 1525: MAX_DAILY_ACTIVITY_ACCESS reference

**Total removable**: ~95 lines from activity limit code

### Lines Using Required Config

**Daily/Weekly period**:
- 1384-1394: Period detection and state setting
- 1551-1574: Period-specific maxActivityReached logic

**restrictAcrossAllActivities**:
- 1199-1214: Daily element count per activity type vs total
- 1245-1282: Weekly element count per activity type vs total
- 1292-1330: Free element allowance calculation
- 1345-1363: Fallback element allowance (within day-based block - REMOVE block)
- 1465-1478: Daily activity count total vs per-type
- 1494-1507: Weekly activity count total vs per-type

**restrictByActivity**:
- 1218-1233: Exam Q element count conditional
- 1264-1279: Exam Q element count conditional (weekly)
- 1529-1540: Activity vs element restriction logic

**ongoingFreeAllowanceActivity**:
- 1456: Existence check
- 1482: Comparison for daily
- 1511: Comparison for weekly

**ongoingFreeAllowanceElements**:
- 1536-1540: Direct usage as maxActivityAllowance

---

## DB Migration Requirements

### Prerequisite for Code Removal

**Action Required**: Ensure ALL non-premium users have one of these 10 configs:
- `guest_daily_0_1a_10_with_lessons`
- `daily_75e_1a_15_with_lessons`
- `daily_75e_1a_10_with_lessons`
- `daily_75e_1a_15_across_all_activities_with_lessons`
- `weekly_50e_25_with_lessons`
- `weekly_50e_50_across_all_activities_with_lessons`
- `weekly_7a_across_all_activities_with_lessons`
- `weekly_10a_across_all_activities_with_lessons`
- `weekly_3a_with_lessons`
- `weekly_5a_with_lessons`

### Migration Script Needed

**Location**: `common-server/scripts/migrate-activity-allowance-configs.js`

**Actions**:
1. Find all users with `activityAllowance.currentFreeType === 'days'`
2. Convert to appropriate element-based cohort
3. Find all users without `activityAllowanceConfig`
4. Assign default cohort based on signup date/account age
5. Find all users with `includeLessons: false`
6. Migrate to `includeLessons: true` cohort

**Query**:
```javascript
// Users with day-based system
db.users.find({
	'activityAllowance.currentFreeType': 'days',
	isPremiumAccount: false
})

// Users without config
db.users.find({
	activityAllowanceConfig: { $exists: false },
	isPremiumAccount: false
})

// Users with lessons excluded (if any)
db.users.find({
	'activityAllowanceConfig.includeLessons': false,
	isPremiumAccount: false
})
```

**Default migration**:
- Older users (signup > 30 days ago) → `weekly_3a_with_lessons` (most restrictive ongoing)
- Newer users (signup ≤ 30 days) → `daily_75e_1a_15_with_lessons` (generous trial)
- Guest users → `guest_daily_0_1a_10_with_lessons` (already handled)

---

## Code Removal Impact

### Before Removal
- Activity limit code: ~400 lines
- Legacy branches: ~95 lines (day-based + fallback)

### After Removal
- Activity limit code: ~305 lines
- Reduction: 24% of limit code

### Remaining Complexity

**Still complex** (but required):
- 4 `restrictAcrossAllActivities` values × 2 periods = 8 branches
- 3 `restrictByActivity` values (true/false/undefined) = 3 branches
- Activity vs element limit switching
- Exam Q special handling with `restrictByActivity`

**Potential simplification**: Consider standardizing all cohorts to `restrictByActivity: false` (explicit) instead of undefined.

---

## Recommendations

### Immediate Actions

1. ✅ **Remove day-based allowance code** (95 lines)
   - Zero cohorts use this
   - No DB migration needed for removal
   - Safe immediate removal

2. ⚠️ **Remove missing config fallback** after DB verification
   - Check: Any users without `activityAllowanceConfig`?
   - If zero found: Safe to remove
   - If any found: Run migration script first

3. ✅ **Confirmed: Keep all other limit code**
   - Required by current cohorts
   - Cannot simplify without changing cohort configs

### Future Simplifications

**If cohort consolidation is acceptable**:
1. Standardize to 3-4 cohorts instead of 9
2. Remove `restrictByActivity` complexity (use element limits only)
3. Consider daily-only (remove weekly complexity)

**Estimated savings**: Additional ~150 lines if cohorts standardized

### Testing Requirements

After removal:
1. Test guest user limits (1 activity, 10 elements daily)
2. Test restrictAcrossAllActivities: true cohorts (3, 5, 6, 7)
3. Test restrictByActivity: true cohorts (6, 7, 8, 9)
4. Test weekly limits (cohorts 4-9)
5. Test exam Q special handling (cohorts 6, 7)

---

## Summary

### Can Remove Immediately
- Day-based allowance system: **95 lines**
- Total with homework + batch saving: **~820 lines**

### Must Keep
- All other activity limit logic: **~305 lines**
- Required by 10 active cohort configurations

### DB Migration Required
- Verify zero users have: `currentFreeType: 'days'`
- Verify zero users without `activityAllowanceConfig`
- Create migration script for any stragglers

### Net Result
From 6461 lines → ~5545 lines after all Phase 1 removals (~14% reduction)

