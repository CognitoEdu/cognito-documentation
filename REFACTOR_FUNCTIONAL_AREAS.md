# SubtopicElements Functional Areas Map

**Component**: SubtopicElements.jsx (6461 lines)  
**Purpose**: Core activity container for lessons, quizzes, flashcards, exam questions  
**Date**: 2025-11-20

---

## Overview

This document maps the 12 core functional areas within SubtopicElements, documenting their current implementation, dependencies, complexity, and recommendations for refactoring into a reusable activity container.

---

## 2.1 Element Data Fetching & Queries

### Current Implementation
**Lines**: 110-233, 3734-3940, 4029-4254  
**Complexity**: ⭐⭐⭐⭐ High

### Components

**9 GraphQL Queries**:
1. `SUBTOPICS_QUERY` (110) - Fetches course subtopics for navigation
2. `MASTERCOURSEELEMENTS_QUERY` (126) - Master course elements (preview/design mode)
3. `COURSEELEMENTS_QUERY` (135) - Standard course elements
4. `RESUMED_COURSEELEMENTS_QUERY` (146) - Resume from in-progress
5. `RESUMED_COURSEELEMENTS_HOMEWORK_QUERY` (157) - **REMOVE** (homework)
6. `ELEMENT_QUERY` (167) - Single element view
7. `COURSEREVISEELEMENTS_QUERY` (175) - Quiz elements
8. `COURSEEXAMQELEMENTS_QUERY` (188) - Exam Q elements
9. `COURSEFLASHCARDSELEMENTS_QUERY` (201) - Flashcard elements
10. `RESUMED_COURSEFLASHCARDS_QUERY` (212) - Resume flashcards
11. `COURSEREVISEELEMENTS_HOMEWORK_QUERY` (223) - **REMOVE** (homework)

**Key Functions**:
- `fetchElements()` (3777-3922): Builds variables and triggers query
- `handleElementsData()` (4029-4254): Transforms query results

### Query Selection Logic

```javascript
// Lines 3734-3774
if (type === 'mastercourse') query = MASTERCOURSEELEMENTS_QUERY;
else if (type === 'course') {
	if (isExamQsRevisionSession) query = COURSEEXAMQELEMENTS_QUERY;
	else if (resumedCourseCode) {
		if (homework) query = RESUMED_COURSEELEMENTS_HOMEWORK_QUERY; // REMOVE
		else if (examQs) query = COURSEELEMENTS_QUERY;
		else query = RESUMED_COURSEELEMENTS_QUERY;
	}
	else query = COURSEELEMENTS_QUERY;
}
else if (type === 'individual') query = ELEMENT_QUERY;
else if (type === 'courserevise') {
	if (resumedCourseCode) {
		if (homework) query = RESUMED_COURSEELEMENTS_HOMEWORK_QUERY; // REMOVE
		else query = RESUMED_COURSEELEMENTS_QUERY;
	}
	else if (homework) query = COURSEREVISEELEMENTS_HOMEWORK_QUERY; // REMOVE
	else query = COURSEREVISEELEMENTS_QUERY;
}
else if (type === 'courseflashcards') {
	if (resumedCourseCode) query = RESUMED_COURSEFLASHCARDS_QUERY;
	else query = COURSEFLASHCARDSELEMENTS_QUERY;
}
```

### Variable Building Complexity

**Example - Course elements** (lines 3833-3844):
```javascript
variables = {
	courseCode: courseCode,
	courseSubtopicCode: parentCode,
	courseSubtopicName: subtopicName,
	subtopicCode: subtopicCode,
	subjectCode: subjectCode,
	subtopicDisplayCode: subtopicDisplayCode,
	homeworkCode: homeworkCode,  // REMOVE
	readOnly: homeworkPreview,   // REMOVE
	showAllElements: showAllElements,
};
```

### Dependencies
- `type` prop (activity type)
- `resumedCourseCode` (resume state)
- `homework` props (REMOVE)
- `examQs`, `examQType` (exam question flags)
- `showAllElements` (display mode)
- `subtopics`, `numberOfQuestions` (configuration)
- `initialElementIds` (persist element selection)

### Refactoring Recommendations

**Extract to**: `useElementFetching()` custom hook

**Benefits**:
- Isolate query selection logic
- Simplify variable building
- Enable query result caching
- Easier testing

**Structure**:
```javascript
const {
	elements,
	loading,
	error,
	refetch,
	subtopicsData
} = useElementFetching({
	activityType,
	courseCode,
	subtopicCode,
	subtopics,
	numberOfQuestions,
	resumedCourseCode,
	resultType,
	showAllElements,
	examQs,
	examQType,
	// ... simplified props
});
```

---

## 2.2 Display Mode Management

### Current Implementation
**Lines**: 741-833, 906-926, 1087-1094, 3942-3957, 4787-4823  
**Complexity**: ⭐⭐⭐ Medium

### Components

**State Management**:
- `showAllElements` (742): Boolean for focus vs all-at-once
- `prevShowAll` (741): Track previous state
- `showAllElementsRef` (743): Ref for event handlers
- `showAllElementsToggle` (745): **REMOVE** - Always false

**localStorage Persistence** (lines 791-833):
```javascript
useEffect(() => {
	if (resultType !== 'restart' || resultType !== 2) {
		const showAllElementsValue = localStorage.getItem(SHOW_ALL_ELEMENTS);
		if (!showAllElementsValue) {
			// Random assignment for A/B testing
			const randomValue = Math.floor(Math.random() * 2);
			if (randomValue === 1) {
				setShowAllElements(true);
				localStorage.setItem(SHOW_ALL_ELEMENTS, '1');
				localStorage.setItem(ORIGINAL_SHOW_ALL_ELEMENTS, '1');
			} else {
				setShowAllElements(false);
				localStorage.setItem(SHOW_ALL_ELEMENTS, '0');
				localStorage.setItem(ORIGINAL_SHOW_ALL_ELEMENTS, '0');
			}
		}
	}
	
	// Handle restart vs redo logic
	if ((resultType === 'restart' || resultType === 2) && resumedCourseCode) {
		setRestartActivity(true);
		setIsRedoingCourse(false);
	} else if (resultType === 0 || resultType === 'levelThree') {
		setIsRedoingCourse(true);
		setRestartActivity(false);
	}
}, [showAllElements, showAllElementsToggle]);
```

**Toggle Handler** (lines 3942-3957):
```javascript
const handleShowAllElementsToggle = (showAll) => {
	activityBegun.current = true;
	
	if (prevShowAll !== showAll) {
		setShowAllElements(showAll);
	}
	
	ReactGA.event(
		'disp_mode_changed_to_' + (showAll ? 'showall_elmnts' : 'onebyone_elmnts'),
		{ category: 'activity_display_mode_changed', context: 'activity' }
	);
};
```

**Mode Detection** (lines 4787-4823):
```javascript
useEffect(() => {
	if (homework || homeworkPreview) {  // REMOVE
		setShowAllElements(false);
	} else if (preview && designMode && (examQs || elementIsExamQ)) {
		setShowAllElements(true);
	} else {
		if (examQs) {
			setShowAllElements(true);  // Exam Qs always all-at-once
		} else if (type === 'course' && isNonInteractiveCourse) {
			setShowAllElements(true);  // A-level revision notes
		} else if (resultType === 'restart') {
			setShowAllElements(false);  // Default to focus mode
		} else {
			const prevShowAll = localStorage.getItem(SHOW_ALL_ELEMENTS) === '1' || false;
			setShowAllElements(prevShowAll);  // Restore preference
		}
	}
}, [/* dependencies */]);
```

### Mode-Specific Behavior

**Focus Mode (showAllElements: false)**:
- Single element visible at a time
- `elementCount` tracks current element
- Footer button controls progression
- Scroll to element after submission
- `continueButton` in footer coordinates with ElementCard

**All-At-Once Mode (showAllElements: true)**:
- All elements rendered simultaneously
- Each element has own submit button
- No element count progression
- Scroll management different
- Activity complete when all submitted

### Context Coordination

**Line 1087-1094**: Sync with ElementsProgressContext
```javascript
useEffect(() => {
	setElementsProgressContext((elementsProgressContext) => ({
		...elementsProgressContext,
		disableScroll: showAllElements,
	}));
	
	showAllElementsRef.current = showAllElements ? true : false;
}, [showAllElements]);
```

### Dependencies
- `localStorage` for persistence
- `elementsProgressContext.disableScroll` coordination
- Element rendering logic (different components)
- Footer button coordination
- Analytics tracking

### Refactoring Recommendations

**Extract to**: `useDisplayMode()` custom hook

**Benefits**:
- Centralize mode detection logic
- Simplify localStorage handling
- Clearer mode switching
- Easier A/B testing

**Structure**:
```javascript
const {
	showAllElements,
	handleToggle,
	isAllAtOnceForced  // For exam qs, revision notes
} = useDisplayMode({
	activityType,
	resultType,
	examQs,
	levelCode,
	resumedCourseCode
});
```

---

## 2.3 Scoring & Progress Tracking

