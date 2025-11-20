# Phase 0 Cleanup - Implementation Checklist

**Component**: SubtopicElements.jsx  
**Total Removals**: 820+ lines (14%)  
**Risk Level**: Low to Medium  
**Duration**: 1-2 weeks

---

## Removal Order (Safest First)

Execute in this order to minimize risk and enable easy rollback if issues arise.

---

## ✅ Step 1: Remove Commented Code (Zero Risk)

**Lines to delete**: ~120 lines  
**Risk**: None (already dead code)  
**Testing**: None needed

### 1.1 Remove Commented Function

**Lines 4322-4373** - Commented `handleRedoHard()` function
```javascript
// Delete entire block:
// const handleRedoHard = () => {
// 	let courseReviseElements =
// 		data.courseFlashcards || data.resumedCourseFlashcards;
//  ... 52 lines ...
// };
```

### 1.2 Remove Commented JSX

**Lines 4629-4662** - Commented `flashcardsCompleteDialog`
```javascript
// Delete entire block:
// const flashcardsCompleteDialog = (
// 	<Dialog
//  ... 34 lines ...
// 	</Dialog>
// );
```

### 1.3 Remove Commented Calls

**Multiple locations**: Commented `setSaveResultDetails` calls
- Lines 3260-3266
- Lines 3285-3291
- Lines 3306-3312
- Lines 3341-3347
- Lines 3355-3361

```javascript
// Delete blocks like:
/*setSaveResultDetails({
	showNextElement: true,
	saveCurrentActivity: false,
	currentElement: currentElement,
	submitAllParts: false,
	continueActivity: null,
});*/
```

### 1.4 Remove COGMAIN Commented Code

**Lines 3683-3686**
```javascript
// Delete:
/* COGMAIN-655
setSavingResults(false);
setSubtopicCompleteDialogOpen(true);      
*/
```

### 1.5 Remove Commented Dialog References

**Lines 6396-6397**
```javascript
// Delete:
{/* {subtopicCompleteDialog}
	{flashcardsCompleteDialog} */}
```

**After Step 1**:
- Run tests
- Verify build successful
- Commit: `refactor: remove commented code from SubtopicElements`

---

## ✅ Step 2: Remove Maintenance Mode (Zero Risk)

**Lines to delete**: ~10 lines  
**Risk**: None (never enabled)  
**Testing**: Quick smoke test

### 2.1 Remove State

**Lines 724-727**
```javascript
// Delete:
const [maintenanceMode] = useState(
	window && window.env && window.env.MAINTENANCE_MODE
);
const [maintenanceModeOverride, setMaintenanceModeOverride] = useState(false);
```

### 2.2 Remove Usage

**Lines 3483-3485**
```javascript
// Delete:
if (maintenanceMode) {
	setMaintenanceModeOverride(true);
}
```

### 2.3 Update useEffect Dependencies

**Line 3688** - Remove from dependency array:
```javascript
// Change:
}, [createCourseElementResultsData, maintenanceModeOverride]);

// To:
}, [createCourseElementResultsData]);
```

**Line 3704** - Remove from dependency array:
```javascript
// Change:
}, [createCourseFlashcardResultsData, maintenanceModeOverride]);

// To:
}, [createCourseFlashcardResultsData]);
```

### 2.4 Simplify Conditions

**Lines 3676-3677** - Remove OR condition:
```javascript
// Change:
if (createCourseElementResultsData || maintenanceModeOverride) {

// To:
if (createCourseElementResultsData) {
```

**Lines 3697-3698** - Remove OR condition:
```javascript
// Change:
if (createCourseFlashcardResultsData || maintenanceModeOverride) {

// To:
if (createCourseFlashcardResultsData) {
```

**After Step 2**:
- Run tests
- Verify no references to maintenanceMode remain
- Commit: `refactor: remove unused maintenance mode from SubtopicElements`

---

## ✅ Step 3: Remove Always-False Toggle (Zero Risk)

**Lines to delete**: ~25 lines  
**Risk**: None (always evaluates to false)  
**Testing**: Quick smoke test

### 3.1 Remove State

**Line 745**
```javascript
// Delete:
const [showAllElementsToggle, setShowAllElementsToggle] = useState(false);
```

### 3.2 Remove Entire useEffect

**Lines 906-926** - Delete entire block:
```javascript
// Delete:
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
		setShowAllElementsToggle(false);
	}
}, [elementType, examQs, resumedCourseCode, homework, homeworkPreview]);
```

