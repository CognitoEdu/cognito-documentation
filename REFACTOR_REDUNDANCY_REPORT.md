# SubtopicElements Redundancy Report

**Date**: 2025-11-20  
**Component**: SubtopicElements.jsx (6461 lines)  
**Total Lines to Remove**: ~800+ lines (estimated 12-15% reduction)

---

## Category 1: Old Batch Saving Mechanism

### Status: REMOVE COMPLETELY
All users now use per-element saving. Batch saving at activity end is obsolete.

### 1.1 Mutations to Remove

**Line 296-324**: `CREATECOURSEELEMENTRESULTS_MUTATION`
```javascript
const CREATECOURSEELEMENTRESULTS_MUTATION = gql`
	mutation createCourseElementResults(
		$elementResults: [CourseElementResultInput]
		$type: String!
		$isHomework: Boolean
		$homeworkId: String
		$homeworkCode: String
		$dateTimeStarted: Date
		$dateTimeCompleted: Date
		$completeSubtopic: Boolean
		$preciseDuration: Int
		$excludeXP: Boolean  // REMOVE
		$examQs: Boolean
	) { ... }
`;
```
**Reason**: Batches all element results to save at end. Replaced by per-element saves.

**Line 326-350**: `CREATE_COURSE_FLASHCARD_RESULTS_MUTATION`
```javascript
const CREATE_COURSE_FLASHCARD_RESULTS_MUTATION = gql`
	mutation createCourseFlashcardResults(
		$flashcardResults: [CourseFlashcardResultInput]
		$levelOneResponseCount: Int
		$levelTwoResponseCount: Int
		$levelThreeResponseCount: Int
		$dateTimeStarted: Date
		$dateTimeCompleted: Date
		$completeSubtopic: Boolean
		$preciseDuration: Int
		$excludeXP: Boolean  // REMOVE
	) { ... }
`;
```
**Reason**: Batches flashcard results at end. Replaced by per-flashcard saves.

### 1.2 Hooks to Remove

**Line 1845-1846**: `createCourseElementResults` hook declaration
**Line 1848-1852**: `createCourseFlashcardResults` hook declaration

### 1.3 State Variables to Remove

**Line 704**: `const [saveResults, setSaveResults] = useState(false);`
- Used to trigger batch save at activity end

**Line 749**: `const [continueToSave, setContinueToSave] = useState(true);`
- Comment says: "only used to stop homework results being calculated before the last result is saved away"
- Tied to homework batch timing

**Line 782-783**: `const [courseElementResultsToSave, setCourseElementResultsToSave] = useState(null);`
- Queues batch results for homework

### 1.4 Logic Blocks to Remove

**Lines 966-1031**: Flashcard completion batch save logic
```javascript
useEffect(() => {
	if (flashcardsComplete) {
		// ... calculates summary ...
		if (!loginContext.isGuestAccount && anyElementSubmitted) {
			createCourseFlashcardResults({ // REMOVE THIS CALL
				variables: {
					flashcardResults: flashcardResults,
					// ...
					excludeXP: /* condition */ // REMOVE
				},
			});
		}
	}
}, [flashcardsComplete]);
```

**Lines 3382-3600**: Entire batch save useEffect
```javascript
useEffect(() => {
	if (saveResults && continueToSave) {
		setSaveResults(false);
		const elementResults = [];
		// ... gathers all element results ...
		// Lines 3386-3551: Complex result gathering logic
		
		if (newElementResults.length > 0) {
			const variables = {
				// ...
				excludeXP: /* condition */, // REMOVE
			};
			
			if (!homework) {
				createCourseElementResults({ variables }); // REMOVE
			} else {
				setCourseElementResultsToSave(variables); // REMOVE
			}
		}
	}
}, [saveResults, continueToSave]);
```
**Reason**: Entire block gathers and batch-saves results. Obsolete with per-element saves.

**Lines 3602-3609**: Homework batch save coordination
```javascript
useEffect(() => {
	if (!savingElementResult && courseElementResultsToSave) {
		createCourseElementResults({ variables: courseElementResultsToSave });
		setCourseElementResultsToSave(null);
	}
}, [savingElementResult, courseElementResultsToSave]);
```