### Current Implementation
**Lines**: 1061-1072, 1976-2416, 2962-3010, 4466-4543  
**Complexity**: ⭐⭐⭐⭐⭐ Very High

### Core State Structure

**elementsContext.currentScores** (line 1061-1072):
```javascript
{
	hideVideoLinks: hideVideoLinks,
	currentScores: {},  // KEY: elementId, VALUE: score object
	currentScoresTotal: 0,
	currentIncorrectScoresTotal: 0,
	totalScore: 0,
	updateCurrentScoresTotal: false,
	currentMultipartScores: {},
	currentMultipartScoresTotal: {},
	currentMultipartElementScores: {},
	currentMultipartComplete: {},
}
```

**Score Object Structure**:
```javascript
currentScores[elementId] = {
	score: 5,  // Marks awarded
	incorrectScore: 2,  // Marks lost
	totalScore: 7,  // Total possible
	submittedAnswer: { /* answer data */ },
	actualAnswers: { /* all answer attempts */ },
	isMultipart: false,
	elementId: "element_id",
	rootElementId: "parent_id" | null,
	courseSubtopicCode: "subtopic_code"
}
```

### Score Calculation Logic

**saveElementResult()** (lines 1976-2416):

**For simple elements** (lines 1993-2094):
1. Extract submitted answers from elementsContext
2. Calculate score from marksAwarded
3. Apply score override for suspicious 0 scores (lines 2024-2072)
4. Construct elementResult object
5. Add to elementResults array

**For multipart elements** (lines 2095-2203):
1. Iterate through all child element scores
2. Aggregate scores from children
3. Create separate elementResult for each child
4. Create parent elementResult with aggregated score

**Score Override Logic** (lines 2024-2072):
```javascript
// If score is 0 or NaN but answers exist, recalculate
if (score === 0 || isNaN(score)) {
	if (submittedAnswer && Object.keys(submittedAnswer).length > 0) {
		let calculatedScore = 0;
		// Count correct answers from submittedAnswer
		for (let j = 0; j < subelementKeys.length; j++) {
			const subelementAnswer = submittedAnswer[subelementKeys[j]];
			
			if (subelementAnswer.answers && Array.isArray(subelementAnswer.answers)) {
				calculatedScore += subelementAnswer.answers.filter(a => a.correct === true).length;
			}
			
			if (subelementAnswer.textAnswer && subelementAnswer.textAnswer.correct === true) {
				calculatedScore += 1;
			}
			
			if (subelementAnswer.isCorrect === true) {
				calculatedScore += 1;
			}
		}
		
		if (calculatedScore > 0) {
			score = calculatedScore;
		}
	}
}
```

### Progress Bar Updates

**updateProgressBar()** (lines 2962-3010):
```javascript
const updateProgressBar = () => {
	setTimeout(() => {
		let numberOfCorrectMarks = 0;
		let numberOfIncorrectMarks = 0;
		
		if (elementsContext.currentScores) {
			const keys = Object.keys(elementsContext.currentScores);
			for (let i = 0; i < keys.length; i++) {
				if (elementsContext.currentScores[keys[i]].totalScore) {
					numberOfCorrectMarks += elementsContext.currentScores[keys[i]].score || 0;
					numberOfIncorrectMarks += elementsContext.currentScores[keys[i]].incorrectScore || 0;
				}
			}
		}
		
		// Add initial scores if resuming
		let initialCorrectValue = 0;
		let initialIncorrectValue = 0;
		let initialTotalScoreValue = 0;
		if (!showAllElements) {
			initialCorrectValue = initialCorrect || 0;
			initialIncorrectValue = initialIncorrect || 0;
			initialTotalScoreValue = initialTotalScore || 0;
		}
		
		setProgressBarHeaderContext((progressBarHeaderContext) => ({
			...progressBarHeaderContext,
			correctValue: numberOfCorrectMarks + initialCorrectValue,
			incorrectValue: numberOfIncorrectMarks + initialIncorrectValue,
			maxValue: initialTotalScoreValue || progressBarHeaderContext.maxValue,
			elementIndex: showAllElements ? numberOfCompletedElements : elementCount,
		}));
	});
};
```

### Score Aggregation

**Lines 4466-4536**: Update scores when element changes
```javascript
useEffect(() => {
	if (elementsContext.updateCurrentScoresTotal) {
		let newCurrentScoresTotal = 0;
		let newCurrentIncorrectScoresTotal = 0;
		
		// Aggregate all element scores
		for (let i = 0; i < Object.keys(elementsContext.currentScores).length; i++) {
			const scoresObjectKey = Object.keys(elementsContext.currentScores)[i];
			if (elementsContext.currentScores[scoresObjectKey]) {
				newCurrentScoresTotal += elementsContext.currentScores[scoresObjectKey].score || 0;
				newCurrentIncorrectScoresTotal += elementsContext.currentScores[scoresObjectKey].incorrectScore || 0;
			}
		}
		
		// Add resumed scores if applicable
		if (resumedScoreTotal) newCurrentScoresTotal += resumedScoreTotal;
		if (resumedIncorrectScoreTotal) newCurrentIncorrectScoresTotal += resumedIncorrectScoreTotal;
		
		setElementsContext((elementsContext) => ({
			...elementsContext,
			updateCurrentScoresTotal: false,
			currentScoresTotal: newCurrentScoresTotal,
			currentIncorrectScoresTotal: newCurrentIncorrectScoresTotal,
		}));
	}
}, [elementsContext.updateCurrentScoresTotal]);
```

### Element Subtopic Mapping

**Lines 4276-4290**: Creates lookup table for mixed-subtopic activities
```javascript
const createElementSubtopicMap = (elements) => {
	const map = {};
	elements.forEach((element) => {
		if (element.type === 'multipart') {
			const courseSubtopicCode = element.courseSubtopicCode;
			// Add subelements to map
			element.subelements.forEach((subelement) => {
				map[subelement.element.id] = courseSubtopicCode;
			});
		}
		map[element.id] = element.courseSubtopicCode;
	});
	
	setElementSubtopicMap(map);
};
```
**Purpose**: For exam Q revision sessions with mixed subtopics, quickly determine which subtopic each element belongs to for correct saving.

### Dependencies
- Apollo Client (`useLazyQuery`)
- `ELEMENT_FIELDS` constant (all element properties)
- Activity type props (`type`, `examQs`, `examQType`)
- Resume props (`resumedCourseCode`, `resultType`)
- Homework props (REMOVE)
- Display mode (`showAllElements`)
- `elementsContext` for score storage
- `ProgressBarHeaderContext` for header updates

### Issues & Technical Debt
1. **Query proliferation**: 11 queries for different combinations
2. **Complex variable building**: 50+ lines of conditional logic
3. **Duplicate transformation**: Similar logic for each activity type
4. **No caching**: Network-only fetch policy

### Refactoring Recommendations

**Priority**: High  
**Extract to**: `useElementFetching()` hook + query consolidation

**Proposed structure**:
```javascript
// Consolidate queries into fewer, more flexible queries
const ACTIVITY_ELEMENTS_QUERY = gql`
  query activityElements(
    $activityConfig: ActivityConfigInput!
    $filters: ElementFiltersInput
  ) {
    activityElements(activityConfig: $activityConfig, filters: $filters) {
      ${ELEMENT_FIELDS}
      metadata {
        showAllElements
        singleSubtopic
        courseSubtopicCode
      }
    }
  }
`;

// Hook encapsulates all fetching logic
const {
	elements,
	loading,
	error,
	subtopicsData,
	refetch
} = useElementFetching({
	activityType,
	courseCode,
	subtopicCode,
	// ... simplified config
});
```

**Benefits**:
- Reduce from 11 queries to 2-3 flexible queries
- Centralize variable building
- Enable proper caching strategies
- Simplify component logic by 200+ lines

---

## 2.4 Per-Element Save Mechanism

### Current Implementation
**Lines**: 1965-2416, 2686-2701, 2721-2833, 3113-3173  
**Complexity**: ⭐⭐⭐⭐ High

### Mutations Used (KEEP)

**SAVECOURSEELEMENTRESULT_MUTATION** (line 232):
```javascript
mutation saveCourseElementResult(
	$elementResults: [CourseElementResultInput]
	$type: String
	$homeworkCode: String  // REMOVE
	$preciseDuration: Int
	$showAllElements: Boolean
	$examQ: Boolean
)
```
**Purpose**: Saves individual element results immediately after submission

**SAVECOURSEFLASHCARDRESULT_MUTATION** (line 282):
```javascript
mutation saveCourseFlashcardResult(
	$flashcardResult: CourseFlashcardResultInput
	$preciseDuration: Int
	$showAllElements: Boolean
)
```
**Purpose**: Saves individual flashcard response immediately

### Key Functions

**saveElementResult()** (lines 1976-2416):
Main function that constructs element result and triggers save.

**Process**:
1. Identify element to save (handle multipart children)
2. Extract submitted answers from `elementsContext.currentScores`
3. Calculate score (with override for suspicious 0s)
4. Build `elementResult` object with:
   - courseCode, topicCode, subtopicCode
   - elementCode
   - marksAwarded, totalScore, scorePercentage
   - submittedAnswer
   - isMultipart, rootElementCode
