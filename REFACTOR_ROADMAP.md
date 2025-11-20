# SubtopicElements Refactoring Roadmap

**Date**: 2025-11-20  
**Current Size**: 6461 lines  
**Post-Cleanup Size**: ~5545 lines (after removing 820+ redundant lines)  
**Target Size**: ~2500-3000 lines core component + extracted modules

---

## Roadmap Overview

### Phase 0: Cleanup (Current)
Remove redundant code to create clean foundation for refactoring.

**Duration**: 1-2 weeks  
**Lines removed**: ~820 lines (14%)  
**Deliverable**: Clean SubtopicElements ready for extraction

### Phase 1: Hook Extractions (Immediate)
Extract complex logic into custom hooks without changing component structure.

**Duration**: 2-3 weeks  
**Lines extracted**: ~1500 lines  
**Deliverable**: SubtopicElements with composition via hooks

### Phase 2: Component Splitting (Follow-up)
Split into specialized activity components with shared container.

**Duration**: 3-4 weeks  
**Lines restructured**: ~3000 lines  
**Deliverable**: ActivityContainer + specialized components

### Phase 3: Context Consolidation (Advanced)
Simplify context architecture and reduce prop drilling.

**Duration**: 2-3 weeks  
**Complexity reduced**: 40-50%  
**Deliverable**: Unified ActivityContext with event-based coordination

---

## Phase 1: Hook Extractions

### Priority 1: Element Fetching (Week 1)

**Extract to**: `hooks/useElementFetching.js`  
**Lines**: ~250 lines  
**Complexity**: High

**Current implementation**:
- Lines 3734-3774: Query selection
- Lines 3777-3922: Variable building and fetching
- Lines 4029-4254: Data transformation

**Hook interface**:
```javascript
const {
	elements,
	loading,
	error,
	refetch,
	subtopicsData,
	totalElementCount,
	isExamQsRevisionSession
} = useElementFetching({
	// Activity config
	activityType,
	courseCode,
	subtopicCode,
	subtopics,
	
	// Display config
	showAllElements,
	numberOfQuestions,
	
	// Resume config
	resumedCourseCode,
	resultType,
	initialElementIds,
	
	// Exam Q config
	examQs,
	examQType,
	
	// Other
	elementType,
	elementsFilter,
	designMode
});
```

**Benefits**:
- Consolidate 9 queries
- Reusable across activity types
- Testable in isolation
- Easier to add caching

**Testing**: Unit tests for query selection logic

---

### Priority 2: Flashcard Logic (Week 2)

**Extract to**: `hooks/useFlashcardLogic.js`  
**Lines**: ~400 lines  
**Complexity**: High

**Current implementation**:
- Lines 843-904: External flip handler
- Lines 1593-1632: Summary calculation
- Lines 2721-2833: Response handling
- Lines 2854-2881: Keyboard controls

**Hook interface**:
```javascript
const {
	// State
	flashcardsProgress,
	flashcardsSummary,
	currentFlashcard,
	flashcardSide,
	selectedResponse,
	
	// Actions
	handleResponse,
	handleFlip,
	resetProgress,
	
	// Computed
	isComplete,
	xpEarned
} = useFlashcardLogic({
	elements,
	resumedProgress,
	onSave: saveFlashcardResult,
	showAllElements,
	activityLimitReached
});
```

**Benefits**:
- Isolate flashcard-specific behavior
- Reusable for different flashcard UIs
- Cleaner keyboard handling
- Simplified response flow

**Testing**: Unit tests for response handling, summary calculation

---

### Priority 3: Exam Q Logic (Week 2)

**Extract to**: `hooks/useExamQLogic.js`  
**Lines**: ~350 lines  
**Complexity**: Very High

**Current implementation**:
- Lines 535-593: Activity-level reset
- Lines 4276-4290: Element subtopic mapping
- Lines 4767-4785: Stepper navigation
- Lines 5808-5894: Progress state tracking

**Hook interface**:
```javascript
const {
	// Navigation
	currentQuestionIndex,
	navigateNext,
	navigatePrevious,
	canNavigateNext,
	canNavigatePrevious,
	
	// Progress
	progressStates,  // ['full', 'partial', 'none'] per element
	completedCount,
	totalCount,
	
	// Reset
	resetAll,
	resetQuestion,
	isResetting,
	
	// Mapping
	getElementSubtopic
} = useExamQLogic({
	elements,
	scores: elementsContext.currentScores,
	examQType,
	isRevisionSession,
	courseCode,
	activityType
});
```