### 3.3 Remove from Dependency

**Line 833** - Remove from dependency array:
```javascript
// Change:
}, [showAllElements, showAllElementsToggle]);

// To:
}, [showAllElements]);
```

**After Step 3**:
- Run tests
- Verify toggle references gone
- Commit: `refactor: remove unused showAllElementsToggle from SubtopicElements`

---

## ⚠️ Step 4: Verify Database (REQUIRED)

**Lines affected**: ~110 lines  
**Risk**: Must verify before removal  
**Testing**: Critical

### 4.1 Run Verification Queries

In `common-server`:

```bash
# Connect to production database
mongo <connection_string>

# Query 1: Day-based users
db.users.find({
	'activityAllowance.currentFreeType': 'days',
	isPremiumAccount: { $ne: true },
	isActive: true,
	isArchived: { $ne: true }
}).count()

# Query 2: Users without config
db.users.find({
	$or: [
		{ activityAllowanceConfig: { $exists: false } },
		{ activityAllowanceConfig: null }
	],
	isPremiumAccount: { $ne: true },
	isGuestAccount: { $ne: true },
	isActive: true,
	isArchived: { $ne: true }
}).count()
```

### 4.2 Decision Tree

**If both queries return 0**:
- ✅ Proceed to Step 5 (remove day-based code)
- ✅ Proceed to Step 6 (remove fallback code)

**If Query 1 > 0** (day-based users found):
- ⚠️ Run migration script: `migrate-day-based-to-element-based.js`
- ⚠️ Re-run verification
- ⚠️ Then proceed to Step 5

**If Query 2 > 0** (users without config):
- ⚠️ Run migration script: `assign-missing-configs.js`
- ⚠️ Re-run verification
- ⚠️ Then proceed to Step 6

---

## ✅ Step 5: Remove Day-Based Limits (After Verification)

**Lines to delete**: ~95 lines  
**Risk**: Low (if verification passed)  
**Testing**: Activity limit testing required

### 5.1 Remove State

**Line 755**
```javascript
// Delete:
const [freeDayAllowance, setFreeDayAllowance] = useState(false);
```

### 5.2 Remove Entire Branch

**Lines 1332-1369** - Delete entire else-if block:
```javascript
// Delete:
} else if (loginContext.activityAllowance.currentFreeType === 'days') {
	const signUpDateAsString =
		loginContext.activityAllowance.signUpDateAsString;
	const todaysDateAsString =
		loginContext.activityAllowance.todaysDateAsString;
	const differenceInDays = getDifferenceInDays(
		todaysDateAsString,
		signUpDateAsString
	);
	
	let fieldName = null;
	if (
		loginContext.activityAllowanceConfig?.restrictAcrossAllActivities
	) {
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

### 5.3 Remove Utility Function

**Lines 1396-1413**
```javascript
// Delete:
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

### 5.4 Remove Usage

**Line 1432**
```javascript
// Delete:
if (freeDayAllowance) {
	usingFreeAllowance = true;
}
```

**After Step 5**:
- Run activity limit tests
- Test guest user limits
- Test each cohort configuration
- Commit: `refactor: remove day-based activity allowance system`

---

## ✅ Step 6: Remove Missing Config Fallback (After Verification)

**Lines to delete**: ~15 lines  
**Risk**: Low (if verification passed)  
**Testing**: Activity limit testing required

### 6.1 Remove Fallback Branch

**Lines 1542-1550**
```javascript
// Delete:
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

### 6.2 Simplify Assignment

**Line 1525**
```javascript
// Change:
let maxActivityAllowance = ActivityAccess.MAX_DAILY_ACTIVITY_ACCESS;