5. Call appropriate mutation immediately

**For Focus Mode**:
- Called after each element submission
- Saves one element at a time
- Progresses to next element after save

**For All-At-Once Mode**:
- Called after each element's submit button
- Multiple saves happen independently
- Activity completes when all elements submitted

**resetIndividualExamQPart()** (lines 2418-2701):
```javascript
const resetIndividualExamQPart = (currentElement) => {
	// 1. Build elementResults from current scores
	// 2. Mark currentElement as reset: true
	// 3. Mark incomplete children as reset: true
	// 4. Call saveCourseElementResult with reset flags
	
	saveCourseElementResult({
		variables: {
			elementResults: elementResults,
			type: type === 'courserevise' ? 'R' : 'L',
			preciseDuration: preciseDuration,
			showAllElements: showAllElements,
			examQ: examQs ? true : false,
		},
	});
};
```
**Purpose**: Resets individual exam Q parts by sending `reset: true` flag

### Flashcard Save Flow

**handleFlashcardResponse()** (lines 2721-2833):
```javascript
const handleFlashcardResponse = (response, event) => {
	// 1. Set selectedResponse for UI feedback
	// 2. Update flashcardsProgress array
	// 3. Save immediately if premium/proxy user
	
	if (loginContext.isProxyPremiumAccount && !isTeacher) {
		const flashcardResult = {
			courseCode: resumedCourseCode ? resumedCourseCode : courseCode,
			topicCode: topicCode,
			subtopicCode: /* determine subtopic */,
			elementCode: currentElement.id,
			response: response,
		};
		
		saveCourseFlashcardResult({
			variables: {
				flashcardResult: flashcardResult,
				preciseDuration: preciseDuration,
				showAllElements: showAllElements,
			},
		});
	}
	
	// 4. Progress to next flashcard or complete activity
	if (elementCount < elements.length) {
		setElementCount(elementCount + 1);
	} else {
		endActivityForFreeUser();  // Complete activity
	}
};
```

### Timing & Idle Tracking

**Precise Duration Calculation**:
```javascript
let preciseDuration = 0;
if (elementTimer && elementTimer.getTotalActiveTime()) {
	preciseDuration = Math.ceil(elementTimer.getTotalActiveTime() / 60000);
	if (addVideoToIdleTimer) {
		preciseDuration += VIDEO_IDLE_TIMEOUT;  // 5 mins
	}
}

if (initialPreciseDuration) {
	preciseDuration += initialPreciseDuration;  // Add resumed time
}
```

**Idle Timer**: 
- `TIMEOUT_ELEMENT = 2 * 60 * 1000` (2 minutes)
- `VIDEO_IDLE_TIMEOUT = 5` (5 minutes) for video elements
- Tracks actual engagement time

### Save Coordination

**Lines 1965-1974**: Trigger from elementsContext
```javascript
useEffect(() => {
	if (elementsContext.saveElement) {
		saveElementResult(elementsContext.saveElement, null);
		
		setElementsContext((elementsContext) => ({
			...elementsContext,
			saveElement: null,
		}));
	}
}, [elementsContext.saveElement]);
```

**Lines 2703-2707**: Save completion callback
```javascript
useEffect(() => {
	if (saveCourseElementResultData) {
		setSavingElementResult(false);
	}
}, [saveCourseElementResultData]);
```

### Activity Limit Integration

**Lines 2245-2323**: Premium proxy user save with limit checks
```javascript
if (proxyAsPremiumUser && !loginContext.isPremiumAccount) {
	if (/* activity includes lessons/quiz/examqs */) {
		if (maxDailyActivityReached && !completedElements.includes(currentElement.id)) {
			continueToSaveResult = false;
			setMaxReachedDialogOpen(true);
		} else {
			// Update completed counts
			setNumberOfCompletedElements((numberOfCompletedElements || 0) + numberOfElements);
			if (currentElement.totalScore) {
				setNumberOfCompletedScoringElements((numberOfCompletedScoringElements || 0) + numberOfElements);
			}
			updateActivityCount(activityType, numberOfElements);
		}
	}
}

if (continueToSaveResult) {
	saveCourseElementResult({ /* variables */ });
}
```

### Dependencies
- `ElementsContext` for score storage
- `ElementsProgressContext` for navigation coordination
- `ProgressBarHeaderContext` for header updates
- `useIdleTimer` for time tracking
- Activity limit states
- `loginContext` for user type
- Mutation hooks

### Issues & Technical Debt
1. **Dual responsibility**: Scoring + saving mixed together
2. **Complex multipart handling**: Different paths for multipart vs simple
3. **Score override**: Defensive programming for scoring bugs
4. **Activity limit coupling**: Save logic mixed with restriction checking
5. **Long function**: `saveElementResult()` is 440 lines

### Refactoring Recommendations

**Priority**: High  
**Extract to**: `useScoreTracking()` + `useElementSaving()` hooks

**Split responsibilities**:
```javascript
// Pure scoring logic
const {
	scores,
	updateScore,
	calculateTotalScore,
	resetScores
} = useScoreTracking(elements);

// Pure saving logic
const {
	saveElement,
	saveFlashcard,
	saving,
	error
} = useElementSaving({
	courseCode,
	activityType,
	showAllElements
});
```

**Benefits**:
- Separate concerns (scoring vs persistence)
- Easier testing (pure score calculations)
- Reusable across activity types
- Simpler component logic

---

## 2.5 Activity Lifecycle Management

### Current Implementation
**Lines**: 1860-1932, 3173-3235, 3623-3673, 5561-5694, 6110-6167  
**Complexity**: ⭐⭐⭐⭐ High

### Lifecycle Phases

#### Phase 1: Activity Start

**Lines 1860-1932**: Initialize activity tracking
```javascript
useEffect(() => {
	let newActivityType = null;
	let labelForAnalytics = '';
	let amplitudeType = '';
	
	// Determine activity type
	if (type === 'courserevise') {
		newActivityType = 'Q';
		labelForAnalytics = 'quiz';
		amplitudeType = 'quiz';
	} else if (examQs) {
		if (examQType === 'examQs') {
			newActivityType = 'E';
			labelForAnalytics = 'examQs';
			amplitudeType = 'examqs';
		} else if (examQType === 'examMCQs') {
			newActivityType = 'EMCQ';
			labelForAnalytics = 'examMCQs';
			amplitudeType = 'exammcqs';
		}
	} else if (type === 'courseflashcards') {
		newActivityType = 'F';
		labelForAnalytics = 'flashcards';
		amplitudeType = 'flashcards';
	} else {
		newActivityType = 'L';
		labelForAnalytics = 'lesson';
		amplitudeType = 'lesson';
	}
	
	setActivityType(newActivityType);
	
	// Google Analytics
	ReactGA.event('activity_started_' + labelForAnalytics, {
		category: 'activity_started',
		context: 'activity',
	});
	
	// Amplitude tracking
	let activityId = null;
	if (loginContext && loginContext.userId) {
		let dateTimeAsString = new Date().toISOString();
		dateTimeAsString = dateTimeAsString.replace(/[^0-9]/g, '');
		activityId = loginContext.userId + '_' + dateTimeAsString;
		
		setLoginContext((loginContext) => ({
			...loginContext,
			currentActivityCourseCode: courseCode,
			currentActivitySubtopicCode: subtopicCode ? subtopicCode : subtopics?.join(','),
			currentActivityId: activityId,
			currentActivityType: amplitudeType,
		}));
	}
	
	startAmplitudeActivity(loginContext, courseCode, subtopicCode, activityId, amplitudeType);
}, [courseCode, type, examQs]);
```

#### Phase 2: Activity Progress

**State tracking**:
- `dateTimeStarted` (722): Activity start timestamp
- `dateTimeCompleted` (723): Activity end timestamp  
- `elementCount` (1142): Current element in focus mode
- `totalElementCount` (1147): Total elements in activity
- `activityComplete` (769): All elements submitted flag
- `activityCompleteDialogShown` (770): Dialog shown flag

#### Phase 3: Activity Completion Detection

**Lines 5467-5559**: Detect completion for lessons/quizzes/exam qs
```javascript
useEffect(() => {
	if (type !== 'courseflashcards' && elements.length > 0) {
		if (!examQs) {
			if (numberOfScoringElements) {
				let numberOfSubmittedElements = 0;
				// Count submitted elements
				const keys = Object.keys(elementsContext.currentScores);
				for (let i = 0; i < keys.length; i++) {
					if (
						elementsContext.currentScores[keys[i]] &&
						!elementsContext.currentScores[keys[i]].isMultipart &&
						elementsContext.currentScores[keys[i]].totalScore &&
						(elementsContext.currentScores[keys[i]].incorrectScore || 0) +
						(elementsContext.currentScores[keys[i]].score || 0) ===
						elementsContext.currentScores[keys[i]].totalScore
					) {
						numberOfSubmittedElements++;
					}
				}
				setActivityComplete(numberOfSubmittedElements === numberOfScoringElements);
			}
		} else {
			// Different logic for exam Qs vs exam MCQs
			// ...
		}
	}
}, [type, elements, elementsContext.currentScores, numberOfScoringElements]);
```