**Benefits**:
- Isolate exam-specific complexity
- Clearer progress tracking
- Simplified reset coordination
- Reusable for future exam features

**Testing**: Unit tests for progress calculation, reset logic

---

### Priority 4: Activity Lifecycle (Week 3)

**Extract to**: `hooks/useActivityLifecycle.js`  
**Lines**: ~300 lines  
**Complexity**: High

**Current implementation**:
- Lines 1860-1932: Activity start tracking
- Lines 3623-3673: Completion handling
- Lines 5467-5559: Completion detection
- Lines 6110-6167: Navigation blocking

**Hook interface**:
```javascript
const {
	// State
	activityStarted,
	activityComplete,
	activityId,
	startTime,
	endTime,
	
	// Actions
	startActivity,
	completeActivity,
	
	// Computed
	duration,
	xpGained,
	shouldBlockNavigation,
	
	// Dialogs
	showCompletionDialog,
	completionDialogMode  // 'full' | 'progress-only'
} = useActivityLifecycle({
	activityType,
	elements,
	scores: elementsContext.currentScores,
	flashcardsProgress,
	showAllElements,
	courseCode,
	subtopicCode,
	onComplete: markActivityComplete
});
```

**Benefits**:
- Centralize lifecycle logic
- Consistent completion detection
- Simplified dialog management
- Cleaner navigation blocking

**Testing**: Unit tests for completion detection, XP calculation

---

### Priority 5: Analytics Tracking (Week 3)

**Extract to**: `hooks/useActivityAnalytics.js`  
**Lines**: ~200 lines  
**Complexity**: Medium

**Current implementation**:
- Lines 377-452: sendUsageToGoogleAnalytics
- Lines 1096-1137: Analytics context setup
- Lines 1860-1932: Activity start events
- Scattered: Element response events

**Hook interface**:
```javascript
const {
	// Setup
	initializeTracking,
	
	// Events
	trackActivityStart,
	trackActivityComplete,
	trackElementResponse,
	trackDisplayModeChange,
	trackActivityRestart,
	trackActivityRetry,
	
	// Context
	analyticsContext
} = useActivityAnalytics({
	activityType,
	courseCode,
	subtopicCode,
	subjectCode,
	showAllElements,
	userId
});
```

**Benefits**:
- Centralize all analytics
- Consistent event naming
- Easier to modify tracking
- Testable independently

**Testing**: Mock tests for event firing

---

### Priority 6: Score Tracking (Week 4)

**Extract to**: `hooks/useScoreTracking.js`  
**Lines**: ~400 lines  
**Complexity**: Very High

**Current implementation**:
- Lines 1061-1072: Score context initialization
- Lines 1976-2416: Score calculation in saveElementResult
- Lines 2962-3010: Progress bar updates
- Lines 4466-4543: Score aggregation

**Hook interface**:
```javascript
const {
	// Current state
	scores,
	totalScore,
	correctScore,
	incorrectScore,
	
	// Multipart handling
	multipartScores,
	
	// Actions
	updateElementScore,
	aggregateScores,
	resetScores,
	resetElementScore,
	
	// Computed
	activityProgress,  // { completed: 5, total: 10, percentage: 50 }
	isActivityComplete
} = useScoreTracking({
	elements,
	resumedScores,
	onScoreUpdate: updateProgressBar
});
```

**Benefits**:
- Separate scoring from saving
- Pure score calculations
- Testable logic
- Reusable across activities

**Testing**: Unit tests for score calculations, multipart aggregation

---

### Priority 7: Element Saving (Week 4)

**Extract to**: `hooks/useElementSaving.js`  
**Lines**: ~200 lines  
**Complexity**: High

**Current implementation**:
- Lines 1965-1974: Save trigger watching
- Lines 1976-2416: Element result construction
- Lines 2686-2701: Reset individual part
- Lines 2721-2833: Flashcard save

**Hook interface**:
```javascript
const {
	// State
	saving,
	error,
	lastSaved,
	
	// Actions
	saveElement,
	saveFlashcard,
	resetElement,
	
	// Coordination
	queueSave,
	flushQueue
} = useElementSaving({
	courseCode,
	topicCode,
	activityType,
	showAllElements,
	onSaveComplete: updateProgressBar
});
```

**Benefits**:
- Isolate persistence logic
- Cleaner error handling
- Save queue management
- Retry logic

**Testing**: Integration tests with mocked mutations