**Lines 2716-2719**: Homework continue-to-save coordination
```javascript
useEffect(() => {
	if (saveCourseElementResultData && !continueToSave) {
		setContinueToSave(true);
	}
}, [saveCourseElementResultData, continueToSave]);
```

**Lines 3676-3688**: Batch save completion handler
```javascript
useEffect(() => {
	if (createCourseElementResultsData || maintenanceModeOverride) {
		setMaintenanceModeOverride(false);
		setNavigationContext((navigationContext) => ({
			...navigationContext,
			refreshCourseResults: true,
		}));
	}
}, [createCourseElementResultsData, maintenanceModeOverride]);
```

**Lines 3697-3704**: Flashcard batch save completion handler  
```javascript
useEffect(() => {
	if (createCourseFlashcardResultsData || maintenanceModeOverride) {
		setMaintenanceModeOverride(false);
		setNavigationContext((navigationContext) => ({
			...navigationContext,
			refreshCourseResults: true,
		}));
	}
}, [createCourseFlashcardResultsData, maintenanceModeOverride]);
```

### 1.5 Parameter Usage to Remove

**Lines 1012-1016**: `excludeXP` in flashcard batch save
**Lines 3569-3573**: `excludeXP` in element batch save
**Lines 307, 320**: `excludeXP` in CREATECOURSEELEMENTRESULTS mutation
**Lines 336, 347**: `excludeXP` in CREATE_COURSE_FLASHCARD_RESULTS mutation

**Reason**: `excludeXP` prevents XP award when:
- Not proxy premium AND not guest AND showAllElements AND (!activityComplete OR activityCompleteDialogShown)
- This condition only matters for batch saves at activity end. Per-element saves don't need this.

### 1.6 Related Cleanup

**Lines 2220-2221**: `setContinueToSave(false)` in homework logic
**Line 2243**: `let continueToSaveResult = true;` local variable
**Line 2262**: `continueToSaveResult = false;`
**Line 2325**: `if (continueToSaveResult)` check
**Line 3660**: `setContinueToSave(true);` in saveActivity

---

## Category 2: Homework Functionality

### Status: REMOVE COMPLETELY (88 occurrences)
Homework is isolated to school platform routes. Not used in these activity components.

### 2.1 Props to Remove (Line 454-515)

From function signature:
- `homeworkPreview` (line 472)
- `setHomeworkElements` (line 473)
- `confirmHomework` (line 474)
- `homework` (line 475)
- `homeworkElements` (line 476)
- `hideHints` (line 477)
- `homeworkId` (line 478)
- `homeworkCode` (line 479)
- `hideMainButton` (line 480)
- `homeworkName` (line 482)
- `resumedHomeworkCode` (line 504)

Also in ElementCard.jsx: `homework`, `hideHints`

### 2.2 GraphQL Queries to Remove

**Lines 157-165**: `RESUMED_COURSEELEMENTS_HOMEWORK_QUERY`
```javascript
export const RESUMED_COURSEELEMENTS_HOMEWORK_QUERY = gql`
  query resumedHomeworkElements($homeworkCode: String!, $resultType: Int, $hash: String!) {
    resumedHomeworkElements(homeworkCode: $homeworkCode,resultType: $resultType, hash: $hash) {
      courseSubtopicCode,
      singleSubtopic
      ${ELEMENT_FIELDS}      
    }
  }
`;
```

**Lines 223-230**: `COURSEREVISEELEMENTS_HOMEWORK_QUERY`
```javascript
const COURSEREVISEELEMENTS_HOMEWORK_QUERY = gql`
  query courseReviseElementsHomework($courseCode: String!, $subjectCode: String!, $elementIds: [String], $subtopics: [String], $homeworkCode: String!, $readOnly: Boolean) {
    courseReviseElementsHomework(...) {
      courseSubtopicCode,
      ${ELEMENT_FIELDS}
    }
  }
`;
```

### 2.3 State Variables to Remove

**Line 714-720**: `elementIds` state (homework-specific)
```javascript
const tempElementIds = [];
if (homeworkElements) {
	for (let i = 0; i < homeworkElements.length; i++) {
		tempElementIds.push(homeworkElements[i].elementId);
	}
}
const [elementIds] = useState(tempElementIds);
```