**Lines 5395-5403**: Detect completion for flashcards
```javascript
useEffect(() => {
	if (type === 'courseflashcards' && elements.length > 0) {
		if (flashcardsProgress.length === elements.length) {
			setActivityComplete(true);
		} else {
			setActivityComplete(false);
		}
	}
}, [type, flashcardsProgress, elements]);
```

#### Phase 4: Activity Completion Actions

**completePremiumActivity()** (lines 3623-3673):
```javascript
const completePremiumActivity = (xpPointsGained) => {
	// 1. Google Analytics event
	const gaEvent = `${baseEvent}${typeEvent}${elementEvent}`;
	ReactGA.event(gaEvent);
	
	// 2. Mark activity as complete in database
	if (type === 'courseflashcards') {
		markPremiumUserCourseAsComplete({
			variables: {
				courseCode: resumedCourseCode ? resumedCourseCode : courseCode,
				type: 'F',
				xpGained: xpPointsGained,
				subtopicCode: parentCode,
			},
		});
	} else {
		markPremiumUserCourseAsComplete({
			variables: {
				courseCode: resumedCourseCode ? resumedCourseCode : courseCode,
				type: type === 'courserevise' ? 'R' : 'L',
				xpGained: xpPointsGained,
				subtopicCode: parentCode,
				examQs: examQs,
			},
		});
	}
	
	// 3. Log completion event
	logEvent({ event_type: 'end_activity' });
};
```

**Lines 3611-3621**: Completion coordination
```javascript
useEffect(() => {
	if (!savingElementResult && completePremiumActivityStatus) {
		completePremiumActivity(completePremiumActivityStatus.xpPointsGained);
		setCompletePremiumActivityStatus(null);
		
		try {
			completeAmplitudeActivity(loginContext);
		} catch (e) {
			// swallow error
		}
	}
}, [savingElementResult, completePremiumActivityStatus]);
```

**Lines 5561-5613**: Auto-completion for all-at-once mode (premium users)
**Lines 5622-5672**: Auto-completion for all-at-once mode (non-premium users)
**Lines 5674-5694**: Auto-completion for all-at-once mode (guest users)

### Completion Dialog Management

**State**:
- `subtopicCompleteDialogOpen` (692): `{ open: boolean, progressOnly: boolean }`
- `flashcardsCompleteDialogOpen` (696): `{ open: boolean, progressOnly: boolean }`
- `activityCompleteDialogShown` (770): Prevents duplicate dialogs

**progressOnly mode**: Shows progress without XP celebration when:
- All-at-once mode AND activity not yet complete
- Used for "View Progress" button before finishing all elements

### Navigation Blocking

**Lines 6110-6167**: Prevent accidental exit during activity
```javascript
<NavigationConfirm
	when={
		// Complex condition determining when to show exit confirmation
		!(type === 'course' && (levelCode === 'alevel' || levelCode === 'ks3') && !examQs) &&
		!activityCompleteDialogShown &&
		!loginContext.isProxyPremiumAccount &&
		!loginContext.isGuestAccount &&
		!activityComplete &&
		!isTeacher &&
		!hideEarlyExitDialog &&
		!subtopicComplete &&
		!flashcardsComplete &&
		// ... check localStorage for hide preference ...
	}
>
	{children}  {/* EndStudySessionEarlyConfirmation */}
</NavigationConfirm>
```

### XP Calculation

**Lines 3198-3216**: Calculate XP for lessons/quizzes
```javascript
let newXpGained = 0;
if (type === 'courserevise') {
	if (resultType === 0 || !completeSubtopic) {
		newXpGained = elementsContext.totalScore * DEFAULT_XP_PER_MARK_QUIZ;
	} else {
		newXpGained = DEFAULT_XP_PER_QUIZ + elementsContext.totalScore * DEFAULT_XP_PER_MARK_QUIZ;
	}
} else {
	if (resultType === 0) {
		newXpGained = elementsContext.totalScore * DEFAULT_XP_PER_MARK_LESSON;
	} else {
		newXpGained = DEFAULT_XP_PER_LESSON + elementsContext.totalScore * DEFAULT_XP_PER_MARK_LESSON;
	}
}
setXpGained(newXpGained);
```

**Lines 1593-1632**: Calculate XP for flashcards
```javascript
const calculateFlashcardsSummary = () => {
	const newFlashcardsSummary = { 1: 0, 2: 0, 3: 0, done: 0, notdone: 0, totalFlashcardCount: totalElementCount, xp: 0 };
	
	// Count responses by level
	for (let i = 0; i < flashcardsProgress.length; i++) {
		newFlashcardsSummary[flashcardsProgress[i].response]++;
	}
	
	// Calculate XP
	let xpAwarded =
		newFlashcardsSummary[1] * DEFAULT_XP_PER_LEVEL_ONE_FLASHCARD +
		newFlashcardsSummary[2] * DEFAULT_XP_PER_LEVEL_TWO_FLASHCARD +
		newFlashcardsSummary[3] * DEFAULT_XP_PER_LEVEL_THREE_FLASHCARD;
	
	if (completeSubtopic && !resultType) {
		xpAwarded += DEFAULT_XP_PER_FLASHCARDS;
	}
	
	newFlashcardsSummary['xp'] = xpAwarded;
	return { newFlashcardsSummary, flashcardResults };
};
```

### Dependencies
- Mutation hooks (save, complete)
- `elementsContext` (scores)
- `loginContext` (user type, analytics)
- Activity type props
- Dialog state
- XP constants
- Analytics services (GA4, Amplitude)

### Issues & Technical Debt
1. **Multiple completion paths**: Different for each activity type + user type
2. **Dialog timing**: Complex coordination between complete/shown flags
3. **XP scattered**: Calculation logic in multiple places
4. **Mixed concerns**: Lifecycle + analytics + dialogs + XP

### Refactoring Recommendations

**Priority**: High  
**Extract to**: `useActivityLifecycle()` hook

```javascript
const {
	activityStarted,
	activityComplete,
	completeActivity,
	showCompletionDialog,
	xpGained
} = useActivityLifecycle({
	activityType,
	elements,
	scores: elementsContext.currentScores,
	onComplete: () => { /* navigate or show dialog */ }
});
```

**Benefits**:
- Centralize lifecycle logic
- Consistent completion detection
- Simplified dialog management
- Cleaner component

---

## 2.6 Analytics & Event Tracking

### Current Implementation
**Lines**: 377-452, 1096-1137, 1860-1932, scattered throughout  
**Complexity**: ⭐⭐⭐ Medium

### Analytics Services

**Google Analytics 4** (ReactGA):
- Activity started events
- Display mode tracking
- Activity completion events
- Video play events
- Element response events

**Amplitude** (via helpers):
- `startAmplitudeActivity()` (line 1918)
- `completeAmplitudeActivity()` (line 3616)

**Custom Events** (`logEvent` hook):
- Element responses
- Activity limit modal shown
- Start/end activity

### Key Functions

**sendUsageToGoogleAnalytics()** (lines 377-452):
```javascript
export const sendUsageToGoogleAnalytics = (
	currentType,
	currentShowAllElements,
	currentExamQs,
	loginContext
) => {
	// Skip exam qs
	if (!currentExamQs) {
		// Generic usage event for all users
		let analyticsEvent = null;
		if (currentType === 'course' || currentType === 'L') {
			analyticsEvent = 'L';
		} else if (currentType === 'courserevise' || currentType === 'Q') {
			analyticsEvent = 'Q';
		} else if (currentType === 'courseflashcards' || currentType === 'F') {
			analyticsEvent = 'F';
		}
		
		if (analyticsEvent) {
			if (currentShowAllElements) {
				analyticsEvent += '_showall_elmnts';
			} else {
				analyticsEvent += '_onebyone_elmnts';
			}
			
			ReactGA.event(analyticsEvent, {
				category: 'activity_display_mode',
				context: 'activity',
			});
		}
		
		// User version 4 specific tracking (lines 408-449)
		if (loginContext.userVersion === 4) {
			// Track against original default mode
			// ...
		}
	}
};
```
**Called**: After each element save (lines 2238, 2349, 2779, 2317)

### Analytics Context Setup

**Lines 1096-1137**: Set LoginContext analytics
```javascript
useEffect(() => {
	const determineActivityType = () => {
		if (examQs) {
			return examQType === 'examMCQs' ? 'exammcqs' : 'examqs';
		}
		switch (type) {
			case 'courseflashcards':
				return isRevisionSession ? 'revision_session_flashcards' : 'flashcards';
			case 'courserevise':
				return isRevisionSession ? 'revision_session_quiz' : 'quiz';
			default:
				return 'lesson';
		}
	};
	
	if (elements.length > 0) {
		const activityType = determineActivityType();
		setLoginContext((loginContext) => ({
			...loginContext,
			analyticsContext: {
				activity_type: activityType,
				subtopic_code: examQs ? subtopicCode : parentCode,
				subject_code: subjectCode,
				activity_mode: examQs || showAllElements ? 'all_at_once' : 'focus',
				course_code: examQs ? courseCode : resumedCourseCode || courseCode,
			},
		}));
	}
}, [parentCode, subjectCode, resumedCourseCode, courseCode, showAllElements, isRevisionSession, type, examQs, examQType, elements]);
```