---

## Phase 2: Component Splitting

### Target Structure

```
src/components/activities/
├── ActivityContainer/
│   ├── ActivityContainer.jsx          # Shared wrapper
│   ├── components/
│   │   ├── ActivityHeader/
│   │   ├── ActivityFooter/
│   │   ├── ActivityProgress/
│   │   └── ActivityComplete/
│   └── hooks/
│       ├── useActivityState.js        # Shared state management
│       └── useActivityCoordination.js
│
├── LessonActivity/
│   ├── LessonActivity.jsx             # Lesson-specific logic
│   ├── components/
│   │   ├── LessonOverview/
│   │   └── LessonControls/
│   └── hooks/
│       └── useLessonLogic.js
│
├── QuizActivity/
│   ├── QuizActivity.jsx               # Quiz-specific logic
│   ├── components/
│   │   ├── QuizOverview/
│   │   ├── QuizSettings/
│   │   └── QuizResults/
│   └── hooks/
│       └── useQuizLogic.js
│
├── FlashcardActivity/
│   ├── FlashcardActivity.jsx          # Flashcard-specific logic
│   ├── components/
│   │   ├── FlashcardDeck/
│   │   ├── FlashcardControls/
│   │   └── FlashcardProgress/
│   └── hooks/
│       └── useFlashcardLogic.js       # (Already extracted)
│
├── ExamQActivity/
│   ├── ExamQActivity.jsx              # Exam Q-specific logic
│   ├── components/
│   │   ├── ExamQCarousel/
│   │   ├── ExamQProgress/
│   │   └── ExamQReset/
│   └── hooks/
│       └── useExamQLogic.js           # (Already extracted)
│
└── shared/
    ├── ElementRenderer/
    │   ├── ElementCard/               # (Existing, simplified)
    │   └── FlashcardCard/             # (Existing, simplified)
    ├── hooks/
    │   ├── useElementFetching.js      # (Extracted in Phase 1)
    │   ├── useScoreTracking.js        # (Extracted in Phase 1)
    │   ├── useElementSaving.js        # (Extracted in Phase 1)
    │   ├── useActivityAnalytics.js    # (Extracted in Phase 1)
    │   └── useActivityLifecycle.js    # (Extracted in Phase 1)
    └── contexts/
        └── ActivityContext.jsx         # Consolidated context
```

---

### Step 1: Create ActivityContainer (Week 5)

**Purpose**: Shared wrapper for all activity types

**Responsibilities**:
- Element fetching coordination
- Display mode management
- Progress bar updates
- Navigation blocking
- Completion detection
- Analytics initialization

**Props**:
```javascript
<ActivityContainer
	activityType="lesson"
	courseCode="MATH101"
	subtopicCode="algebra_1"
	showAllElements={showAllElements}
	onDisplayModeChange={handleModeChange}
	onComplete={handleComplete}
>
	{({ elements, scores, saveElement, isComplete }) => (
		<LessonActivity
			elements={elements}
			scores={scores}
			onSave={saveElement}
			isComplete={isComplete}
		/>
	)}
</ActivityContainer>
```

**Benefits**:
- Render prop pattern for flexibility
- Shared functionality extracted
- Activity-specific logic stays separate

---

### Step 2: Create LessonActivity (Week 6)

**Purpose**: Lesson-specific behavior

**Unique to lessons**:
- Revision notes mode (A-level, all-at-once forced)
- KS3 lessons (all-at-once forced)
- Non-interactive courses (content-only)
- Video elements with idle timeout

**Component structure**:
```javascript
export default function LessonActivity({
	elements,
	scores,
	showAllElements,
	onSave,
	onComplete
}) {
	// Lesson-specific state
	const [includeVideoTime, setIncludeVideoTime] = useState(false);
	
	// Non-interactive course detection
	const isNonInteractive = elements.every(el => !el.totalScore);
	
	// Force all-at-once for revision notes/KS3
	const forceAllAtOnce = levelCode === 'alevel' || levelCode === 'ks3';
	
	return (
		<div className="lesson-activity">
			{isNonInteractive ? (
				<NonInteractiveLessonView elements={elements} />
			) : (
				<InteractiveLessonView
					elements={elements}
					scores={scores}
					showAllElements={forceAllAtOnce || showAllElements}
					onSave={onSave}
				/>
			)}
		</div>
	);
}
```