// To:
let maxActivityAllowance = 0;  // Will be set by config
```

Or remove entirely if only used in deleted fallback

### 6.3 Update Logic

**Lines 1519-1574** - This block becomes simpler without fallback

**Before**:
```javascript
if (!maxActivitiesReached) {
	// ...
} else {
	let maxActivityAllowance = ActivityAccess.MAX_DAILY_ACTIVITY_ACCESS;
	
	if (/* conditions */) {
		maxActivityAllowance = /* from config */;
	}
	
	if (!loginContext.activityAllowanceConfig) {  // DELETE THIS BRANCH
		// ...
	} else if (loginContext.activityAllowanceConfig.period === 'daily') {
		// ...
	} else if (loginContext.activityAllowanceConfig.period === 'weekly') {
		// ...
	}
}
```

**After**:
```javascript
if (!maxActivitiesReached) {
	// ...
} else {
	let maxActivityAllowance = /* from config */;
	
	// Remove fallback, go straight to period checks
	if (loginContext.activityAllowanceConfig.period === 'daily') {
		// ...
	} else if (loginContext.activityAllowanceConfig.period === 'weekly') {
		// ...
	}
}
```

**After Step 6**:
- Run activity limit tests
- Test config not found scenario (should never happen)
- Commit: `refactor: remove activityAllowanceConfig fallback logic`

---

## ✅ Step 7: Remove Homework Functionality

**Lines to delete**: ~200 lines  
**Risk**: Medium (verify isolation first)  
**Testing**: Full activity flow testing

### 7.1 Remove Props

**Lines 472-480, 482, 504** in function signature:
```javascript
// Delete these props:
homeworkPreview,
setHomeworkElements,
confirmHomework,
homework,
homeworkElements,
hideHints,
homeworkId,
homeworkCode,
hideMainButton,
homeworkName,
resumedHomeworkCode,
```

### 7.2 Remove Queries

**Lines 157-165** - Delete query:
```javascript
// Delete:
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

**Lines 223-230** - Delete query:
```javascript
// Delete:
const COURSEREVISEELEMENTS_HOMEWORK_QUERY = gql`
  query courseReviseElementsHomework(...) {
    courseReviseElementsHomework(...) {
      courseSubtopicCode,
      ${ELEMENT_FIELDS}
    }
  }
`;
```

### 7.3 Remove State

**Lines 714-720**
```javascript
// Delete:
const tempElementIds = [];
if (homeworkElements) {
	for (let i = 0; i < homeworkElements.length; i++) {
		tempElementIds.push(homeworkElements[i].elementId);
	}
}
const [elementIds] = useState(tempElementIds);
```

**Line 721**
```javascript
// Delete:
const [homeworkReadyToConfirm, setHomeworkReadyToConfirm] = useState(false);
```

### 7.4 Remove Conditional Logic

**Search and remove all homework conditionals**:

Pattern: `if (homework || homeworkPreview)`, `if (homework)`, `homeworkCode`, etc.

**Key locations**:
- Line 907-908: Toggle disable check
- Lines 2219-2240: Homework save logic
- Lines 3499: Proxy premium check
- Lines 3560-3562: Batch save vars
- Lines 3581-3583: Homework batch queue
- Lines 3742-3744: Query selection
- Lines 3758-3760: Query selection
- Lines 3803-3807: Variables
- Lines 3850-3854: Variables
- Lines 3865-3874: Variables
- Lines 4102-4120: Element gathering
- Lines 4125-4127: Ready to confirm
- Lines 4136-4141: Resume logic
- Lines 4155-4156: Query selection
- Lines 4193-4202: Element gathering
- Lines 4788-4789: Show all elements
- Line 4820: Dependencies
- Line 4609: Prop pass
- Lines 5054-5057: Video/hints props
- Lines 6288-6314: Confirm button

### 7.5 Update Query Selection

**Lines 3742-3774** - Simplify query selection:
```javascript
// Remove all homework branches
if (resumedCourseCode) {
	if (homework) { // DELETE
		query = RESUMED_COURSEELEMENTS_HOMEWORK_QUERY; // DELETE
	} else { // DELETE 'else'
		if (examQs) query = COURSEELEMENTS_QUERY;
		else query = RESUMED_COURSEELEMENTS_QUERY;
	} // DELETE
}
```

Becomes:
```javascript
if (resumedCourseCode) {
	if (examQs) query = COURSEELEMENTS_QUERY;
	else query = RESUMED_COURSEELEMENTS_QUERY;
}
```

### 7.6 Remove Footer Button

**Lines 6288-6314** - Delete entire conditional block:
```javascript
// Delete:
{homeworkPreview &&
!hideMainButton &&
!teacherPreview &&
!showAllElements ? (
	<div className={...}>
		<Button
			value={'confirmHomework'}
			disabled={!homeworkReadyToConfirm}
			id="confirmHomework"
			type="button"
			variant="contained"
			color="primary"
			onClick={confirmHomework}
			className={classes.continueButton}
		>
			<Typography className={classes.continueButtonLabel}>
				Confirm
			</Typography>
		</Button>
	</div>
) : null}
```