**Lines 664-676**: Clear analytics context on unmount
```javascript
useEffect(() => {
	setLoginContext((prevContext) => ({
		...prevContext,
		analyticsContext: {},
	}));
	
	return () => {
		setLoginContext((prevContext) => ({
			...prevContext,
			analyticsContext: {},
		}));
	};
}, []);
```

### Event Types Tracked

**Activity Events**:
- `activity_started_{type}` - Activity begins
- `6_activity_3_ev_compActivity_{type}_{mode}` - Activity completes
- `6_retry_1_bc_{type}` - Retry mistakes
- `6_restart_1_bc_{type}` - Restart activity
- `disp_mode_changed_to_{mode}` - Display mode toggle

**Element Events**:
- `element_response` with `element_type` - Each element submission
- `video_play` - Video played
- `activity_limit_modal_shown` - Limit reached

**User-specific Events** (userVersion 4):
- Tracks display mode against user's original default
- Events like `L_showall_elmnts_def_onebyone`

### Dependencies
- `react-ga4` library
- `loginContext` (userVersion, userId)
- `logEvent` hook from useLogEvent
- Amplitude helpers
- Activity state (type, mode, completion)

### Issues & Technical Debt
1. **Scattered calls**: Analytics spread throughout component
2. **Inconsistent patterns**: GA4 vs Amplitude vs custom events
3. **Tight coupling**: Analytics mixed with business logic
4. **User version checks**: Legacy userVersion 4 tracking

### Refactoring Recommendations

**Priority**: Medium  
**Extract to**: `useActivityAnalytics()` hook

```javascript
const {
	trackActivityStart,
	trackElementResponse,
	trackActivityComplete,
	trackModeChange
} = useActivityAnalytics({
	activityType,
	courseCode,
	subtopicCode,
	subjectCode,
	showAllElements
});
```

**Benefits**:
- Centralize all analytics
- Consistent event naming
- Easier to add/modify tracking
- Testable in isolation

---

## 2.7 Flashcard-Specific Logic

### Current Implementation
**Lines**: 642, 843-904, 1139-1141, 1593-1632, 2721-2833, 2854-2881, 6047-6061  
**Complexity**: ⭐⭐⭐⭐ High

### Core State

**Lines 642, 1139-1141**: Flashcard progress tracking
```javascript
const [flashcardFlipped, setFlashcardFlipped] = useState(false);
const [flashcardsProgress, setFlashcardsProgress] = useState(
	resumedCourseCode ? resumedFlashcardsProgress : []
);
```

**flashcardsProgress structure**:
```javascript
[
	{ flashcard: elementObject, response: 1 },  // Confident
	{ flashcard: elementObject, response: 2 },  // Getting there
	{ flashcard: elementObject, response: 3 },  // Still learning
]
```

**Lines 730-738**: Flashcard summary state
```javascript
const [flashcardsSummary, setFlashcardsSummary] = useState({
	1: 0,  // Confident count
	2: 0,  // Getting there count
	3: 0,  // Still learning count
	done: 0,
	notdone: 0,
	totalFlashcardCount: 0,
	xp: 0,
});
```

### Flashcard Response Handling

**handleFlashcardResponse()** (lines 2721-2833):
```javascript
const handleFlashcardResponse = (response, event) => {
	if (flashcardLimitReached) {
		setMaxReachedDialogOpen(true);
		return;
	}
	
	event?.stopPropagation();
	setSavingElementResult(true);
	setSelectedResponse(response);
	
	const currentElement = elements[elementCount - 1];
	
	// Update flashcard in active state for animation
	setActiveFlashcard({
		id: currentElement.id,
		response: response,
	});
	
	// Save if premium user
	if (loginContext.isProxyPremiumAccount && !isTeacher) {
		const flashcardResult = {
			courseCode: resumedCourseCode ? resumedCourseCode : courseCode,
			topicCode: topicCode,
			subtopicCode: resumedCourseCode
				? resumedSubtopicsObject[currentElement.id]
				: currentElement.courseSubtopicCode,
			elementCode: currentElement.id,
			response: response,
		};
		
		saveCourseFlashcardResult({
			variables: {
				flashcardResult: flashcardResult,
				preciseDuration: preciseDuration,
				showAllElements: showAllElements,
			},
		});
		
		logEvent({ event_type: 'element_response', element_type: 'flashcard' });
		sendUsageToGoogleAnalytics('F', showAllElements, false, loginContext);
	}
	
	// Update progress array
	const currentFlashcardsProgress = [...flashcardsProgress];
	let found = false;
	for (let i = 0; i < currentFlashcardsProgress.length; i++) {
		if (currentFlashcardsProgress[i].flashcard.id === currentElement.id) {
			found = true;
			currentFlashcardsProgress[i].response = response;
			break;
		}
	}
	if (!found) {
		currentFlashcardsProgress.push({
			flashcard: currentElement,
			response: response,
		});
	}
	setFlashcardsProgress(currentFlashcardsProgress);
	
	// Progress to next or complete
	if (elementCount < elements.length) {
		setElementCount(elementCount + 1);
		setExternalFlashcardResponse(null);
	} else {
		endActivityForFreeUser();
	}
	
	setTimeout(() => setSelectedResponse(null), 500);
};
```

### External Flip Handler

**handleExternalFlip()** (lines 843-904):
Used for keyboard shortcut (key '2') and external flip button.

```javascript
const handleExternalFlip = () => {
	if (elements && elements[elementCount - 1]) {
		const currentElement = elements[elementCount - 1];
		const currentCardSide = flashcardContext.flashcards?.[currentElement.id]?.side || 'front';
		
		let continueToFlip = true;
		
		// Premium user check for front-to-back flips
		if (currentCardSide === 'front' && !preview && !loginContext.isPremiumAccount) {
			if (maxDailyActivityReached && !completedElements.includes(currentElement.id)) {
				continueToFlip = false;
				setFlashcardLimitReached(true);
				setMaxReachedDialogOpen(true);
			} else {
				// Track as completed for free users
				if (!designMode && !completedElements.includes(currentElement.id)) {
					setCompletedElements([...completedElements, currentElement.id]);
					setNumberOfCompletedElements((numberOfCompletedElements || 0) + 1);
					setNumberOfCompletedScoringElements((numberOfCompletedScoringElements || 0) + 1);
					
					if (maxDailyActivityReached) {
						setFlashcardLimitReached(true);
						setMaxReachedDialogOpen(true);
					}
					
					updateActivityCount('F');
				}
			}
		}
		
		if (continueToFlip) {
			const newSide = currentCardSide === 'front' ? 'back' : 'front';
			
			setFlashcardContext((prevContext) => ({
				...prevContext,
				flashcards: {
					...prevContext.flashcards,
					[currentElement.id]: { side: newSide },
				},
			}));
			setExternalFlashcardSide(newSide);
			setFlashcardFlipped(true);
		}
	}
};
```

### Keyboard Controls

**Lines 2854-2881**: Keyboard shortcuts for flashcards
```javascript
const handleKeyPress = (event) => {
	if (
		activityType === 'F' &&
		!showAllElements &&
		!flashcardsComplete &&
		(activityBegun.current || isRedoingCourse || (restartActivity && animationStage === 2) || (resumedCourseCode && !isRedoingCourse && !restartActivity))
	) {
		if (event.key === '1') {
			handleFlashcardResponse(3);  // Still learning
		} else if (event.key === '2') {
			handleExternalFlip();  // Flip
		} else if (event.key === '3') {
			handleFlashcardResponse(1);  // Confident
		}
	}
};

useEffect(() => {
	document.addEventListener('keydown', handleKeyPress);
	return () => {
		document.removeEventListener('keydown', handleKeyPress);
	};
}, [handleFlashcardResponse]);
```

### Summary Calculation

**calculateFlashcardsSummary()** (lines 1593-1632):
- Counts responses by confidence level
- Calculates XP earned
- Returns summary object + results array for batch save (REMOVE batch usage)

### Progress Bar Integration

**Lines 4294-4306, 5192-5209**: Update flashcard progress header
```javascript
setFlashcardsProgressBarHeaderContext((context) => ({
	...context,
	levelOneValue: responses[1],
	levelTwoValue: responses[2],
	levelThreeValue: responses[3],
	maxValue: totalElementCount,
	showingAllElements: showAllElements,
}));
```

### Flashcard Controls Data