**Extract from SubtopicElements**:
- Non-interactive course handling (lines 4708-4713, 5983-6027)
- Video idle timer logic (lines 1649-1658)
- Revision notes mode detection
- Lesson-specific XP calculation

---

### Step 3: Create QuizActivity (Week 6)

**Purpose**: Quiz-specific behavior

**Unique to quizzes**:
- Start screen before beginning
- Question selection (incorrect/correct/new mix)
- Subtopic selection interface
- Retry mistakes flow

**Component structure**:
```javascript
export default function QuizActivity({
	elements,
	scores,
	showAllElements,
	onSave,
	onComplete,
	subtopics,
	numberOfQuestions,
	resultType
}) {
	const [quizStarted, setQuizStarted] = useState(false);
	
	// Quiz start screen
	if (!quizStarted) {
		return (
			<QuizSettings
				subtopics={subtopics}
				numberOfQuestions={numberOfQuestions}
				onStart={() => setQuizStarted(true)}
			/>
		);
	}
	
	return (
		<QuizView
			elements={elements}
			scores={scores}
			showAllElements={showAllElements}
			onSave={onSave}
			onComplete={onComplete}
			resultType={resultType}
		/>
	);
}
```

**Extract from SubtopicElements**:
- Start screen logic (lines 1676-1705)
- Quiz-specific XP calculation (lines 3199-3206)
- Retry mistakes handling (lines 4375-4401)
- Question selection persistence

---

### Step 4: FlashcardActivity (Week 7)

**Purpose**: Flashcard-specific behavior  
**Status**: Most logic already extractable via `useFlashcardLogic` hook

**Component structure**:
```javascript
export default function FlashcardActivity({
	elements,
	showAllElements,
	onSave,
	onComplete,
	resumedProgress
}) {
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
		onResponse: onSave,
		showAllElements
	});
	
	// Keyboard controls
	useFlashcardKeyboard({
		onStillLearning: () => handleResponse(3),
		onFlip: handleFlip,
		onConfident: () => handleResponse(1),
		enabled: !showAllElements && !isComplete
	});
	
	return (
		<div className="flashcard-activity">
			{showAllElements ? (
				<FlashcardDeckView
					flashcards={elements}
					progress={progress}
					onResponse={handleResponse}
				/>
			) : (
				<FlashcardFocusView
					currentCard={currentCard}
					onResponse={handleResponse}
					onFlip={handleFlip}
				/>
			)}
			
			<FlashcardProgress summary={summary} />
		</div>
	);
}
```

**Extract from SubtopicElements**:
- Already identified in useFlashcardLogic
- Keyboard controls (extract to `useFlashcardKeyboard`)
- Progress display coordination

---

### Step 5: ExamQActivity (Week 7)

**Purpose**: Exam Q-specific behavior

**Unique to exam qs**:
- Carousel navigation
- Part-by-part submission
- Mixed subtopic support
- Activity-level reset
- Progress indicator per question

**Component structure**:
```javascript
export default function ExamQActivity({
	elements,
	scores,
	examQType,
	onSave,
	onComplete,
	isRevisionSession
}) {
	const {
		currentQuestionIndex,
		navigateNext,
		navigatePrevious,
		progressStates,
		resetAll,
		getElementSubtopic
	} = useExamQLogic({
		elements,
		scores,
		examQType,
		isRevisionSession
	});
	
	return (
		<div className="examq-activity">
			<ExamQProgress
				current={currentQuestionIndex}
				total={elements.length}
				states={progressStates}
				onReset={resetAll}
			/>
			
			{examQType === 'examQs' ? (
				<ExamQCarousel
					questions={elements}
					currentIndex={currentQuestionIndex}
					onNavigate={(index) => navigateNext(index)}
					scores={scores}
					onSave={onSave}
				/>
			) : (
				<ExamMCQList
					questions={elements}
					scores={scores}
					onSave={onSave}
				/>
			)}
		</div>
	);
}
```

**Extract from SubtopicElements**:
- Already identified in useExamQLogic
- Carousel rendering (lines 4826-5155)
- Stepper UI (lines 5918-5957)

---

## Phase 3: Context Consolidation

### Current Context Problems

**11 separate contexts**:
1. ElementsContext (scores, totals)
2. ElementsProgressContext (button coordination)
3. FlashcardContext (flip state)
4. ProgressBarHeaderContext (header updates)
5. FlashcardsProgressBarHeaderContext
6. MultipartElementsContext
7. ExamQActivityResetContext
8. NavigationContext
9. SystemContext
10. CourseContext
11. LoginContext