**Line 721**: `const [homeworkReadyToConfirm, setHomeworkReadyToConfirm] = useState(false);`

### 2.4 Conditional Logic to Remove

**Lines 907-908**: `if (homework || homeworkPreview) { setShowAllElementsToggle(false); }`

**Lines 2219-2240**: Homework per-element save logic
```javascript
if (homework) {
	if (elementCount === elements.length) {
		setContinueToSave(false);
	}
	if (!preview) {
		saveCourseElementResult({
			variables: {
				elementResults: elementResults,
				homeworkCode: homeworkCode, // REMOVE
				preciseDuration: preciseDuration,
				showAllElements: showAllElements,
				examQ: examQs ? true : false,
			},
		});
	}
}
```

**Lines 3560-3583**: Homework batch save conditional
```javascript
isHomework: homework,
homeworkId: homeworkId,
homeworkCode: homeworkCode,
// ...
if (!homework) {
	createCourseElementResults({ variables });
} else {
	setCourseElementResultsToSave(variables);
}
```

**Lines 3499**: `if (proxyAsPremiumUser && !homework)`

**Lines 3742-3744, 3758-3760**: Homework query selection
```javascript
if (resumedCourseCode) {
	if (homework) {
		query = RESUMED_COURSEELEMENTS_HOMEWORK_QUERY;
	}
}
```

**Lines 3803-3807, 3850-3854, 3865-3874**: Homework query variables

**Lines 4102-4120**: Homework element gathering for preview

**Lines 4125-4127**: `if (homeworkPreview) { setHomeworkReadyToConfirm(true); }`

**Lines 4136-4141**: Homework resume logic

**Lines 4155-4156**: Homework query selection

**Lines 4193-4202**: Homework preview element gathering

**Lines 4788-4789**: `if (homework || homeworkPreview) { setShowAllElements(false); }`

**Lines 4820**: Homework in dependencies

**Lines 5054-5056**: `hideVideoLinks={(homework && hideVideoLinks) || ...}`

**Lines 5057**: `hideHints={homework && hideHints}`

**Lines 6288-6314**: Homework confirm button footer
```javascript
{homeworkPreview && !hideMainButton && !teacherPreview && !showAllElements ? (
	<div className={...}>
		<Button
			value={'confirmHomework'}
			disabled={!homeworkReadyToConfirm}
			onClick={confirmHomework}
		>
			<Typography>Confirm</Typography>
		</Button>
	</div>
) : null}
```

**Line 4609**: `isHomework={homework}` prop

### 2.5 Query Variable Removal

Throughout `fetchElements()` function (lines 3777-3922):
- Remove `homeworkCode` from variables
- Remove homework query selection branches
- Remove `readOnly: homeworkPreview` parameters
- Remove `elementIds` variable usage (homework-only)

---

## Category 3: Legacy Activity Limits

### Status: REMOVE SPECIFIC BRANCHES
Based on activity-allowance-config.js, certain restriction patterns are no longer used.

### 3.1 Day-Based Allowance System (REMOVE - 0 cohorts use)

**Line 755**: `const [freeDayAllowance, setFreeDayAllowance] = useState(false);`

**Lines 1332-1369**: Entire day-based calculation branch
```javascript
else if (loginContext.activityAllowance.currentFreeType === 'days') {
	const signUpDateAsString = loginContext.activityAllowance.signUpDateAsString;
	const todaysDateAsString = loginContext.activityAllowance.todaysDateAsString;
	const differenceInDays = getDifferenceInDays(todaysDateAsString, signUpDateAsString);
	
	let fieldName = null;
	if (loginContext.activityAllowanceConfig?.restrictAcrossAllActivities) {
		fieldName = 'currentFreeAllowanceTotal';
	} else {
		if (type === 'course') {
			fieldName = 'currentFreeAllowanceLessonElements';
		} else if (type === 'courseflashcards') {
			fieldName = 'currentFreeAllowanceFlashcards';
		} else if (type === 'courserevise') {
			fieldName = 'currentFreeAllowanceQuizElements';
		} else if (examQs) {
			if (examQType === 'examQs') {
				fieldName = 'currentFreeAllowanceExamQParts';
			} else if (examQType === 'examMCQs') {
				fieldName = 'currentFreeAllowanceExamMCQs';
			}
		}
	}
	
	if (differenceInDays < loginContext.activityAllowance[fieldName]) {
		setFreeDayAllowance(true);
	} else {
		setFreeDayAllowance(false);
	}
}
```
**Reason**: No cohorts use `currentFreeType: 'days'`. All use `currentFreeType: 'elements'`.