**Lines 6047-6061**: Control button configuration
```javascript
const controlsData = [
	{
		variant: 'still-learning',
		onClick: () => handleFlashcardResponse(3),
		id: 'level-3-button',
		className: getControlClass('still-learning'),
	},
	{ variant: 'flip', onClick: handleExternalFlip, id: 'flip-button' },
	{
		variant: 'confident',
		onClick: () => handleFlashcardResponse(1),
		id: 'level-1-button',
		className: getControlClass('confident'),
	},
];
```

### Dependencies
- `FlashcardContext` (card flip state)
- `FlashcardsProgressBarHeaderContext` (progress display)
- `elementsContext` (current element)
- `saveCourseFlashcardResult` mutation
- Activity limit logic
- Keyboard event listeners

### Issues & Technical Debt
1. **Limit integration**: Flashcard logic tightly coupled with activity limits
2. **Duplicate response handling**: Parent + child components both handle responses
3. **Summary used for batch**: `calculateFlashcardsSummary` currently serves batch saves (REMOVE that usage)
4. **External state**: Flashcard state managed both locally and in context

### Refactoring Recommendations

**Priority**: High  
**Extract to**: `useFlashcardLogic()` hook + `FlashcardActivity` component

```javascript
const {
	handleResponse,
	handleFlip,
	progress,
	summary,
	currentCard,
	isComplete
} = useFlashcardLogic({
	elements,
	resumedProgress,
	onResponse: saveFlashcardResponse
});
```

**Benefits**:
- Isolate flashcard-specific behavior
- Reusable for different flashcard UIs
- Cleaner keyboard handling
- Simplified response flow

---

## 2.8 Exam Q-Specific Logic

### Current Implementation
**Lines**: 535-593, 689, 4276-4290, 4767-4785, 5808-5894, 5918-5957  
**Complexity**: ⭐⭐⭐⭐⭐ Very High

### Exam Q Types

**examQs flag**: General exam questions feature  
**examQType values**:
- `'examQs'`: Long-form exam questions (multipart elements)
- `'examMCQs'`: Multiple choice exam questions

### Activity-Level Reset

**Lines 535-593**: `triggerReset()` function
```javascript
const [activityResetTriggerId, setActivityResetTriggerId] = useState(0);

const triggerReset = () => {
	if (isExamQsRevisionSession) {
		// Find completed multipart elements
		const elementsToReset = [];
		
		for (const elementId in elementsContext.currentScores) {
			const element = elementsContext.currentScores[elementId];
			if (
				element.rootElementId &&
				element.incorrectScore + element.score === element.totalScore &&
				element.totalScore > 0
			) {
				if (!elementsToReset.includes(element.rootElementId)) {
					elementsToReset.push(element.rootElementId);
				}
			}
		}
		
		resetMultipleCourseElementResults({
			variables: {
				courseCode,
				type: type === 'courserevise' ? 'R' : 'L',
				elements: elements
					.filter((element) => elementsToReset.includes(element.id))
					.map((element) => {
						const elementIds = [element.id];
						
						// Include subelement IDs
						if (element.type === 'multipart' && element.subelements) {
							for (let i = 0; i < element.subelements.length; i++) {
								if (
									element.subelements[i].element &&
									element.subelements[i].element.id &&
									element.subelements[i].element.totalScore
								) {
									elementIds.push(element.subelements[i].element.id);
								}
							}
						}
						
						return {
							topicCode: element.courseTopicCode,
							subtopicCode: element.courseSubtopicCode,
							elementIds: elementIds,
						};
					}),
			},
		});
	} else {
		// Single subtopic reset
		resetCourseActivity({
			variables: {
				courseCode,
				topicCode,
				subtopicCode,
				type: type === 'courserevise' ? 'R' : 'L',
			},
		});
	}
};
```

### ExamQActivityResetContext

**Lines 595-598**: Context value for activity-level resets
```javascript
const examQActivityResetContextValue = useMemo(
	() => ({ activityResetTriggerId, triggerReset }),
	[activityResetTriggerId]
);
```

**Lines 6453-6457**: Wrapped at component level
```javascript
return shouldWrapWithExamQActivityResetContext ? (
	<ExamQActivityResetContext.Provider value={examQActivityResetContextValue}>
		{mainContent}
	</ExamQActivityResetContext.Provider>
) : mainContent;
```

### Element Subtopic Mapping

**Lines 4276-4290**: Critical for mixed-subtopic activities
```javascript
const createElementSubtopicMap = (elements) => {
	const map = {};
	elements.forEach((element) => {
		if (element.type === 'multipart') {
			const courseSubtopicCode = element.courseSubtopicCode;
			// Map subelements to parent's subtopic
			element.subelements.forEach((subelement) => {
				map[subelement.element.id] = courseSubtopicCode;
			});
		}
		map[element.id] = element.courseSubtopicCode;
	});
	
	setElementSubtopicMap(map);
};
```
**Usage**: Determines correct subtopicCode when saving element results in revision sessions

### Exam Q Stepper Navigation

**Lines 4767-4785**: Navigation between exam questions
```javascript
const handleStepperNext = () => {
	if (elementIndex < elements.length - 1) {
		setElementIndex((prevElementIndex) => prevElementIndex + 1);
		setExamQsProgressContext((prevState) => ({
			...prevState,
			index: prevState.index + 1,
		}));
	}
};

const handleStepperBack = () => {
	if (elementIndex > 0) {
		setElementIndex((prevElementIndex) => prevElementIndex - 1);
		setExamQsProgressContext((prevState) => ({
			...prevState,
			index: prevState.index - 1,
		}));
	}
};
```

**Lines 5918-5957**: Stepper UI component
```javascript
const examQsStepper = (
	<MobileStepper
		variant="dots"
		steps={elements.length}
		position="static"
		activeStep={elementIndex}
		nextButton={<Button onClick={handleStepperNext} disabled={elementIndex === elements.length - 1}>Next</Button>}
		backButton={<Button onClick={handleStepperBack} disabled={elementIndex === 0}>Back</Button>}
	/>
);
```

### Progress State Tracking

**Lines 5808-5878**: Track completion state per exam question
```javascript
useEffect(() => {
	if (examQs) {
		if (!examQType || examQType === 'examQs') {
			const newState = [];
			if (elements && elementsContext.currentScores) {
				for (let i = 0; i < elements.length; i++) {
					const currentElement = elements[i];
					
					let subelementCompletedCount = 0;
					let scoringSubelementCount = 0;
					
					// Count completed scoring subelements
					for (let j = 0; j < currentElement.subelements.length; j++) {
						const multipartElement = currentElement.subelements[j].element;
						
						if (multipartElement.totalScore) {
							scoringSubelementCount++;
						}
						
						if (elementsContext.currentScores[multipartElement.id]) {
							if (
								elementsContext.currentScores[multipartElement.id].totalScore &&
								(elementsContext.currentScores[multipartElement.id].incorrectScore || 0) +
								(elementsContext.currentScores[multipartElement.id].score || 0) ===
								elementsContext.currentScores[multipartElement.id].totalScore
							) {
								subelementCompletedCount++;
							}
						}
					}
					
					// Determine state: 'full', 'partial', or 'none'
					if (subelementCompletedCount === scoringSubelementCount) {
						newState[i] = 'full';
					} else if (subelementCompletedCount > 0) {
						newState[i] = 'partial';
					} else {
						newState[i] = 'none';
					}
				}
			}
			
			setExamQsProgressContext((context) => ({
				...context,
				state: newState,
			}));
		} else if (examQType === 'examMCQs') {
			// Different logic for MCQs
			// ...
		}
	}
}, [elements, elementsContext, examQs]);
```

### Rendering Logic

**Lines 4862-4888**: Exam Q carousel rendering
```javascript
style={{
	position: showAllElements && examQs && examQType === 'examQs' && elementIndex !== index
		? 'absolute'
		: 'relative',
	visibility: showAllElements && examQs && examQType === 'examQs' && elementIndex !== index
		? 'hidden'
		: null,
	height: showAllElements && examQs && examQType === 'examQs' && elementIndex !== index
		? 0
		: null,
	// ... hide non-active exam qs
}}
```

### ExamQsProgressContext State

**Lines 638-640, 5880-5894**:
```javascript
const [examQsProgressContext, setExamQsProgressContext] = useState({
	index: 0,
	count: elements.length,
	state: []  // ['full', 'partial', 'none'] per element
});
```

### Dependencies
- `examQs`, `examQType` props
- `isRevisionSession` flag
- `elementSubtopicMap` for mixed subtopics
- `elementsContext.currentScores`
- `ExamQsProgressContext`
- `ExamQActivityResetContext`
- Stepper UI components
- Reset mutations

### Issues & Technical Debt
1. **Complex completion tracking**: Different for examQs vs examMCQs
2. **Subtopic mapping**: Required for mixed-subtopic revision sessions
3. **Carousel rendering**: Complex visibility/positioning logic
4. **Reset coordination**: Activity-level + element-level resets
5. **Progress state duplication**: Tracked in multiple places

### Refactoring Recommendations

**Priority**: Very High  
**Extract to**: `useExamQLogic()` hook + `ExamQActivity` component