**Issues**:
- Flag-based coordination (set flag → watch flag → trigger action)
- Stale closure risks
- Complex synchronization
- Unclear data flow

### Proposed: Unified ActivityContext

**Create**: `contexts/ActivityContext.jsx`

**Structure**:
```javascript
const ActivityContext = createContext();

const activityReducer = (state, action) => {
	switch (action.type) {
		case 'START_ACTIVITY':
			return { ...state, started: true, startTime: Date.now() };
		
		case 'UPDATE_ELEMENT_SCORE':
			return {
				...state,
				scores: {
					...state.scores,
					[action.elementId]: action.score
				}
			};
		
		case 'COMPLETE_ACTIVITY':
			return { ...state, complete: true, endTime: Date.now() };
		
		// ... more actions
	}
};

export function ActivityProvider({ children, activityType, courseCode }) {
	const [state, dispatch] = useReducer(activityReducer, initialState);
	
	// Derived state
	const totalScore = useMemo(() => 
		Object.values(state.scores).reduce((sum, score) => sum + score.correct, 0),
		[state.scores]
	);
	
	// Event handlers (replace flag coordination)
	const handleElementSubmit = useCallback((elementId, score) => {
		dispatch({ type: 'UPDATE_ELEMENT_SCORE', elementId, score });
		saveElementResult(elementId, score);  // Direct call instead of flag
	}, [saveElementResult]);
	
	const value = {
		// State
		...state,
		totalScore,
		
		// Actions (event-based, not flag-based)
		startActivity: () => dispatch({ type: 'START_ACTIVITY' }),
		submitElement: handleElementSubmit,
		completeActivity: () => dispatch({ type: 'COMPLETE_ACTIVITY' }),
		resetElement: (elementId) => dispatch({ type: 'RESET_ELEMENT', elementId }),
	};
	
	return <ActivityContext.Provider value={value}>{children}</ActivityContext.Provider>;
}
```

**Benefits**:
- Single source of truth
- Event-based coordination (callbacks instead of flags)
- Predictable state transitions
- Easier debugging
- Better performance (fewer re-renders)

### Migration Strategy

**Step 1**: Create new ActivityContext alongside old contexts  
**Step 2**: Migrate ElementsContext → ActivityContext.scores  
**Step 3**: Migrate ElementsProgressContext → ActivityContext (event handlers)  
**Step 4**: Migrate activity-specific contexts (flashcard, examQ)  
**Step 5**: Remove old contexts  

**Duration**: 2-3 weeks  
**Risk**: High (touches all state management)

---

## Phase 4: Final Cleanup

### Remove Legacy Patterns

1. **Flag-based coordination** → Event callbacks
2. **Prop drilling** → Context access
3. **Multiple useEffects watching same state** → Single reducer
4. **Bidirectional context updates** → Unidirectional flow

### Simplify Component Interface

**Current**: 50+ props  
**Target**: 15-20 props

**Simplified interface**:
```javascript
<ActivityContainer
	// Essential config
	activityType="lesson" | "quiz" | "flashcards" | "examqs" | "exammcqs"
	courseCode={string}
	subtopicCode={string}
	subtopics={string[]}  // For multi-subtopic
	
	// Resume
	resumedCourseCode={string}
	resultType="restart" | "retry" | 0 | 2
	
	// Display
	numberOfQuestions={number}
	
	// Callbacks
	onComplete={function}
	onNavigateAway={function}
>
	{renderProps => <ActivitySpecificComponent {...renderProps} />}
</ActivityContainer>
```

---

## Testing Strategy

### Phase 1 Testing (Per Hook)
- Unit tests for each extracted hook
- Mock dependencies
- Test all branches
- Coverage target: 80%+

### Phase 2 Testing (Per Component)
- Component tests for each activity type
- Integration tests for ActivityContainer
- E2E tests for complete flows
- Coverage target: 70%+

### Regression Testing
- Existing test suite must pass
- Manual testing checklist:
  - [ ] Start each activity type
  - [ ] Complete each activity type
  - [ ] Hit activity limits
  - [ ] Resume activities
  - [ ] Retry mistakes
  - [ ] Restart activities
  - [ ] Switch display modes
  - [ ] Navigate between elements
  - [ ] Reset exam questions
  - [ ] Submit feedback

---

## Success Metrics

### Code Quality
- Main component: < 3000 lines
- Average function length: < 50 lines
- Max function length: < 150 lines
- Cyclomatic complexity: < 15 per function