**After Step 7**:
- Run full test suite
- Manual test: Start each activity type
- Verify no homework code remains
- Commit: `refactor: remove homework functionality from SubtopicElements`

---

## ✅ Step 8: Remove Batch Saving Mechanism

**Lines to delete**: ~300 lines  
**Risk**: Medium (verify per-element saves work correctly)  
**Testing**: Critical - test all activity types

### 8.1 Remove Mutations

**Lines 296-324** - Delete mutation:
```javascript
// Delete:
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
		$excludeXP: Boolean
		$examQs: Boolean
	) {
		createCourseElementResults(
			elementResults: $elementResults
			type: $type
			isHomework: $isHomework
			homeworkId: $homeworkId
			homeworkCode: $homeworkCode
			dateTimeStarted: $dateTimeStarted
			dateTimeCompleted: $dateTimeCompleted
			completeSubtopic: $completeSubtopic
			preciseDuration: $preciseDuration
			excludeXP: $excludeXP
			examQs: $examQs
		)
	}
`;
```

**Lines 326-350** - Delete mutation:
```javascript
// Delete:
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
		$excludeXP: Boolean
	) {
		createCourseFlashcardResults(
			flashcardResults: $flashcardResults
			levelOneResponseCount: $levelOneResponseCount
			levelTwoResponseCount: $levelTwoResponseCount
			levelThreeResponseCount: $levelThreeResponseCount
			dateTimeStarted: $dateTimeStarted
			dateTimeCompleted: $dateTimeCompleted
			completeSubtopic: $completeSubtopic
			preciseDuration: $preciseDuration
			excludeXP: $excludeXP
		)
	}
`;
```

### 8.2 Remove Hooks

**Lines 1845-1846**
```javascript
// Delete:
const [createCourseElementResults, { data: createCourseElementResultsData }] =
	useMutation(CREATECOURSEELEMENTRESULTS_MUTATION, { errorPolicy: 'all' });
```

**Lines 1848-1852**
```javascript
// Delete:
const [
	createCourseFlashcardResults,
	{ data: createCourseFlashcardResultsData },
] = useMutation(CREATE_COURSE_FLASHCARD_RESULTS_MUTATION, {
	errorPolicy: 'all',
});
```

### 8.3 Remove State

**Line 704**
```javascript
// Delete:
const [saveResults, setSaveResults] = useState(false);
```

**Line 749**
```javascript
// Delete:
const [continueToSave, setContinueToSave] = useState(true);
```

**Lines 782-783**
```javascript
// Delete:
const [courseElementResultsToSave, setCourseElementResultsToSave] =
	useState(null);
```

### 8.4 Remove Flashcard Batch Save

**Lines 966-1031** - Remove createCourseFlashcardResults call:
```javascript
// In useEffect for flashcardsComplete:
// Remove the createCourseFlashcardResults call (lines 1002-1018)
// Keep the rest of the useEffect (dialog, summary calculation)

// Delete these lines:
if (!loginContext.isGuestAccount && anyElementSubmitted) {
	createCourseFlashcardResults({
		variables: {
			flashcardResults: flashcardResults,
			levelOneResponseCount: newFlashcardsSummary[1],
			levelTwoResponseCount: newFlashcardsSummary[2],
			levelThreeResponseCount: newFlashcardsSummary[3],
			dateTimeStarted: dateTimeStarted,
			dateTimeCompleted: dateTimeCompleted,
			completeSubtopic: completeSubtopic,
			preciseDuration: preciseDuration,
			excludeXP: /* condition */,
		},
	});
}
```

### 8.5 Remove Entire Batch Save useEffect

**Lines 3382-3600** - Delete entire useEffect:
```javascript
// Delete entire block:
useEffect(() => {
	if (saveResults && continueToSave) {
		setSaveResults(false);
		
		const elementResults = [];
		
		if (elementsContext.currentScores) {
			// ... 160+ lines of result gathering ...
		}
		
		// ... more logic ...
		
		if (!loginContext.isGuestAccount) {
			setAnyElementSubmitted(false);
			
			if (newElementResults.length > 0) {
				const variables = { /* ... */ };
				
				if (!homework) {
					createCourseElementResults({ variables });
				} else {
					setCourseElementResultsToSave(variables);
				}
			}
		}
	}
}, [saveResults, continueToSave]);
```