**Lines 1396-1413**: `getDifferenceInDays()` utility function
```javascript
function getDifferenceInDays(date1, date2) {
	const year1 = date1.substring(0, 4);
	const month1 = date1.substring(4, 6);
	const day1 = date1.substring(6, 8);
	
	const year2 = date2.substring(0, 4);
	const month2 = date2.substring(4, 6);
	const day2 = date2.substring(6, 8);
	
	const date1AsDate = new Date(year1, month1 - 1, day1);
	const date2AsDate = new Date(year2, month2 - 1, day2);
	
	const differenceInMilliseconds = date1AsDate - date2AsDate;
	const differenceInDays = differenceInMilliseconds / (1000 * 60 * 60 * 24);
	
	return Math.ceil(differenceInDays);
}
```

**Line 1432**: `if (freeDayAllowance) { usingFreeAllowance = true; }`
- Part of complex allowance calculation

### 3.2 Missing Config Fallback (REMOVE - all users have config)

**Lines 1542-1550**: Fallback for users without `activityAllowanceConfig`
```javascript
if (!loginContext.activityAllowanceConfig) {
	if (
		(numberOfNonFreeScoringComplete || 0) +
			(currentDailyElementCount || 0) >=
		maxActivityAllowance
	) {
		setMaxDailyActivityJustReached(true);
		setMaxDailyActivityReached(true);
	}
}
```
**Reason**: All users now have `activityAllowanceConfig` via cohort assignment. Fallback unnecessary.

**Impacts**: 
- Remove fallback to `ActivityAccess.MAX_DAILY_ACTIVITY_ACCESS` (line 1525)
- Simplify lines 1551-1574 (daily period check) 
- Simplify lines 1561-1570 (weekly period check)

### 3.3 Non-Existent Config Variations (VERIFY & REMOVE)

**`includeLessons: false` checks**: 
All cohorts have `includeLessons: true`. Search for:
- `!loginContext.activityAllowanceConfig?.includeLessons`
- `includeLessons === false`
- Any logic assuming lessons are excluded

**Current config matrix**:
```
All 9 cohorts + guest: includeLessons: true
No cohorts have: includeLessons: false
```

### 3.4 Config Branches to KEEP

**period**:
- `'daily'`: Cohorts 1-3, guest ✓
- `'weekly'`: Cohorts 4-9 ✓

**restrictAcrossAllActivities**:
- `true`: Cohorts 3, 5, 6, 7 ✓
- `false`: Cohorts 1, 2, 4, 8, 9, guest ✓

**restrictByActivity**:
- `true`: Cohorts 6, 7, 8, 9 ✓
- `false` or undefined: Cohorts 1-5, guest ✓

**Element/Activity limits**: Keep all tracking for current values (10, 15, 25, 50 elements; 1, 3, 5, 7, 10 activities)

---

## Category 4: Other Redundant Code

### 4.1 Maintenance Mode (REMOVE - Never Used)

**Line 724-726**: State initialization
```javascript
const [maintenanceMode] = useState(
	window && window.env && window.env.MAINTENANCE_MODE
);
```

**Line 727**: `const [maintenanceModeOverride, setMaintenanceModeOverride] = useState(false);`

**Line 3483-3485**: Single usage
```javascript
if (maintenanceMode) {
	setMaintenanceModeOverride(true);
}
```

**Lines 3676-3677, 3697-3698**: Used in batch save handlers (already being removed)

**Reason**: Never actually enabled in production. Feature not used.

### 4.2 Commented Out Code (REMOVE)

**Lines 4322-4373**: Commented `handleRedoHard()` function
```javascript
// const handleRedoHard = () => {
// 	let courseReviseElements = ...
//  ... 52 lines of commented code ...
// };
```