### Maintainability
- Clear separation of concerns
- Reusable hooks across activities
- Consistent patterns
- Well-documented

### Performance
- No regression in render performance
- Reduced unnecessary re-renders
- Improved initial load time

### Developer Experience
- Easier to add new activity types
- Clearer codebase navigation
- Faster onboarding for new developers
- Simpler debugging

---

## Risk Assessment

### High Risk
- **Context consolidation** (Phase 3)
  - Touches all state management
  - Risk of breaking subtle interactions
  - Requires extensive testing

### Medium Risk
- **Component splitting** (Phase 2)
  - Changes component hierarchy
  - Routing updates needed
  - Potential for missed edge cases

### Low Risk
- **Hook extractions** (Phase 1)
  - Logic stays in same component
  - Incremental changes
  - Easy to test

### Mitigation
- Feature flag for new structure
- Parallel implementation during transition
- Gradual rollout (10% → 50% → 100%)
- Easy rollback plan

---

## Timeline Summary

| Phase | Duration | Risk | Lines Changed |
|-------|----------|------|---------------|
| 0. Cleanup | 1-2 weeks | Low | 820 removed |
| 1. Hooks | 2-3 weeks | Low | 1500 extracted |
| 2. Components | 3-4 weeks | Medium | 3000 restructured |
| 3. Contexts | 2-3 weeks | High | 500 consolidated |
| **Total** | **10-14 weeks** | - | **~5800 lines refactored** |

### Incremental Delivery

- **After Phase 0**: Cleaner code, easier to understand
- **After Phase 1**: More maintainable, reusable hooks
- **After Phase 2**: Clear separation, easier to extend
- **After Phase 3**: Optimal architecture

### Checkpoints

**Checkpoint 1** (End of Phase 0):
- All redundant code removed
- Tests passing
- No regressions

**Checkpoint 2** (End of Phase 1):
- 5+ hooks extracted and tested
- Component logic simplified
- Hooks reused in at least 2 places

**Checkpoint 3** (End of Phase 2):
- 4 specialized components created
- ActivityContainer working for all types
- E2E tests passing

**Checkpoint 4** (End of Phase 3):
- Contexts consolidated
- Event-based coordination
- Performance benchmarks met

---

## Appendix: Current vs Target Architecture

### Current (6461 lines)

```
SubtopicElements.jsx
├── 9 GraphQL queries
├── 50+ props
├── 80+ state variables
├── 40+ useEffects
├── 20+ functions
├── 11 context providers
└── Complex conditional rendering
```

**Issues**:
- God component anti-pattern
- Mixed concerns
- Difficult to test
- Hard to extend

### Target (~2500 lines core + modules)

```
ActivityContainer (500 lines)
├── Uses hooks:
│   ├── useElementFetching (250 lines)
│   ├── useActivityLifecycle (300 lines)
│   ├── useActivityAnalytics (200 lines)
│   └── useScoreTracking (400 lines)
├── Renders:
│   ├── LessonActivity (400 lines)
│   ├── QuizActivity (450 lines)
│   ├── FlashcardActivity (350 lines)
│   └── ExamQActivity (500 lines)
└── Uses:
    ├── ActivityContext (unified)
    └── useElementSaving (200 lines)
```

**Benefits**:
- Clear separation of concerns
- Testable components
- Reusable hooks
- Extensible architecture
- Easier onboarding
- Better performance

---

## Next Steps

1. ✅ Complete Phase 0 cleanup (remove 820 lines)
2. ⏳ Get approval for Phase 1 hook extractions
3. ⏳ Create spike/POC for ActivityContainer pattern
4. ⏳ Validate approach with team
5. ⏳ Begin Phase 1 implementation

---

## Questions for Decision

### Before Phase 1
1. Should we extract hooks before or after removing redundant code?
   - **Recommendation**: After cleanup (cleaner foundation)

2. Which hook should we extract first?
   - **Recommendation**: useElementFetching (highest impact)

### Before Phase 2
1. Do we keep SubtopicElements as wrapper or create new ActivityContainer?
   - **Recommendation**: Create new, deprecate SubtopicElements gradually

2. How to handle routing updates?
   - **Recommendation**: Keep existing routes, update internals

### Before Phase 3
1. Breaking change acceptable for context consolidation?
   - **Recommendation**: Feature flag + gradual migration

2. Keep backward compatibility during transition?
   - **Recommendation**: Yes, dual support for 1-2 sprints