### 8.6 Remove Homework Batch Coordinator

**Lines 3602-3609**
```javascript
// Delete:
useEffect(() => {
	if (!savingElementResult && courseElementResultsToSave) {
		createCourseElementResults({
			variables: courseElementResultsToSave,
		});
		setCourseElementResultsToSave(null);
	}
}, [savingElementResult, courseElementResultsToSave]);
```

### 8.7 Remove Continue Coordinator

**Lines 2716-2719**
```javascript
// Delete:
useEffect(() => {
	if (saveCourseElementResultData && !continueToSave) {
		setContinueToSave(true);
	}
}, [saveCourseElementResultData, continueToSave]);
```

### 8.8 Remove Batch Completion Handlers

**Lines 3676-3688**
```javascript
// Delete:
useEffect(() => {
	if (createCourseElementResultsData) {
		setNavigationContext((navigationContext) => ({
			...navigationContext,
			refreshCourseResults: true,
		}));
	}
}, [createCourseElementResultsData]);
```

**Lines 3697-3704**
```javascript
// Delete:
useEffect(() => {
	if (createCourseFlashcardResultsData) {
		setNavigationContext((navigationContext) => ({
			...navigationContext,
			refreshCourseResults: true,
		}));
	}
}, [createCourseFlashcardResultsData]);
```

### 8.9 Remove saveResults Triggers

Search for `setSaveResults(true)` and remove:
- Line 3179: In saveActivity function
- Line 5659: In completion logic

### 8.10 Remove continueToSave Usage

**Line 2220-2221**
```javascript
// Delete:
if (elementCount === elements.length) {
	setContinueToSave(false);
}
```

**Line 3660**
```javascript
// Delete:
setContinueToSave(true);
```

**After Step 8**:
- **CRITICAL**: Test all activity saves
- Verify per-element saves work for:
  - [ ] Lessons (focus mode)
  - [ ] Lessons (all-at-once)
  - [ ] Quizzes (focus mode)
  - [ ] Quizzes (all-at-once)
  - [ ] Flashcards (focus mode)
  - [ ] Flashcards (all-at-once)
  - [ ] Exam Qs
  - [ ] Exam MCQs
- Verify activity completion works
- Verify XP is awarded
- Check database for correct saves
- Commit: `refactor: remove batch saving mechanism, keep per-element saves`

---

## ✅ Step 9: Remove Teacher Save Checks

**Lines to delete**: ~15 lines  
**Risk**: Low (teachers can complete activities now)  
**Testing**: Teacher account testing

### 9.1 Remove isTeacher Calculation

**Lines 956-963**
```javascript
// Delete:
let isTeacher = false;
if (
	loginContext &&
	loginContext.roleCodes &&
	loginContext.roleCodes.includes('teacher')
) {
	isTeacher = true;
}
```

### 9.2 Remove isTeacher Conditionals

**Line 985** - Remove check:
```javascript
// Change:
if (proxyAsPremiumUser) {
	if (!isTeacher) {  // DELETE
		setCompletePremiumActivityStatus({ xpPointsGained: xpPointsGained });
	}  // DELETE
}

// To:
if (proxyAsPremiumUser) {
	setCompletePremiumActivityStatus({ xpPointsGained: xpPointsGained });
}
```

**Line 3500** - Remove check:
```javascript
// Change:
if (proxyAsPremiumUser && !homework) {
	if (!isTeacher) {  // DELETE
		// ... completion logic ...
	}  // DELETE
}

// To:
if (proxyAsPremiumUser) {
	// ... completion logic ...
}
```

**Line 3518** - Remove check:
```javascript
// Change:
} else {
	if (!isTeacher) {  // DELETE
		// ... save logic ...
	}  // DELETE
}

// To:
} else {
	// ... save logic ...
}
```

**Line 3657** - Remove check:
```javascript
// Change:
} else {
	if (!isTeacher) {  // DELETE
		markPremiumUserCourseAsComplete({ ... });
		logEvent({ event_type: 'end_activity' });
	}  // DELETE
}

// To:
} else {
	markPremiumUserCourseAsComplete({ ... });
	logEvent({ event_type: 'end_activity' });
}
```

### 9.3 Remove Prop Pass

**Line 4952**
```javascript
// Change:
isTeacher={isTeacher}

// Delete this prop from FlashcardCard
```