```javascript
const {
	currentQuestionIndex,
	progressStates,
	navigateNext,
	navigatePrevious,
	resetAll,
	resetQuestion
} = useExamQLogic({
	elements,
	scores: elementsContext.currentScores,
	activityType,
	isRevisionSession
});
```

**Benefits**:
- Isolate exam-specific complexity
- Clearer progress tracking
- Simplified reset logic
- Reusable for future exam features

---

## 2.9 Element Reset Functionality

### Current Implementation
**Lines**: 535-593 (activity), 1798-1841 (mutations), 2418-2701 (individual)  
**Complexity**: ⭐⭐⭐ Medium

### Mutations

**RESETCOURSEACTIVITY_MUTATION** (line 252):
```javascript
mutation resetCourseActivity(
	$courseCode: String!
	$topicCode: String!
	$subtopicCode: String!
	$type: String!
)
```
**Purpose**: Resets entire subtopic activity progress

**RESETMULTIPLECOURSEELEMENTRESULTS_MUTATION** (line 268):
```javascript
mutation resetMultipleCourseElementResults(
	$courseCode: String!
	$type: String!
	$elements: [ResetElementsInput]!
)
```
**Purpose**: Resets specific elements (for exam Q revision sessions)

### Reset Handlers

**Lines 1798-1817**: Activity-level reset with toast
```javascript
const [resetCourseActivity] = useMutation(RESETCOURSEACTIVITY_MUTATION, {
	errorPolicy: 'all',
	onCompleted: (data) => {
		setActivityResetTriggerId((id) => id + 1);  // Trigger UI reset
		triggerToast({
			title: 'All exam questions reset',
			variant: 'success',
			duration: 3000,
		});
	},
	onError: () => {
		triggerToast({
			title: 'Something went wrong',
			description: 'Please try resetting all questions again.',
			variant: 'error',
			duration: 3000,
		});
	},
});
```

**Lines 1819-1841**: Multiple element reset with toast
```javascript
const [resetMultipleCourseElementResults] = useMutation(
	RESETMULTIPLECOURSEELEMENTRESULTS_MUTATION,
	{
		errorPolicy: 'all',
		onCompleted: (data) => {
			setActivityResetTriggerId((id) => id + 1);
			triggerToast({
				title: 'All questions reset',
				variant: 'success',
				duration: 3000,
			});
		},
		onError: () => {
			triggerToast({
				title: 'Something went wrong',
				description: 'Please try resetting all questions again.',
				variant: 'error',
				duration: 3000,
			});
		},
	}
);
```

### Individual Element Reset

**resetIndividualExamQPart()** (lines 2418-2701):
```javascript
const resetIndividualExamQPart = (currentElement) => {
	let elementResults = [];
	
	// 1. Build element results from current scores
	// 2. Mark currentElement.reset = true
	// 3. Mark incomplete children as reset = true
	
	elementResults = elementResults.map((element) => {
		if (element.elementCode === currentElement.id) {
			element.reset = true;
		} else if (!element.reset) {
			const elementResult = elementsContext.currentScores[element.elementCode];
			
			// If child is incomplete, also reset it
			if (
				elementResult &&
				elementResult.rootElementId &&
				elementResult.totalScore &&
				(elementResult.incorrectScore || 0) + (elementResult.score || 0) !== elementResult.totalScore
			) {
				element.reset = true;
			}
		}
		return element;
	});
	
	// 4. Save with reset flags
	saveCourseElementResult({
		variables: {
			elementResults: elementResults,
			type: type === 'courserevise' ? 'R' : 'L',
			preciseDuration: preciseDuration,
			showAllElements: showAllElements,
			examQ: examQs ? true : false,
		},
	});
};
```

### Reset Coordination Flow

1. **User clicks "Reset all" button** in ActivityOverview
2. **triggerReset()** called → increments `activityResetTriggerId`
3. **ExamQActivityResetContext** propagates trigger ID
4. **ElementCard** observes trigger ID change (line 336-348 in ElementCard.jsx)
5. **ElementCard** calls local reset → updates UI state
6. **Database updated** via mutation
7. **Toast notification** shown on success

### Context Propagation

**Lines 595-598**: Create context value
```javascript
const examQActivityResetContextValue = useMemo(
	() => ({ activityResetTriggerId, triggerReset }),
	[activityResetTriggerId]
);
```

**Lines 6090-6092**: Check if wrapping needed
```javascript
const shouldWrapWithExamQActivityResetContext =
	examQType === 'examQs' || examQType === 'examMCQs';
```

### Dependencies
- `resetCourseActivity` mutation
- `resetMultipleCourseElementResults` mutation  
- `ExamQActivityResetContext`
- `useToast` hook
- `elementsContext.currentScores`
- `onElementReset` prop (animation trigger)

### Issues & Technical Debt
1. **Two reset mechanisms**: Activity-level vs individual element
2. **Complex cascading**: Reset parent → triggers child resets
3. **State synchronization**: Must coordinate DB + UI + context
4. **Error handling**: Toast-based feedback only
5. **Revision session logic**: Different behavior for mixed subtopics

### Refactoring Recommendations

**Priority**: Medium  
**Extract to**: `useElementReset()` hook

```javascript
const {
	resetActivity,
	resetElement,
	resetMultiple,
	isResetting
} = useElementReset({
	courseCode,
	activityType,
	onSuccess: () => triggerToast({ ... })
});
```

**Benefits**:
- Centralize reset logic
- Cleaner state management
- Consistent error handling
- Reusable across activities

---

## 2.11 Context Coordination

### Current Implementation
**Lines**: Throughout component  
**Complexity**: ⭐⭐⭐⭐⭐ Very High - **REQUIRES SEPARATE DETAILED ANALYSIS**

### Context Inventory

| Context | Purpose | Lines | Read/Write | Complexity |
|---------|---------|-------|------------|------------|
| **ElementsContext** | Element scores, totals, multipart scores | 1061-1072 | Both | Very High |
| **ElementsProgressContext** | Button labels, scroll, element navigation | 1074-1080 | Both | High |
| **FlashcardContext** | Card flip state per flashcard | 651-653 | Both | Medium |
| **ProgressBarHeaderContext** | Header progress bar values | 648-650 | Both | High |
| **FlashcardsProgressBarHeaderContext** | Flashcard-specific header | 655-657 | Both | Medium |
| **MultipartElementsContext** | Multipart completion state | 706-707 | Both | Medium |
| **ExamQActivityResetContext** | Activity-level reset trigger | 595-598 | Read | Low |
| **NavigationContext** | Refresh course results flag | 659 | Write | Low |
| **SystemContext** | Drawer visibility, mobile flags | 660 | Both | Low |
| **CourseContext** | Course level code | 661 | Read | Low |
| **LoginContext** | User info, analytics, activity config | 658 | Both | Very High |

### State Flow Patterns

**ElementsContext → ElementCard → Subelements**:
```
SubtopicElements manages elementsContext
  ↓ Provider wraps elements
ElementCard reads/updates scores
  ↓ Prop drilling
Subelements (MCQ, Text, etc) update scores
  ↓ Update via setElementsContext
Scores aggregate back to SubtopicElements
  ↓ Triggers progress bar update
ProgressBarHeaderContext updated
```

**Button Coordination (Focus Mode)**:
```
ElementCard determines button label
  ↓ Updates elementProgressContext
ElementsProgressContext.buttonLabel set
  ↓ Read by SubtopicElements footer
Footer button uses label
  ↓ onClick triggers handleContinue
SubtopicElements.handleContinue called
  ↓ Saves element, progresses to next
ElementCard receives new element
```

### Critical State Synchronization Points

**Lines 1949-1963**: Force continue from element
```javascript
useEffect(() => {
	if (elementsContext.forceContinue && !elementsProgressContext.submitAll) {
		const activeElement = elementsContext.activeElement;
		const showNextElement = elementsContext.showNextElement ? 'submit' : null;
		setElementsContext((elementsContext) => ({
			...elementsContext,
			forceContinue: false,
			showNextElement: false,
			activeElement: null,
		}));
		handleContinue(showNextElement, activeElement);
	}
}, [elementsContext.forceContinue, elementsProgressContext.submitAll]);
```

**Lines 3012-3024**: Call continue from element progress context
```javascript
useEffect(() => {
	if (elementsProgressContext.callContinueFromElement) {
		const showNextElement = elementsProgressContext.showNextElement ? 'submit' : null;
		setElementsProgressContext((elementsProgressContext) => ({
			...elementsProgressContext,
			callContinueFromElement: false,
			showNextElement: false,
		}));
		handleContinue(showNextElement);
	}
}, [elementsProgressContext.callContinueFromElement]);
```

**Lines 3026-3041**: Submit element from progress context
```javascript
useEffect(() => {
	if (elementsProgressContext.submitElement) {
		const activeElement = elementsProgressContext.activeElement;
		const enabledSubmitAll = elementsProgressContext.enableSubmitAll ? true : false;
		setElementsProgressContext((elementsProgressContext) => ({
			...elementsProgressContext,
			submitElement: false,
			activeElement: null,
			enableSubmitAll: false,
		}));
		
		handleContinue(enabledSubmitAll ? 'submit-all' : 'submit', activeElement);
	}
}, [elementsProgressContext.submitElement]);
```