**Lines 4629-4662**: Commented `flashcardsCompleteDialog` JSX
```javascript
// const flashcardsCompleteDialog = (
// 	<Dialog ... >
//  ... 34 lines of commented JSX ...
// 	</Dialog>
// );
```

**Line 3260-3266, 3285-3291, 3306-3312, 3341-3347, 3355-3361**: Multiple commented `setSaveResultDetails` calls

**Line 6396-6397**: Commented dialog references

### 4.3 Always-False Toggle (REMOVE)

**Lines 906-926**: `showAllElementsToggle` logic
```javascript
useEffect(() => {
	if (homework || homeworkPreview) {
		setShowAllElementsToggle(false);
	} else if (examQs) {
		setShowAllElementsToggle(false);
	} else if (
		type === 'course' &&
		(levelCode === 'alevel' || levelCode === 'ks3') &&
		!examQs
	) {
		setShowAllElementsToggle(false);
	} else if (resumedCourseCode) {
		if (type === 'courseflashcards' && resultType === 'restart') {
			setShowAllElementsToggle(false);
		} else {
			setShowAllElementsToggle(false);
		}
	} else {
		setShowAllElementsToggle(false); // Always false
	}
}, [elementType, examQs, resumedCourseCode, homework, homeworkPreview]);
```
**Reason**: Comments indicate this was changed to always be false. Variable serves no purpose.

**Line 745**: `const [showAllElementsToggle, setShowAllElementsToggle] = useState(false);`

### 4.4 Teacher Save Prevention (REMOVE)

**Lines 956-963**: `isTeacher` calculation for save prevention
```javascript
let isTeacher = false;
if (
	loginContext &&
	loginContext.roleCodes &&
	loginContext.roleCodes.includes('teacher')
) {
	isTeacher = true;
}
```

**Usage to remove**:
- Line 985: `if (!isTeacher)` before calling completePremiumActivity
- Line 3500: `if (!isTeacher)` before save logic
- Line 3518: `if (!isTeacher)` before save
- Line 3657: `if (!isTeacher)` before markPremiumUserCourseAsComplete
- Line 4952: `isTeacher={isTeacher}` prop pass

**Reason**: Teachers can now complete activities. Save prevention no longer needed.

### 4.5 Additional Cleanup

**Line 3683-3686**: Commented out COGMAIN-655 code block
```javascript
/* COGMAIN-655
setSavingResults(false);
setSubtopicCompleteDialogOpen(true);      
*/
```

---

## Summary Statistics

### Lines to Remove by Category

| Category | Line Count | Percentage |
|----------|-----------|------------|
| Batch Saving | ~300 lines | 4.6% |
| Homework | ~200 lines | 3.1% |
| Legacy Limits | ~150 lines | 2.3% |
| Maintenance Mode | ~10 lines | 0.2% |
| Commented Code | ~120 lines | 1.9% |
| Always-False Toggle | ~25 lines | 0.4% |
| Teacher Checks | ~15 lines | 0.2% |
| **TOTAL** | **~820 lines** | **12.7%** |

### Impact on File Size

- **Current**: 6461 lines
- **After cleanup**: ~5640 lines (estimated)
- **Reduction**: 820 lines (12.7%)

### Mutations Impact

**Remove 2 mutations**: 
- CREATE_COURSE_FLASHCARD_RESULTS_MUTATION
- CREATECOURSEELEMENTRESULTS_MUTATION

**Keep 3 mutations**:
- SAVECOURSEELEMENTRESULT_MUTATION (per-element)
- SAVECOURSEFLASHCARDRESULT_MUTATION (per-flashcard)
- MARK_PREMIUM_USER_COURSE_COMPLETE_MUTATION (activity summary)

### Props Impact

**Remove 11 homework props** from function signature (9% of props)

**Remaining props**: ~43 props (many candidates for further refactoring in Phase 2)

---

## Next Steps

1. Execute removals in safe order:
   - Remove commented code (safest)
   - Remove maintenance mode
   - Remove homework functionality  
   - Remove day-based allowance
   - Remove batch saving mechanism
   - Remove teacher save prevention

2. Run tests after each category removal

3. Update prop types and component documentation

4. Proceed to Phase 2: Functional areas analysis