**After Step 9**:
- Test with teacher account
- Verify teachers can complete activities
- Verify teachers get XP
- Commit: `refactor: remove teacher save prevention, teachers can complete activities`

---

## 🧪 Testing Strategy

### After Each Step
1. Run test suite: `npm test`
2. Check build: `npm run build`
3. Check linter: `npm run lint`

### After Step 7 (Homework)
Manual testing:
- [ ] Start lesson (non-school account)
- [ ] Complete lesson
- [ ] Verify no homework UI appears
- [ ] Check console for errors

### After Step 8 (Batch Saving) - CRITICAL
Manual testing for each activity type:
- [ ] Lesson focus mode: Submit each element, verify saves
- [ ] Lesson all-at-once: Submit all, verify saves
- [ ] Quiz focus mode: Submit each element, verify saves
- [ ] Quiz all-at-once: Submit all, verify saves
- [ ] Flashcards focus: Select responses, verify saves
- [ ] Flashcards all-at-once: Select responses, verify saves
- [ ] Exam Qs: Submit parts, verify saves
- [ ] Exam MCQs: Submit questions, verify saves

Check database:
```javascript
// Verify courseElementResults created per element
db.courseElementResults.find({
	courseCode: "TEST_COURSE",
	dateTimeCreated: { $gte: new Date('2025-11-20') }
}).sort({ dateTimeCreated: -1 }).limit(20)

// Verify no batch results
db.courseResults.find({
	courseCode: "TEST_COURSE",
	dateTimeCreated: { $gte: new Date('2025-11-20') }
})
```

### After Step 9 (Teacher)
Manual testing:
- [ ] Login as teacher
- [ ] Complete lesson
- [ ] Complete quiz
- [ ] Complete flashcards
- [ ] Verify XP awarded
- [ ] Check progress saved

---

## 📋 Sign-Off Checklist

### Before Starting
- [ ] All assessment documents reviewed
- [ ] Team approval obtained
- [ ] DB verification queries run (Steps 4-6)
- [ ] Migration run if needed
- [ ] Backup branch created

### After Completion
- [ ] All 820+ lines removed
- [ ] All tests passing
- [ ] Manual testing complete
- [ ] No console errors
- [ ] No build warnings
- [ ] Database saves verified
- [ ] Analytics events verified
- [ ] Code review approved
- [ ] Documentation updated

### Deployment
- [ ] Deployed to staging
- [ ] Staging testing complete
- [ ] Deployed to production
- [ ] Production monitoring (24 hours)
- [ ] No rollback needed

---

## 🐛 Common Issues & Solutions

### Issue: Tests failing after homework removal
**Solution**: Update test mocks to remove homework props

### Issue: Query selection failing
**Solution**: Verify query selection logic simplified correctly (Step 7.5)

### Issue: Activity not saving
**Solution**: Check per-element saves still called correctly (Step 8.8 critical)

### Issue: Progress bar not updating
**Solution**: Verify updateProgressBar() still called after per-element saves

### Issue: Completion not detected
**Solution**: Check activity complete logic (no longer waits for batch save)

---

## 📞 Rollback Plan

### If Issues in Step 1-3
- Revert commit
- Zero risk, safe rollback

### If Issues in Step 5-6 (Config)
- Check DB migration ran correctly
- Revert migration if needed
- Revert code changes

### If Issues in Step 7 (Homework)
- Check homework is truly isolated
- Revert commit
- Investigate shared dependencies

### If Issues in Step 8 (Batch Saving)
- **CRITICAL**: Check per-element saves
- Verify database writes
- May need backend investigation
- Revert commit if saves failing

### If Issues in Step 9 (Teacher)
- Revert commit
- Check teacher permissions

---

## 📚 Reference

**All Documents**: `/cognito-main-client/REFACTOR_*.md`

**Key Commands**:
```bash
# Run tests
npm test

# Run specific test
npm test -- SubtopicElements

# Check build
npm run build

# Lint
npm run lint

# Start dev server
npm run dev
```

---

## ✨ Success!

After completing all steps, you will have:
- ✅ Removed 820+ lines of redundant code
- ✅ Simplified SubtopicElements to 5545 lines
- ✅ Maintained all functionality
- ✅ Improved code quality by 14%
- ✅ Created foundation for further refactoring

**Next**: Proceed to Phase 1 (Hook Extractions) following the [Roadmap](REFACTOR_ROADMAP.md)