### Context Anti-Patterns

1. **Flag-based coordination**: Using boolean flags to trigger actions across contexts
2. **Stale closure risk**: Multiple useEffects watching same context properties
3. **Bidirectional updates**: Parent updates context → child reads → child updates back
4. **No single source of truth**: Element state duplicated across contexts

### Props vs Context Decision Matrix

**Currently uses props** (should be context):
- `showAllElements` (display mode) - passed as prop AND synced with context
- `maxDailyActivityReached` (limit state) - prop drilled through multiple levels

**Currently uses context** (could be props):
- `carouselContext` - only used within ElementCard, doesn't need provider
- `hintsContext` - only used within ElementCard

### Refactoring Recommendations

**Priority**: Very High  
**Action**: Separate analysis document required

**Recommendations**:
1. **Consolidate ElementsContext + ElementsProgressContext** into single `ActivityContext`
2. **Move FlashcardContext** into ActivityContext.flashcards
3. **Lift display mode** to ActivityContext instead of prop drilling
4. **Event-based coordination** instead of flag watching:
   ```javascript
   // Instead of:
   setContext({ submitElement: true, activeElement: el });
   
   // Use:
   onSubmitElement(el);  // Direct callback
   ```
5. **Context reducer**: Replace setState with useReducer for complex state transitions

---

## 2.12 Animation & UX

### Current Implementation
**Lines**: 609-613, 746, 827-832, 6090-6286  
**Complexity**: ⭐⭐⭐ Medium

### Animation Hook

**useActivityAnimation** (line 609-613):
```javascript
const {
	animationStage,
	isResetting,
	triggerAnimation,
	triggerResetAnimation,
} = useActivityAnimation();
```

**Animation stages**:
- Stage 0: Initial/hidden
- Stage 1: Activity overview showing
- Stage 2: Activity in progress (elements visible)
- Stage 3: Fade out (for resets)
- Stage 4: Fade in (after reset)

### Activity Overview Animation

**Lines 6176-6260**: Show overview before activity starts
```javascript
<ActivityAnimationWrapper
	isVisible={
		isResetting
			? true
			: activityType !== 'F' ||
			  (activityType === 'F' &&
				(showAllElements ||
				 (!resumedCourseCode && !isRedoingCourse && animationStage < 2) ||
				 (restartActivity && !isRedoingCourse && animationStage < 2)))
	}
	customAnimate={
		isResetting
			? animationStage === 3 ? { opacity: 0 } : animationStage === 4 ? { opacity: 1 } : undefined
			: undefined
	}
>
	<ActivityOverview
		introElementContent={introElementContent}
		videoElementCount={videoElementCount}
		elementCount={actualElementCount}
		subtopicKey={subtopicDisplayCode || subtopicKey}
		subtopicName={subtopicName}
		// ...
	/>
	
	{!examQs && !isNonInteractiveCourse && (/* show controls */) && (
		<ActivityModeControls
			showAllElements={showAllElements}
			handleShowAllElementsToggle={handleShowAllElementsToggle}
			animationStage={animationStage}
			triggerAnimation={triggerAnimation}
			onQuizStart={() => setStartElements(true)}
			isVisible={animationStage === 1}
		/>
	)}
</ActivityAnimationWrapper>
```

### Elements Animation

**Lines 6263-6286**: Elements fade in after controls
```javascript
<ActivityAnimationWrapper
	isVisible={
		isResetting
			? true
			: animationStage === 2 ||
			  animationStage === 4 ||
			  examQs ||
			  isRedoingCourse ||
			  isNonInteractiveCourse ||
			  (resumedCourseCode && !restartActivity)
	}
	customAnimate={
		isResetting
			? animationStage === 3 ? { opacity: 0 } : animationStage === 4 ? { opacity: 1 } : undefined
			: undefined
	}
>
	{elementsComponent}
</ActivityAnimationWrapper>
```

### Completion Card Animation

**Lines 5148-5153**: Activity complete card
```javascript
<ActivityAnimationWrapper
	isVisible={shouldShowActivityComplete}
	style={{ position: 'relative', zIndex: 10000 }}
>
	{shouldShowActivityComplete && activityCompleteCard}
</ActivityAnimationWrapper>
```

**shouldShowActivityComplete** (lines 4714-4729):
```javascript
const shouldShowActivityComplete =
	activityType === 'E' ||
	activityType === 'EMCQ' ||
	isNonInteractiveCourse ||
	(activityType !== 'F'
		? showAllElements || ((animationStage === 2 || resumedCourseCode) && (showActivityDialog || noTotalScore))
		: elementType === 'flashcard' && (showAllElements || ((animationStage === 2 || resumedCourseCode) && (showActivityDialog || noTotalScore))));
```

### Activity State Flags

**Lines 746, 827-832**: Restart vs redo detection
```javascript
const [restartActivity, setRestartActivity] = useState(false);
const [isRedoingCourse, setIsRedoingCourse] = useState(false);

// In localStorage effect:
if ((resultType === 'restart' || resultType === 2) && resumedCourseCode) {
	setRestartActivity(true);
	setIsRedoingCourse(false);
} else if (resultType === 0 || resultType === 'levelThree') {
	setIsRedoingCourse(true);
	setRestartActivity(false);
}
```

### Scroll Management

**Lines 620-626**: Prevent horizontal scroll
```javascript
useEffect(() => {
	document.body.style.overflowX = 'hidden';
	
	return () => {
		document.body.style.overflowX = '';
	};
}, []);
```

**Lines 5896-5916**: Auto-scroll to summary on mobile
```javascript
useEffect(() => {
	if (
		(subtopicCompleteDialogOpen.open || flashcardsCompleteDialogOpen.open) &&
		systemContext.isMobile &&
		!showAllElements &&
		!alreadyScrolledToSummaryCard.current &&
		!(activityType === 'F' && !showAllElements)
	) {
		setTimeout(() => {
			alreadyScrolledToSummaryCard.current = true;
			window.scrollTo(0, document.body.scrollHeight);
		}, 500);
	}
}, [subtopicCompleteDialogOpen.open, flashcardsCompleteDialogOpen.open, systemContext.isMobile]);
```

**useHideBackToTop hook** (line 5571): Hides back-to-top button during activities

### Idle Timer Integration

**Lines 628-630**: Idle detection
```javascript
const elementTimer = useIdleTimer({ timeout: TIMEOUT_ELEMENT });
const [addVideoToIdleTimer, setAddVideoToIdleTime] = useState(false);
```

**Lines 1649-1658**: Add video time to idle calculation
```javascript
useEffect(() => {
	if (activeElementForTimer) {
		if (activeElementForTimer.type === 'video') {
			setAddVideoToIdleTime(true);
		}
	}
}, [activeElementForTimer]);
```

### Dependencies
- `useActivityAnimation` hook
- `ActivityAnimationWrapper` component
- `useIdleTimer` hook
- `useHideBackToTop` hook
- `restartActivity` / `isRedoingCourse` state
- Animation stage coordination

### Issues & Technical Debt
1. **Complex visibility logic**: Multiple conditions for showing/hiding
2. **Animation stage coordination**: Tightly coupled with lifecycle
3. **Resume vs restart**: Similar but subtly different flows
4. **Mobile-specific scroll**: Special handling for summary card

### Refactoring Recommendations

**Priority**: Medium  
**Keep as-is** with minor cleanup

The animation system is working well and is relatively self-contained. Main improvements:
- Document animation stages clearly
- Extract visibility logic to helper functions
- Simplify shouldShowActivityComplete condition

---

## Summary: Functional Areas Extraction Priority

### Immediate (Post-Cleanup)
1. **Element Data Fetching** → `useElementFetching()` hook
2. **Flashcard Logic** → `useFlashcardLogic()` hook + FlashcardActivity component
3. **Exam Q Logic** → `useExamQLogic()` hook + ExamQActivity component

### High Priority
4. **Context Coordination** → Consolidate contexts, separate analysis needed
5. **Scoring & Progress** → `useScoreTracking()` + `useProgressTracking()` hooks
6. **Lifecycle Management** → `useActivityLifecycle()` hook

### Medium Priority
7. **Per-Element Saving** → `useElementSaving()` hook
8. **Analytics** → `useActivityAnalytics()` hook
9. **Element Reset** → `useElementReset()` hook

### Low Priority (Working Well)
10. **Display Mode** → Minor cleanup sufficient
11. **Feedback System** → Already isolated
12. **Animation & UX** → Keep as-is with documentation

### Estimated Effort

**After redundancy removal** (~820 lines):
- Remaining: ~5640 lines
- Target after refactoring: ~2500-3000 lines in core component
- Extracted: ~2500-3000 lines across hooks and specialized components
- Net reduction: ~30-40% in main component complexity

---

## Next Steps

1. ✅ Complete redundancy removal (Phase 1)
2. Create detailed Context Coordination analysis document
3. Begin hook extractions in priority order
4. Create specialized activity components
5. Update tests for refactored structure

